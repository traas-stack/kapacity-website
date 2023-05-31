---
title: "定时画像示例"
linkTitle: "定时画像示例"
description: "定时弹性画像示例介绍"
weight: 15

---

## 前置准备

- Kubernetes cluster
- Kapacity installed on the cluster

## 安装步骤

### 1、部署测试用的服务

可以通过下载 [nginx-statefulset.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/nginx-statefulset.yaml)
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

### 2、配置定时画像&部署IHPA

下载 [cron-portrait-sample.yaml](https://github.com/traas-stack/kapacity/blob/main/examples/autoscaling/cron-portrait-sample.yaml)
文件，并按照需求修改Yaml文件中的 spec.portraitProviders 字段和 spec.scaleTargetRef
字段。（样例里包含定时画像，每10分钟修改一次副本数）

```yaml
apiVersion: autoscaling.kapacity.traas.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: cron-portrait-sample
spec:
  paused: false
  minReplicas: 0
  maxReplicas: 10
  portraitProviders:
    - type: Cron
      priority: 0
      cron:
        crons:
          - name: "cron-1"
            start: "0 * * * *"
            end: "10 * * * *"
            replicas: 1
          - name: "cron-2"
            start: "10 * * * *"
            end: "20 * * * *"
            replicas: 2
          - name: "cron-3"
            start: "20 * * * *"
            end: "30 * * * *"
            replicas: 3
          - name: "cron-4"
            start: "30 * * * *"
            end: "40 * * * *"
            replicas: 4
          - name: "cron-5"
            start: "40 * * * *"
            end: "50 * * * *"
            replicas: 5
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

执行命令创建IHPA CR

```bash
kubectl apply -f <your-file-director>/cron-portrait-sample.yaml
```

查看IHPA CR是否创建成功

```bash
kubectl get ihpa

NAME                   AGE
cron-portrait-sample   13s
```

### 3、验证结果

通过查看 IHPA 的事件您可以看到如下结果：

```bash
kubectl describe ihpa cron-portrait-sample

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

## 清理资源

您可以执行以下命令清理相关资源

```bash
kubectl delete -f <your-file-director>/cron-portrait-sample.yaml 
kubectl delete -f <your-file-director>/nginx-statefulset.yaml 
```

如果您使用自己的服务，请您单独清理