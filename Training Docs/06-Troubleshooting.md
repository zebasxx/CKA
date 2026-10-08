# 06 — Troubleshooting

## Goal

Diagnose failures systematically across workloads, nodes, networking, storage,
and control-plane components.

## Learning objectives

- Start with symptoms and narrow the failing layer.
- Use events, status, logs, and node services efficiently.
- Recover broken workloads and cluster components.
- Validate each repair from the user's perspective.

## 1. A repeatable workflow

1. Confirm the kubeconfig context, namespace, and reported symptom.
2. Check the broad state with `get` before changing anything.
3. Use `describe` and events to find scheduling and lifecycle failures.
4. Read current and previous logs.
5. Inspect dependencies: configuration, Services, DNS, storage, and nodes.
6. Make the smallest justified repair.
7. Re-run the original failing action to prove recovery.

```bash
kubectl config current-context
kubectl get nodes
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

## 2. Pod failures

```bash
kubectl describe pod <pod>
kubectl logs <pod> --all-containers
kubectl logs <pod> -c <container> --previous
kubectl get pod <pod> -o yaml
```

Common states:

| Symptom | Likely investigation |
| --- | --- |
| `Pending` | Resources, selectors, taints, PVCs, scheduler events. |
| `ImagePullBackOff` | Image name, tag, registry access, pull secret. |
| `CrashLoopBackOff` | Logs, command, configuration, probes, dependencies. |
| `CreateContainerConfigError` | Missing ConfigMap, Secret, key, or volume. |
| `Terminating` | Finalizers, volumes, node health, grace period. |

For minimal images without a shell, attach an ephemeral debug container when
the cluster permits it:

```bash
kubectl debug -it pod/<pod> --image=busybox:1.36 --target=<container>
```

## 3. Node failures

```bash
kubectl describe node <node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
sudo systemctl status kubelet
sudo journalctl -u kubelet --since "20 minutes ago"
sudo crictl ps -a
sudo crictl logs <container-id>
```

Check disk pressure, memory pressure, PID pressure, runtime status, certificate
errors, and connectivity to the API server. Confirm that the kubelet uses the
expected configuration and container runtime endpoint.

## 4. Control-plane failures

On a kubeadm control plane, verify static manifests and containers locally:

```bash
sudo ls -l /etc/kubernetes/manifests
sudo crictl ps -a
sudo crictl logs <control-plane-container-id>
sudo journalctl -u kubelet --since "20 minutes ago"
```

Check listening ports and certificates:

```bash
sudo ss -lntp
sudo kubeadm certs check-expiration
```

If the API server is unavailable, `kubectl` cannot show its Pods or logs;
inspect the runtime and manifest files directly on the node.

## 5. Service and DNS failures

Trace the full request path:

```text
client Pod -> DNS -> Service -> EndpointSlice -> ready backend Pod -> port
```

```bash
kubectl get service <service> -o yaml
kubectl get endpointslices -l kubernetes.io/service-name=<service>
kubectl get pods --show-labels
kubectl exec <client> -- nslookup <service>
kubectl exec <client> -- wget -qO- http://<service>:<port>
```

Compare `port`, `targetPort`, container listening ports, selectors, readiness,
and NetworkPolicies.

## 6. Storage failures

```bash
kubectl get pv,pvc
kubectl describe pvc <claim>
kubectl describe pod <pod>
kubectl get storageclass
kubectl get volumeattachments
```

Events usually distinguish provisioning, binding, attachment, and mount
failures. Continue to CSI controller/node logs or kubelet logs as indicated.

## 7. Useful output techniques

```bash
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get pods -o wide --show-labels
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].restartCount}'
kubectl explain pod.spec.containers.livenessProbe
kubectl diff -f repaired.yaml
```

Avoid deleting evidence too early. A forced Pod deletion can hide the event,
status, or previous logs that explain the failure.

## Broken-cluster drills

Practice identifying and repairing each fault without being told the cause:

1. Deployment with a nonexistent image tag.
2. Pod referencing a missing ConfigMap key.
3. Service selector that matches no Pods.
4. Incorrect Service `targetPort`.
5. Pod blocked by a node taint.
6. PVC referencing a nonexistent StorageClass.
7. Kubelet stopped on a worker.
8. Invalid flag added to a static Pod manifest.
9. NetworkPolicy blocking required DNS or application traffic.
10. Readiness probe using the wrong path or port.

## Completion checklist

- [ ] You can diagnose a Pending or crashing Pod from evidence.
- [ ] You can inspect components when the API server is unavailable.
- [ ] You can trace Service traffic to a ready backend.
- [ ] You can separate PVC binding, attachment, and mount failures.
- [ ] You validate repairs by repeating the original failing action.
