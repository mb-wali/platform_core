# Grafana Alloy

## Links
* [Offical Docs](https://grafana.com/docs/alloy/latest/)
* [Tutorial](https://www.youtube.com/watch?v=E654LPrkCjo)
* [Docs on Tutorial](https://github.com/christianlempa/boilerplates)
---

## Installation
You can [install](https://grafana.com/docs/alloy/latest/set-up/install/) it via multiple ways.

* Helm Chart: https://github.com/grafana/alloy/tree/main/operations/helm/charts/alloy


## kubernetes basic rules for logs

```yaml
alloy:
  configMap:
    content: |
      logging {
          level  = "info"
          format = "logfmt"
      }

      discovery.kubernetes "pods" {
          role = "pod"
      }

      discovery.relabel "pods" {
        targets = discovery.kubernetes.pods.targets
        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }
        rule {
          source_labels = ["__meta_kubernetes_node_name"]
          target_label  = "node"
        }
      }

      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pods.output
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
          endpoint {
              url = "http://loki-gateway/loki/api/v1/push"
          }
      }
```

# What else can Alloy do?
* **Metrics:** Alloy can scrape Prometheus-style metrics from pods, nodes, or endpoints. Can forward metrics to: Prometheus remote write endpoints. e.g. `nginx_requests_total{method="POST",status="500"} 3`
* **Traces:** Alloy can collect OpenTelemetry traces from applications.
  ```bash
  #  Can enrich traces with metadata from Kubernetes pods (namespace, pod, node, labels).
  # e.g.
  TraceID: abc123
  Span 1: HTTP GET /login  (frontend pod)
  Span 2: DB query SELECT … (backend pod)
  Span 3: Token validation    (backend pod)
  ```
* **Custom Enrichment & Observability Pipelines:**

  Custom enrichment and observability pipelines let Alloy turn raw logs/metrics/traces into structured, labeled, queryable observability data while keeping everything in a single agent.

## What OpenTelemetry Traces Are
* Modern applications are often made of multiple services/microservices.
* A single user request may touch many services (e.g., API → backend → database → cache).
* Distributed tracing tracks the path of that request across all these services.

## Extra documentation
* Components: https://grafana.com/docs/alloy/latest/introduction/how-alloy-works/#component-based-architecture
