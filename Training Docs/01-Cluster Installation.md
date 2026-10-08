# 01 — Cluster Installation

## Goal

Build a working Kubernetes cluster with `kubeadm`, connect a worker node, and
verify that the control plane, networking, and DNS are healthy.

## Learning objectives

By the end of this module, you should be able to:

- Prepare Linux nodes for Kubernetes.
- Explain the roles of `kubeadm`, `kubelet`, and `kubectl`.
- Initialize a control plane and save its join command.
- Install a Container Network Interface (CNI) plugin.
- Join and remove worker nodes safely.
- Reset and rebuild a lab cluster.

## 1. Understand the components

| Component | Purpose |
| --- | --- |
| `kubeadm` | Bootstraps and upgrades a conformant cluster. |
| `kubelet` | Runs on every node and manages Pods assigned to that node. |
| `kubectl` | Sends commands to the Kubernetes API server. |
| Container runtime | Runs containers; `containerd` is a common choice. |
| CNI plugin | Implements Pod networking and network policy capabilities. |

`kubeadm` creates the cluster, but it does not install a CNI plugin or manage
the operating system prerequisites for you.

## 2. Prepare every node

Use at least one control-plane node and one worker node. Give each node a
unique hostname, IP address, and MAC address.

### Disable swap

```bash
sudo swapoff -a
swapon --show
```

Also remove or comment out persistent swap entries in `/etc/fstab` for the
lab. The command `swapon --show` should return no active swap devices.

### Load kernel modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Persist the modules in `/etc/modules-load.d/k8s.conf`:

```text
overlay
br_netfilter
```

### Configure networking sysctls

Create `/etc/sysctl.d/k8s.conf` with:

```text
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
```

Apply and verify the settings:

```bash
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

### Install the runtime and Kubernetes tools

Install a CRI-compatible runtime, then install `kubelet`, `kubeadm`, and
`kubectl` from the Kubernetes package repository for the version used by your
lab. Package repository URLs and supported versions change over time, so use
the official installation guide for this step.

For `containerd`, ensure its CRI plugin is enabled and its cgroup driver is
compatible with the kubelet. A common configuration uses systemd cgroups:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

Restart the services after changing their configuration:

```bash
sudo systemctl restart containerd
sudo systemctl enable --now kubelet
```

The kubelet may restart repeatedly before `kubeadm init`; that is expected.

## 3. Initialize the control plane

Choose a Pod CIDR compatible with your CNI plugin:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

For a cluster with a stable API endpoint, add a DNS name or virtual IP:

```bash
sudo kubeadm init \
  --control-plane-endpoint=cluster-api.example.internal:6443 \
  --pod-network-cidr=10.244.0.0/16
```

Save the printed `kubeadm join` command. It contains the discovery token and
CA certificate hash needed by worker nodes.

Configure `kubectl` for the current non-root user:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
kubectl get nodes
```

## 4. Install a CNI plugin

Apply the manifest supplied by the CNI provider selected for your lab:

```bash
kubectl apply -f <cni-manifest-url-or-file>
```

The plugin's configured Pod CIDR must agree with the value passed to
`kubeadm init`. Watch the system Pods until they become ready:

```bash
kubectl get pods -n kube-system -w
kubectl get nodes -o wide
```

## 5. Join a worker node

Run the saved join command as root on the worker:

```bash
sudo kubeadm join <api-server>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If the token expired, generate a new complete command on the control plane:

```bash
sudo kubeadm token create --print-join-command
```

Verify the result:

```bash
kubectl get nodes
kubectl describe node <worker-name>
```

## 6. Validate the cluster

Create a small deployment and test its Service and DNS:

```bash
kubectl create deployment web --image=nginx:alpine --replicas=2
kubectl expose deployment web --port=80
kubectl run client --image=busybox:1.36 --restart=Never \
  --command -- sleep 3600
kubectl wait --for=condition=Ready pod/client --timeout=90s
kubectl exec client -- nslookup web
kubectl exec client -- wget -qO- http://web
```

Clean up the validation workload:

```bash
kubectl delete deployment web
kubectl delete service web
kubectl delete pod client
```

## 7. Remove or rebuild a node

Drain a worker before planned removal:

```bash
kubectl drain <worker-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <worker-name>
```

Reset the removed node locally:

```bash
sudo kubeadm reset
```

`kubeadm reset` does not necessarily remove CNI configuration, firewall
rules, or `$HOME/.kube/config`; inspect those separately when rebuilding a lab.

## Practice tasks

1. Build a one-control-plane, one-worker cluster.
2. Generate a new join token and add the worker without copying the original
   command.
3. Deploy an application and prove that Pod-to-Service DNS works.
4. Drain, remove, reset, and rejoin the worker.
5. Locate the static Pod manifests and kubelet configuration created by
   `kubeadm`.

## Completion checklist

- [ ] All nodes report `Ready`.
- [ ] CoreDNS Pods report `Running` and `Ready`.
- [ ] A test Pod can resolve and reach a Service.
- [ ] You can generate a new worker join command.
- [ ] You can identify `/etc/kubernetes/manifests` and kubelet logs.
