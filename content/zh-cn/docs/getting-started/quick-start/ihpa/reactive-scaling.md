---
title: "使用响应式扩缩容"
weight: 2

---

## 准备开始

你需要拥有一个安装了 Kapacity 与 Prometheus 的 Kubernetes 集群。

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

## 创建配置了动态响应式画像源的 IHPA

下载 [dynamic-reactive-portrait-sample.yaml](/examples/ihpa/dynamic-reactive-portrait-sample.yaml) 文件，其内容如下所示：

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: dynamic-reactive-portrait-sample
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
      portraitType: Reactive
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 30
      algorithm:
        type: KubeHPA
```

执行以下命令创建该 IHPA：

```shell
kubectl apply -f dynamic-reactive-portrait-sample.yaml
```

## 增加负载

执行以下命令获取 NGINX 服务的 ClusterIP 和端口：

```shell
kubectl get svc nginx
```

```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
nginx        ClusterIP   10.111.21.74   <none>        80/TCP     13m
```

启动一个不同的 Pod 作为客户端，该 Pod 会不断地向 NGINX 服务发出请求，其中的服务地址和端口请替换为上一步中得到的值：

```shell
# 在单独的终端中运行它以便负载生成继续，你可以继续执行其余步骤
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://<service-ip>:<service-port> > /dev/null; done"
```

等待几分钟后，可以通过 IHPA 的事件看到工作负载被扩容了：

```shell
kubectl describe ihpa dynamic-reactive-portrait-sample
```

```
...
Events:
  Type    Reason                Age    From             Message
  ----    ------                ----   ----             -------
  Normal  CreateReplicaProfile  6m58s  ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal  UpdateReplicaProfile  3m45s  ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 6, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

## 停止产生负载

在我们创建 `busybox` 容器的终端中，输入 `<Ctrl> + C` 来终止负载的产生。

等待几分钟后，可以通过 IHPA 的事件看到工作负载被缩容了：

```shell
kubectl describe ihpa dynamic-reactive-portrait-sample
```

```
...
Events:
  Type    Reason                Age    From             Message
  ----    ------                ----   ----             -------
  Normal  CreateReplicaProfile  9m58s  ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal  UpdateReplicaProfile  6m45s  ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 6, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  3m15s  ihpa_controller  update ReplicaProfile with onlineReplcas: 6 -> 4, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  2m45s  ihpa_controller  update ReplicaProfile with onlineReplcas: 4 -> 1, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

## 清理资源

执行以下命令清理所有资源：

```shell
kubectl delete -f dynamic-reactive-portrait-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```
