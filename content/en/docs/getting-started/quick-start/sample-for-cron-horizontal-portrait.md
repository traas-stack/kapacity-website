---
title: "Example for Cron Scaling"
linkTitle: "Example for Cron Scaling"
description: "Example for Cron Scaling"
weight: 15

---

## Pre-requisites

- Kubernetes cluster
- Kapacity installed on the cluster

## Steps

### 1.Deploying Test Service

Download [nginx-statefulset.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/nginx-statefulset.yaml)
file, and execute the following command to quickly deploy an Nginx service. You can also deploy your own service, which
can be modified later when deploying IHPA yaml
Contents of ScaleTargetRef.

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

### 2.Scaling with Cron Portraits

Download or copy the following configuration
to [cron-portrait-sample.yaml](https://github.com/traas-stack/kapacity/blob/main/examples/autoscaling/cron-portrait-sample.yaml)
file, and modify the spec.portraitProviders field and spec.scaleTargetRef field in the Yaml file as required.

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
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

Execute command to create IHPA CR

```bash
kubectl apply -f cron-portrait-sample.yaml
```

View IHPA CR Creation Results

```bash
kubectl get ihpa

NAME                   AGE
cron-portrait-sample   13s
```

### 3.Validation Results

By looking at IHPA's events you can see the following results:

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

## Clean-Up

You can execute the following command to clean up related resources

```bash
kubectl delete -f cron-portrait-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```

If you use your own service, please clean up separately