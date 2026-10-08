# 04 — Networking

## Goal

Connect workloads reliably and securely using Services, DNS, NetworkPolicies,
Ingress, and the Gateway API.

## Learning objectives

- Explain Pod, Service, and node networking.
- Create and troubleshoot Services and DNS records.
- Restrict traffic with NetworkPolicies.
- Route external HTTP traffic with Ingress or Gateway API resources.

## 1. Networking model

Kubernetes expects:

- Every Pod to have its own cluster IP address.
- Pods to communicate with other Pods without address translation in the
  Kubernetes networking model.
- Agents on a node to communicate with all Pods on that node.
- A CNI plugin to implement Pod networking.

Useful inspection commands:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get endpointslices -A
kubectl get networkpolicies -A
```

## 2. Services

| Type | Exposure |
| --- | --- |
| ClusterIP | Inside the cluster only; the default. |
| NodePort | A port on each node forwards to the Service. |
| LoadBalancer | Requests an external load balancer from an integration. |
| ExternalName | Returns an external DNS name through cluster DNS. |

```bash
kubectl create deployment web --image=nginx:alpine --replicas=2
kubectl expose deployment web --name=web --port=80 --target-port=80
kubectl get service web
kubectl get endpointslices -l kubernetes.io/service-name=web
```

A Service selects Pods by labels. If it has no endpoints, compare its selector
with the labels on ready Pods:

```bash
kubectl describe service web
kubectl get pods --show-labels
```

## 3. DNS

Typical Service names are:

```text
<service>.<namespace>.svc.cluster.local
```

Test lookup and connectivity from inside the cluster:

```bash
kubectl run netcheck --image=busybox:1.36 --restart=Never \
  --command -- sleep 3600
kubectl exec netcheck -- nslookup web
kubectl exec netcheck -- wget -qO- http://web
```

When DNS fails, inspect CoreDNS, its Service, the Pod resolver configuration,
and network connectivity:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get service -n kube-system kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl exec netcheck -- cat /etc/resolv.conf
```

## 4. NetworkPolicies

NetworkPolicies require support from the installed CNI plugin. Policies are
additive: allowed traffic is the union of all policies that select a Pod.

Default-deny ingress for a namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Allow traffic from Pods labeled `access=frontend` to TCP port 8080:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: app
spec:
  podSelector:
    matchLabels:
      role: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Be precise with namespace selectors, Pod selectors, and YAML indentation;
different nesting can mean AND or OR matching.

## 5. Ingress

An Ingress resource requires an installed Ingress controller.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  rules:
    - host: web.example.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

Inspect both the resource and the controller logs when routing fails:

```bash
kubectl describe ingress web
kubectl get ingressclass
kubectl get pods -A | grep -i ingress
```

## 6. Gateway API

Gateway API separates infrastructure ownership from application routes:

- `GatewayClass`: controller implementation.
- `Gateway`: listening infrastructure and allowed routes.
- `HTTPRoute`: application HTTP routing rules.

Gateway API custom resources and a compatible controller must be installed
before these objects can operate.

```bash
kubectl get gatewayclasses
kubectl get gateways -A
kubectl get httproutes -A
```

## Practice tasks

1. Expose a Deployment through a ClusterIP Service and verify its endpoints.
2. Break the Service selector, diagnose the missing endpoints, and repair it.
3. Test cross-namespace Service DNS using the fully qualified name.
4. Apply default-deny ingress and permit only one labeled client.
5. Route a hostname to a Service through the controller available in your lab.

## Completion checklist

- [ ] You can trace traffic from a Service to its EndpointSlices and Pods.
- [ ] You can diagnose common CoreDNS failures.
- [ ] You can write and test a NetworkPolicy.
- [ ] You understand that Ingress and Gateway resources require controllers.
- [ ] You can distinguish Ingress from Gateway API ownership models.
