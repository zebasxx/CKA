# Kubernetes Edge Proxy and Traefik

These notes add application ingress to the kubeadm cluster built in
`ClusterConfiguration + Proxy.md`.

The two proxy nodes form the external **edge load-balancer** layer:

- Keepalived provides the shared VIP `10.11.1.56`;
- HAProxy accepts connections on the VIP and forwards them into Kubernetes;
- Traefik runs inside Kubernetes and routes HTTP requests to Services.

```text
Application client
        |
        | DNS: <application>-k8s-lab.argostranslations.com -> 10.11.1.56
        v
Keepalived VIP 10.11.1.56
        |
        v
HAProxy on proxy01/proxy02
        |
        | TCP 80  -> worker NodePort 30080
        | TCP 443 -> worker NodePort 30443
        v
Traefik on int01/int02
        |
        | Ingress hostname and path rules
        v
Kubernetes Services and Pods
```

HAProxy remains a layer-4 TCP proxy. Traefik performs HTTP routing and TLS
termination. HAProxy sends PROXY protocol v2 so Traefik can record the original
client address.

## Versions

| Component | Version |
| --- | --- |
| Kubernetes | `v1.37.2` |
| Helm | `v3.22.1` |
| Traefik Helm chart | `42.3.1` |
| Traefik Proxy | `v3.7.17` |

Keep versions pinned and review release notes before upgrading. The chart
version and Traefik application version are separate version numbers.

## Prerequisites

Before continuing, verify:

```bash
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get --raw=/readyz
```

All five Kubernetes nodes should be `Ready`, all Calico components should be
healthy, and `/readyz` should return `ok`.

The following network paths must be allowed:

| Source | Destination | Ports | Purpose |
| --- | --- | --- | --- |
| Application clients | `10.11.1.56` | TCP `80`, `443` | Application ingress. |
| Both proxy nodes | `10.11.1.49`, `10.11.1.50` | TCP `30080`, `30443` | Traefik NodePorts. |
| Kubernetes nodes | Internet through Squid | TCP `3128` | Charts and container images. |

Create an individual DNS record for each LAB application. Do not change the
corporate-wide `*.argostranslations.com` DNS record:

```text
whoami-k8s-lab.argostranslations.com  A  10.11.1.56
grafana-k8s-lab.argostranslations.com A  10.11.1.56
```

The naming pattern `<application>-k8s-lab.argostranslations.com` keeps the
entire application name in one DNS label, allowing the existing
`*.argostranslations.com` certificate to match it.

Keep the Kubernetes API record separate:

```text
k8s-api-lab.argostranslations.com  A  10.11.1.56
```

## 1. Configure the installation shell

Run the Kubernetes commands from `krk-sw-mgr01-l` as `cicdadm`.

LAB requires the outbound Squid proxy. The Kubernetes networks, internal
domains, and API endpoint must bypass it:

```bash
export http_proxy="http://10.0.7.10:3128"
export https_proxy="http://10.0.7.10:3128"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export no_proxy="localhost,127.0.0.1,::1,10.11.1.0/24,10.96.0.0/12,10.244.0.0/16,k8s-api-lab.argostranslations.com,.argostranslations.com,.argosmultilingual.com"
export NO_PROXY="$no_proxy"
```

Verify both paths:

```bash
curl -fsSIL --max-time 20 https://get.helm.sh | head
kubectl get --raw=/readyz
```

## 2. Install Helm

Download and verify the pinned Helm release:

```bash
HELM_VERSION="v3.22.1"

curl -fLO \
  "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz"

curl -fLO \
  "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz.sha256sum"

sha256sum --check \
  "helm-${HELM_VERSION}-linux-amd64.tar.gz.sha256sum"
```

Extract and install the client:

```bash
tar -xzf "helm-${HELM_VERSION}-linux-amd64.tar.gz"
sudo install -o root -g root -m 0755 linux-amd64/helm /usr/local/bin/helm
helm version
```

Helm uses the same kubeconfig as `kubectl`:

```bash
helm list --all-namespaces
```

## 3. Select the Traefik worker nodes

Run one Traefik Pod on each current worker. Label the nodes explicitly so a
future general-purpose worker is not automatically added to the ingress tier:

```bash
kubectl label node krk-sw-int01-l ingress-ready=true
kubectl label node krk-sw-int02-l ingress-ready=true
```

Verify:

```bash
kubectl get nodes --show-labels | grep 'ingress-ready=true'
```

Traefik will use a DaemonSet with this label selector. Combined with
`externalTrafficPolicy: Local`, each HAProxy backend has a local Traefik Pod.
If a local Traefik Pod is unavailable, that worker's NodePort stops being a
healthy backend.

