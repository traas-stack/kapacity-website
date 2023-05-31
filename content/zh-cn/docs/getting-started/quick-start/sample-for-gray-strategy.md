---
title: "灰度变更示例"
linkTitle: "灰度变更示例"
description: "灰度变更示例介绍"
weight: 17

---

## 前置环境准备

- Kubernetes cluster
- Kapacity installed on the cluster

## 安装步骤

### 1、部署测试用的服务

下载 [nginx-statefulset.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/nginx-statefulset.yaml)
文件到本地，然后执行以下命令

```bash
kubectl apply -f <your-file-directory>/nginx-statefulset.yaml
```

验证服务部署完成

```bash
kubectl get po

NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          43s
```

### 2、配置静态画像&部署IHPA

下载 [gray-strategy-sample.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/autoscaling/gray-strategy-sample.yaml)
文件到本地。

```yaml
apiVersion: autoscaling.kapacity.traas.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: gray-strategy-sample
spec:
  paused: false
  minReplicas: 0
  maxReplicas: 10
  portraitProviders:
    - type: Static
      priority: 0
      static:
        replicas: 10
  behavior:
    scaleDown:
      grayStrategy:
        grayState: Cutoff            #灰度变更中间状态
        changeIntervalSeconds: 10    #每批次变更间隔时间，单位为秒
        changePercent: 50            #每批次变更粒度，单位为百分比
        observationSeconds: 60       #从中间状态变更到终态前的观察时间
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

执行以下命令，生成IHPA CR并扩容副本数到10

```bash
kubectl apply -f <your-file-director>/gray-strategy-sample.yaml
```

查看执行结果

```bash
kubectl get po

NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          36s
nginx-1   1/1     Running   0          34s
nginx-2   1/1     Running   0          11s
nginx-3   1/1     Running   0          10s
nginx-4   1/1     Running   0          9s
nginx-5   1/1     Running   0          8s
nginx-6   1/1     Running   0          7s
nginx-7   1/1     Running   0          6s
nginx-8   1/1     Running   0          5s
nginx-9   1/1     Running   0          2s
```

### 3、进行灰度渐进式缩容

执行以下命令编辑IHPA CR gray-strategy-sample，修改目标副本数为2。

```bash
kubectl edit ihpa gray-strategy-sample
```

等待10s后，可以看到4个pod变为了Cutoff状态

```bash
kubectl get po 'kapacity.traas.io/pod-state'

NAME      READY   STATUS    RESTARTS   AGE   POD-STATE
nginx-0   1/1     Running   0          70s
nginx-1   1/1     Running   0          68s
nginx-2   1/1     Running   0          45s
nginx-3   1/1     Running   0          44s
nginx-4   1/1     Running   0          43s
nginx-5   1/1     Running   0          42s
nginx-6   1/1     Running   0          41s   Cutoff
nginx-7   1/1     Running   0          40s   Cutoff
nginx-8   1/1     Running   0          39s   Cutoff
nginx-9   1/1     Running   0          36s   Cutoff
```

再等待10s后，可以看到8个pod变为Cutoff状态

```bash
kubectl get po 'kapacity.traas.io/pod-state'

NAME      READY   STATUS    RESTARTS   AGE   POD-STATE
nginx-0   1/1     Running   0          78s
nginx-1   1/1     Running   0          76s
nginx-2   1/1     Running   0          53s   Cutoff
nginx-3   1/1     Running   0          52s   Cutoff
nginx-4   1/1     Running   0          51s   Cutoff
nginx-5   1/1     Running   0          50s   Cutoff
nginx-6   1/1     Running   0          49s   Cutoff
nginx-7   1/1     Running   0          48s   Cutoff
nginx-8   1/1     Running   0          47s   Cutoff
nginx-9   1/1     Running   0          44s   Cutoff
```

再观察1m后，可以看到pod被缩容到2个

```bash
kubectl get po 'kapacity.traas.io/pod-state'

NAME      READY   STATUS    RESTARTS   AGE     POD-STATE
nginx-0   1/1     Running   0          2m28s
nginx-1   1/1     Running   0          2m26s
```

您也可以通过IHPA Events看到缩容的整个流程

```bash
kubectl describe ihpa gray-strategy-sample

Events:
  Type    Reason                Age    From             Message
  ----    ------                ----   ----             -------
  Normal  CreateReplicaProfile  2m52s  ihpa_controller  create ReplicaProfile with onlineReplcas: 10, cutoffReplicas: 0, standbyReplicas: 0
  Normal  UpdateReplicaProfile  2m15s  ihpa_controller  update ReplicaProfile with onlineReplcas: 10 -> 6, cutoffReplicas: 0 -> 4, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  2m5s   ihpa_controller  update ReplicaProfile with onlineReplcas: 6 -> 2, cutoffReplicas: 4 -> 8, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  65s    ihpa_controller  update ReplicaProfile with onlineReplcas: 2 -> 2, cutoffReplicas: 8 -> 0, standbyReplicas: 0 -> 0
```

## 清理资源

您可以通过执行以下命令清理样例相关资源

```bash
kubectl delete -f <your-file-director>/gray-strategy-sample.yaml 
kubectl delete -f <your-file-director>/nginx-statefulset.yaml 
```