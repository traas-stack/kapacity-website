---
title: "IHPA 整体架构"
weight: 1
---

## 组件架构图

<img src="/images/en/ihpa-architecture.png" width="1000"/>

图例：

- 蓝底的为 IHPA 自身组件
- 绿底的为 IHPA 自身的 [Kubernetes 定制资源（CR）](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/api-extension/custom-resources/)对象
- 黄底的为 IHPA 依赖的其他 Kubernetes 资源对象
- 白底的为相关外部系统

部分缩写的全称：

- IHPA: IntelligentHorizontalPodAutoscaler
- HPortrait: HorizontalPortrait
- CM: ConfigMap

## 组件概述

### Replica Controller

IHPA 执行层组件，负责具体工作负载副本数量和状态控制，其通过对接不同的原生和第三方组件支持 Pod 的扩缩容、摘挂流和激保活等操作。

### IHPA Controller

IHPA 控制面组件，直接接受用户或外部系统的 IHPA 配置（包括目标工作负载、指标、算法、变更与稳定性配置等），下发画像任务并整合画像结果，再根据画像结果执行多级分批弹性伸缩。

### HPortrait Controller

内置水平弹性伸缩算法管理组件，负责运行和管理针对不同工作负载的不同弹性伸缩算法的工作流，并将其输出结果转换为标准画像格式。具体的算法子任务则会作为单独的 Kubernetes Job 或者其他大数据/算法平台的任务被调度执行。这些子任务会从外部监控系统中获取历史与实时指标数据进行计算并生成画像结果。

特别的，部分简单算法（如响应式算法等）的逻辑是直接实现在该组件中，不再走单独的算法子任务。

### Metrics Provider Server

统一监控指标查询组件，屏蔽底层监控系统差异，为外部运行的组件（如算法任务等）提供统一监控指标查询服务。

其提供的 API 与 [Kubernetes Metrics API](https://github.com/kubernetes/metrics) 类似，但不同的是它能够同时支持实时和历史指标查询。

### Agent（暂未引入）

运行在 Kubernetes 集群节点上的 agent 组件，主要负责执行 Pod 的激保活等需要与底层操作系统进行交互的操作。