## 4. Add the official Traefik chart repository

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm search repo traefik/traefik --versions | head
```

Set the pinned chart version:

```bash
TRAEFIK_CHART_VERSION="42.3.1"
```

Inspect the selected release before installation:

```bash
helm show chart traefik/traefik \
  --version "$TRAEFIK_CHART_VERSION"
```

## 5. Create the Traefik Helm values

Create a working directory:

```bash
mkdir -p "$HOME/k8s-files/traefik"
cd "$HOME/k8s-files/traefik"
```

Create `values.yaml`:

```bash
tee values.yaml >/dev/null <<'EOF'
deployment:
  enabled: true
  kind: DaemonSet

nodeSelector:
  ingress-ready: "true"

service:
  enabled: true
  spec:
    type: NodePort
    externalTrafficPolicy: Local

ports:
  web:
    port: 8000
    expose:
      default: true
    exposedPort: 80
    nodePort: 30080
    proxyProtocol:
      trustedIPs:
        - "10.11.1.54/32"
        - "10.11.1.55/32"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    port: 8443
    expose:
      default: true
    exposedPort: 443
    nodePort: 30443
    proxyProtocol:
      trustedIPs:
        - "10.11.1.54/32"
        - "10.11.1.55/32"
    http:
      tls:
        enabled: true

ingressClass:
  enabled: true
  isDefaultClass: false
  name: traefik

providers:
  kubernetesIngress:
    enabled: true
    publishedService:
      enabled: false
    ingressEndpoint:
      ip: "10.11.1.56"
    strictPrefixMatching: true
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: false
    defaultTLSResourcesNamespace: traefik
  kubernetesGateway:
    enabled: false

core:
  strictTLSOptions: true

api:
  dashboard: true
  insecure: false

ingressRoute:
  dashboard:
    enabled: false

log:
  level: INFO
  format: json

accessLog:
  enabled: true
  format: json
  fields:
    headers:
      defaultMode: drop

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
EOF
```

Important properties of this configuration:

- fixed NodePorts make the HAProxy backends stable;
- the DaemonSet creates one Traefik Pod on each `ingress-ready` worker;
- `externalTrafficPolicy: Local` prevents cross-node forwarding before
  Traefik receives the connection;
- only the two HAProxy node addresses are trusted to send PROXY protocol;
- HTTP redirects to HTTPS at the Traefik `web` entrypoint;
- the dashboard is not exposed through the edge proxy;
- applications must explicitly select `ingressClassName: traefik`.

Render the chart locally and review the resource summary before installing:

```bash
helm template traefik traefik/traefik \
  --namespace traefik \
  --version "$TRAEFIK_CHART_VERSION" \
  --values values.yaml >/dev/null
```

## 6. Install Traefik

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --version "$TRAEFIK_CHART_VERSION" \
  --values values.yaml \
  --wait \
  --timeout 5m
```

Verify the release and Kubernetes resources:

```bash
helm list --namespace traefik
kubectl get daemonset,pods,service --namespace traefik --output=wide
kubectl get ingressclass traefik
```

Expected results:

- one Traefik Pod runs on `krk-sw-int01-l`;
- one Traefik Pod runs on `krk-sw-int02-l`;
- the Traefik Service is type `NodePort`;
- `web` uses `30080` and `websecure` uses `30443`;
- the `traefik` IngressClass exists.

Check the actual Service ports:

```bash
kubectl get service traefik --namespace traefik \
  --output=jsonpath='{range .spec.ports[*]}{.name}{" -> "}{.nodePort}{"\n"}{end}'
```

Inspect logs if either Pod is not ready:

```bash
kubectl logs --namespace traefik \
  --selector=app.kubernetes.io/name=traefik \
  --tail=100
```

## 7. Validate the Traefik NodePorts

From both `krk-sw-proxy01-l` and `krk-sw-proxy02-l`, verify all four backend
connections:

```bash
nc -zv -w 2 10.11.1.49 30080
nc -zv -w 2 10.11.1.49 30443
nc -zv -w 2 10.11.1.50 30080
nc -zv -w 2 10.11.1.50 30443
```

Do not modify HAProxy until all four checks succeed.

## 8. Add application ingress to HAProxy

Apply the following complete HAProxy configuration on both proxy nodes. It
retains the Kubernetes API frontend and adds HTTP and HTTPS frontends for
Traefik.

Configure and restart `krk-sw-proxy02-l` first. Validate it, then repeat on
`krk-sw-proxy01-l` so the active VIP always has a prepared peer.

