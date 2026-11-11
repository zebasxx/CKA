# Kubernetes Storage

Kubernetes provides a flexible storage abstraction that lets you decouple
application data from the lifecycle of individual Pods. This note covers the
recommended storage types for different data categories and the key differences
between block and object storage.

## Tiered Storage Design

| Data Type | Recommended Storage |
| --- | --- |
| Declarative configuration | `ConfigMap` |
| Credentials and keys | `Secret` |
| Disposable temporary / cache files | `emptyDir` |
| Durable files used by one application replica | RWO block-storage PVC |
| Files shared by multiple replicas | RWX NFS PVC |
| Large downloads / generated artifacts | Object storage when supported; otherwise RWX NFS |
| Databases | Dedicated RWO block storage, not general-purpose NFS |

## Core Differences

| Feature | Block Storage | Object Storage |
| --- | --- | --- |
| Data Structure | Splits data into raw, fixed-size blocks with unique addresses but no metadata. | Stores data as self-contained objects in a flat address space (no folders). |
| Access Method | Accessed like a local hard drive via protocols like iSCSI or Fibre Channel. | Accessed over the web via HTTP/HTTPS RESTful APIs. |
| Metadata | Minimal (only basic storage characteristics). | Highly customizable, rich metadata attached to every single object. |
| Performance | Extremely low latency and fast random access. | Higher throughput for large files but higher initial latency. |
| Scalability | Harder to scale horizontally; limited by the volume capacity. | Near-infinite horizontal scalability across nodes and data centers. |
| Modifiability | Mutable: You can change a single byte or block within a file. | Immutable: To change a file, you must overwrite the entire object. |
| Best Used For | Databases, Virtual Machine boot disks, and ERP software. | Backups, archives, logs, videos, images, and AI training datasets. |

## Persistent Volumes - PV

In our K8 infrastructure, we need to provide at least one PV option so applications can define PVC and use the selected storages options for our cluster.
Below a popular production block-storage options list:

- Ceph RBD
  Distributed, replicated block storage commonly integrated with Kubernetes through Ceph CSI. It can be deployed inside Kubernetes using Rook or operated as an external Ceph cluster. Highly capable but operationally complex.
- Longhorn
  Kubernetes-native replicated block storage. It stores volume replicas across dedicated disks on multiple Kubernetes nodes. Easier to deploy and operate than Ceph, making it suitable for smaller on-premises clusters.
- Portworx
  Commercial Kubernetes-native storage platform from Pure Storage. Provides replicated block volumes, snapshots, backups, encryption, disaster recovery, and enterprise support.
- Red Hat OpenShift Data Foundation
  Red Hat’s supported storage platform, primarily built around Ceph and Rook. Common in production OpenShift environments and provides block, filesystem, and object storage.
- VMware vSphere CSI
  Creates virtual disks on vSphere datastores and attaches them to Kubernetes VMs. Common when Kubernetes nodes run on VMware infrastructure.
- Proxmox CSI
  Provisions virtual disks using Proxmox storage backends. Most available implementations are community projects, so maturity, compatibility, and support must be evaluated carefully before production use.
- Enterprise SAN CSI
  Vendor CSI drivers for storage arrays from NetApp, Dell, HPE, Pure Storage, IBM, Hitachi, and others. Kubernetes dynamically creates LUNs or volumes on the enterprise array. Often the preferred choice when supported storage infrastructure already exists.
- Amazon EBS CSI
  AWS-managed block storage for Kubernetes and EKS. Volumes reside in an AWS Availability Zone and attach to worker instances.
- Azure Disk CSI
  Azure-managed block storage for AKS and Kubernetes running on Azure virtual machines.
- Google Persistent Disk CSI
  Google Cloud block storage for GKE and Kubernetes clusters running on Google Compute Engine.
- OpenEBS
  Kubernetes-native storage framework supporting several storage engines. Mayastor provides replicated NVMe-oriented block storage, while LocalPV exposes node-local disks.
- LINSTOR with DRBD
  Replicated block storage managed by LINSTOR and DRBD. Provides synchronous block replication between Linux nodes and integrates with Kubernetes through a CSI driver.
