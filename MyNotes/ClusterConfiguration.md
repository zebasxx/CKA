# Kubernetes Cluster Configuration

These notes build the primary CKA practice cluster with `kubeadm` on Ubuntu.
Commands are intended for Kubernetes **v1.37** on `amd64` unless noted
otherwise.

> Keep component versions pinned while building the cluster. Before reusing
> these notes for another Kubernetes minor release, review the upstream
> installation and version-skew documentation.

## Lab topology

| Node | Role | Address | vCPU | RAM |
| --- | --- | --- | ---: | ---: |
| `cka-cp01` | Control plane | `192.168.124.51` | 2 | 3 GiB |
| `cka-worker01` | Worker | `192.168.124.61` | 2 | 3 GiB |
| `cka-worker02` | Worker | `192.168.124.62` | 2 | 3 GiB |

Network ranges:

| Network | CIDR | Notes |
| --- | --- | --- |
| VM network | `192.168.124.0/24` | Libvirt `k8s-lab` network. |
| Pod network | `10.244.0.0/16` | Must not overlap the VM or Service network. |
| Service network | `10.96.0.0/12` | kubeadm default. |

The Pod, Service, and VM networks must not overlap.

## Core components

| Component | Purpose |
| --- | --- |
| `kubeadm` | Bootstraps, joins, upgrades, and resets Kubernetes nodes. |
| `kubelet` | Node agent that reconciles assigned Pods through the container runtime. |
| `kubectl` | Command-line client for the Kubernetes API. |
| `containerd` | CRI-compatible container runtime used to manage containers and images. |
| `runc` | Low-level OCI runtime used by containerd to create containers. |
| CNI plugins | Executables used to configure network interfaces inside Pod sandboxes. |
| Calico | Cluster network implementation and NetworkPolicy engine. |
| `crictl` | Troubleshooting client for CRI-compatible container runtimes. |

`kubectl` creates and manages API objects such as Pods, Deployments, Services,
Secrets, and Ingresses. It does not manage containers directly.

`kubelet` runs on every node. It:

- watches the API for Pods assigned to its node;
- asks the container runtime to create, stop, and restart containers;
- runs startup, readiness, and liveness probes;
- mounts volumes and injects configuration;
- reports node and Pod status to the API server.

## Installation overview

1. Prepare every node.
2. Install containerd, runc, and the base CNI binaries on every node.
3. Configure containerd to use the systemd cgroup driver.
4. Install `kubelet`, `kubeadm`, `kubectl`, and `crictl`.
5. Initialize the control plane.
6. Install Calico.
7. Join the workers.
8. Validate node readiness, DNS, and Service connectivity.

---

## 1. Prepare every node

Run this section on `cka-cp01`, `cka-worker01`, and `cka-worker02`.

### Confirm identity and connectivity

```bash
hostnamectl --static
ip -brief address
ip route
```

Confirm that every node has a unique hostname, IP address, and MAC address.
Test connectivity to the other nodes by IP. Name resolution is convenient but
not required for this configuration because the API endpoint uses an IP address.

```bash
ping -c 2 192.168.124.51
ping -c 2 192.168.124.61
ping -c 2 192.168.124.62
```

### Disable swap permanently

The kubelet fails on active swap by default. `swapoff` changes the current
runtime state only; the swap configuration must also be disabled for future
boots.

Inspect the current configuration:

```bash
swapon --show
grep -nE '[[:space:]]swap[[:space:]]' /etc/fstab || true
```

Back up and edit `/etc/fstab`:

```bash
sudo cp --preserve /etc/fstab /etc/fstab.before-kubernetes
sudoedit /etc/fstab
```

Comment out any active swap entry, for example:

```text
# /swap.img none swap sw 0 0
```

Disable swap now and verify:

```bash
sudo swapoff --all
swapon --show
```

`swapon --show` should produce no output. If swap returns after reboot, inspect
systemd swap units or zram services:

```bash
systemctl list-units --type=swap --all
```

### Load required kernel modules

`overlay` is used by the container runtime's overlay filesystem. The
`br_netfilter` module exposes bridged traffic to the netfilter hooks used by
Kubernetes networking components.

Persist both modules:

```bash
sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<'EOF'
overlay
br_netfilter
EOF
```

Load them immediately and verify:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
lsmod | grep -E '^(overlay|br_netfilter)'
```

### Configure networking sysctls

```bash
sudo tee /etc/sysctl.d/99-kubernetes.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Verify the effective values:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

Each value should be `1`.

### Install download prerequisites

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gpg tar wget
```

Avoid a general `apt upgrade` in the middle of cluster installation. Apply
operating-system updates deliberately and reboot before continuing when the
kernel or core libraries require it.

### Create a temporary working directory

```bash
mkdir -p "$HOME/k8s-files"
cd "$HOME/k8s-files"
```

---

## 2. Install the container runtime

Run this section on every node.

### Install containerd

Set an explicit version so all nodes receive the same runtime:

```bash
CONTAINERD_VERSION="2.4.1"

