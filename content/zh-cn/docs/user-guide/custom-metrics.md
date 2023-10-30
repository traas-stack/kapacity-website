---
title: "使用自定义指标"
weight: 3
---

## 背景

[Kubernetes Metrics API](https://github.com/kubernetes/metrics) 提供了一套 K8s 体系内的通用指标查询接口，但它只支持查询指标的当前实时值，而不支持查询历史值，此外，它也不支持基于工作负载维度的 Pod 指标聚合查询，无法满足各类智能算法的数据需求。因此，Kapacity 对 Metrics API 进行了进一步的抽象和扩展，在最大程度兼容用户使用习惯的同时支持了通用的指标历史查询与工作负载维度聚合查询等高阶查询能力。

目前，Kapacity 提供的 Metrics API 支持下面两种指标查询后端：

## 使用 Prometheus 作为指标查询后端（默认）

将 Kapacity Manager 的启动参数 `--metric-provider` 设置为 `prometheus` 以使用 Prometheus 作为指标查询后端。

如果使用此后端，你只需要有一个 Prometheus，**无需安装** Kubernetes Metrics Server 或其他 Metrics Adapter（包括 Prometheus Adapter）。

你可以在 Kapacity Manager 所在命名空间的 ConfigMap `kapacity-config` 中找到 `prometheus-metrics-config.yaml` 这份配置，通过修改这份配置，你可以完全自定义不同指标类型的 Prometheus 查询语句。这份配置的格式完全兼容 [Prometheus Adapter 的配置](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md)，因此，如果你此前使用 Prometheus Adapter 配置了自定义 HPA 指标，可以直接复用以前的配置。

当前 Kapacity 提供的默认配置如下所示：

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

为了支持工作负载维度聚合查询等高阶查询能力，我们在 Prometheus Adapter 配置之上扩展了部分字段，下面对这些扩展字段作简要说明：

- `workloadPodNamePatterns`：Kapacity 的部分算法会需要查询工作负载维度的指标信息，如某工作负载 Pods 的 CPU 总用量、某工作负载的 Ready Pods 数量等，此时 Kapacity 会以工作负载 Pod 名称正则匹配的方式对 Pod 维度的指标做聚合查询，因此需要通过该字段配置不同类型工作负载的 Pod 名称正则匹配规则。如果你使用了默认配置以外的其他工作负载，需要在该字段中添加相应配置。
- `readyPodsOnlyContainerQuery`：Kapacity 的部分算法在查询工作负载 Pods 的资源总用量时会有额外的条件，如仅查询某工作负载 Ready Pods 的 CPU 总用量，此时我们需要使用该字段提供一条单独的 PQL 语句来做此特殊条件下的查询。Kapacity 默认提供了基于 [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) 所提供指标的查询语句，你也可以按需修改为其他实现。

## 使用 Kubernetes Metrics API 作为指标查询后端（不推荐）

将 Kapacity Manager 的启动参数 `--metric-provider` 设置为 `metrics-api` 以使用 Kubernetes Metrics API 作为指标查询后端。

如果使用此后端，你需要安装 [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server) 或其他 Metrics Adapter（如 [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter)），由它们来屏蔽下层监控系统差异。

但需要注意的是，**此后端不支持指标历史值查询，也不支持基于工作负载维度的 Pod 指标聚合查询**，因此其可用范围非常有限，只适用于部分仅使用简单算法的场景，如[响应式扩缩容](/zh-cn/docs/getting-started/quick-start/ihpa/reactive-scaling/)。
