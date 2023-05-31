---
title: "安装指南"
linkTitle: "安装指南"
description: "Kapacity安装和卸载"
weight: 12

---

本文档主要介绍环境准备、如何快速安装部署Kapacity。

## 环境准备

- Go 1.19+
- Kubernetes 1.19+
- Cert-manager 1.11+
- Helm 3.1.0

## 安装流程

我们提供了多种可选的安装方式，主要包括：

- [基于Helm安装](#基于helm安装)
- [基于Yaml安装](#基于yaml安装)

### 基于Helm安装

使用Helm安装Kapacity

- 添加Helm仓库地址

```bash
helm repo add kapacity https://kapacity.github.io/charts
```

- 更新Helm仓库到最新版本

```bash
helm repo update
```

- 使用Helm Charts安装Kapacity

```bash
kubectl create namespace kapacity-system
helm install kapacity kapacity --namespace kapacity-system
```

- 验证Kapacity安装是否成功

```bash
kubectl get deploy -n kapacity-system

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
kapacity-controller-manager   1/1     1            1           3d23h
```

### 基于Yaml安装

我们在 [kapacity-deploy.yaml]() 中提供了 CRD 和相关资源，您可以通过下载文件并执行 **kubectl** 命令来安装 Kapacity。

- 通过Yaml安装Kapacity

```bash
kubectl apply -f kapacity-deploy.yaml
```

- 验证Kapacity安装是否成功

```bash
kubectl get deploy -n kapacity-system

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
kapacity-controller-manager   1/1     1            1           3d23h
```

## 卸载安装

- 基于Helm安装的卸载

```bash
helm uninstall kapacity -n kapacity-system
```

- 基于Yaml安装的卸载

```bash
kubectl delete -f kapacity-deploy.yaml
```