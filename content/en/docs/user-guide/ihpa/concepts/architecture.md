---
title: "IHPA Architecture Overview"
weight: 1
---

{{% pageinfo color="primary" %}}
Note: Most contents of this page are translated by AI from [Chinese](/zh-cn/docs/user-guide/ihpa/concepts/architecture/). It would be very appreciated if you could help us improve by editing this page (click the corresponding link on the right side).
{{% /pageinfo %}}

## Component Architecture

<img src="/images/en/ihpa-architecture.png" width="1000"/>

Legend:

- Components of IHPA itself are in blue background
- [Kubernetes CR](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)s of IHPA are in green background
- Other Kubernetes resources that IHPA depends on are in yellow background
- Related external systems of IHPA are in white background

Full name of some abbreviations:

- IHPA: IntelligentHorizontalPodAutoscaler
- HPortrait: HorizontalPortrait
- CM: ConfigMap

## Component Overview

### Replica Controller

Replica Controller is the IHPA execution layer component. It is responsible for the control of the specific workload replica quantity and status. It supports the operations such as scaling, traffic cutoff, and activation of Pods by docking with different native and third-party components.

### IHPA Controller

IHPA Controller is the IHPA control plane component. It directly accepts IHPA configurations from users or external systems (including target workload, metrics, algorithms, change and stability configurations, etc.), issues profiling tasks and integrates profiling results, and then performs multi-level batched elasticity scaling based on the profiling results.

### HPortrait Controller

HPortrait Controller is the built-in horizontal elasticity scaling algorithm management component. It is responsible for running and managing the workflows of different elasticity scaling algorithms for different workloads, and converting their output results into standard profiling format. The specific algorithm sub-tasks are scheduled to be executed as separate Kubernetes Jobs or tasks on other big data/algorithm platforms. These sub-tasks obtain historical and real-time metric data from external monitoring systems for computation and generation of profiling results.

Specifically, the logic of some simple algorithms (such as reactive algorithms, etc.) is directly implemented in this component, without going through separate algorithm sub-tasks.

### Metrics Provider Server

Metrics Provider Server is the unified monitoring metrics query component. It shields the differences of the underlying monitoring system, providing a unified monitoring metrics query service for external running components (such as algorithm tasks, etc.).

The API it provides is similar to [Kubernetes Metrics API](https://github.com/kubernetes/metrics), but the difference is that it supports both real-time and historical metric queries.

### Agent (not included yet)

Agent is the agent component running on the nodes of the Kubernetes cluster. It is mainly responsible for executing operations that require interaction with the underlying operating system, such as activating and de-activating Pods.
