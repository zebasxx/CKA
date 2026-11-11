# Kubernetes Cluster Configuration with HA Proxy Nodes

These notes build the primary CKA practice cluster with `kubeadm` on Ubuntu.
Commands are intended for Kubernetes **v1.37** on `amd64` unless noted
otherwise.

> Keep component versions pinned while building the cluster. Before reusing
> these notes for another Kubernetes minor release, review the upstream
> installation and version-skew documentation.

## Lab topology

| Node | Role | Address | vCPU | RAM |
| --- | --- | --- | ---: | ---: |
| `krk-sw-mgr01-l` | Control plane | `10.11.1.46` | 2 | 4 GiB |
| `krk-sw-mgr02-l` | Control plane | `10.11.1.47` | 2 | 4 GiB |
| `krk-sw-mgr03-l` | Control plane | `10.11.1.48` | 2 | 4 GiB |
| `krk-sw-int01-l` | Worker | `10.11.1.49` | 4 | 8 GiB |
| `krk-sw-int02-l` | Worker | `10.11.1.50` | 4 | 8 GiB |
| `krk-sw-proxy01-l` | External load balancer | `10.11.1.54` | 2 | 4 GiB |
| `krk-sw-proxy02-l` | External load balancer | `10.11.1.55` | 2 | 4 GiB |

The proxy nodes share `10.11.1.56` through Keepalived. HAProxy listens on the
VIP and distributes Kubernetes API connections across all three control-plane
nodes. The proxy nodes are not Kubernetes nodes and do not run Pods or etcd.
This initial configuration exposes only the API on port `6443`. After an
in-cluster Traefik deployment exists, HAProxy can also expose ports `80` and
`443` and forward them to Traefik's worker-node endpoints.

| Endpoint | Address | Purpose |
| --- | --- | --- |
| Kubernetes API DNS | `k8s-api-lab.argostranslations.com` | Stable kubeadm and client endpoint. |
| Kubernetes API VIP | `10.11.1.56:6443` | Keepalived VIP and HAProxy frontend. |
| Outbound Squid proxy | `10.0.7.10:3128` | Internet downloads and container image pulls. |

Network ranges:

| Network | CIDR | Notes |
| --- | --- | --- |
| VM network | `10.11.1.0/24` | Network used by the Terraform-created VMs. |
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
| Keepalived | Moves the shared VIP between the two external proxy nodes. |
| HAProxy | Load-balances TCP connections to healthy API servers. |

`kubectl` creates and manages API objects such as Pods, Deployments, Services,
Secrets, and Ingresses. It does not manage containers directly.

`kubelet` runs on every node. It:

- watches the API for Pods assigned to its node;
- asks the container runtime to create, stop, and restart containers;
- runs startup, readiness, and liveness probes;
- mounts volumes and injects configuration;
- reports node and Pod status to the API server.

## Installation overview

1. Configure DNS, HAProxy, Keepalived, and the API VIP on both proxy nodes.
2. Prepare all five Kubernetes nodes, including outbound proxy access.
3. Install containerd, runc, and the base CNI binaries on every Kubernetes node.
4. Configure containerd to use systemd cgroups and the outbound proxy.
5. Install `kubelet`, `kubeadm`, `kubectl`, and `crictl`.
6. Initialize the first control-plane node through the API VIP.
7. Join the other two control-plane nodes through the API VIP.
8. Install Calico.
9. Join the workers.
10. Validate node readiness, API failover, DNS, and Service connectivity.

---

## 1. Configure the external API load balancers

Complete this section before running `kubeadm init`. The API DNS record must
exist in the internal DNS service:

```text
k8s-api-lab.argostranslations.com  A  10.11.1.56
```

Verify it from both proxy nodes and all five Kubernetes nodes:

```bash
getent ahostsv4 k8s-api-lab.argostranslations.com
```

The result must contain `10.11.1.56`.

Network firewalls must allow TCP `6443` from Kubernetes nodes and authorized
administration clients to both proxy nodes, TCP `6443` from both proxy nodes
to all three managers, and unicast VRRP traffic between the two proxy nodes.

### Install HAProxy and Keepalived

Run on both `krk-sw-proxy01-l` and `krk-sw-proxy02-l`:

```bash
sudo apt-get update
sudo apt-get install -y haproxy keepalived netcat-openbsd iputils-arping

id -u keepalived_script >/dev/null 2>&1 || \
  sudo useradd \
    --system \
    --no-create-home \
    --shell /usr/sbin/nologin \
    keepalived_script

sudo install -d -m 0755 /usr/local/libexec
```

Apply the Keepalived networking settings on both proxy nodes:

