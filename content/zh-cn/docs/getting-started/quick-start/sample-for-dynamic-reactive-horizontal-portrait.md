---
title: "响应式画像示例"
linkTitle: "响应式画像示例"
description: "响应式弹性画像示例介绍"
weight: 16

---

## 前置环境准备

- Kubernetes cluster
- Kapacity installed on the cluster
- Promethues

## 安装步骤

### 1、部署测试用的服务

您可以通过下载 [nginx-statefulset.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/nginx-statefulset.yaml)
部署一个Nginx服务，您也可以部署自己的服务，只需要在后边部署IHPA yaml时修改下 ScaleTargetRef 的内容。

```bash
kubectl apply -f <your-file-directory>/nginx-statefulset.yaml
```

验证服务部署完成

```bash
kubectl get po

NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          5s
```


### 2、配置响应式画像&部署IHPA

下载 [dynamic-portrait-reactive-sample.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/autoscaling/dynamic-portrait-reactive-sample.yaml)
文件到本地。

```yaml
apiVersion: autoscaling.kapacity.traas.io/v1alpha1
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

执行命令创建IHPA CR

```bash
kubectl apply -f <your-file-director>/dynamic-portrait-reactive-sample.yaml
```

查看IHPA CR是否创建成功

```bash
kubectl get ihpa

NAME                               AGE
dynamic-portrait-reactive-sample   29m
```

### 3、压测服务

获取暴露的服务和端口号

```bash
kubectl get service nginx

NAME         TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
nginx        NodePort   10.111.21.74   <none>        80:32591/TCP    13m
```

通过压测工具或压测脚本对服务进行压测，比如通过Apache Benchmark(ab)进行压力测试，注意url中ip和port替换为您自己服务。

```bash
ab -n 10000 -c 100 http://<your-node-ip>:<your-port>/index
```

### 4、观察结果

等待3-10分钟后，通过IHPA Events可以看到具体的弹性结果

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