```bash
sudo cp --preserve /etc/haproxy/haproxy.cfg \
  /etc/haproxy/haproxy.cfg.before-traefik

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

frontend kubernetes_http
    bind *:80
    default_backend traefik_http

backend traefik_http
    balance roundrobin
    option tcp-check
    server int01 10.11.1.49:30080 check fall 3 rise 2 send-proxy-v2
    server int02 10.11.1.50:30080 check fall 3 rise 2 send-proxy-v2

frontend kubernetes_https
    bind *:443
    default_backend traefik_https

backend traefik_https
    balance roundrobin
    option tcp-check
    server int01 10.11.1.49:30443 check fall 3 rise 2 send-proxy-v2
    server int02 10.11.1.50:30443 check fall 3 rise 2 send-proxy-v2
EOF

sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
systemctl is-active haproxy
sudo ss -lntp | grep -E ':(80|443|6443)[[:space:]]'
```

Check recent HAProxy messages:

```bash
sudo journalctl -u haproxy -n 50 --no-pager
```

The API, HTTP, and HTTPS frontends should listen on both proxy nodes. HAProxy
must report both Traefik backends as healthy.

## 9. Deploy a test application

Create a small application with two replicas, a ClusterIP Service, and a
standard Kubernetes Ingress:

```bash
kubectl apply --filename=- <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: ingress-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami:v1.11.0
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: ingress-demo
spec:
  selector:
    app: whoami
  ports:
    - name: http
      port: 80
      targetPort: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  namespace: ingress-demo
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - whoami-k8s-lab.argostranslations.com
  rules:
    - host: whoami-k8s-lab.argostranslations.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
EOF
```

Wait for the application and inspect the route:

```bash
kubectl rollout status deployment/whoami \
  --namespace ingress-demo \
  --timeout=120s

kubectl get pods,service,ingress \
  --namespace ingress-demo \
  --output=wide

kubectl describe ingress whoami --namespace ingress-demo
```

The Ingress address should be `10.11.1.56`.

## 10. Test through the edge VIP

If the application DNS record is not ready yet, `--resolve` supplies a
temporary client-side mapping without changing `/etc/hosts`.

> At this stage Traefik serves its generated `TRAEFIK DEFAULT CERT`, so a
> browser reports the site as untrusted. This is expected. Complete section
> 11 to install the existing `*.argostranslations.com` certificate as the
> shared default certificate.

Confirm that HTTP redirects to HTTPS:

```bash
curl --head \
  --resolve whoami-k8s-lab.argostranslations.com:80:10.11.1.56 \
  http://whoami-k8s-lab.argostranslations.com/
```

Expected status:

```text
HTTP/1.1 308 Permanent Redirect
```

Test HTTPS through HAProxy and Traefik:

```bash
curl --insecure \
  --resolve whoami-k8s-lab.argostranslations.com:443:10.11.1.56 \
  https://whoami-k8s-lab.argostranslations.com/
```

`--insecure` is required only because this initial test uses Traefik's default
certificate. The response should identify a `whoami` Pod and show the incoming
request information.

Watch Traefik access logs while sending requests:

```bash
kubectl logs --namespace traefik \
  --selector=app.kubernetes.io/name=traefik \
  --follow \
  --tail=20
```

## 11. Configure the shared wildcard TLS certificate

Kubernetes distributes certificates to every Traefik replica through Secrets.
Do not place TLS private keys on shared NFS, mount a certificate directory into
Traefik, or copy certificate files onto worker nodes.

Kubernetes Secrets require strict RBAC and should be encrypted at rest in etcd
for a production cluster. Base64 encoding alone is not encryption.

### Select a compatible hostname pattern

TLS wildcards match exactly one DNS label:

| Certificate | Covered | Not covered |
| --- | --- | --- |
| `*.argostranslations.com` | `whoami-k8s-lab.argostranslations.com` | `whoami.k8s-lab.argostranslations.com` |

The LAB design uses the existing corporate wildcard certificate with explicit
application DNS records:

```text
DNS:         <application>-k8s-lab.argostranslations.com -> 10.11.1.56
Certificate: *.argostranslations.com
```

Examples include `whoami-k8s-lab.argostranslations.com` and
`grafana-k8s-lab.argostranslations.com`. Hyphens remain within one DNS label,
so the wildcard certificate matches. A dotted prefix such as
`whoami.k8s-lab.argostranslations.com` has two labels and does not match.

### Inspect the certificate before importing it

The certificate file should contain the leaf certificate followed by any
intermediate CA certificates. The private key must match the leaf certificate.

```bash
openssl x509 \
  --in /secure/path/wildcard.crt \
  --noout \
  --subject \
  --issuer \
  --dates \
  --ext subjectAltName
```

Verify that the certificate and key have the same public key without printing
the private key:

```bash
openssl x509 \
  --in /secure/path/wildcard.crt \
  --pubkey \
  --noout | sha256sum

openssl pkey \
  --in /secure/path/wildcard.key \
  --pubout | sha256sum
```

