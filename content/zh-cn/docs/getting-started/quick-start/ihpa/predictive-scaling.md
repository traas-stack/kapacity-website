---
title: "使用预测式扩缩容"
weight: 3

---

## 准备开始

你需要拥有一个安装了 Kapacity、Prometheus 的 Kubernetes 集群。

## 安装 Ingress Controller

参考 <a href="https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/" target="_blank"> Ingress Controller
官方文档 </a> 进行安装，并修改配置以支持 Prometheus 采集 Ingress 指标数据，注意在 Prometheus 控制台验证是否正确采集。

## 添加 Ingress 自定义指标

在 kapacity 的 metrics config 里加入 ingress 自定义指标 **nginx_ingress_controller_requests_rate**，然后重启 kapacity

```shell
kubectl edit cm kapacity-config -nkapacity-system
```

```yaml
kind: ConfigMap
apiVersion: v1
...
data:
  prometheus-metrics-config.yaml: |
    # 新增 nginx ingress 请求自定义指标
    rules:
    - seriesQuery: '{__name__=~"^nginx_ingress_controller_requests.*",namespace!=""}'
      seriesFilters: []
      resources:
        template: "<<.Resource>>"
      name:
        matches: ""
        as: "nginx_ingress_controller_requests_rate"
      metricsQuery: "round(sum(rate(<<.Series>>{<<.LabelMatchers>>}[3m])) by (<<.GroupBy>>), 0.001)"
    externalRules:
      ...
```

## 运行示例工作负载

1. 下载 [nginx-statefulset.yaml](/examples/workload/nginx-statefulset.yaml) 文件，并通过 kubectl 创建一个 NGINX 服务：

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

2. 下载 [nginx-ingress.yaml](/examples/workload/nginx-ingress.yaml) 文件，并通过 kubectl 命令创建一个 ingress 服务：

```shell
kubectl apply -f nginx-ingress.yaml
```

验证 ingress 创建成功

```shell
kubectl get ingress
```

```shell
NAME           CLASS   HOSTS                  ADDRESS     PORTS   AGE
nginx-server   nginx   nginx.example.com      localhost   80      2d
```

3. 运行脚本模拟业务请求，产生训练所需数据集。

下载 [test-client.yaml](/examples/workload/test-client.yaml)，通过 kubectl 命令启动一个不同的 Pod
作为客户端，通过脚本周期性(每小时为1个周期)向 NGINX 服务发出请求，模拟周期性业务负载：

```shell
kubectl apply -f test-client.yaml
```

周期性流量示意图：

<img src="/images/en/workload-metrics.png" width="80%"/>


由于训练所需数据集较大，建议至少运行24小时以后再进行模型训练

## 训练预测模型

通过 [时序预测模型训练](/zh-cn/docs/user-guide/algorithm/predictive-model-training) 提前生成好模型，保存模型文件到单独目录。

```shell
-rw-r--r--@ 1 admin  staff   316K Oct 11 17:29 checkpoint.pth
-rw-r--r--@ 1 admin  staff   286B Oct 11 17:30 estimator_config.yaml
-rw-r--r--@ 1 admin  staff    29B Oct 11 17:30 item2id.json
```

保存预测模型到 configmap

```shell
kubectl -n <your-namespace> create configmap kapacity-tsf-model --from-file=<your-model-path>
```

其中 your-namespace 和 your-model-path 需要替换为自己的命名空间和模型目录。

## 创建配置了预测式画像源的 IHPA

