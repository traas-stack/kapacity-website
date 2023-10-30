---
title: "Use Custom Metrics"
weight: 3
---

{{% pageinfo color="primary" %}}
Note: Most contents of this page are translated by AI from [Chinese](/zh-cn/docs/user-guide/custom-metrics/). It would be very appreciated if you could help us improve by editing this page (click the corresponding link on the right side).
{{% /pageinfo %}}

## Background

The [Kubernetes Metrics API](https://github.com/kubernetes/metrics) provides a set of common metric query interfaces within the K8s system, but it only supports querying the current real-time value of metrics, doesn't support querying historical values, and also doesn't support aggregated querying of Pod metrics based on the workload dimension, which cannot meet the data needs of various intelligent algorithms. Therefore, Kapacity has further abstracted and extended the Metrics API, supporting general metric history query and workload dimension aggregation query and other advanced query capabilities while being maximally compatible with user usage habits.

Currently, the Metrics API provided by Kapacity supports the following two metric provider backends:

## Use Prometheus as metric provider (default)

Set the startup parameter `--metric-provider` of Kapacity Manager to `prometheus` to use Prometheus as metric provider.

If you use this provider, you only need a Prometheus, **no need to install** Kubernetes Metrics Server or other Metrics Adapters (including Prometheus Adapter).

You can find the `prometheus-metrics-config.yaml` configuration in the ConfigMap `kapacity-config` in the namespace where Kapacity Manager is located. By modifying this configuration, you can fully customize Prometheus query statements for different metric types. The format of this configuration is fully compatible with the [configuration of Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md), so if you have previously configured custom HPA metrics with Prometheus Adapter, you can directly reuse the previous configuration.

The default configuration provided by Kapacity is as follows:

```yaml
resourceRules:
  cpu:
    containerQuery: |-
      sum by (<<.GroupBy>>) (
        irate(container_cpu_usage_seconds_total{container!="",container!="POD",<<.LabelMatchers>>}[3m])
      )
    readyPodsOnlyContainerQuery: |-
      sum by (<<.GroupBy>>) (
          (kube_pod_status_ready{condition="true"} == 1)
        * on (namespace, pod) group_left ()
          sum by (namespace, pod) (
            irate(container_cpu_usage_seconds_total{container!="",container!="POD",<<.LabelMatchers>>}[3m])
          )
      )
    resources:
      overrides:
        namespace:
          resource: namespace
        pod:
          resource: pod
    containerLabel: container
  memory:
    containerQuery: |-
      sum by (<<.GroupBy>>) (
        container_memory_working_set_bytes{container!="",container!="POD",<<.LabelMatchers>>}
      )
    readyPodsOnlyContainerQuery: |-
      sum by (<<.GroupBy>>) (
          (kube_pod_status_ready{condition="true"} == 1)
        * on (namespace, pod) group_left ()
          sum by (namespace, pod) (
            container_memory_working_set_bytes{container!="",container!="POD",<<.LabelMatchers>>}
          )
      )
    resources:
      overrides:
        namespace:
          resource: namespace
        pod:
          resource: pod
    containerLabel: container
  window: 3m
rules: []
externalRules:
- seriesQuery: '{__name__="kube_pod_status_ready"}'
  metricsQuery: sum(<<.Series>>{condition="true",<<.LabelMatchers>>})
  name:
    as: ready_pods_count
  resources:
    overrides:
      namespace:
        resource: namespace
workloadPodNamePatterns:
- group: apps
  kind: ReplicaSet
  pattern: ^%s-[a-z0-9]+$
- group: apps
  kind: Deployment
  pattern: ^%s-[a-z0-9]+-[a-z0-9]+$
- group: apps
  kind: StatefulSet
  pattern: ^%s-[0-9]+$
```

To support advanced query capabilities such as workload dimension aggregation query, we have extended some fields on top of the Prometheus Adapter configuration, and below is a brief explanation of these extended fields:

- `workloadPodNamePatterns`: Some algorithms of Kapacity will need to query workload dimension metric information, such as the total CPU usage of a workload's Pods, the number of Ready Pods of a workload, etc. At this time, Kapacity will aggregate queries on Pod dimension metrics by matching the workload Pod name with regular expressions, so it is necessary to configure the regular matching rules of Pod names for different types of workloads through this field. If you use workloads other than the default configuration, you need to add the corresponding configuration in this field.
- `readyPodsOnlyContainerQuery`: Some algorithms of Kapacity have extra conditions when querying the total resource usage of workload Pods, such as only querying the total CPU usage of some workload's Ready Pods. In this case, we need to provide a separate PQL statement through this field for this special condition query. Kapacity default provides a query statement based on the metrics provided by [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics), you can also modify it to other implementations as needed.

## Use Kubernetes Metrics API as metric provider (not recommended)

Set the startup parameter `--metric-provider` of Kapacity Manager to `metrics-api` to use Kubernetes Metrics API as metric provider.

If you use this provider, you need to install [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server) or other Metrics Adapters (such as [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter)) to shield the differences of the underlying monitoring system.

However, it should be noted that **this backend does not support the query of historical values of metrics, nor does it support the aggregation query of Pod metrics based on the workload dimension**, so its usable range is very limited, only suitable for some scenarios that only use simple algorithms, such as [Reactive Scaling](/docs/getting-started/quick-start/ihpa/reactive-scaling/).
