---
title: "使用定时扩缩容"
weight: 15

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

## 创建配置了定时画像源的 IHPA

下载 [cron-portrait-sample.yaml](/examples/ihpa/cron-portrait-sample.yaml) 文件，其内容如下所示：

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: cron-portrait-sample
spec:
  minReplicas: 1
  maxReplicas: 10
  portraitProviders:
  - type: Cron
    priority: 1
    cron:
      crons:
      - name: cron-1
        start: 0 * * * *
        end: 10 * * * *
        replicas: 1
      - name: cron-2
        start: 10 * * * *
        end: 20 * * * *
        replicas: 2
      - name: cron-3
        start: 20 * * * *
        end: 30 * * * *
        replicas: 3
      - name: cron-4
        start: 30 * * * *
        end: 40 * * * *
        replicas: 4
      - name: cron-5
        start: 40 * * * *
        end: 50 * * * *
        replicas: 5
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

执行以下命令创建该 IHPA：

```shell
kubectl apply -f cron-portrait-sample.yaml
```

## 验证结果

通过查看 IHPA 的事件可以看到工作负载的副本数正按我们的配置进行动态调整：

```shell
kubectl describe ihpa cron-portrait-sample
```

```
...
Events:
  Type     Reason                Age                From             Message
  ----     ------                ----               ----             -------
  Normal   CreateReplicaProfile  38m                ihpa_controller  create ReplicaProfile with onlineReplcas: 3, cutoffReplicas: 0, standbyReplicas: 0
  Normal   UpdateReplicaProfile  33m (x2 over 33m)  ihpa_controller  update ReplicaProfile with onlineReplcas: 3 -> 4, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  23m                ihpa_controller  update ReplicaProfile with onlineReplcas: 4 -> 5, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Warning  NoValidPortraitValue  13m                ihpa_controller  no valid portrait value for now
  Normal   UpdateReplicaProfile  3m15s              ihpa_controller  update ReplicaProfile with onlineReplcas: 5 -> 1, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

你也可以通过直接观察工作负载的副本数变化来验证。

{{% alert title="说明" %}}
可以看到，由于上面的 cron 表达式没有覆盖所有时间段，因此在某段时间内出现了 `NoValidPortraitValue` 事件，此时工作负载的副本数将保持不变。
{{% /alert %}}

## 清理资源

执行以下命令清理所有资源：

```shell
kubectl delete -f cron-portrait-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```