- Local Persistent Volumes
  Exposes a disk directly from one Kubernetes node. Provides strong local performance but no built-in replication or automatic movement to another node. Usually combined with application-level replication.

For our cluster we will use Longhorn for now.

## Longhorn configuration:

Longhorn will provide dynamically provisioned, replicated block storage for
applications that need `ReadWriteOnce` (`RWO`) volumes. The storage layer will
use these three Kubernetes nodes:

| Node | Address | Purpose |
| --- | --- | --- |
| `krk-sw-db01-l` | `10.11.1.51` | Kubernetes worker and Longhorn replica host |
| `krk-sw-db02-l` | `10.11.1.52` | Kubernetes worker and Longhorn replica host |
| `krk-sw-db03-l` | `10.11.1.53` | Kubernetes worker and Longhorn replica host |

Each node must have a dedicated virtual disk for Longhorn data. The three
replicas must reside on independent disks and nodes:

```text
krk-sw-db01-l ── dedicated disk ── Longhorn replica 1
krk-sw-db02-l ── dedicated disk ── Longhorn replica 2
krk-sw-db03-l ── dedicated disk ── Longhorn replica 3
```

The DB VMs must be Kubernetes nodes. Longhorn is Kubernetes-native storage;
it does not use an external standalone Longhorn storage server.

The normal application workflow is dynamic provisioning:

```text
Dedicated disks on the three DB nodes
                ↓
       Longhorn storage pool
                ↓
Application creates a PVC using longhorn-rwo
                ↓
Longhorn CSI creates the block volume
                ↓
Kubernetes automatically creates and binds the PV
                ↓
The volume is attached and mounted in the application Pod
```

Applications therefore create PVCs. Administrators do not manually create a
PV for every application.

### 1. Prepare the three DB nodes as Kubernetes workers

Follow the common node preparation, containerd, kubelet, kubeadm and proxy
instructions in [ClusterConfiguration + Proxy.md](<ClusterConfiguration + Proxy.md>)
on all three DB nodes. Do not run `kubeadm init` on them.

On an existing control-plane node, generate a fresh worker join command:

```bash
sudo kubeadm token create --print-join-command --ttl 2h
```

Run the complete command that it prints on each DB node. It should use the
shared API endpoint:

```text
k8s-api-lab.argostranslations.com:6443
```

From `krk-sw-mgr01-l`, confirm that all three nodes joined and are ready:

```bash
kubectl get nodes -o wide
```

The result must contain `krk-sw-db01-l`, `krk-sw-db02-l` and
`krk-sw-db03-l` with status `Ready`.

Label their operational role. This label is informational and does not taint
the nodes:

```bash
kubectl label node krk-sw-db01-l node-role.kubernetes.io/storage=storage
kubectl label node krk-sw-db02-l node-role.kubernetes.io/storage=storage
kubectl label node krk-sw-db03-l node-role.kubernetes.io/storage=storage
```

### 2. Create and mount the Longhorn data partition

Each DB VM already has one `240 GiB` virtual disk, `/dev/sda`. The cloned image
uses only its first `40 GiB`; the remaining `200 GiB` is unallocated. The
current layout on all three DB nodes is:

```text
/dev/sda       240 GiB virtual disk
├─/dev/sda1     1 GiB EFI
├─/dev/sda2     2 GiB /boot
├─/dev/sda3  36.9 GiB LVM for / and /var
└─             200 GiB unallocated space
```

For this lab, create `/dev/sda4` in the unallocated space and dedicate that
partition to Longhorn. No additional Proxmox disk is required. This provides
filesystem separation, but it is not failure isolation from the OS disk.

> **Warning:** These commands modify the partition table of the OS disk. Run
> them only on `krk-sw-db01-l`, `krk-sw-db02-l` and `krk-sw-db03-l`, whose
> layouts have been verified as shown above. A wrong device or partition
> command can make the VM unbootable.

Run the following on each DB node. First save a copy of its current GPT and
display the unused space:

```bash
sudo sgdisk --backup=/root/sda-gpt-before-longhorn.bin /dev/sda
sudo parted -s /dev/sda unit GiB print free
```

