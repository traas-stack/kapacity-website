---
title: "Installation"
description: "How to install or uninstall Kapacity"
weight: 12

---

## Prerequisites

- [Kubernetes](https://kubernetes.io/) 1.16+ (1.22+ recommended)
- [Helm](https://helm.sh/) 3

## Install

### Install cert-manager

Install [cert-manager](https://cert-manager.io/) by Helm.

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Install Prometheus

Install [Prometheus](https://prometheus.io/) with [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) by Helm.

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

Run following command to view Prometheus Server's ClusterIP and port:

```shell
kubectl get svc -n prometheus
```

```
NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
prometheus-kube-state-metrics         ClusterIP   10.97.190.94     <none>        8080/TCP   5m
prometheus-server                     ClusterIP   10.104.214.48    <none>        80/TCP     5m
```

### Install Kapacity

Install Kapacity by Helm. The Prometheus Server address params are those got in previous step.

```shell
helm repo add kapacity https://traas-stack.github.io/kapacity-charts
helm repo update

helm install \
  kapacity-manager kapacity/kapacity-manager \
  --namespace kapacity-system \
  --create-namespace \
  --set prometheus.address=http://<prometheus-server-clusterip>:<prometheus-server-port> 
```

### Verify Kapacity installation

```shell
kubectl get deploy -n kapacity-system
```

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
kapacity-manager   1/1     1            1           5m
```

## Uninstall

```shell
helm uninstall kapacity-manager -n kapacity-system
helm uninstall prometheus -n prometheus
helm uninstall cert-manager -n cert-manager
```
