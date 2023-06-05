---
title: "Gray Scaling"
linkTitle: "Gray Scaling"
description: "Gray Scaling"
weight: 17

---

## Pre-requisites

- Kubernetes cluster
- Kapacity installed on the cluster

## Steps

### 1.Deploying Test Service

Download the [nginx-statefulset.yaml](/examples/nginx-statefulset.yaml) file and execute the following command to
quickly deploy an Nginx service. You can also deploy your own service, just modify the content of ***scaleTargetRef***
when deploying IHPA yaml later.

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

### 2.Use Multi-Stage Gray Scaling

Download or copy the following configuration to [gray-strategy-sample.yaml](/examples/ihpa/gray-strategy-sample.yaml)
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
        changeIntervalSeconds: 30
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
kubectl get po -L 'kapacitystack.io/pod-state' -owide

NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          50m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          56s   10.1.5.68   docker-desktop   <none>           1/1
nginx-2   1/1     Running   0          54s   10.1.5.69   docker-desktop   <none>           1/1
nginx-3   1/1     Running   0          52s   10.1.5.70   docker-desktop   <none>           1/1
nginx-4   1/1     Running   0          50s   10.1.5.71   docker-desktop   <none>           1/1
```

The number of service EndPoints is also changed to 5

```bash
kubectl get ep nginx

NAME    ENDPOINTS                                            AGE
nginx   10.1.5.52:80,10.1.5.68:80,10.1.5.69:80 + 2 more...   3d3h
```

In the 10th minute, you can see that both pods have become Cutoff and have been removed from the EndPoint list of the
service

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

After waiting for another 30 seconds, you can see that the 4 pods have changed to the Cutoff state and have been removed
from the EndPoint list of the service.

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

After observing another 1m, you can see that the pod has been scaled down to 1

```bash
kubectl get po -L 'kapacitystack.io/pod-state' -owide

NAME      READY   STATUS    RESTARTS   AGE    IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          52m    10.1.5.52   docker-desktop   <none>           1/1
```

You can also see the entire process of downsizing through IHPA Events

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

## Clean-Up

You can clean up the sample related resources by executing the following command

```bash
kubectl delete -f gray-strategy-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```