The output must show approximately `200 GiB` free from `40 GiB` to `240 GiB`.
The cloned GPT backup header still records the template's original `40 GiB`
disk boundary. Relocate that backup header to the actual end of the virtual
disk, then create partition 4 using all remaining space:

```bash
sudo sgdisk --move-second-header /dev/sda
sudo sgdisk --new=4:0:0 --typecode=4:8300 --change-name=4:longhorn /dev/sda
sudo partprobe /dev/sda
sudo udevadm settle
```

Verify the result before creating a filesystem:

```bash
sudo sgdisk --print /dev/sda
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS
findmnt --source /dev/sda4 || true
sudo wipefs --no-act /dev/sda4
```

The result must contain an unmounted `/dev/sda4` of approximately `200 GiB`
with no filesystem. Do not continue if `/dev/sda4` is mounted or has an
unexpected signature.

Format and mount only partition 4:

```bash
sudo mkfs.ext4 -L longhorn /dev/sda4
sudo install -d -m 0755 /var/lib/longhorn
export LONGHORN_UUID="$(sudo blkid -s UUID -o value /dev/sda4)"
echo "UUID=${LONGHORN_UUID} /var/lib/longhorn ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo mount /var/lib/longhorn
findmnt /var/lib/longhorn
df -h /var/lib/longhorn
```

Test the persistent mount before installing Longhorn:

```bash
sudo umount /var/lib/longhorn
sudo systemctl daemon-reload
sudo mount -a
findmnt /var/lib/longhorn
```

The final command must show `/dev/sda4` mounted at `/var/lib/longhorn`.

### 3. Install the host prerequisites

Longhorn's V1 data engine uses iSCSI to present block devices. Install these
packages on every worker that may run an application using a Longhorn volume:
`krk-sw-int01-l`, `krk-sw-int02-l` and all three DB nodes.

```bash
sudo apt-get update
sudo apt-get install -y open-iscsi nfs-common
echo iscsi_tcp | sudo tee /etc/modules-load.d/longhorn.conf
sudo modprobe iscsi_tcp
sudo systemctl enable --now iscsid
systemctl is-active iscsid
lsmod | grep '^iscsi_tcp'
```

`nfs-common` is required when Longhorn RWX volumes are used later. The initial
StorageClass in this document provides RWO block volumes.

### 4. Mark only the DB nodes as Longhorn storage nodes

Run these commands before installing Longhorn. The Helm setting used in the
next section makes Longhorn create its default data disk only on nodes carrying
this label:

```bash
kubectl label node krk-sw-db01-l node.longhorn.io/create-default-disk=true
kubectl label node krk-sw-db02-l node.longhorn.io/create-default-disk=true
kubectl label node krk-sw-db03-l node.longhorn.io/create-default-disk=true
```

Verify the labels:

```bash
kubectl get nodes -L node.longhorn.io/create-default-disk
```

The label controls where Longhorn stores replicas. It does not prevent an
application on `krk-sw-int01-l` or `krk-sw-int02-l` from mounting a Longhorn
volume over the cluster network.

### 5. Install Longhorn with Helm

Run this section from `krk-sw-mgr01-l`, where `kubectl` is configured. Helm 3
must be installed first:

```bash
helm version
kubectl get --raw=/readyz
```

The containerd systemd proxy config applies only to containerd image pulls; it
does not configure the interactive shell or Helm. Export the proxy variables
in the current shell before accessing the chart repository:

```bash
export http_proxy=http://10.0.7.10:3128 https_proxy=http://10.0.7.10:3128 HTTP_PROXY=http://10.0.7.10:3128 HTTPS_PROXY=http://10.0.7.10:3128 no_proxy=localhost,127.0.0.1,::1,10.11.1.0/24,10.96.0.0/12,10.244.0.0/16,k8s-api-lab.argostranslations.com,.argostranslations.com,.argosmultilingual.com NO_PROXY=localhost,127.0.0.1,::1,10.11.1.0/24,10.96.0.0/12,10.244.0.0/16,k8s-api-lab.argostranslations.com,.argostranslations.com,.argosmultilingual.com
```

Verify both external HTTPS access through the proxy and direct access to the
Kubernetes API before running Helm:

