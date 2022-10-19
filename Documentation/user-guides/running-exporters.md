<br>
<div class="alert alert-info" role="alert">
    <i class="fa fa-exclamation-triangle"></i><b> Note:</b> Starting with v0.39.0, Prometheus Operator requires use of Kubernetes v1.16.x and up.
</div>

# Running Exporters

Running exporters and scraping them with Prometheus configured by the prometheus-operator.

## The goal of ServiceMonitor(s)

The goal for one ServiceMonitor should be to cover a large number of Services. This can be achieved by creating a generic `ServiceMonitor`.

## Create a service that exposes Pod(s)

Scraping an exporter or separate metrics port requires a service that targets the Pod(s) of the exporter or application.

The following examples create a Service for the `kube-state-metrics`, and a generic ServiceMonitor which may be used for more than simply the `kube-state-metrics` Service.

The order in which the `Service` and `ServiceMonitor` is created is not important.

### kube-state-metrics Service example

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kube-state-metrics
    k8s-app: kube-state-metrics
  annotations:
    alpha.monitoring.coreos.com/non-namespaced: "true"
  name: kube-state-metrics
spec:
  ports:
  - name: http-metrics
    port: 8080
    targetPort: metrics
    protocol: TCP
  selector:
    k8s-app: kube-state-metrics
```

This Service targets all Pods with the label `k8s-app: kube-state-metrics`.

### Generic ServiceMonitor example

This ServiceMonitor targets **all** Services with the label `k8s-app` (`spec.selector`) any value, in the namespaces `kube-system` and `monitoring` (`spec.namespaceSelector`).

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: k8s-apps-http
  labels:
    k8s-apps: http
spec:
  jobLabel: k8s-app
  selector:
    matchExpressions:
    - {key: k8s-app, operator: Exists}
  namespaceSelector:
    matchNames:
    - kube-system
    - monitoring
  endpoints:
  - port: http-metrics
    interval: 15s
```

## Default Labels

By default, `PodMonitors` and `ServiceMonitors` include runtime metadata in the scraped results.

### PodMonitors

| Label | Description |
| ----- | ----------- |
| job | `{metadata.namespace}/{metadata.name}`  of the `PodMonitor` or read from `jobLabel` if specified |
| namespace | `{metadata.namespace}` of the scraped pod |
| container | `{metadata.namespace}` of the container in the scraped pod |
| pod | `{metadata.name}` of the scraped pod |
| endpoint | `{spec.Port}` or `{spec.TargetPort}` if specified |

### ServiceMonitors

| Label | Description |
| ----- | ----------- |
| job | `{metadata.name}` of the scraped service or read from `jobLabel` if specified |
| node/pod | Set depending on the endpoint responding to service request |
| namespace | `{metadata.namespace}` of the scraped pod |
| service | `{metadata.name}` of the scraped service |
| pod | `{metadata.name}` of the scraped pod |
| container | `{metadata.namespace}` of the container in the scraped pod |
| endpoint | `{spec.Port}` or `{spec.TargetPort}` if specified |

### Configuration

The default labels may be configured by adding annotations to the `PodMonitor` or `ServiceMonitor` object. Each annotation will rename or disable the related labels. Setting the annotation to blank will disable the label.

| Annotation | |
| ---------- | ----------- |
| `operator.prometheus.io/enableServiceEndpointLabel` | Set to `false` to disable the `node`, `pod` and `endpoint` labels. |
| `operator.prometheus.io/scrapedNamespaceLabel` | Configures the `namespace` label |
| `operator.prometheus.io/scrapedServiceNameLabel` | Configures the `service` label |
| `operator.prometheus.io/scrapedPodNameLabel` | Configures the `pod` label |
| `operator.prometheus.io/scrapedContainerNameLabel` | Configures the `container` label |

## Troubleshooting

### Namespace "limits"/things to keep in mind

See the ServiceMonitor Documentation:

> By default and **before the version v0.19.0**, `ServiceMonitors` must be installed in the same namespace as the Prometheus resource. With the Prometheus Operator **v0.19.0 and above**, `ServiceMonitors` can be selected outside the Prometheus namespace via the `serviceMonitorNamespaceSelector` field of the Prometheus resource. The discovered targets may come from any namespace. This allows cross-namespace monitoring use cases, for example, for meta-monitoring.