curl -fLO \
  "https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz"

sudo tar -C /usr/local -xzvf \
  "containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz"
```

In a production workflow, compare the archive's SHA-256 checksum with the
checksum published on the release page before extracting it.

Install the systemd unit from the same pinned release tag:

```bash
sudo install -d -m 0755 /usr/local/lib/systemd/system

curl -fsSL \
  "https://raw.githubusercontent.com/containerd/containerd/v${CONTAINERD_VERSION}/containerd.service" \
  -o containerd.service

sudo install -o root -g root -m 0644 \
  containerd.service \
  /usr/local/lib/systemd/system/containerd.service
```

Reload systemd and start containerd:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
systemctl status containerd --no-pager
```

> `systemctl daemon-reload` and `systemctl enable` require `sudo`. The
> passwordless SSH user cannot authenticate through a PolicyKit password
> prompt.

### Install runc

```bash
RUNC_VERSION="v1.5.2"

curl -fLO \
  "https://github.com/opencontainers/runc/releases/download/${RUNC_VERSION}/runc.amd64"

sudo install -o root -g root -m 0755 \
  runc.amd64 /usr/local/sbin/runc
```

Verify it:

```bash
runc --version
```

### Install the base CNI binaries

These binaries provide low-level network operations such as loopback, bridge,
host-local IPAM, and port mapping. They do **not** create the cluster Pod
network by themselves; Calico is installed after the API server is running.

```bash
CNI_PLUGINS_VERSION="v1.9.1"

curl -fLO \
  "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_VERSION}/cni-plugins-linux-amd64-${CNI_PLUGINS_VERSION}.tgz"

sudo install -d -m 0755 /opt/cni/bin
sudo tar -C /opt/cni/bin -xzvf \
  "cni-plugins-linux-amd64-${CNI_PLUGINS_VERSION}.tgz"
```

Verify several expected binaries:

```bash
ls -l /opt/cni/bin/{bridge,host-local,loopback,portmap}
```

### Configure containerd

Generate the default configuration:

```bash
sudo install -d -m 0755 /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

Configure the runc runtime to use systemd cgroups, matching the kubelet:

```bash
sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml
```

Confirm that the CRI plugin is not listed in `disabled_plugins` and that the
cgroup setting changed:

```bash
grep -nE 'disabled_plugins|SystemdCgroup' /etc/containerd/config.toml
```

Restart and validate containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl is-active containerd
containerd --version
```

If it does not start:

```bash
sudo journalctl -u containerd -n 100 --no-pager
```

---

## 3. Install Kubernetes tools

Run this section on every node. Installing `kubectl` on workers is optional,
but keeping the same toolset on all lab nodes is convenient.

### Configure the Kubernetes v1.37 repository

```bash
KUBERNETES_MINOR="v1.37"

sudo install -d -m 0755 /etc/apt/keyrings
curl -fsSL "https://pkgs.k8s.io/core:/stable:/${KUBERNETES_MINOR}/deb/Release.key" | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBERNETES_MINOR}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list >/dev/null
```

Install and hold the packages:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

The kubelet may restart repeatedly before `kubeadm init` or `kubeadm join`.
That is expected because it does not have a cluster configuration yet.

Check installed versions:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

### Install and configure crictl

Use the same minor version as Kubernetes:

```bash
CRICTL_VERSION="v1.37.0"

curl -fLO "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz"

sudo tar -C /usr/local/bin -xzvf "crictl-${CRICTL_VERSION}-linux-amd64.tar.gz"
```

Configure its containerd endpoint:

```bash
sudo tee /etc/crictl.yaml >/dev/null <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

Verify runtime access:

```bash
sudo crictl version
sudo crictl info
```

### All-node preflight check

Before initializing the cluster, verify on every node:

```bash
swapon --show
systemctl is-active containerd
systemctl is-enabled containerd
sysctl net.ipv4.ip_forward
lsmod | grep -E '^(overlay|br_netfilter)'
sudo crictl info >/dev/null && echo "CRI is healthy"
```

Expected results:

- no active swap;
- containerd is active and enabled;
- IP forwarding is `1`;
- both kernel modules are loaded;
- `crictl info` succeeds.

---

## 4. Initialize the control plane

Run this section only on `cka-cp01`.

### Optional preflight inspection

```bash
sudo kubeadm config images list
sudo kubeadm config images pull
```

### Create the cluster

```bash
sudo kubeadm init --apiserver-advertise-address=192.168.124.51 --pod-network-cidr=10.244.0.0/16
```

Save the complete `kubeadm join` command printed at the end. It contains a
short-lived bootstrap token and the CA certificate hash required by workers.

If initialization fails, inspect the kubelet and control-plane containers
before resetting anything:

```bash
sudo journalctl -u kubelet -n 100 --no-pager
sudo crictl ps -a
```

### Configure kubectl for the regular user

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 0600 "$HOME/.kube/config"
```

Verify API access:

