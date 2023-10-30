---
title: "使用预测式扩缩容"
weight: 3

---

## 准备开始

你需要拥有一个安装了 Kapacity 与 Prometheus 的 Kubernetes 集群。

请确保你的 Kubernetes 集群中有可用的 DNS（如 [CoreDNS](https://coredns.io/)）来解析 Service 域名。如果没有，则需要对 Kapacity 做如下配置调整：

使用如下命令查看 Kapacity gRPC Server 的 ClusterIP 和端口：

```shell
kubectl get svc -n kapacity-system kapacity-grpc-service
```

```
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kapacity-grpc-service   ClusterIP   192.168.38.172   <none>        9090/TCP   5m
```

使用如下命令更新 Kapacity 的配置，其中的 Kapacity gRPC Server 地址相关参数即为上一步查看到的值：

```shell
helm upgrade \
  kapacity-manager kapacity/kapacity-manager \
  --reuse-values \
  --set algorithmJob.defaultMetricsServerAddr=<kapacity-grpc-server-clusterip>:<kapacity-grpc-server-port> 
```

## 安装并配置 Ingress NGINX Controller

Kapacity IHPA 的预测式扩缩容使用「[基于流量驱动的副本数预测](/zh-cn/docs/user-guide/ihpa/concepts/predictive-scaling-principles/#基于流量驱动的副本数预测)」算法，因此我们需要至少一条流量指标来使用预测式扩缩容。这里我们使用 Ingress NGINX 作为工作负载入口流量的例子。

如果你的 Kubernetes 集群中还没有 Ingress NGINX Controller，请参考[官方文档](https://kubernetes.github.io/ingress-nginx/deploy/)进行安装。

安装完成后，请按照[此文档](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/)进行配置以确保 Prometheus 能够采集 Ingress NGINX 的指标。

## 配置 Kapacity 以识别 Ingress NGINX 指标

使用如下命令在 Kapacity 的自定义 Prometheus 指标配置中加入 Ingress NGINX 的指标：

```shell
kubectl edit cm -n kapacity-system kapacity-config
```

```yaml
apiVersion: v1
data:
  prometheus-metrics-config.yaml: |
    resourceRules:
      ...
    # 在 rules 中加入 Ingress NGINX 的指标
    rules:
    - seriesQuery: '{__name__="nginx_ingress_controller_requests"}'
      metricsQuery: round(sum(irate(<<.Series>>{<<.LabelMatchers>>}[3m])) by (<<.GroupBy>>), 0.001)
      name:
        as: nginx_ingress_controller_requests_rate
      resources:
        template: <<.Resource>>
        # 注意：当且仅当你的 Prometheus 是用 Prometheus Operator 安装的，才需要写下面的 overrides 字段
        overrides:
          exported_namespace:
            resource: namespace
    externalRules:
      ...
kind: ConfigMap
...
```

可以看到，该配置与 [Prometheus Adapter 的配置](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md)完全兼容，更多背景信息可参考[此用户指南](/zh-cn/docs/user-guide/custom-metrics)。

随后，使用如下命令重启 Kapacity Manager 以加载最新配置：

```shell
kubectl rollout restart -n kapacity-system deploy/kapacity-manager
```

## 运行示例工作负载

1. 下载 [nginx-statefulset.yaml](/examples/workload/nginx-statefulset.yaml) 文件，并执行以下命令以运行一个 NGINX 服务：

```shell
kubectl apply -f nginx-statefulset.yaml
```

验证服务部署完成：

```shell
kubectl get po
```

```
NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          5s
```

2. 下载 [nginx-ingress.yaml](/examples/workload/nginx-ingress.yaml) 文件，并执行以下命令以为 NGINX 服务创建一个 Ingress：

```shell
kubectl apply -f nginx-ingress.yaml
```

验证 Ingress 创建成功，并记录下 Ingress 的 ADDRESS：

```shell
kubectl get ing
```

```
NAME           CLASS   HOSTS               ADDRESS           PORTS   AGE
nginx-server   nginx   nginx.example.com   139.224.120.211   80      2d
```

3. 下载 [periodic-client.yaml](/examples/workload/periodic-client.yaml) 文件，将其中的 `<nginx-ingress-address>` 替换为上一步记录的 Ingress 的 ADDRESS，随后执行以下命令创建一个按周期性（以 1 小时为 1 个周期）规律向 NGINX 服务发送请求的客户端 Pod：

```shell
kubectl apply -f periodic-client.yaml
```

它会产生如下图所示的周期性流量：

<img src="/images/en/periodic-traffic-metric.png" width="1000"/>

由于算法学习需要一定数据量，建议至少运行 24 小时后再进行后续步骤。

## 训练时序预测模型

请参考[用户指南](/zh-cn/docs/user-guide/algorithm/train-tsf-model/)使用[该配置](/examples/algorithm/tsf-model-train-config.yaml)完成时序预测模型的训练，随后执行以下命令将模型及其附属文件保存为一个 ConfigMap 供后续算法任务使用，其中的 `<model-save-path>` 请替换为实际的模型保存目录路径：

```shell
kubectl create cm -n kapacity-system example-model --from-file=<model-save-path>
```

{{% alert title="说明" %}}
在实际使用中我们可能会得到更大的模型，此时建议将模型文件存储到[持久卷](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/)而非 ConfigMap。
{{% /alert %}}

## 创建配置了动态预测式画像源的 IHPA

下载 [dynamic-predictive-portrait-sample.yaml](/examples/ihpa/dynamic-predictive-portrait-sample.yaml) 文件，其内容如下所示：

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: dynamic-predictive-portrait-sample
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  portraitProviders:
  - type: Dynamic
    priority: 1
    dynamic:
      portraitType: Predictive
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: AverageValue
            averageValue: 1m
      - type: External
        external:
          metric:
            name: ready_pods_count
          target:
            type: NA
      - name: qps
        type: Object
        object:
          describedObject:
            apiVersion: networking.k8s.io/v1
            kind: Ingress
            name: nginx-server
          metric:
            name: nginx_ingress_controller_requests_rate
          target:
            type: NA
      algorithm:
        type: ExternalJob
        externalJob:
          job:
            type: CronJob
            cronJob:
              template:
                spec:
                  schedule: "0/30 * * * *"
                  jobTemplate:
                    spec:
                      template:
                        spec:
                          containers:
                          - name: algorithm
                            args:
                            - --tsf-model-path=/opt/kapacity/timeseries/forecasting/model
                            - --tsf-freq=10min
                            - --re-history-len=24H
                            - --re-time-delta-hours=8
                            - --re-test-dataset-size-in-seconds=3600
                            - --scaling-freq=10min
                            volumeMounts:
                            - name: model
                              mountPath: /opt/kapacity/timeseries/forecasting/model
                              readOnly: true
                          volumes:
                          - name: model
                            configMap:
                              name: example-model
                          restartPolicy: OnFailure
          resultSource:
            type: ConfigMap
```

请将算法参数 `--re-time-delta-hours` 的值替换成你所在时区的 UTC 偏移值，如 UTC+8 时区则填写 `8`，UTC-7 时区则填写 `-7`。

下面简单解释该 IHPA 的一些配置：

先来看指标，在「基于流量驱动的副本数预测」算法中，我们需要多类指标来共同驱动该算法，因此我们约定了下面的指标配置规范：

- 第一个指标应当配置为该工作负载的目标资源指标，因此类型只能为 `Resource` 或者 `ContainerResource`。它指定了我们期望 IHPA 帮我们维持的目标资源水位。
- 第二个指标应当配置为该工作负载的在线副本数指标，算法会使用该指标查询该工作负载的历史 Ready Pod（即承载流量的 Pod）的数量。该指标的类型只能为 `External`，它会在工作负载维度按照 Pod 名称正则匹配做聚合查询，Kapacity 默认配置了基于 kube-state-metrics 的 `ready_pods_count` 指标可供直接使用。需要注意的是，由于该指标仅用于历史查询，我们不需要为它指定目标值，因此这里我们将其 `target` 的 `type` 写为一个占位符 `NA`。
- 第三个及以后的指标应当配置为和该工作负载的目标资源指标存在正相关的流量指标（如 QPS、RPC 等），算法会对这些指标进行时序预测，随后基于历史资源水位和副本数，给出未来能够满足目标资源水位的预测副本数。这些指标的类型可以是除了 `Resource` 和 `ContainerResource` 的任意类型，**但注意必须为这些指标设置与训练时设置的相同的 `name`**。同样地，这些指标也仅用于历史查询，因此不需要设定目标值。

再来看算法参数，这里简单说明其中几个关键参数的作用，更多信息可参考算法脚本自身的 flags 说明：

- `--tsf-freq`：该参数指定了流量时序预测的精度，需与训练模型时的 `freq` 参数保持一致。
- `--re-history-len`：该参数指定了副本数推荐算法学习的历史长度，一般建议至少覆盖应用的两个行为周期。
- `--re-time-delta-hours`：该参数指定了应用所在时区的 UTC 偏移值，副本数推荐算法需要感知时区信息以学习时间特征。
- `--re-test-dataset-size-in-seconds`：该参数指定了副本数推荐算法学习的测试集大小，默认为一天（86400），只有历史长度不足一天时才需要将其改短，如本示例中设置为一小时（3600）。
- `--scaling-freq`：该参数指定了算法最终输出的副本数预测结果的精度，即最终实际扩缩容的最高频率，因此其不能短于算法的原始预测精度 `--tsf-freq`。算法会按照给定的精度对原始预测结果按最大值做重采样后输出，比如如果该参数设置为 1 小时，则算法最终会给出每小时该工作负载所需的最大副本数，最终该工作负载最多每小时进行一次扩缩容。

执行以下命令创建该 IHPA：

```shell
kubectl apply -f dynamic-predictive-portrait-sample.yaml
```

## 验证结果

1. 验证 IHPA 自动创建了运行算法任务的 CronJob，且上一次任务运行成功：

```shell
kubectl get cj -n kapacity-system
```

```
NAME                                                    SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
default-dynamic-predictive-portrait-sample-predictive   0/30 * * * *   False     1        26m             2d1h
```

```shell
kubectl get job -n kapacity-system
```

```
NAME                                                             COMPLETIONS   DURATION   AGE
default-dynamic-predictive-portrait-sample-predictive-28286564   1/1           16s        28m
```

2. 验证算法结果成功写入了 IHPA 的预测式水平画像：

```shell
kubectl get hp dynamic-predictive-portrait-sample-predictive -o yaml
```

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: HorizontalPortrait
metadata:
  name: dynamic-predictive-portrait-sample-predictive
  namespace: default
  ...
spec:
  ...
status:
  conditions:
  - lastTransitionTime: "2023-10-25T11:00:00Z"
    message: portrait has been successfully generated
    observedGeneration: 1
    reason: SucceededGeneratePortrait
    status: "True"
    type: PortraitGenerated
  portraitData:
    expireTime: "2023-10-25T11:30:00Z"
    timeSeries:
      timeSeries:
      - replicas: 4
        timestamp: 1698231600
      - replicas: 3
        timestamp: 1698232200
      - replicas: 2
        timestamp: 1698232800
    type: TimeSeries
```

3. 验证 IHPA 按算法的预测结果对工作负载的副本数进行调整：

```shell
kubectl describe ihpa dynamic-predictive-portrait-sample
```

```
...
Events:
  Type     Reason                Age                 From             Message
  ----     ------                ----                ----             -------
  Warning  NoValidPortraitValue  29m (x10 over 85m)  ihpa_controller  no valid portrait value for now
  Normal   UpdateReplicaProfile  25m                 ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 4, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  15m                 ihpa_controller  update ReplicaProfile with onlineReplcas: 4 -> 3, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  5m9s                ihpa_controller  update ReplicaProfile with onlineReplcas: 3 -> 2, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

## 清理资源

执行以下命令清理所有资源：

```shell
kubectl delete -f dynamic-predictive-portrait-sample.yaml
kubectl delete -f periodic-client.yaml
kubectl delete -f nginx-ingress.yaml
kubectl delete -f nginx-statefulset.yaml
```