```bash
sudo tee /etc/sysctl.d/99-keepalived.conf >/dev/null <<'EOF'
net.ipv4.ip_nonlocal_bind=1
net.ipv4.conf.all.arp_ignore=1
net.ipv4.conf.all.arp_announce=2
net.ipv4.conf.default.arp_ignore=1
net.ipv4.conf.default.arp_announce=2
net.ipv4.conf.all.arp_notify=1
EOF

sudo sysctl --system
```

### Configure HAProxy

Run on both proxy nodes:

```bash
sudo cp --preserve /etc/haproxy/haproxy.cfg \
  /etc/haproxy/haproxy.cfg.before-kubernetes

sudo tee /etc/haproxy/haproxy.cfg >/dev/null <<'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend kubernetes_api
    bind *:6443
    default_backend kubernetes_api_servers

backend kubernetes_api_servers
    balance roundrobin
    option tcp-check
    server mgr01 10.11.1.46:6443 check fall 3 rise 2
    server mgr02 10.11.1.47:6443 check fall 3 rise 2
    server mgr03 10.11.1.48:6443 check fall 3 rise 2
EOF

sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable haproxy
sudo systemctl restart haproxy
systemctl is-active haproxy
sudo ss -lntp | grep ':6443'
```

Before kubeadm initializes the API servers, HAProxy will report that its
backends are unavailable. That is expected; the frontend must still listen on
TCP port `6443`.

### Install the Keepalived HAProxy health check

Run on both proxy nodes:

```bash
sudo tee /usr/local/libexec/check-haproxy.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -o errexit

nc -z 127.0.0.1 6443
EOF

sudo chown root:root /usr/local/libexec/check-haproxy.sh
sudo chmod 0755 /usr/local/libexec/check-haproxy.sh
```

### Configure Keepalived node variables

On `krk-sw-proxy01-l`, set:

```bash
export KEEPALIVED_NODE_IP="10.11.1.54"
export KEEPALIVED_PEER_IP="10.11.1.55"
export KEEPALIVED_PRIORITY="200"
```

On `krk-sw-proxy02-l`, set:

```bash
export KEEPALIVED_NODE_IP="10.11.1.55"
export KEEPALIVED_PEER_IP="10.11.1.54"
export KEEPALIVED_PRIORITY="150"
```

On each node, obtain the same Keepalived authentication value securely and
place it in an environment variable. Keepalived uses only the first eight
characters for `auth_type PASS`:

```bash
while :; do read -rsp 'Keepalived shared password: ' KEEPALIVED_AUTH_PASSWORD; echo; [[ "$KEEPALIVED_AUTH_PASSWORD" =~ ^[A-Za-z0-9]{8,}$ ]] && break; echo 'Use at least eight alphanumeric characters'; done
```

Validate all required values before writing the configuration:

```bash
: "${KEEPALIVED_NODE_IP:?required}"
: "${KEEPALIVED_PEER_IP:?required}"
: "${KEEPALIVED_PRIORITY:?required}"
: "${KEEPALIVED_AUTH_PASSWORD:?required}"
```

### Configure and start Keepalived

Run the following common block on both proxy nodes after setting their
node-specific variables. Environment variables do not survive an SSH
reconnection, so set and validate them in the same terminal session that
writes the configuration:

```bash
sudo tee /etc/keepalived/keepalived.conf >/dev/null <<EOF
global_defs {
  enable_script_security
  script_user keepalived_script
}

vrrp_script chk_haproxy {
  script "/usr/local/libexec/check-haproxy.sh"
  interval 2
  fall 2
  rise 2
}

vrrp_instance VI_56 {
  state BACKUP
  interface eth0
  virtual_router_id 56
  priority ${KEEPALIVED_PRIORITY}
  advert_int 1
  preempt_delay 10

  authentication {
    auth_type PASS
    auth_pass ${KEEPALIVED_AUTH_PASSWORD:0:8}
  }

  unicast_src_ip ${KEEPALIVED_NODE_IP}
  unicast_peer {
    ${KEEPALIVED_PEER_IP}
  }

  virtual_ipaddress {
    10.11.1.56/24 dev eth0
  }

  track_script {
    chk_haproxy
  }

  garp_master_delay 1
  garp_master_refresh 10
  garp_master_repeat 2
}
EOF

unset KEEPALIVED_AUTH_PASSWORD

sudo chown root:root /etc/keepalived/keepalived.conf
sudo chmod 0600 /etc/keepalived/keepalived.conf
sudo keepalived -t -f /etc/keepalived/keepalived.conf
sudo systemctl enable keepalived
sudo systemctl restart keepalived
systemctl is-active keepalived
```

