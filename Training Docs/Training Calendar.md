# CKA Training Calendar

This calendar divides the complete training path into **35 daily sessions of
one hour each**. At one session per day, the plan takes five weeks. If rest
days are added, allow six to seven calendar weeks.

## Estimated effort

| Module | Sessions | Hours |
| --- | ---: | ---: |
| [Cluster Installation](<01-Cluster Installation>) | 5 | 5 |
| [Cluster Administration](<02-Cluster Administration>) | 6 | 6 |
| [Workloads and Scheduling](<03-Workloads and Scheduling>) | 5 | 5 |
| [Networking](<04-Networking>) | 5 | 5 |
| [Storage](<05-Storage>) | 4 | 4 |
| [Troubleshooting](<06-Troubleshooting>) | 5 | 5 |
| [CKA Simulations](<07-CKA Simulations>) | 5 | 5 |
| **Total** | **35** | **35** |

The estimate assumes that a disposable Kubernetes lab is already available
or can be rebuilt during the installation sessions. Extra time may be needed
for initial virtual machine or cloud infrastructure setup.

## Standard one-hour session

Unless a day specifies a different split, use this structure:

- **5 minutes:** Recall the previous session without looking at notes.
- **15 minutes:** Read and summarize the assigned material.
- **35 minutes:** Perform the commands or practical exercise.
- **5 minutes:** Verify the result and record questions or mistakes.

Do not mark a day complete just because the material was read. Complete the
stated outcome in the lab whenever the required feature is available.

## Step 1 — Cluster installation

### Day01 — Components and lab design

- [ ] Read **Goal**, **Learning objectives**, and **Understand the components**
  in [Cluster Installation](<01-Cluster Installation>).
- [ ] Identify the control-plane node, worker node, container runtime, and CNI
  plugin that the lab will use.
- [ ] Draw or write the expected traffic path between `kubectl`, the API
  server, kubelet, runtime, and Pods.
- **Outcome:** Explain the purpose of `kubeadm`, `kubelet`, `kubectl`, the
  runtime, and the CNI without consulting the document.

### Day02 — Prepare the nodes

- [ ] Cover **Prepare every node** through the kernel module and sysctl steps.
- [ ] Disable swap and verify the required kernel networking settings.
- [ ] Install or inspect the container runtime and Kubernetes tools.
- [ ] Confirm that `containerd` and `kubelet` are enabled.
- **Outcome:** Both nodes satisfy the operating system prerequisites for
  `kubeadm`.

### Day03 — Initialize the control plane

- [ ] Cover **Initialize the control plane** and **Install a CNI plugin**.
- [ ] Run `kubeadm init` with a Pod CIDR compatible with the selected CNI.
- [ ] Configure the non-root user's kubeconfig.
- [ ] Install the CNI and observe the system Pods becoming ready.
- **Outcome:** The control-plane node reports `Ready`, and CoreDNS is running.

### Day04 — Join and validate a worker

- [ ] Cover **Join a worker node** and **Validate the cluster**.
- [ ] Generate a fresh join command and add the worker.
- [ ] Deploy the example web workload and client Pod.
- [ ] Prove that Service DNS and HTTP connectivity work.
- **Outcome:** All nodes are `Ready`, and the client reaches the Service.

### Day05 — Node lifecycle and installation review

- [ ] Cover **Remove or rebuild a node**, **Practice tasks**, and the module
  checklist.
- [ ] Drain, remove, reset, and rejoin the worker.
- [ ] Locate the static Pod manifests and kubelet logs.
- [ ] Complete every item in the installation checklist.
- **Outcome:** Rebuild a worker and verify cluster health without following a
  copied step-by-step procedure.

## Step 2 — Cluster administration

### Day06 — Kubeconfig and contexts

- [ ] Cover **Kubeconfig and contexts** in
  [Cluster Administration](<02-Cluster Administration>).
- [ ] List, inspect, and switch contexts.
- [ ] Set a default namespace on a practice context.
- [ ] Practice targeting a context with `--context` without changing the
  current context.
