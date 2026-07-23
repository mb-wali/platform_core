# Contour

## References

- Helm Chart:
  - https://github.com/projectcontour/helm-charts/tree/main/charts/contour
- Source Code:
  - https://github.com/projectcontour/contour

## What is `contour-envoy`?

`contour-envoy` is the Envoy proxy deployment managed by Contour.

Example service:

```text
contour-envoy   LoadBalancer   10.43.50.248   188.166.116.113   80:30785/TCP,443:30207/TCP
```

### Components

- **Contour**: Kubernetes ingress controller and control plane.
- **Envoy**: Data plane proxy that handles incoming HTTP/HTTPS traffic.

Contour watches Kubernetes resources such as:

- HTTPProxy
- Ingress
- Gateway API resources

It then generates Envoy configuration dynamically.

### Why is it exposed as a LoadBalancer?

The `contour-envoy` service is typically exposed as a `LoadBalancer` because Envoy is the component that receives external traffic and routes it to services inside the cluster.

Traffic flow:

```text
Internet
    │
    ▼
LoadBalancer (contour-envoy)
    │
    ▼
Envoy Proxy
    │
    ▼
Kubernetes Services
    │
    ▼
Pods
```

---

## TODO

### Prometheus Metrics

Reference:

https://github.com/projectcontour/helm-charts/tree/main/charts/contour#prometheus-metrics

Tasks:

- [ ] Enable Contour metrics
- [ ] Enable Envoy metrics
- [ ] Configure Prometheus scraping
- [ ] Create Grafana dashboards
- [ ] Add alerting rules

---

## Contour Configuration

Reference:

https://projectcontour.io/docs/1.33/configuration/

Example configuration:

```yaml
configInline:
  # Controls whether Contour ignores PermitInsecure field in HTTPProxy
  disablePermitInsecure: false

  # Envoy-specific configuration
  envoy:
    # Trust the first upstream proxy hop (e.g. HAProxy)
    # for X-Forwarded-* headers
    num-trusted-hops: 1

    # Strip trailing dot in Host header
    strip-trailing-host-dot: false

  # TLS defaults
  tls:
    # Configure a fallback certificate if desired
    fallback-certificate: {}

  # Access log format
  # Supported values: envoy, json
  accesslog-format: envoy
```

### Notes

#### `disablePermitInsecure`

When set to `false`, HTTPProxy resources may use the `permitInsecure` option to allow HTTP traffic.

#### `envoy.num-trusted-hops`

Used when traffic passes through one or more upstream proxies (for example HAProxy, Cloudflare, or a cloud load balancer).

Example:

```text
Client
  │
  ▼
HAProxy
  │
  ▼
Envoy
```

Set:

```yaml
num-trusted-hops: 1
```

so Envoy trusts the first proxy's `X-Forwarded-*` headers.

#### `accesslog-format`

Options:

```yaml
accesslog-format: envoy
```

Traditional Envoy log format.

```yaml
accesslog-format: json
```

Structured JSON logs for centralized logging systems such as:

- Loki
- Elasticsearch
- OpenSearch
- Datadog

---

# Gateway API

Reference:

https://projectcontour.io/docs/1.23/guides/gateway-api/

## Overview

Contour supports the Kubernetes Gateway API as an alternative to Ingress and HTTPProxy.

Key resources:

- `GatewayClass`
- `Gateway`
- `HTTPRoute`
- `TLSRoute`
- `ReferenceGrant`

Example architecture:

```text
GatewayClass
      │
      ▼
   Gateway
      │
      ▼
  HTTPRoute
      │
      ▼
   Services
      │
      ▼
     Pods
```

## Benefits

- Standard Kubernetes networking API
- Multi-team delegation support
- Better separation between infrastructure and application ownership
- Successor to the traditional Ingress API

## Useful Commands

```bash
kubectl get gatewayclass
kubectl get gateways -A
kubectl get httproutes -A
kubectl describe gateway <gateway-name>
```

---

## My Deployment Configuration

Reference:

https://github.com/projectcontour/helm-charts/blob/main/charts/contour/values.yaml

```yaml
envoy:
  service:
    type: NodePort
    nodePorts:
      http: {{ contour_node_port_http }}
      https: {{ contour_node_port_https }}

ingressClass:
  enabled: true
  name: {{ clusterIngressClassName }}
  isDefaultClass: true
```

### What does this configuration do?

#### Envoy Service

```yaml
envoy:
  service:
    type: NodePort
```

Instead of creating a cloud LoadBalancer, Kubernetes exposes Envoy on fixed ports on every node.

Example:

```text
Node IP: 10.0.0.10
HTTP: 30080
HTTPS: 30443
```

Traffic flow:

```text
Internet
    │
    ▼
HAProxy / External Load Balancer
    │
    ▼
NodeIP:NodePort
    │
    ▼
contour-envoy
    │
    ▼
Application Services
    │
    ▼
Pods
```

This model is commonly used when:

- Using HAProxy, NGINX, F5, or another external load balancer.
- Running on bare metal.
- Using Hetzner, OVH, Proxmox, VMware, or other environments without a cloud LoadBalancer integration.
- Managing ingress traffic outside Kubernetes.

### Ingress Class

```yaml
ingressClass:
  enabled: true
  name: {{ clusterIngressClassName }}
  isDefaultClass: true
```

This creates an IngressClass resource and configures Contour as the default ingress controller.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
spec:
  ingressClassName: my-ingress
```

If `isDefaultClass: true` is set, Ingress resources without an explicit `ingressClassName` will automatically be handled by Contour.

### Verify Installation

Check the Envoy service:

```bash
kubectl get svc -n projectcontour
```

Expected output:

```text
NAME            TYPE       CLUSTER-IP      PORT(S)
contour         ClusterIP  10.x.x.x        8001/TCP
contour-envoy   NodePort   10.x.x.x        80:<nodeport>/TCP,443:<nodeport>/TCP
```

Check the ingress class:

```bash
kubectl get ingressclass
```

Expected output:

```text
NAME          CONTROLLER
my-ingress    projectcontour.io/ingress-controller
```

### Notes

- `contour` is the control plane.
- `contour-envoy` is the data plane that receives traffic.
- Envoy listens on ports `80` and `443` inside the cluster.
- Kubernetes exposes those ports through the configured NodePorts.
- External traffic should be directed to the NodePorts, usually through HAProxy or another load balancer.