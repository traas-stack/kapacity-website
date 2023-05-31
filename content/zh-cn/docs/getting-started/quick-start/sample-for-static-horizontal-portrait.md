---
title: "静态画像示例"
linkTitle: "静态画像示例"
description: "静态画像示例介绍"
weight: 14

---

## 前置环境准备

- Kubernetes cluster
- Kapacity installed on the cluster

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

### 2、配置静态画像&部署IHPA

下载 [static-portrait-sample.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/autoscaling/static-portrait-sample.yaml)
文件到本地，如果您希望弹性自己的服务还请修改 spec.scaleTargetRef 字段。

```yaml
apiVersion: autoscaling.kapacity.traas.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: static-portrait-sample
spec:
  paused: false
  minReplicas: 0
  maxReplicas: 10
  portraitProviders:
    - type: Static
      priority: 0
      static:
        replicas: 2
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

执行命令创建IHPA CR

```bash
kubectl apply -f <your-file-director>/static-portrait-sample.yaml
```

查看IHPA CR是否创建成功

```bash
kubectl get ihpa

NAME                     AGE
static-portrait-sample   11m
```

### 3、验证结果

执行以下命令，可以看到副本数从1扩容到2

```bash
kubectl get po

NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          7m18s
nginx-1   1/1     Running   0          7s
```

## 清理资源

您可以通过执行以下命令清理样例相关资源

```bash
kubectl delete -f <your-file-director>/static-portrait-sample.yaml 
kubectl delete -f <your-file-director>/nginx-statefulset.yaml 
```

如果您使用自己的服务，请您单独清理