- **Outcome:** Always identify the target cluster and namespace before making
  a change.

### Day07 — RBAC foundations

- [ ] Cover the Role, ClusterRole, RoleBinding, and ClusterRoleBinding concepts.
- [ ] Create the `team-a` namespace, `pod-reader` Role, and user binding.
- [ ] Test allowed and denied actions with `kubectl auth can-i`.
- **Outcome:** Explain the scope of each RBAC object and create a namespaced
  least-privilege permission.

### Day08 — ServiceAccounts and RBAC practice

- [ ] Finish **Role-based access control**.
- [ ] Create the example `reporter` ServiceAccount and binding.
- [ ] Build a new Role that can read ConfigMaps but cannot update or delete
  them.
- [ ] Verify permissions by impersonating the ServiceAccount.
- **Outcome:** Grant and verify workload identity permissions without using an
  unnecessarily broad binding.

### Day09 — Certificates and static Pods

- [ ] Cover **Certificates** and **Static Pods and control-plane components**.
- [ ] Check kubeadm-managed certificate expiration.
- [ ] Inspect the API server certificate with OpenSSL.
- [ ] Map each file in `/etc/kubernetes/manifests` to its running static Pod.
- **Outcome:** Locate control-plane certificates and manifests and explain how
  the kubelet manages static Pods.

### Day10 — etcd backup and restore

- [ ] Cover **etcd backup and restore**.
- [ ] Find the etcd endpoint and certificate paths in its static Pod manifest.
- [ ] Save an etcd snapshot and validate its status.
- [ ] In a disposable environment, restore the snapshot into a new data
  directory and identify the required manifest change.
- **Outcome:** Produce a validated backup and describe the restore procedure.

### Day11 — Maintenance and upgrades

- [ ] Cover **Node maintenance**, **Cluster upgrades**, the practice tasks, and
  the checklist.
- [ ] Cordon, drain, and uncordon a worker while watching workload placement.
- [ ] Write the correct upgrade order for control-plane and worker nodes.
- [ ] Run `kubeadm upgrade plan` if the lab supports it.
- **Outcome:** Place a node into maintenance safely and explain a staged
  kubeadm upgrade.

## Step 3 — Workloads, scheduling, and networking

### Day12 — Workload controllers

- [ ] Cover **Workload controllers** in
  [Workloads and Scheduling](<03-Workloads and Scheduling>).
- [ ] Compare Deployments, StatefulSets, DaemonSets, Jobs, and CronJobs.
- [ ] Generate starter YAML for a Deployment, Job, and CronJob.
- [ ] Apply the objects and inspect their Pods and status.
- **Outcome:** Choose the appropriate controller for a described workload.

### Day13 — Updates and rollbacks

- [ ] Cover **Updates and rollbacks**.
- [ ] Perform a successful rolling update.
- [ ] Introduce an invalid image, diagnose the rollout, and roll it back.
- [ ] Inspect rollout history and the related ReplicaSets.
- **Outcome:** Update and recover a Deployment while preserving availability.

### Day14 — ConfigMaps and Secrets

- [ ] Cover **ConfigMaps and Secrets**.
- [ ] Create configuration from literals and a file.
- [ ] Create a Secret without exposing its decoded value in source control.
- [ ] Consume both objects as environment variables and mounted files.
- **Outcome:** Configure a Pod externally and diagnose a missing key.

### Day15 — Resources and probes

- [ ] Cover **Resources and probes**.
- [ ] Add CPU and memory requests and limits to a Deployment.
- [ ] Configure readiness, liveness, and startup probes.
- [ ] Break one probe deliberately and diagnose the resulting behavior.
- **Outcome:** Explain how requests, limits, and each probe affect a workload.

### Day16 — Scheduling and autoscaling

- [ ] Cover **Scheduling**, **Autoscaling**, and the remaining module sections.
- [ ] Practice a node selector, taint and toleration, and one affinity rule.
- [ ] Create and inspect an HPA if a metrics provider is available.
- [ ] Complete the module checklist.
- **Outcome:** Predict where a Pod can schedule and identify why a Pending Pod
  cannot be placed.

