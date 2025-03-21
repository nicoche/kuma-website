---
title: Traffic Metrics
---

{{site.mesh_product_name}} facilitates consistent traffic metrics across all data plane proxies in your mesh.

You can add metrics to a mesh configuration, or to an individual data plane proxy configuration. For example, you might need metrics for individual data plane proxies to override the default metrics port if it's already in use on the specified machine.

{{site.mesh_product_name}} provides full integration with Prometheus:

* Each proxy can expose its metrics in [Prometheus format](https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format).
* Because metrics are part of the mesh configuration, {{site.mesh_product_name}} exposes an API called the monitoring assignment service (MADS) which exposes every proxy in the mesh.

To collect metrics from {{site.mesh_product_name}}, you need to expose metrics from proxies and applications. 

{% tip %}
In the rest of this page we assume you have already configured your observability tools to work with {{site.mesh_product_name}}.
If you haven't already read the [observability docs](/docs/{{ page.version }}/explore/observability).
{% endtip %}

## Expose metrics from data plane proxies

To expose metrics from every proxy in the mesh, configure the `Mesh` resource:


{% tabs expose-metrics-data-plane-proxies useUrlFragment=false %}
{% tab expose-metrics-data-plane-proxies Kubernetes %}

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus
```

which is a shortcut for:

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus
      conf:
        skipMTLS: false
        port: 5670
        path: /metrics
        tags: # tags that can be referred in Traffic Permission when metrics are secured by mTLS  
          kuma.io/service: dataplane-metrics
```
{% endtab %}
{% tab expose-metrics-data-plane-proxies Universal %}

```yaml
type: Mesh
name: default
metrics:
  enabledBackend: prometheus-1
  backends:
  - name: prometheus-1
    type: prometheus
    conf:
      skipMTLS: true # by default mTLS metrics are also protected by mTLS. Scraping metrics with mTLS without transparent proxy is not supported at the moment.
```

which is a shortcut for:

```yaml
type: Mesh
name: default
metrics:
  enabledBackend: prometheus-1
  backends:
  - name: prometheus-1
    type: prometheus
    conf:
      skipMTLS: true
      port: 5670
      path: /metrics
      tags: # tags that can be referred in Traffic Permission when metrics are secured by mTLS  
        kuma.io/service: dataplane-metrics
```
{% endtab %}
{% endtabs %}

This tells {{site.mesh_product_name}} to configure every proxy in the `default` mesh to expose an HTTP endpoint with Prometheus metrics on port `5670` and URI path `/metrics`.

