---
title: "Example for Multi-Stage Grayscale Scaling"
linkTitle: "Example for Multi-Stage Grayscale Scaling"
description: "Example for Multi-Stage Grayscale Scaling"
weight: 17

---

## Pre-requisites

- Kubernetes cluster
- Kapacity installed on the cluster

## Steps

### 1.Deploying Test Service

Download
the [nginx-statefulset.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/nginx-statefulset.yaml)
file and execute the following command to quickly deploy an Nginx service. You can also deploy your own service, just
modify the content of ***scaleTargetRef*** when deploying IHPA yaml later.

```bash
cd <your-file-directory>
kubectl apply -f nginx-statefulset.yaml
```

Verify service deployment results

```bash
kubectl get po

NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          5s
```

### 2.Use Multi-Stage Grayscale Scaling

Download or copy the following configuration
to [gray-strategy-sample.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/autoscaling/gray-strategy-sample.yaml)
document. This configuration contains 2 portraits:

- Static Portrait：
    - The priority is 1 and the number of replicas 1
- Cron Portrait
    - The priority is 2, the number of replicas is 5, and it takes effect from the 0th minute to the 10th minute of each
      hour

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
        grayState: Cutoff
        changeIntervalSeconds: 10
        changePercent: 50
        observationSeconds: 60
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

Execute the following command to generate IHPA CR

```bash
kubectl apply -f gray-strategy-sample.yaml
```

If the current time is 0-9 minutes, you can see that the scheduled portrait takes effect (high priority), and the number
of pods has expanded from 1 to 5

```bash
kubectl get po

NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          3m4s
nginx-1   1/1     Running   0          52s
nginx-2   1/1     Running   0          45s
nginx-3   1/1     Running   0          37s
nginx-4   1/1     Running   0          30s
```

In the 10th minute, you can see that the 2 pods have changed to the Cutoff state

```bash
kubectl get po -L 'kapacitystack.io/pod-state'

NAME      READY   STATUS    RESTARTS   AGE     POD-STATE
nginx-0   1/1     Running   0          3m14s
nginx-1   1/1     Running   0          62s
nginx-2   1/1     Running   0          55s
nginx-3   1/1     Running   0          47s     Cutoff
nginx-4   1/1     Running   0          40s     Cutoff
```

After waiting for another 10 seconds, you can see that 4 pods have changed to Cutoff status

```bash
kubectl get po -L 'kapacitystack.io/pod-state'

NAME      READY   STATUS    RESTARTS   AGE     POD-STATE
nginx-0   1/1     Running   0          3m22s
nginx-1   1/1     Running   0          70s     Cutoff
nginx-2   1/1     Running   0          63s     Cutoff
nginx-3   1/1     Running   0          55s     Cutoff
nginx-4   1/1     Running   0          48s     Cutoff
```

After observing another 1m, you can see that the pod has been scaled down to 1

```bash
kubectl get po -L 'kapacitystack.io/pod-state'

NAME      READY   STATUS    RESTARTS   AGE     POD-STATE
nginx-0   1/1     Running   0          4m26s
```

You can also see the entire process of downsizing through IHPA Events

```bash
kubectl describe ihpa gray-strategy-sample

Events:
  Type    Reason                Age    From             Message
  ----    ------                ----   ----             -------
  Normal  CreateReplicaProfile  4m35s  ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal  UpdateReplicaProfile  2m23s  ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 5, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  83s    ihpa_controller  update ReplicaProfile with onlineReplcas: 5 -> 3, cutoffReplicas: 0 -> 2, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  73s    ihpa_controller  update ReplicaProfile with onlineReplcas: 3 -> 1, cutoffReplicas: 2 -> 4, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  13s    ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 1, cutoffReplicas: 4 -> 0, standbyReplicas: 0 -> 0
```

## Clean-Up

You can clean up the sample related resources by executing the following command

```bash
kubectl delete -f gray-strategy-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```