---
title: "使用多阶段灰度扩缩容"
weight: 3

---

## 准备开始

你需要拥有一个安装了 Kapacity 的 Kubernetes 集群。

## 运行示例工作负载

下载 [nginx-statefulset.yaml](/examples/workload/nginx-statefulset.yaml) 文件，并执行以下命令以运行一个 NGINX 服务：

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

## 创建配置了多阶段灰度缩容的 IHPA

下载 [gray-strategy-sample.yaml](/examples/ihpa/gray-strategy-sample.yaml) 文件，其内容如下所示：

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: gray-strategy-sample
spec:
  minReplicas: 1
  maxReplicas: 10
  portraitProviders:
  - priority: 1
    static:
      replicas: 1
    type: Static
  - cron:
      crons:
      - name: cron-1
        replicas: 5
        start: 0 * * * *
        end: 10 * * * *
    priority: 2
    type: Cron
  behavior:
    scaleDown:
      grayStrategy:
        grayState: Cutoff         # GrayState is the desired state of pods that in gray stage.
        changeIntervalSeconds: 30 # ChangeIntervalSeconds is the interval time between each gray change.
        changePercent: 50         # ChangePercent is the percentage of the total change of replica numbers which is used to calculate the amount of pods to change in each gray change.
        observationSeconds: 60    # ObservationSeconds is the additional observation time after the gray change reaching 100%.
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

该 IHPA 配置了以下两个画像源：

- 静态画像源：优先级为 1，副本数始终为 1。
- 定时画像源：优先级为 2，每小时第 0 分钟到第 10 分钟的副本数为 5。

由于定时画像源的优先级**高于**静态画像源，因此在其生效期间指定的副本数会覆盖静态画像源的副本数。

执行以下命令创建该 IHPA：

```shell
kubectl apply -f gray-strategy-sample.yaml
```

## 验证结果

在任意小时的第 0~9 分钟，我们可以看到定时画像源生效，工作负载的副本数从 1 扩容到了 5：

```shell
kubectl get po -L 'kapacitystack.io/pod-state' -o wide
```

```
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          50m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          56s   10.1.5.68   docker-desktop   <none>           1/1
nginx-2   1/1     Running   0          54s   10.1.5.69   docker-desktop   <none>           1/1
nginx-3   1/1     Running   0          52s   10.1.5.70   docker-desktop   <none>           1/1
nginx-4   1/1     Running   0          50s   10.1.5.71   docker-desktop   <none>           1/1
```

该工作负载对应服务的 Endpoint 数量也变为 5 个：

```shell
kubectl get ep nginx
```

```
NAME    ENDPOINTS                                            AGE
nginx   10.1.5.52:80,10.1.5.68:80,10.1.5.69:80 + 2 more...   3d3h
```

在第 10 分钟我们可以看到多阶段灰度缩容开始，其中 2 个 Pod 变为了 Cutoff 状态，并且从服务的 Endpoint 中摘除：

```shell
kubectl get po -L 'kapacitystack.io/pod-state' -o wide
```

```
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          51m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          63s   10.1.5.68   docker-desktop   <none>           1/1
nginx-2   1/1     Running   0          61s   10.1.5.69   docker-desktop   <none>           1/1
nginx-3   1/1     Running   0          59s   10.1.5.70   docker-desktop   <none>           0/1               Cutoff
nginx-4   1/1     Running   0          57s   10.1.5.71   docker-desktop   <none>           0/1               Cutoff
```

```shell
kubectl get ep nginx
```

```
NAME    ENDPOINTS                                AGE
nginx   10.1.5.52:80,10.1.5.68:80,10.1.5.69:80   3d3h
```

再过 30 秒后，可以看到 4 个 Pod 变为了 Cutoff 状态，并且从服务的 Endpoint 中摘除：

```shell
kubectl get po -L 'kapacitystack.io/pod-state' -o wide
```

```
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          51m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          96s   10.1.5.68   docker-desktop   <none>           0/1               Cutoff
nginx-2   1/1     Running   0          94s   10.1.5.69   docker-desktop   <none>           0/1               Cutoff
nginx-3   1/1     Running   0          92s   10.1.5.70   docker-desktop   <none>           0/1               Cutoff
nginx-4   1/1     Running   0          90s   10.1.5.71   docker-desktop   <none>           0/1               Cutoff
```

```shell
kubectl get ep nginx
```

```
NAME    ENDPOINTS      AGE
nginx   10.1.5.52:80   3d3h
```

再过 1 分钟后，可以看到工作负载最终被缩容到 1 个 Pod：

```shell
kubectl get po -L 'kapacitystack.io/pod-state' -o wide
```

```
NAME      READY   STATUS    RESTARTS   AGE    IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          52m    10.1.5.52   docker-desktop   <none>           1/1
```

你也可以通过 IHPA 的事件看到缩容的整个流程：

```shell
kubectl describe ihpa gray-strategy-sample
```

```
...
Events:
  Type    Reason                Age    From             Message
  ----    ------                ----   ----             -------
  Normal  CreateReplicaProfile  3m53s  ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal  UpdateReplicaProfile  2m44s  ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 5, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  104s   ihpa_controller  update ReplicaProfile with onlineReplcas: 5 -> 3, cutoffReplicas: 0 -> 2, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  74s    ihpa_controller  update ReplicaProfile with onlineReplcas: 3 -> 1, cutoffReplicas: 2 -> 4, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  14s    ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 1, cutoffReplicas: 4 -> 0, standbyReplicas: 0 -> 0
```

## 清理资源

您可以通过执行以下命令清理样例相关资源

```shell
kubectl delete -f gray-strategy-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```