Both hashes must match.

### Create the Kubernetes TLS Secret

Run from a secure administrative system that has the certificate files and an
authorized kubeconfig. The command sends the certificate and key to the API;
it does not store their literal contents in shell history or repository files:

```bash
kubectl create secret tls wildcard-default-tls \
  --namespace traefik \
  --cert=/secure/path/wildcard.crt \
  --key=/secure/path/wildcard.key
```

Verify only the Secret metadata and type. Do not print its data:

```bash
kubectl get secret wildcard-default-tls \
  --namespace traefik \
  --output=custom-columns='NAME:.metadata.name,TYPE:.type'
```

Expected type:

```text
kubernetes.io/tls
```

### Configure Traefik's default TLSStore

The Helm values restrict the cluster-wide default TLS resources to the
`traefik` namespace. Create exactly one TLSStore named `default`:

```bash
kubectl apply --filename=- <<'EOF'
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
  name: default
  namespace: traefik
spec:
  defaultCertificate:
    secretName: wildcard-default-tls
EOF
```

Verify it and check Traefik logs for certificate errors:

```bash
kubectl get tlsstore default --namespace traefik --output=yaml
kubectl logs --namespace traefik \
  --selector=app.kubernetes.io/name=traefik \
  --tail=100
```

Ingress resources with TLS enabled and no route-specific `secretName` will use
this default certificate. A particular application can still override it with
a TLS Secret in the application's namespace.

After installing the `*.argostranslations.com` certificate, test without
`--insecure`:

```bash
curl \
  --resolve whoami-k8s-lab.argostranslations.com:443:10.11.1.56 \
  https://whoami-k8s-lab.argostranslations.com/
```

For automatic issuance and renewal, deploy cert-manager with an approved
internal issuer. Prefer cert-manager over Traefik's local ACME storage when
running multiple Traefik replicas.

## 12. Access the dashboard locally

Do not expose the dashboard publicly without authentication. Use a local
port-forward for administrative inspection:

```bash
TRAEFIK_POD="$(kubectl get pods \
  --namespace traefik \
  --selector=app.kubernetes.io/name=traefik \
  --output=jsonpath='{.items[0].metadata.name}')"

kubectl port-forward \
  --namespace traefik \
  "pod/${TRAEFIK_POD}" \
  9000:8080
```

Open:

```text
http://127.0.0.1:9000/dashboard/
```

The port-forward remains attached to the terminal until it is stopped.

## Operational checks

```bash
helm status traefik --namespace traefik
kubectl get daemonset,pods,service --namespace traefik --output=wide
kubectl get ingress --all-namespaces
kubectl logs --namespace traefik \
  --selector=app.kubernetes.io/name=traefik \
  --tail=100
```

On both edge proxies:

```bash
systemctl is-active haproxy keepalived
sudo ss -lntp | grep -E ':(80|443|6443)[[:space:]]'
sudo journalctl -u haproxy -u keepalived -n 50 --no-pager
```

## Troubleshooting

### Ingress returns 404

Check the Ingress class, hostname, and Traefik provider:

```bash
kubectl get ingressclass
kubectl describe ingress whoami --namespace ingress-demo
kubectl logs --namespace traefik \
  --selector=app.kubernetes.io/name=traefik \
  --tail=100
```

The request `Host` header must exactly match the Ingress rule.

### HAProxy reports a Traefik backend down

```bash
kubectl get pods --namespace traefik --output=wide
kubectl get endpointslices --namespace traefik \
  --selector=kubernetes.io/service-name=traefik
nc -zv -w 2 10.11.1.49 30080
nc -zv -w 2 10.11.1.50 30080
```

With `externalTrafficPolicy: Local`, each worker must have a ready local
Traefik Pod.

### Client address is incorrect

Confirm that:

- HAProxy backends use `send-proxy-v2`;
- Traefik trusts only `10.11.1.54/32` and `10.11.1.55/32` for PROXY protocol;
- the Traefik Service uses `externalTrafficPolicy: Local`;
- HAProxy connects directly to the worker NodePorts.

## Ingress versus Gateway API

This guide starts with the stable `networking.k8s.io/v1` Ingress API because it
is directly relevant to CKA practice and remains widely deployed. Kubernetes
has frozen the Ingress API and recommends Gateway API for new capabilities.
Traefik supports both, but enable and learn Gateway API separately after this
Ingress path is working.

## References

- [Traefik Kubernetes installation](https://doc.traefik.io/traefik/setup/kubernetes/)
- [Official Traefik Helm chart](https://github.com/traefik/traefik-helm-chart)
- [Traefik Kubernetes Ingress provider](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes Services and networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Helm installation](https://helm.sh/docs/intro/install/)