Do not start or restart Keepalived if `keepalived -t` reports an error. Stop
the service, correct or regenerate the configuration, and repeat the test
until it exits successfully with no configuration errors.

### Validate the API VIP

One proxy node, normally `krk-sw-proxy01-l`, must own the VIP:

```bash
ip -brief address show dev eth0
```

From any Kubernetes node, verify DNS and the HAProxy listener:

```bash
getent ahostsv4 k8s-api-lab.argostranslations.com
nc -zv -w 2 k8s-api-lab.argostranslations.com 6443
```

The TCP connection must succeed. HAProxy will have no healthy API backend
until the first control-plane node is initialized.

Do not continue until the VIP, DNS record, HAProxy, and Keepalived are all
working.

---

## 2. Prepare every Kubernetes node

Run this section on all five nodes:

- `krk-sw-mgr01-l`
- `krk-sw-mgr02-l`
- `krk-sw-mgr03-l`
- `krk-sw-int01-l`
- `krk-sw-int02-l`

### Confirm identity and connectivity

```bash
hostnamectl --static
ip -brief address
ip route
```

Confirm that every node has a unique hostname, IP address, and MAC address.
Test connectivity to the other nodes and the load balancer. Name resolution is
required because the shared API endpoint uses its DNS name.

```bash
ping -c 2 10.11.1.46
ping -c 2 10.11.1.47
ping -c 2 10.11.1.48
ping -c 2 10.11.1.49
ping -c 2 10.11.1.50
ping -c 2 10.11.1.54
ping -c 2 10.11.1.55
ping -c 2 10.11.1.56
getent ahostsv4 k8s-api-lab.argostranslations.com
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

### Configure temporary outbound proxy access

LAB nodes do not have a direct Internet route. APT already uses its configured
proxy, but interactive tools such as `curl`, `wget`, and `kubectl` require
proxy environment variables. Run on every Kubernetes node in each shell used
for installation:

```bash
export http_proxy="http://10.0.7.10:3128"
export https_proxy="http://10.0.7.10:3128"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"

export no_proxy="localhost,127.0.0.1,::1,10.11.1.0/24,10.96.0.0/12,10.244.0.0/16,k8s-api-lab.argostranslations.com,.argostranslations.com,.argosmultilingual.com"
export NO_PROXY="$no_proxy"
```

The Kubernetes node, Service, and Pod networks must remain in `NO_PROXY` so
that Squid does not intercept internal API or workload traffic.

Verify both outbound and direct API endpoint handling:

```bash
curl -fsSIL --max-time 20 https://github.com | head
curl --noproxy '*' -kfsS --max-time 5 \
  https://k8s-api-lab.argostranslations.com:6443/readyz || true
```

Before `kubeadm init`, HAProxy has no healthy API-server backend. The second
command may therefore report an empty reply or `SSL_ERROR_SYSCALL` after
HAProxy accepts and closes the connection. It must not time out or return a
Squid `Forbidden` response. After the first API server starts, `/readyz`
should return `ok`.

### Create a temporary working directory

```bash
mkdir -p "$HOME/k8s-files"
cd "$HOME/k8s-files"
```

---

## 3. Install the container runtime

Run this section on every Kubernetes node. Do not install the container runtime
on the external proxy nodes for this architecture.

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

Configure the containerd systemd service to use Squid for external image
registries while bypassing all cluster-local networks. Shell proxy variables
alone are not inherited by the containerd service:

```bash
sudo install -d -m 0755 /etc/systemd/system/containerd.service.d

sudo tee /etc/systemd/system/containerd.service.d/proxy.conf >/dev/null <<'EOF'
[Service]
Environment="HTTP_PROXY=http://10.0.7.10:3128"
Environment="HTTPS_PROXY=http://10.0.7.10:3128"
Environment="NO_PROXY=localhost,127.0.0.1,::1,10.11.1.0/24,10.96.0.0/12,10.244.0.0/16,k8s-api-lab.argostranslations.com,.argostranslations.com,.argosmultilingual.com"
EOF

