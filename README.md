# kube-prometheus

This repository collects Kubernetes manifests, [Grafana](http://grafana.com/) dashboards, and
[Prometheus rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
combined with documentation and scripts to provide single-command deployments of end-to-end
Kubernetes cluster monitoring with [Prometheus](https://prometheus.io/) (Operator).

This repository is fine tuned for running [Rancher](https://github.com/rancher/rancher).

## Prerequisites

First, you need a running Kubernetes cluster, provisioned by rancher. 

## Monitoring Kubernetes

The manifests here use the [Prometheus Operator](https://github.com/coreos/prometheus-operator),
which manages Prometheus servers and their configuration in a cluster. With a single command we can
install

* The Operator itself
* The Prometheus [node_exporter](https://github.com/prometheus/node_exporter)
* [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
* The [Prometheus specification](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#prometheus) based on which the Operator deploys a Prometheus setup
* A Prometheus configuration covering monitoring of all Kubernetes core components and exporters
* A default set of alerting rules on the cluster components' health
* A Grafana instance serving dashboards on cluster metrics
* A three node highly available Alertmanager cluster

Simply run:

```bash
export KUBECONFIG=<path> # defaults to "~/.kube/config"
hack/cluster-monitoring/deploy
```

After all pods are ready, you can reach:

* Prometheus UI on node port `30900`
* Alertmanager UI on node port `30903`
* Grafana on node port `30902`

To tear it all down again, run:

```bash
hack/cluster-monitoring/teardown
```

## Monitoring custom services

The example manifests in [manifests/examples/example-app](/contrib/kube-prometheus/manifests/examples/example-app)
deploy a fake service exposing Prometheus metrics. They additionally define a new Prometheus
server and a [`ServiceMonitor`](https://github.com/coreos/prometheus-operator/blob/master/Documentation/design.md#servicemonitor),
which specifies how the example service should be monitored.
The Prometheus Operator will deploy and configure the desired Prometheus instance and continuously
manage its life cycle.

```bash
hack/example-service-monitoring/deploy
```

After all pods are ready you can reach the Prometheus server on node port `30100` and observe
how it monitors the service as specified. Same as before, this Prometheus server automatically
discovers the Alertmanager cluster deployed in the [Monitoring Kubernetes](#Monitoring-Kubernetes)
section.

Teardown:

```bash
hack/example-service-monitoring/teardown
```

## Dashboarding

The provided manifests deploy a Grafana instance serving dashboards provided via ConfigMaps.
Said ConfigMaps are generated from Python scripts in assets/grafana, that all have the extension
.dashboard.py as they are loaded by the [grafanalib](https://github.com/aknuds1/grafanalib)
Grafana dashboard generator. Bear in mind that we are for now using a fork of grafanalib as
we needed to make extensive changes to it, in order to be able to generate our dashboards. We are
hoping to be able to consolidate our version with the original.

As such, in order to make changes to the dashboard bundle, you need to change the \*.dashboard.py 
files in assets/grafana, eventually add your own, and then run `make generate` in the
kube-prometheus root directory.
 
To read more in depth about developing dashboards, read the
[Developing Prometheus Rules and Grafana Dashboards](docs/developing-alerts-and-dashboards.md)
documentation.

### Reloading of dashboards

Currently, Grafana does not support serving dashboards from static files. Instead, the `grafana-watcher`
sidecar container aims to emulate the behavior, by keeping the Grafana database always in sync
with the provided ConfigMap. Hence, the Grafana pod is effectively stateless.
This allows managing dashboards via `git` etc. and easily deploying them via CD pipelines.

In the future, a separate Grafana operator will support gathering dashboards from multiple
ConfigMaps based on label selection.

WARNING: If you deploy multiple Grafana instances for HA, you must use session affinity.
Otherwise if pods restart the prometheus datasource ID can get out of sync between the pods,
breaking the UI

## Roadmap

* Grafana Operator that dynamically discovers and deploys dashboards from ConfigMaps
* KPM/Helm packages to easily provide production-ready cluster-monitoring setup (essentially contents of `hack/cluster-monitoring`)
* Add meta-monitoring to default cluster monitoring setup
* Build out the provided dashboards and alerts for cluster monitoring to have full coverage of all system aspects

