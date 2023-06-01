---
title: "Installation"
description: "How to install or uninstall Kapacity"
weight: 12

---

This document mainly introduces environment preparation and how to quickly install and deploy Kapacity.

## Pre-requisites

- Kubernetes 1.19+
- Helm 3+
- Cert-manager 1.11+
- Prometheus

## Steps

### Installing CertManager

Refer to [Cert-Manager Official Documentation](https://cert-manager.io/docs/installation/helm/) to install

### Installing Prometheus

Prometheus can be installed using helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus  prometheus-community/prometheus -n prometheus  \
   --create-namespace \
   --set pushgateway.enabled=false \
   --set alertmanager.enabled=false \
   --set prometheus-node-exporter.hostRootFsMount.enabled=false
```

### Installing Kapacity

Install Kapacity using Helm

- Add Helm repo

```bash
helm repo add kapacity https://kapacity.github.io/charts
```

- Update Helm repo

```bash
helm repo update
```

- Install Kapacity using Helm Charts

```bash
kubectl create namespace kapacity-system
helm install kapacity kapacity --namespace kapacity-system
```

- Verify Kapacity installation results

```bash
kubectl get deploy -n kapacity-system

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
kapacity-controller-manager   1/1     1            1           3d23h
```

## Uninstall

```bash
helm uninstall kapacity -n kapacity-system
helm uninstall prometheus -n prometheus
