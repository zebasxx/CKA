# 03 — Workloads and Scheduling

## Goal

Create, configure, scale, and schedule workloads using Kubernetes controllers
and Pod specifications.

## Learning objectives

- Choose the correct workload controller.
- Generate and edit manifests quickly.
- Configure applications with ConfigMaps and Secrets.
- Use requests, limits, probes, and autoscaling.
- Control Pod placement with selectors, affinity, taints, and tolerations.

## 1. Workload controllers

| Controller | Best suited for |
| --- | --- |
| Deployment | Stateless applications and rolling updates. |
| StatefulSet | Stable identity and storage for stateful applications. |
| DaemonSet | One Pod on every eligible node. |
| Job | Finite work that must complete. |
| CronJob | Jobs started on a schedule. |

Create starter manifests imperatively, then edit them:

```bash
kubectl create deployment web --image=nginx:alpine \
  --replicas=3 --dry-run=client -o yaml > web.yaml
kubectl create job report --image=busybox:1.36 \
  --dry-run=client -o yaml -- echo done
kubectl create cronjob cleanup --image=busybox:1.36 \
  --schedule="*/10 * * * *" --dry-run=client -o yaml -- echo cleanup
```

Apply and inspect:

```bash
kubectl apply -f web.yaml
kubectl get deploy,rs,pods
kubectl rollout status deployment/web
```

## 2. Updates and rollbacks

```bash
kubectl set image deployment/web nginx=nginx:1.27-alpine
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```

Use `spec.strategy.rollingUpdate.maxUnavailable` and `maxSurge` to control a
Deployment rollout.

## 3. ConfigMaps and Secrets

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-file=config.properties

kubectl create secret generic db-credentials \
  --from-literal=username=app \
  --from-literal=password='<password>'
```

Consume values as environment variables or mounted files. Secrets are
base64-encoded in manifests, not automatically encrypted at rest. Avoid
putting decoded secret values in terminal history or source control.

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

## 4. Resources and probes

Requests influence scheduling; limits constrain resource use.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```

- Readiness decides whether a Pod receives Service traffic.
- Liveness decides whether a container should be restarted.
- Startup probes protect slow-starting applications from premature liveness
  failures.

## 5. Scheduling

### Node selectors

```bash
kubectl label node <node> disk=ssd
```

```yaml
spec:
  nodeSelector:
    disk: ssd
```

### Taints and tolerations

```bash
kubectl taint node <node> dedicated=batch:NoSchedule
```

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: batch
    effect: NoSchedule
```

A toleration permits scheduling on a tainted node; it does not guarantee that
the Pod will be placed there. Combine it with node affinity when placement must
be targeted.

### Affinity

- Node affinity selects nodes by labels with required or preferred rules.
- Pod affinity attracts Pods toward other Pods.
- Pod anti-affinity spreads Pods away from one another.
- Topology spread constraints distribute replicas across zones or nodes.

## 6. Autoscaling

The HorizontalPodAutoscaler typically depends on a metrics provider and
resource requests:

```bash
kubectl autoscale deployment web --cpu-percent=60 --min=2 --max=10
kubectl get hpa
kubectl top pods
```

## 7. Fast inspection and editing

```bash
kubectl get pod <pod> -o yaml
kubectl describe pod <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> -c <container> --previous
kubectl edit deployment <name>
kubectl explain deployment.spec.template.spec.affinity
```

## Practice tasks

1. Create a three-replica Deployment and perform a rolling image update.
2. Roll back the update after deliberately choosing an invalid image.
3. Inject one ConfigMap key and one Secret key into a Pod.
4. Schedule a workload only on SSD-labeled nodes.
5. Taint a node for batch work and create a matching workload.
6. Create a Job that completes successfully and a CronJob that runs it.

## Completion checklist

- [ ] You can choose among Deployment, StatefulSet, DaemonSet, and Job.
- [ ] You can diagnose a failed rollout.
- [ ] You can configure probes and resource requests.
- [ ] You can predict the effect of taints, tolerations, and affinity.
- [ ] You can create and inspect an HPA.
