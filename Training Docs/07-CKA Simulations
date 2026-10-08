# 07 — CKA Simulations

## Goal

Combine all modules in timed, context-aware exercises that reward accuracy,
verification, and efficient use of Kubernetes documentation.

## Simulation rules

- Use a disposable lab cluster.
- Set a timer and do not pause it while researching.
- Read each task completely before changing the cluster.
- Confirm the kubeconfig context and namespace for every task.
- Verify the requested end state rather than assuming a command succeeded.
- Record skipped tasks and return to them after completing faster work.

## Before starting

```bash
kubectl config get-contexts
kubectl config current-context
kubectl get nodes
```

Useful shell setup for a temporary exam-like session:

```bash
alias k=kubectl
export do='--dry-run=client -o yaml'
```

Do not depend on aliases or editor settings you have not practiced using.

## Simulation 1 — Core operations

Suggested time: 45 minutes.

1. Create namespace `blue`.
2. In `blue`, create a three-replica Deployment named `web` using an nginx
   image available in the lab.
3. Add CPU and memory requests to the container.
4. Expose the Deployment internally on port 80.
5. Create a temporary client Pod and prove Service DNS and HTTP connectivity.
6. Label one worker `workload=blue` and require the Deployment to run there.
7. Perform a rolling image update and confirm it completes.
8. Save all objects in namespace `blue` to a YAML output file for review.

Verification ideas:

```bash
kubectl get all -n blue -o wide
kubectl get endpointslices -n blue
kubectl rollout status deployment/web -n blue
```

## Simulation 2 — Administration and recovery

Suggested time: 60 minutes.

1. Create a namespace-scoped Role that permits reading Pods and their logs.
2. Bind the Role to a ServiceAccount and verify allowed and denied actions.
3. Cordon and drain a worker while preserving DaemonSets.
4. Take and validate an etcd snapshot.
5. Find the expiration dates of kubeadm-managed certificates.
6. Diagnose a worker whose kubelet has stopped.
7. Return the drained worker to service and verify all nodes are Ready.

Verification ideas:

```bash
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<account> \
  -n <namespace>
kubectl get nodes
kubectl get pods -A -o wide
```

## Simulation 3 — Networking and storage

Suggested time: 60 minutes.

1. Create `frontend` and `backend` namespaces.
2. Deploy a backend application and expose it with a ClusterIP Service.
3. Prove connectivity from a client in `frontend`.
4. Apply a default-deny ingress policy in `backend`.
5. Permit traffic only from properly labeled frontend client Pods.
6. Create a PVC using an available StorageClass.
7. Mount it into a Pod, write a marker file, recreate the Pod, and confirm the
   marker remains.
8. Diagnose a prepared Service with no endpoints.

Verification ideas:

```bash
kubectl get networkpolicy -A
kubectl get service,endpointslices -A
kubectl get pv,pvc -A
```

## Simulation 4 — Mixed incident set

Suggested time: 90 minutes.

Prepare or ask a study partner to prepare these faults, then solve them without
seeing the setup steps:

1. One Pod is Pending because of an unmatched node selector.
2. One Deployment is unavailable because of a broken readiness probe.
3. One application cannot start because a Secret key is misspelled.
4. One Service sends traffic to the wrong container port.
5. One PVC cannot bind because its StorageClass does not exist.
6. DNS fails because CoreDNS cannot schedule.
7. A worker reports NotReady because its kubelet is inactive.
8. A control-plane component fails due to a malformed static Pod manifest.

For every incident, record:

- The visible symptom.
- The command that revealed the cause.
- The smallest repair made.
- The command or test that proved recovery.

## Multi-context drill

Use at least two disposable clusters or kubeconfig contexts.

1. List contexts and identify the current one.
2. Create namespace `context-check` in only the requested cluster.
3. Read a Pod in one context without changing the persistent current context:

   ```bash
   kubectl --context=<context> get pods -A
   ```

4. Switch context explicitly, perform a task, and switch back.
5. Verify that no object was created in the wrong cluster.

## Scoring rubric

| Result | Score |
| --- | ---: |
| Correct result with explicit verification | 2 |
| Correct result without verification | 1 |
| Incomplete, incorrect, or wrong context | 0 |

Also track time lost to syntax lookup, incorrect namespaces, and unverified
assumptions. Repeat weak tasks until they can be completed reliably.

## Final readiness checklist

- [ ] I check context and namespace before each task.
- [ ] I generate starter YAML quickly and edit it safely.
- [ ] I use `get`, `describe`, events, and logs in a consistent order.
- [ ] I can troubleshoot workloads, nodes, networking, and storage.
- [ ] I can back up etcd and maintain kubeadm nodes.
- [ ] I verify every completed task.
- [ ] I can complete mixed scenarios within the chosen time limit.
