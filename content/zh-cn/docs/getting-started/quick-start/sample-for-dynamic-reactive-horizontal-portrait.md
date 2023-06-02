---
title: "使用响应式扩缩容"
linkTitle: "使用响应式扩缩容"
description: "使用响应式扩缩容示例"
weight: 16

---

## 前置环境准备

- Kubernetes cluster
- Kapacity installed on the cluster
- Promethues

## 安装步骤

### 1.部署测试服务

下载 [nginx-statefulset.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/nginx-statefulset.yaml)
文件，并执行以下命令可以快速部署一个 Nginx 服务。 您也可以部署自己的服务，只需要在后边部署 IHPA yaml 时修改下
***scaleTargetRef*** 的内容。

```bash
cd <your-file-directory>
kubectl apply -f nginx-statefulset.yaml
```

验证服务部署完成

```bash
kubectl get po

NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          5s
```

### 2.使用响应式画像扩缩容

下载或拷贝以下配置到 [dynamic-portrait-reactive-sample.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/autoscaling/dynamic-portrait-reactive-sample.yaml)
文件。

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: dynamic-portrait-reactive-sample
spec:
  paused: false
  minReplicas: 0
  maxReplicas: 10
  portraitProviders:
    - type: Dynamic
      priority: 0
      dynamic:
        portraitType: Reactive
        metrics:
          - name: metrics-scale-out
            type: Resource
            resource:
              name: cpu
              target:
                type: Utilization
                averageUtilization: 30
        algorithm:
          type: KubeHPA
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

执行命令创建 IHPA CR

```bash
kubectl apply -f dynamic-portrait-reactive-sample.yaml
```

查看 IHPA CR 是否创建成功

```bash
kubectl get ihpa

NAME                               AGE
dynamic-portrait-reactive-sample   29m
```

### 3.压测服务

获取 Nginx 服务暴露的端口号（样例里是32591）

```bash
kubectl get service nginx

NAME         TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
nginx        NodePort   10.111.21.74   <none>        80:32591/TCP    13m
```

通过压测工具或压测脚本对服务进行压测，比如通过 Apache Benchmark(ab) 进行压力测试，注意 url 中 ip 和 port 替换为您自己的。

```bash
ab -n 10000 -c 100 http://<your-node-ip>:<your-port>/index
```

### 4.观察结果

等待3-10分钟后，通过 IHPA Events 可以看到具体的弹性结果

```bash
kubectl describe ihpa dynamic-portrait-reactive-sample

Events:
  Type     Reason                             Age                From             Message
  ----     ------                             ----               ----             -------
  Warning  FailedUpdateAndFetchPortraitValue  27m                ihpa_controller  failed to update and fetch portrait value: failed to fetch portrait value: failed to get HorizontalPortrait "default/dynamic-portrait-reactive-sample-reactive": HorizontalPortrait.autoscaling.kapacity.traas.io "dynamic-portrait-reactive-sample-reactive" not found
  Warning  NoValidPortraitValue               10m (x9 over 27m)  ihpa_controller  no valid portrait value for now
  Normal   CreateReplicaProfile               9m58s              ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal   UpdateReplicaProfile               6m45s              ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 6, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile               3m15s              ihpa_controller  update ReplicaProfile with onlineReplcas: 6 -> 4, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile               2m45s              ihpa_controller  update ReplicaProfile with onlineReplcas: 4 -> 1, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

## 清理资源

您可以通过执行以下命令清理样例相关资源

```bash
kubectl delete -f <your-file-director>/dynamic-portrait-reactive-sample.yaml 
kubectl delete -f <your-file-director>/nginx-statefulset.yaml 
```