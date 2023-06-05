---
title: "使用多阶段灰度扩缩容"
linkTitle: "使用多阶段灰度扩缩容"
description: "使用多阶段灰度扩缩容示例"
weight: 17

---

## 前置环境准备

- Kubernetes cluster
- Kapacity installed on the cluster

## 安装步骤

### 1.部署测试服务

下载 [nginx-statefulset.yaml](/examples/nginx-statefulset.yaml) 文件，并执行以下命令可以快速部署一个 Nginx 服务。
您也可以部署自己的服务，只需要在后边部署 IHPA yaml 时修改下 ***scaleTargetRef*** 的内容。

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

### 2.使用多阶段灰度扩缩容

下载或复制以下配置到 [gray-strategy-sample.yaml](/examples/ihpa/gray-strategy-sample.yaml) 文件。该配置里包含2个画像：

- 静态画像：
    - 优先级为1，副本数为1
- 定时画像
    - 优先级为2，副本数为5，每个小时第0分钟到第10分钟生效

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: gray-strategy-sample
spec:
  paused: false
  minReplicas: 0
  maxReplicas: 10
  portraitProviders:
    - priority: 1
      static:
        replicas: 1
      type: Static
    - cron:
        crons:
          - end: 10 * * * *
            name: test
            replicas: 5
            start: 0 * * * *
            timeZone: Asia/Shanghai
      priority: 2
      type: Cron
  behavior:
    scaleDown:
      grayStrategy:
        grayState: Cutoff            #灰度变更中间状态
        changeIntervalSeconds: 30    #每批次变更间隔时间，单位为秒
        changePercent: 50            #每批次变更粒度，单位为百分比
        observationSeconds: 60       #从中间状态变更到终态前的观察时间
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

执行以下命令，生成 IHPA CR

```bash
kubectl apply -f gray-strategy-sample.yaml
```

假如当前时间为第0-9分钟，可以看到定时画像生效(优先级高)，pod 数从1扩容到了5

```bash
kubectl get po -L 'kapacitystack.io/pod-state' -owide

NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          50m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          56s   10.1.5.68   docker-desktop   <none>           1/1
nginx-2   1/1     Running   0          54s   10.1.5.69   docker-desktop   <none>           1/1
nginx-3   1/1     Running   0          52s   10.1.5.70   docker-desktop   <none>           1/1
nginx-4   1/1     Running   0          50s   10.1.5.71   docker-desktop   <none>           1/1
```

服务 EndPoint 数也变为5个

```bash
kubectl get ep nginx

NAME    ENDPOINTS                                            AGE
nginx   10.1.5.52:80,10.1.5.68:80,10.1.5.69:80 + 2 more...   3d3h
```

第10分钟可以看到，2个 pod 变为了 Cutoff 状态，并且从服务的 EndPoint 列表里摘除

```bash
kubectl get po -L 'kapacitystack.io/pod-state' -owide

NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          51m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          63s   10.1.5.68   docker-desktop   <none>           1/1
nginx-2   1/1     Running   0          61s   10.1.5.69   docker-desktop   <none>           1/1
nginx-3   1/1     Running   0          59s   10.1.5.70   docker-desktop   <none>           0/1               Cutoff
nginx-4   1/1     Running   0          57s   10.1.5.71   docker-desktop   <none>           0/1               Cutoff

------
kubectl get ep nginx

NAME    ENDPOINTS                                AGE
nginx   10.1.5.52:80,10.1.5.68:80,10.1.5.69:80   3d3h
```

再等待30s后，可以看到4个 pod 变为 Cutoff 状态，并且从服务的 EndPoint 列表里摘除

```bash
kubectl get po -L 'kapacitystack.io/pod-state' -owide

NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          51m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          96s   10.1.5.68   docker-desktop   <none>           0/1               Cutoff
nginx-2   1/1     Running   0          94s   10.1.5.69   docker-desktop   <none>           0/1               Cutoff
nginx-3   1/1     Running   0          92s   10.1.5.70   docker-desktop   <none>           0/1               Cutoff
nginx-4   1/1     Running   0          90s   10.1.5.71   docker-desktop   <none>           0/1               Cutoff

------
kubectl get ep nginx

NAME    ENDPOINTS      AGE
nginx   10.1.5.52:80   3d3h
```

再观察1m后，可以看到 pod 被缩容到1个

```bash
kubectl get po -L 'kapacitystack.io/pod-state' -owide

NAME      READY   STATUS    RESTARTS   AGE    IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          52m    10.1.5.52   docker-desktop   <none>           1/1
```

您也可以通过 IHPA Events 看到缩容的整个流程

```bash
kubectl describe ihpa gray-strategy-sample

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

```bash
kubectl delete -f gray-strategy-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```