```bash
kubectl cluster-info
kubectl get nodes --output=wide
kubectl get pods --namespace=kube-system
```

The control-plane node remains `NotReady` until a cluster CNI is installed.

---

## 5. Install Calico networking

Run this section only on `cka-cp01` after `kubeadm init`.

Calico is used instead of plain Flannel because the training plan includes
Kubernetes NetworkPolicy enforcement. Calico v3.33 supports Kubernetes v1.37.

The upstream Calico custom resource defaults to `192.168.0.0/16`, which would
overlap this lab's `192.168.124.0/24` VM network. Download it and change the
pool to the same `10.244.0.0/16` CIDR supplied to `kubeadm init`.

```bash
CALICO_VERSION="v3.33.0"

kubectl create --filename \
  "https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/v3_projectcalico_org.yaml"

kubectl create --filename \
  "https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/tigera-operator.yaml"

kubectl wait \
  --for=condition=Established \
  crd/installations.operator.tigera.io \
  --timeout=120s

curl -fsSL \
  "https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/custom-resources.yaml" \
  -o calico-custom-resources.yaml

sed -i \
  's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#' \
  calico-custom-resources.yaml

grep -A6 'ipPools:' calico-custom-resources.yaml
kubectl create --filename=calico-custom-resources.yaml
```

Wait for Calico and the node to become healthy:

```bash
watch -n 2 kubectl get pods --namespace calico-system
```

In another terminal, inspect status:

```bash
kubectl get tigerastatus
kubectl get nodes --output=wide
kubectl get pods --namespace=kube-system
```

Do not install a second cluster CNI, such as Flannel, on top of Calico.

---

## 6. Join the worker nodes

Run the saved command as root on both `cka-worker01` and `cka-worker02`:

```bash
sudo kubeadm join 192.168.124.51:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If the token expires, generate a new complete command on `cka-cp01`:

```bash
sudo kubeadm token create --print-join-command
```

Back on `cka-cp01`, watch the nodes:

```bash
watch -n2 kubectl get nodes --output=wide
```

All three nodes should eventually report `Ready`.

---

## 7. Validate the cluster

### Check system health

```bash
kubectl get nodes --output=wide
kubectl get pods --all-namespaces --output=wide
kubectl get events --all-namespaces --sort-by=.metadata.creationTimestamp
```

### Test scheduling, Service routing, and DNS

```bash
kubectl create deployment web --image=nginx:alpine --replicas=2
kubectl expose deployment web --port=80 --target-port=80

kubectl run client \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600

kubectl wait --for=condition=Ready pod/client --timeout=120s
kubectl get pods --output=wide
kubectl get service web
kubectl get endpointslices \
  --selector=kubernetes.io/service-name=web

kubectl exec client -- nslookup web
kubectl exec client -- wget -qO- http://web
```

Clean up after successful validation:

```bash
kubectl delete deployment web
kubectl delete service web
kubectl delete pod client
```

### Record useful cluster information

```bash
kubectl version
kubectl config current-context
kubectl cluster-info
kubeadm version
containerd --version
```

---

## Troubleshooting quick reference

### Kubelet or node problems

```bash
sudo systemctl status kubelet --no-pager
sudo journalctl -u kubelet -n 100 --no-pager
kubectl describe node <node-name>
```

### Runtime problems

```bash
sudo systemctl status containerd --no-pager
sudo journalctl -u containerd -n 100 --no-pager
sudo crictl info
sudo crictl ps -a
sudo crictl logs <container-id>
```

### Control-plane problems

The kubeadm control plane runs as static Pods defined in
`/etc/kubernetes/manifests`:

```bash
sudo ls -l /etc/kubernetes/manifests
sudo crictl ps -a
sudo journalctl -u kubelet -n 100 --no-pager
```

When the API server is unavailable, use `crictl` and local service logs rather
than `kubectl`.

### Networking problems

```bash
kubectl get pods --namespace=calico-system --output=wide
kubectl get pods --namespace=kube-system --output=wide
kubectl get tigerastatus
kubectl get endpointslices --all-namespaces
ip route
```

Check that the Pod CIDR in the Calico installation matches the CIDR passed to
`kubeadm init`.

---

## Reset a lab node

Use reset only when intentionally rebuilding a disposable lab node.

On the control plane, drain and remove a worker first:

```bash
kubectl drain <worker-name> \
  --ignore-daemonsets \
  --delete-emptydir-data
kubectl delete node <worker-name>
```

On the node being rebuilt:

```bash
sudo kubeadm reset
```

`kubeadm reset` does not necessarily remove CNI configuration, network rules,
or the user's kubeconfig. Inspect them explicitly before reusing the node:

```bash
sudo ls -la /etc/cni/net.d 2>/dev/null || true
sudo iptables-save | less
ls -la "$HOME/.kube" 2>/dev/null || true
```

## References

- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [containerd getting started](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [cri-tools releases](https://github.com/kubernetes-sigs/cri-tools/releases)
- [Calico Kubernetes installation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/)