### Day17 — Networking model and Services

- [ ] Cover **Networking model** and **Services** in
  [Networking](<04-Networking>).
- [ ] Create a Deployment and ClusterIP Service.
- [ ] Trace the Service selector to its EndpointSlices and ready Pods.
- [ ] Break the selector, observe the missing endpoints, and repair it.
- **Outcome:** Trace Service traffic to the correct backend Pods.

### Day18 — Cluster DNS

- [ ] Cover **DNS**.
- [ ] Resolve a Service by short name and fully qualified name.
- [ ] Inspect the client Pod's `/etc/resolv.conf`.
- [ ] Inspect CoreDNS Pods, Service, and logs.
- **Outcome:** Diagnose whether a name-resolution failure is caused by the
  client, DNS, or the destination Service.

### Day19 — NetworkPolicies

- [ ] Cover **NetworkPolicies**.
- [ ] Apply default-deny ingress in a practice namespace.
- [ ] Permit traffic only from a correctly labeled client to one port.
- [ ] Prove both the permitted and denied paths.
- **Outcome:** Write and validate a least-privilege ingress policy.

### Day20 — Ingress

- [ ] Cover **Ingress**.
- [ ] Identify the IngressClass and controller available in the lab.
- [ ] Route a test hostname and path to a Service.
- [ ] Inspect the Ingress and controller logs if routing fails.
- **Outcome:** Explain the resource/controller relationship and verify an HTTP
  route end to end.

### Day21 — Gateway API and networking review

- [ ] Cover **Gateway API**, the networking practice tasks, and checklist.
- [ ] Identify GatewayClasses, Gateways, and HTTPRoutes in the lab.
- [ ] Compare the ownership model of Ingress and Gateway API.
- [ ] Repeat one Service, DNS, or policy task without notes.
- **Outcome:** Complete the networking checklist and identify the correct tool
  for each routing requirement.

## Step 4 — Storage and troubleshooting

### Day22 — Persistent storage fundamentals

- [ ] Cover **Storage objects** and **Static provisioning** in
  [Storage](<05-Storage>).
- [ ] Compare PVs, PVCs, volumes, and StorageClasses.
- [ ] Create the practice static PV and matching PVC.
- [ ] Inspect the binding and explain why the objects match.
- **Outcome:** Bind a claim to a suitable static volume.

### Day23 — Mounting and persistence

- [ ] Cover **Mount a claim in a Pod**.
- [ ] Mount the claim, write a marker file, and verify its contents.
- [ ] Recreate the Pod and prove that the data remains.
- [ ] Test and explain the effect of the PV reclaim policy.
- **Outcome:** Demonstrate that storage persists independently of a Pod.

### Day24 — Dynamic provisioning and StatefulSets

- [ ] Cover **Dynamic provisioning**, **Reclaim policies and expansion**, and
  **StatefulSets**.
- [ ] Create a dynamically provisioned claim using the lab StorageClass.
- [ ] Inspect its provisioner, binding mode, and reclaim policy.
- [ ] Inspect the per-replica claims created by a StatefulSet.
- **Outcome:** Explain the lifecycle of dynamically provisioned storage and
  StatefulSet claims.

### Day25 — Storage troubleshooting

- [ ] Cover **Troubleshooting workflow**, practice tasks, and the checklist.
- [ ] Diagnose a PVC with a nonexistent StorageClass.
- [ ] Practice separating provisioning, binding, attachment, and mount errors.
- [ ] Use events, VolumeAttachments, CSI Pods, and kubelet logs as appropriate.
- **Outcome:** Identify the failed storage stage from observable evidence.

### Day26 — Troubleshooting method and Pods

- [ ] Cover **A repeatable workflow** and **Pod failures** in
  [Troubleshooting](<06-Troubleshooting>).
