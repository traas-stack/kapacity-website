---
title: "安装指南"
description: "了解如何安装或卸载 Kapacity"
weight: 12

---

## 环境准备

- [Kubernetes](https://kubernetes.io/zh-cn/) 1.16+ (1.22+ recommended)
- [Helm](https://helm.sh/zh/) 3

## 安装流程

### 安装 cert-manager

使用 Helm 安装 [cert-manager](https://cert-manager.io/)。

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### 安装 Prometheus

使用 Helm 安装 [Prometheus](https://prometheus.io/) 与 [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)。

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install \
  prometheus prometheus-community/prometheus \
  --namespace prometheus \
  --create-namespace \
  --set alertmanager.enabled=false \
  --set prometheus-node-exporter.enabled=false \
  --set prometheus-pushgateway.enabled=false
```

可以使用如下命令查看 Prometheus Server 的 ClusterIP 和端口：

```shell
kubectl get svc -n prometheus
```

```
NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
prometheus-kube-state-metrics         ClusterIP   10.97.190.94     <none>        8080/TCP   5m
prometheus-server                     ClusterIP   10.104.214.48    <none>        80/TCP     5m
```

### 安装 Kapacity

使用 Helm 安装 Kapacity。其中的 Prometheus Server 地址相关参数即为上一步查看到的值。

```shell
helm repo add kapacity https://traas-stack.github.io/kapacity-charts
helm repo update

helm install \
  kapacity-manager kapacity/kapacity-manager \
  --namespace kapacity-system \
  --create-namespace \
  --set prometheus.address=http://<prometheus-server-clusterip>:<prometheus-server-port> 
```

### 验证 Kapacity 安装是否成功

```shell
kubectl get deploy -n kapacity-system
```

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
kapacity-manager   1/1     1            1           5m
```

## 卸载流程

```shell
helm uninstall kapacity-manager -n kapacity-system
helm uninstall prometheus -n prometheus
helm uninstall cert-manager -n cert-manager
```