```bash
curl -fsSIL --connect-timeout 5 --max-time 20 https://charts.longhorn.io/index.yaml | head
curl --noproxy '*' -kfsS --max-time 5 https://k8s-api-lab.argostranslations.com:6443/readyz
```

The first command must return an HTTP success response and the second must
print `ok`.

Add the official repository and install the pinned Longhorn release:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm upgrade --install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.13.0 \
  --set defaultSettings.createDefaultDiskLabeledNodes=true \
  --set defaultSettings.defaultDataPath=/var/lib/longhorn \
  --set persistence.defaultClass=false \
  --set persistence.defaultClassReplicaCount=3
```

The shell and containerd proxy configuration from the cluster build must be in
place so that Helm can reach the chart repository and the nodes can pull the
Longhorn images.

Wait for the installation and inspect its state:

```bash
kubectl --namespace longhorn-system get pods --watch
```

Press `Ctrl+C` after all Pods are `Running`. Then verify the Longhorn nodes:

```bash
kubectl --namespace longhorn-system get nodes.longhorn.io
kubectl --namespace longhorn-system get pods -o wide
```

All three DB nodes must appear in the Longhorn node list. Check the disk paths
that Longhorn registered:

```bash
kubectl --namespace longhorn-system get nodes.longhorn.io \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.disks[*]}{"  "}{.path}{"\n"}{end}{end}'
```

Only the DB nodes should show `/var/lib/longhorn` as a schedulable replica
disk. Do not continue until the three storage nodes and their disks are
available.

### 6. Create the application StorageClass

Create a dedicated StorageClass instead of making Longhorn the implicit
cluster default. This makes each application's storage choice explicit.

```bash
kubectl apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-rwo
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  dataLocality: "disabled"
  fsType: "ext4"
EOF
```

Important properties:

- `numberOfReplicas: "3"` stores one replica on each DB storage node.
- `reclaimPolicy: Retain` protects the underlying volume when its PVC is
  accidentally deleted. Releasing a PV does not make it reusable
  automatically; an administrator must deliberately recover or remove it.
- `allowVolumeExpansion: true` permits a PVC to request a larger volume later.
- The class is not marked as the default. Applications must specify
  `storageClassName: longhorn-rwo`.

Verify it:

```bash
kubectl get storageclass longhorn-rwo
```

### 7. Verify dynamic provisioning with a PVC

Create a test namespace and PVC:

```bash
kubectl create namespace storage-test

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test
  namespace: storage-test
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-rwo
  resources:
    requests:
      storage: 1Gi
EOF
```

Kubernetes and Longhorn should dynamically create and bind the PV:

```bash
kubectl --namespace storage-test get pvc longhorn-test --watch
kubectl get pv
```

Press `Ctrl+C` after the PVC status becomes `Bound`. No PV was manually
defined; it was created by the Longhorn CSI provisioner.

Create a Pod that mounts the claim and writes a file:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-test
  namespace: storage-test
spec:
  containers:
    - name: test
      image: busybox:1.37
      command: ["sh", "-c", "echo 'Longhorn storage works' > /data/result.txt; sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: longhorn-test
EOF
```

Verify the mount and data:

```bash
kubectl --namespace storage-test wait \
  --for=condition=Ready pod/longhorn-test \
  --timeout=180s
kubectl --namespace storage-test exec longhorn-test -- cat /data/result.txt
kubectl --namespace storage-test describe pvc longhorn-test
```

The expected file content is:

```text
Longhorn storage works
```

Because this StorageClass uses `Retain`, do not delete the test PVC unless its
retained PV and Longhorn volume will also be reviewed and deliberately cleaned
up.

### 8. PVC template for an application

An application that needs durable RWO block storage can use this template:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: application-data
  namespace: application-namespace
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-rwo
  resources:
    requests:
      storage: 20Gi
```

The application Deployment or StatefulSet then references the claim:

```yaml
spec:
  containers:
    - name: application
      image: example/application:version
      volumeMounts:
        - name: data
          mountPath: /var/lib/application
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: application-data
```

Use one RWO PVC per database replica unless the database operator explicitly
documents a different storage design. Longhorn replica count protects against
a storage-node failure; it does not replace application-aware database
replication or off-cluster backups.