- [ ] Diagnose one Pending Pod and one crashing Pod.
- [ ] Use current logs, previous logs, events, and `describe` output.
- [ ] Attach an ephemeral debug container if the lab supports it.
- **Outcome:** Move from symptom to root cause without deleting the evidence.

### Day27 — Node failures

- [ ] Cover **Node failures**.
- [ ] Stop the kubelet on a disposable worker and observe the cluster state.
- [ ] Inspect node conditions, workloads, systemd logs, and runtime containers.
- [ ] Restore the kubelet and verify node and workload recovery.
- **Outcome:** Diagnose and recover a NotReady worker.

### Day28 — Control-plane failures

- [ ] Cover **Control-plane failures**.
- [ ] Map static manifests to runtime containers and local logs.
- [ ] In a disposable lab, introduce a reversible manifest error and repair it.
- [ ] Verify ports, certificate state, and API availability after recovery.
- **Outcome:** Troubleshoot a control-plane component without depending on a
  functioning Kubernetes API.

## Step 5 — Troubleshooting and simulations

### Day29 — Network and storage incidents

- [ ] Cover **Service and DNS failures**, **Storage failures**, and **Useful
  output techniques**.
- [ ] Diagnose one Service with no endpoints and one Pending PVC.
- [ ] Trace each problem through its dependency chain.
- [ ] Validate both repairs from a client workload.
- **Outcome:** Use a layered workflow for network and storage incidents.

### Day30 — Broken-cluster drills

- [ ] Cover **Broken-cluster drills** and the completion checklist.
- [ ] Select at least five faults from the list and solve them without being
  told their causes.
- [ ] Record the symptom, diagnostic command, repair, and verification for each.
- **Outcome:** Solve five mixed incidents in one hour with evidence for every
  repair.

### Day31 — Simulation 1: Core operations

- [ ] Read the rules and preparation section in
  [CKA Simulations](<07-CKA Simulations>).
- [ ] Spend 45 minutes completing **Simulation 1 — Core operations**.
- [ ] Spend 15 minutes verifying and scoring every task.
- **Outcome:** Complete the core workload scenario within the time limit and
  identify any task that needs repetition.

### Day32 — Simulation 2: Administration and recovery

- [ ] Spend the full hour on **Simulation 2 — Administration and recovery**.
- [ ] Verify RBAC, node health, snapshot status, and certificate information.
- [ ] Score the session immediately when the timer ends.
- **Outcome:** Complete the administrative scenario with no wrong-context
  changes.

### Day33 — Simulation 3: Networking and storage

- [ ] Spend the full hour on **Simulation 3 — Networking and storage**.
- [ ] Test allowed and denied network paths explicitly.
- [ ] Prove data persistence rather than relying only on PVC status.
- [ ] Score the session immediately when the timer ends.
- **Outcome:** Complete and verify the combined network and storage scenario.

### Day34 — Simulation 4: Mixed incidents, part 1

- [ ] Start **Simulation 4 — Mixed incident set** with a clean 60-minute timer.
- [ ] Diagnose the first five prepared incidents.
- [ ] For each incident, record the symptom, decisive evidence, smallest
  repair, and verification.
- **Outcome:** Recover at least five incidents without seeing their setup.

### Day35 — Simulation 4, contexts, and final assessment

- [ ] Spend 30 minutes completing the remaining mixed incidents.
- [ ] Spend 20 minutes on the **Multi-context drill**.
- [ ] Spend 10 minutes scoring the work and completing the **Final readiness
  checklist**.
- [ ] List weak topics that need another focused session before the exam.
- **Outcome:** Finish the entire training path with a documented readiness
  assessment and a short list of targeted follow-up work.

## Progress summary

- [ ] Days 01–05: Cluster Installation
- [ ] Days 06–11: Cluster Administration
- [ ] Days 12–16: Workloads and Scheduling
- [ ] Days 17–21: Networking
- [ ] Days 22–25: Storage
- [ ] Days 26–30: Troubleshooting
- [ ] Days 31–35: CKA Simulations