sudo systemctl daemon-reload
sudo systemctl restart containerd
sudo systemctl --no-pager show containerd --property=Environment
systemctl is-active containerd
```

If it does not start:

```bash
sudo journalctl -u containerd -n 100 --no-pager
```

---

## 4. Install Kubernetes tools

Run this section on every Kubernetes node. Do not install kubelet or kubeadm on
the external proxy nodes. Installing `kubectl` on workers is optional, but
keeping the same toolset on all Kubernetes nodes is convenient.

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

## 5. Initialize the first control-plane node

Run this section only on `krk-sw-mgr01-l`.

The three control-plane nodes use stacked etcd, which is the kubeadm default.
The shared endpoint is `k8s-api-lab.argostranslations.com:6443`, which resolves
to the Keepalived VIP. HAProxy distributes connections to healthy API servers.

### Optional preflight inspection

```bash
sudo kubeadm config images list --kubernetes-version=v1.37.2
sudo kubeadm config images pull --kubernetes-version=v1.37.2
```

Using an explicit version prevents kubeadm from trying to discover the latest
stable version from `dl.k8s.io`. Image downloads are performed by containerd,
which uses the systemd proxy drop-in configured earlier.

### Create the cluster

```bash
sudo kubeadm init \
  --apiserver-advertise-address=10.11.1.46 \
  --control-plane-endpoint=k8s-api-lab.argostranslations.com:6443 \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs \
  --kubernetes-version=v1.37.2
```

Save both complete `kubeadm join` commands printed at the end:

- the control-plane command contains `--control-plane` and a short-lived
  `--certificate-key`;
- the worker command does not contain those options.

If initialization fails, inspect the kubelet and control-plane containers
before changing cluster state:

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

## 6. Join the remaining control-plane nodes

Run the control-plane join command printed by `kubeadm init` as root on both
`krk-sw-mgr02-l` and `krk-sw-mgr03-l`. It has this form:

```bash
sudo kubeadm join k8s-api-lab.argostranslations.com:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

The uploaded certificate secret is deleted after two hours, after which its
decryption key can no longer be used for a join. If necessary, upload the
certificates again on `krk-sw-mgr01-l`:

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

Copy the new certificate key from the output. If the bootstrap token also
expired, generate a new worker-style join command and add `--control-plane`
and the new `--certificate-key` when joining each control-plane node:

```bash
sudo kubeadm token create --print-join-command
```

After both joins complete, verify the control-plane and etcd static Pods from
`krk-sw-mgr01-l`:

```bash
kubectl get nodes --output=wide
kubectl get pods --namespace=kube-system --output=wide
```

All three nodes may remain `NotReady` until Calico is installed.

---

## 7. Install Calico networking

Run this section only on `krk-sw-mgr01-l` after all three control-plane nodes
have joined.

Calico is used instead of plain Flannel because the training plan includes
Kubernetes NetworkPolicy enforcement. Calico v3.33 supports Kubernetes v1.37.

The upstream Calico custom resource defaults to `192.168.0.0/16`. Change the
pool to the same `10.244.0.0/16` CIDR supplied to `kubeadm init` so the
configuration is explicit and consistent.

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

## 8. Join the worker nodes

Run the saved worker command as root on both `krk-sw-int01-l` and
`krk-sw-int02-l`:

```bash
sudo kubeadm join k8s-api-lab.argostranslations.com:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If the token expires, generate a new complete command on
`krk-sw-mgr01-l`:

```bash
sudo kubeadm token create --print-join-command
```

Back on `krk-sw-mgr01-l`, watch the nodes:

```bash
watch -n2 kubectl get nodes --output=wide
```

All five nodes should eventually report `Ready`.

---

## 9. Validate the cluster

### Check system health

```bash
kubectl get nodes --output=wide
kubectl get pods --all-namespaces --output=wide
kubectl get events --all-namespaces --sort-by=.metadata.creationTimestamp
```

Confirm that kubeconfig uses the load-balanced DNS endpoint rather than an
individual manager address:

```bash
kubectl config view \
  --minify \
  --output=jsonpath='{.clusters[0].cluster.server}{"\n"}'
```

Expected value:

```text
https://k8s-api-lab.argostranslations.com:6443
```

### Test API load balancing and VIP failover

On both proxy nodes, verify HAProxy and Keepalived:

```bash
systemctl is-active haproxy keepalived
sudo journalctl -u haproxy -u keepalived -n 50 --no-pager
```

Identify which proxy currently owns `10.11.1.56`:

```bash
ip -brief address show dev eth0
```

From `krk-sw-mgr01-l`, continuously query the API while testing failover:

```bash
watch -n1 kubectl get --raw=/readyz
```

On the proxy that currently owns the VIP, stop HAProxy:

```bash
sudo systemctl stop haproxy
```

Within several seconds, the VIP should move to the other proxy and API queries
should resume after only a brief interruption. Verify the VIP on the other
proxy, then restore HAProxy on the original node:

```bash
sudo systemctl start haproxy
systemctl is-active haproxy
```

Repeat the test in the other direction if both failover paths must be proven.

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

## References

- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Creating highly available clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [HAProxy health checks](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/health-checks/)
- [containerd getting started](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [cri-tools releases](https://github.com/kubernetes-sigs/cri-tools/releases)
- [Calico Kubernetes installation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/)
