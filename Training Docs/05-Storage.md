# 05 — Storage

## Goal

Provision and troubleshoot persistent storage for Kubernetes workloads.

## Learning objectives

- Explain volumes, PersistentVolumes, and PersistentVolumeClaims.
- Use static and dynamic provisioning.
- Select access modes, reclaim policies, and volume modes.
- Attach storage to Pods and StatefulSets.
- Diagnose claims that remain Pending and volumes that fail to mount.

## 1. Storage objects

| Object | Purpose |
| --- | --- |
| Volume | Storage attached directly through a Pod specification. |
| PersistentVolume (PV) | Cluster storage resource with an independent lifecycle. |
| PersistentVolumeClaim (PVC) | A workload's request for persistent storage. |
| StorageClass | Defines a dynamic provisioning method and its policy. |

Common access modes are `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`, and
`ReadWriteOncePod`. Actual support depends on the storage driver.

## 2. Static provisioning

The following `hostPath` PV is suitable only for a single-node practice lab:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lab-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /var/lib/lab-data
```

Request matching storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
```

Inspect binding:

```bash
kubectl get pv,pvc
kubectl describe pvc lab-data
```

## 3. Mount a claim in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "date >> /data/log && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: lab-data
```

Verify data from the container:

```bash
kubectl exec writer -- cat /data/log
```

## 4. Dynamic provisioning

List available StorageClasses and identify the default:

```bash
kubectl get storageclass
kubectl describe storageclass <name>
```

A dynamic claim names a StorageClass and does not require you to create a PV:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: <storage-class>
  resources:
    requests:
      storage: 2Gi
```

`volumeBindingMode: WaitForFirstConsumer` delays provisioning or binding until
a Pod is scheduled, allowing topology-aware placement.

## 5. Reclaim policies and expansion

- `Retain` preserves the backing data after the claim is deleted and requires
  manual reclamation.
- `Delete` removes the dynamically provisioned volume with the claim, subject
  to the driver and policy.

PVC expansion requires support from the driver and
`allowVolumeExpansion: true` on the StorageClass. Request expansion by
increasing `spec.resources.requests.storage`; do not reduce a PVC's size.

## 6. StatefulSets

StatefulSets use `volumeClaimTemplates` to create one stable claim per replica.
Deleting or scaling down a StatefulSet does not necessarily delete its claims,
which protects data but requires deliberate cleanup.

```bash
kubectl get statefulsets
kubectl get pvc
kubectl describe statefulset <name>
```

## 7. Troubleshooting workflow

For a Pending PVC:

```bash
kubectl describe pvc <claim>
kubectl get pv
kubectl get storageclass
kubectl get events --sort-by=.metadata.creationTimestamp
```

Check for:

- Missing or misspelled StorageClass names.
- Capacity or access-mode mismatch.
- A selector that matches no PV.
- Topology constraints with `WaitForFirstConsumer`.
- Provisioner or CSI controller errors.

For mount failures:

```bash
kubectl describe pod <pod>
kubectl get volumeattachments
kubectl get pods -A | grep -i csi
sudo journalctl -u kubelet --since "15 minutes ago"
```

## Practice tasks

1. Create a static PV and bind it to a PVC.
2. Mount the claim, write data, recreate the Pod, and prove persistence.
3. Create a dynamically provisioned claim using the lab's StorageClass.
4. Diagnose a claim with an incorrect StorageClass name.
5. Inspect the reclaim policy before and after deleting a test claim.

## Completion checklist

- [ ] You can match a PVC to a suitable PV.
- [ ] You understand static versus dynamic provisioning.
- [ ] You can explain access modes and reclaim policies.
- [ ] You can identify scheduling and CSI-related storage failures.
- [ ] You can verify that data survives Pod recreation.