下载 [dynamic-predictive-portrait-sample.yaml](/examples/ihpa/dynamic-predictive-portrait-sample.yaml) 文件，编辑该yaml文件，其中
algorithm-image-id 需要替换为真实的算法镜像ID，具体内容如下所示：

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: dynamic-predictive-portrait-sample
spec:
  maxReplicas: 10
  minReplicas: 1
  portraitProviders:
  - dynamic:
      algorithm:
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
                            image: <algorithm-image-id>
                            imagePullPolicy: IfNotPresent
                            args:
                            - --tsf-model-path=/opt/kapacity/timeseries/forecasting/model
                            - --tsf-freq=10min
                            - --re-history-len=48H
                            - --re-time-delta-hours=8
                            - --scaling-freq=10min
                            - --re-test-dataset-size-in-seconds=86400
                            volumeMounts:
                            - name: model
                              mountPath: /opt/kapacity/timeseries/forecasting/model
                              readOnly: true
                          volumes:
                          - name: model
                            configMap:
                              name: kapacity-tsf-model
                          restartPolicy: OnFailure
          resultSource:
            type: ConfigMap
        type: ExternalJob
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            averageUtilization: 30
            type: Utilization
      - type: External
        external:
          metric:
            name: ready_pods_count
          target:
            type: NA
      - type: Object
        name: qps
        object:
          describedObject:
            apiVersion: networking.k8s.io/v1
            kind: Ingress
            name: nginx-server
          metric:
            name: nginx_ingress_controller_requests_rate
          target:
            type: NA
      portraitType: Predictive
    priority: 10
    type: Dynamic
  scaleMode: Auto
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nginx
```

在预测算法中我们约定了 metrics 的指标类型和顺序：

- 第一个指标为资源指标，类型为 Resource ，用于查询资源（cpu、mem）历史水位。
- 第二个指标为副本数指标，类型为 External ，用于查询可用副本数历史数据。
- 第三个及其他为流量指标，用户可以自己根据需要选择类型，用于查询流量（rpc、msg、pv等）历史数据。

执行以下命令创建该 IHPA：

```shell
kubectl apply -f dynamic-predictive-portrait-sample.yaml
```

## 验证结果

1. IHPA 的 portrait provider 创建 CronJob 启动预测式算法 job

```shell
kubectl get cronjob
```

```shell
NAME                                 SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
dynamic-predictive-portrait-sample   0/30 * * * *   False     1        26m             2d1h
```

```shell
kubectl get job
```

```shell
NAME                                          COMPLETIONS   DURATION   AGE
dynamic-predictive-portrait-sample-28286564   1/1           16s        28m
```

2. 预测式算法 job 运行成功后回写结果到 configmap

```shell
kubectl get cm dynamic-predictive-portrait-sample-result -oyaml
```

可以看到生成了未来 3 个时间点的副本数预测结果

```yaml
apiVersion: v1
data:
  expireTime: "2023-10-13T13:00:00.00Z"
  timeSeries: '{"1697200200": 3, "1697200800": 2, "1697201400": 1}'
  type: TimeSeries
kind: ConfigMap
metadata:
  name: dynamic-predictive-portrait-sample-result
  ownerReferences:
  - apiVersion: autoscaling.kapacitystack.io/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: HorizontalPortrait
    name: dynamic-predictive-portrait-sample
    uid: 8e0ea542-6e7d-4529-83dc-efcfdec8ec6e
```

3. 根据预测的结果执行副本变更

```shell
 kubectl describe ihpa dynamic-predictive-portrait-sample
```

```shell
Events:
  Type     Reason                Age                 From             Message
  ----     ------                ----                ----             -------
  Warning  NoValidPortraitValue  29m (x10 over 85m)  ihpa_controller  no valid portrait value for now
  Normal   UpdateReplicaProfile  25m                 ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 3, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  15m                 ihpa_controller  update ReplicaProfile with onlineReplcas: 3 -> 2, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  5m9s                ihpa_controller  update ReplicaProfile with onlineReplcas: 2 -> 1, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0

```

## 清理资源

执行以下命令清理所有资源：

```shell
kubectl delete -f dynamic-predictive-portrait-sample.yaml
kubectl delete -f test-client.yaml
kubectl delete -f nginx-ingress.yaml
kubectl delete -f nginx-statefulset.yaml
```
