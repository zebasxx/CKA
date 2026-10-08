# 02 — Cluster Administration

## Goal

Operate and maintain a Kubernetes cluster securely, including access control,
certificates, etcd, node maintenance, and version upgrades.

## Learning objectives

- Switch safely between kubeconfig contexts.
- Create RBAC permissions with the principle of least privilege.
- Inspect and renew cluster certificates.
- Back up and restore etcd.
- Drain nodes and perform a staged `kubeadm` upgrade.

## 1. Kubeconfig and contexts

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context-name>
kubectl config set-context --current --namespace=<namespace>
```

Always confirm the context before changing a cluster:

```bash
kubectl config current-context
kubectl cluster-info
```

## 2. Role-based access control

RBAC combines permissions with subjects:

- `Role`: permissions inside one namespace.
- `ClusterRole`: cluster-wide permissions or reusable namespaced permissions.
- `RoleBinding`: grants a Role or ClusterRole inside one namespace.
- `ClusterRoleBinding`: grants a ClusterRole across the cluster.

Create a read-only Role and bind it to a user:

```bash
kubectl create namespace team-a
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n team-a
kubectl create rolebinding alice-pod-reader \
  --role=pod-reader \
  --user=alice \
  -n team-a
```

Test authorization before relying on it:

```bash
kubectl auth can-i list pods -n team-a --as=alice
kubectl auth can-i delete pods -n team-a --as=alice
kubectl auth can-i --list -n team-a --as=alice
```

ServiceAccounts represent workloads rather than human users:

```bash
kubectl create serviceaccount reporter -n team-a
kubectl create rolebinding reporter-read \
  --clusterrole=view \
  --serviceaccount=team-a:reporter \
  -n team-a
```

## 3. Certificates

Inspect kubeadm-managed certificate expiration:

```bash
sudo kubeadm certs check-expiration
```

Inspect an individual certificate:

```bash
sudo openssl x509 \
  -in /etc/kubernetes/pki/apiserver.crt \
  -noout -subject -issuer -dates
```

Renew certificates only as part of an approved maintenance procedure:

```bash
sudo kubeadm certs renew all
```

After renewal, restart affected static Pods or the kubelet as required, and
refresh copied kubeconfig files such as the user's admin configuration.

## 4. Static Pods and control-plane components

Control-plane components installed by kubeadm are usually static Pods defined
under `/etc/kubernetes/manifests`.

```bash
sudo ls -l /etc/kubernetes/manifests
kubectl get pods -n kube-system -o wide
sudo journalctl -u kubelet --since "10 minutes ago"
```

The kubelet watches the manifest directory. Back up a manifest before editing
it, and keep backup files outside that directory so the kubelet does not try to
run them as additional static Pods.

## 5. etcd backup and restore

Discover endpoint and certificate paths from the etcd static Pod manifest:

```bash
sudo grep -E -- '--(listen-client-urls|cert-file|key-file|trusted-ca-file)' \
  /etc/kubernetes/manifests/etcd.yaml
```

Create and verify a snapshot using the values from that manifest:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot save /var/backups/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=<ca-file> \
  --cert=<cert-file> \
  --key=<key-file>

sudo ETCDCTL_API=3 etcdctl snapshot status /var/backups/etcd.db \
  --write-out=table
```

Restore into a new data directory, then update the etcd static Pod manifest to
use it:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot restore /var/backups/etcd.db \
  --data-dir=/var/lib/etcd-restored
```

Before a real restore, back up the current manifest and data and confirm the
cluster topology. A restore is a disruptive operation.

## 6. Node maintenance

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

Review PodDisruptionBudgets when a drain cannot evict workloads:

```bash
kubectl get pdb -A
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
```

## 7. Cluster upgrades

Read the version-specific release and upgrade notes first. Upgrade one minor
version at a time and keep kubelet versions from exceeding the API server
version.

Typical control-plane sequence:

1. Upgrade `kubeadm`.
2. Run `kubeadm upgrade plan`.
3. Run `kubeadm upgrade apply <version>` on the first control-plane node.
4. Drain the node, upgrade `kubelet` and `kubectl`, restart kubelet, and
   uncordon it.
5. Upgrade additional control-plane nodes, then workers one at a time.

Core commands include:

```bash
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply <version>
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl get nodes
```

## Practice tasks

1. Grant a ServiceAccount permission to read ConfigMaps in only one namespace.
2. Prove with `kubectl auth can-i` that it cannot delete them.
3. Check all kubeadm-managed certificate expiration dates.
4. Take an etcd snapshot and validate it.
5. Cordon, drain, and uncordon a worker while observing workload movement.

## Completion checklist

- [ ] You can explain RoleBinding versus ClusterRoleBinding.
- [ ] You can locate control-plane manifests and certificates.
- [ ] You can create and validate an etcd snapshot.
- [ ] You can place a node into and out of maintenance.
- [ ] You know the safe order for a kubeadm cluster upgrade.