The metrics endpoint is forwarded to the standard Envoy [Prometheus metrics endpoint](https://www.envoyproxy.io/docs/envoy/latest/operations/admin#get--stats?format=prometheus) and supports the same query parameters.
You can pass the `filter` query parameter to limit the results to metrics whose names match a given regular expression.
By default all available metrics are returned.

## Expose metrics from applications
 
In addition to exposing metrics from the data plane proxies, you might want to expose metrics from applications running next to the proxies. {{site.mesh_product_name}} allows scraping Prometheus metrics from the applications endpoint running in the same `Pod` or `VM`.
Later those metrics are aggregated and exposed at the same `port/path` as data plane proxy metrics.
It is possible to configure it at the `Mesh` level, for all the applications in the `Mesh`, or just for specific applications.

Here are reasons where you'd want to use this feature:

- Application metrics are labelled with your mesh parameters (tags, mesh, data plane name...), this means that in mixed Universal and Kubernetes mode metrics are reported with the same types of labels.
- Both application and sidecar metrics are scraped at the same time. This makes sure they are coherent (with 2 different scrapers they can end up scraping at different intervals and make metrics harder to correlate).
- If you disable [passthrough](/docs/{{ page.version }}/networking/non-mesh-traffic#outgoing) and your mesh uses mTLS but Prometheus is outside the mesh (`skipMTLS: true`) this will be the only way to retrieve these metrics as the application is completely hidden behind the sidecar. 

{% warning %}
Any configuration change requires redeployment of the data plane.
{% endwarning %}

{% tabs expose-metrics-apps useUrlFragment=false %}
{% tab expose-metrics-apps Kubernetes %}
```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus
      conf:
        skipMTLS: false
        port: 5670
        path: /metrics
        tags: # tags that can be referred in Traffic Permission when metrics are secured by mTLS 
          kuma.io/service: dataplane-metrics
        aggregate:
          - name: my-service # name of the metric, required to later disable/override with pod annotations 
            path: "/metrics/prometheus"
            port: 8888
          - name: other-sidecar
            # default path is going to be used, default: /metrics
            port: 8000
```
{% endtab %}
{% tab expose-metrics-apps Universal %}
```yaml
type: Mesh
name: default
metrics:
  enabledBackend: prometheus-1
  backends:
  - name: prometheus-1
    type: prometheus
    conf:
      port: 5670
      path: /metrics
      skipMTLS: true # by default mTLS metrics are also protected by mTLS. Scraping metrics with mTLS without transparent proxy is not supported at the moment.
      aggregate:
      - name: my-service # name of the metric, required to later disable/override in the Dataplane resource
        path: "/metrics/prometheus"
        port: 8888
      - name: other-sidecar
        # default path is going to be used, default: /metrics
        port: 8000
```
{% endtab %}
{% endtabs %}

This configuration will cause every application in the mesh to be scrapped for metrics by the data plane proxy. If you need to expose metrics only for the specific application it is possible through `annotation` for Kubernetes or `Dataplane` resource for Universal deployment.

{% tabs annotations useUrlFragment=false %}
{% tab annotations Kubernetes %}
Kubernetes allows to configure it through annotations. In case to configure you can use `prometheus.metrics.kuma.io/aggregate-<name>-(path/port/enabled)`, where name is used to match the `Mesh` configuration and override or disable it.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 namespace: kuma-example
 name: kuma-tcp-echo
spec:
 ...
 template:
   metadata:
     ...
     annotations:
       prometheus.metrics.kuma.io/aggregate-my-service-enabled: "false"  # causes that configuration from Mesh to be disabled and result in this endpoint's metrics to not be exposed
       prometheus.metrics.kuma.io/aggregate-other-sidecar-port: "1234" # override port from Mesh
       prometheus.metrics.kuma.io/aggregate-application-port: "80"
       prometheus.metrics.kuma.io/aggregate-application-path: "/stats"
   spec:
     containers:
     ...
```
{% endtab %}
{% tab annotations Universal %}
```yaml
type: Dataplane
mesh: default
name: example
metrics:
  type: prometheus
  conf:
    path: /metrics/overridden
    aggregate:
    - name: my-service # causes that configuration from Mesh to be disabled and result in this endpoint's metrics to not be exposed
      enabled: false
    - name: other-sidecar
      port: 1234 # override port from Mesh
      address: "127.0.0.1"
    - name: application
      path: "/stats"
      port: 80`
```

{% warning %}
By default, sidecar scrapes metrics on `dataplane.networking.address`. If you set `dataplane.networking.inbound[].address.serviceAddress`, and you want to scrape metrics from the application, you also need to set `dataplane.metrics.conf.aggregate[].address`.
{% endwarning %}

{% endtab %}
{% endtabs %}

## Override Prometheus settings per data plane proxy

{% tabs override-prometheus useUrlFragment=false %}
{% tab override-prometheus Kubernetes %}
To override mesh-wide defaults for a particular `Pod`, use the following annotations:
* `prometheus.metrics.kuma.io/port` - to override mesh-wide default port
* `prometheus.metrics.kuma.io/path` - to override mesh-wide default path

For example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: kuma-example
  name: kuma-tcp-echo
spec:
  ...
  template:
    metadata:
      ...
      annotations:
        prometheus.metrics.kuma.io/port: "1234"               # override Mesh-wide default port
        prometheus.metrics.kuma.io/path: "/non-standard-path" # override Mesh-wide default path
    spec:
      containers:
      ...
```

Proxies for this Pod expose an HTTP endpoint with Prometheus metrics on port `1234` and URI path `/non-standard-path`.
{% endtab %}
{% tab override-prometheus Universal %}

To override mesh-wide defaults on a particular machine, configure the `Dataplane` resource:

```yaml
type: Dataplane
mesh: default
name: example
metrics:
  type: prometheus
  conf:
    skipMTLS: false
    port: 1234
    path: /non-standard-path
```

This proxy exposes an HTTP endpoint with Prometheus metrics on port `1234` and URI path `/non-standard-path`.
{% endtab %}
{% endtabs %}

## Filter Envoy metrics

In case you don't want to retrieve all Envoy's metrics, it's possible to filter them. Configuration is dynamic and doesn't require a restart of a sidecar. You are able to specify [`regex`](https://www.envoyproxy.io/docs/envoy/latest/operations/admin#get--stats?filter=regex) which causes that metric's endpoint returns only matching metrics. Also, you can set flag [`usedOnly`](https://www.envoyproxy.io/docs/envoy/latest/operations/admin#get--stats?usedonly) that returns only metrics updated by Envoy.

{% tabs envoy useUrlFragment=false %}
{% tab envoy Kubernetes %}
```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus
      conf:
        skipMTLS: false
        port: 5670
        path: /metrics
        envoy:
          filterRegex: http2_act.*
          usedOnly: true
```
{% endtab %}
{% tab envoy Universal %}
```yaml
type: Mesh
name: default
metrics:
  enabledBackend: prometheus-1
  backends:
  - name: prometheus-1
    type: prometheus
    conf:
      port: 5670
      path: /metrics
      envoy:
        filterRegex: http2_act.*
        usedOnly: true
```
{% endtab %}
{% endtabs %}

## Secure data plane proxy metrics

{{site.mesh_product_name}} lets you expose proxy metrics in a secure way by leveraging mTLS. Prometheus needs to be a part of the mesh for this feature to work, which is the default deployment mode on Kubernetes when using [`kumactl install observability`](/docs/{{ page.version }}/explore/observability#demo-setup).

{% tabs secure-data-plane-proxy-metrics useUrlFragment=false %}
{% tab secure-data-plane-proxy-metrics Kubernetes %}
Make sure that mTLS is enabled in the mesh.
```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus
      conf:
        port: 5670
        path: /metrics
        skipMTLS: false
        tags: # tags that can be referred in a TrafficPermission resource 
          kuma.io/service: dataplane-metrics
```

If you have strict [traffic permissions](/docs/{{ page.version }}/policies/traffic-permissions) you will want to allow the traffic from Grafana to Prometheus and from Prometheus to data plane proxy metrics:

```yaml
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  name: metrics-permissions
spec:
  sources:
    - match:
       kuma.io/service: prometheus-server_mesh-observability_svc_80
  destinations:
    - match:
       kuma.io/service: dataplane-metrics
---
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  name: grafana-to-prometheus
spec:
   sources:
   - match:
      kuma.io/service: "grafana_mesh-observability_svc_80"
   destinations:
   - match:
      kuma.io/service: "prometheus-server_mesh-observability_svc_80"
```
{% endtab %}
{% tab secure-data-plane-proxy-metrics Universal %}
Make sure that mTLS is enabled in the mesh.
```yaml
type: Mesh
name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus
      conf:
        port: 5670
        path: /metrics
        skipMTLS: false
        tags: # tags that can be referred in a TrafficPermission resource 
          kuma.io/service: dataplane-metrics
```

If you have strict [traffic permissions](/docs/{{ page.version }}/policies/traffic-permissions) you will want to allow the traffic from Grafana to Prometheus and from Prometheus to data plane proxy metrics:

```yaml
type: TrafficPermission
mesh: default
name: metrics-permissions
spec:
  sources:
    - match:
       kuma.io/service: prometheus-server
  destinations:
    - match:
       kuma.io/service: dataplane-metrics
---
type: TrafficPermission
mesh: default
name: grafana-to-prometheus
spec:
   sources:
   - match:
      kuma.io/service: "grafana"
   destinations:
   - match:
      kuma.io/service: "prometheus-server"
```
{% endtab %}
{% endtabs %}
