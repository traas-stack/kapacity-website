---
title: "Multi-Stage Gray Scaling"
weight: 4

---

## Before you begin

You need to have a Kubernetes cluster with Kapacity installed.

## Run sample workload

Download [nginx-statefulset.yaml](/examples/workload/nginx-statefulset.yaml) and run following command to run an NGINX workload:

```shell
kubectl apply -f nginx-statefulset.yaml
```

Check if the workload is running:

```shell
kubectl get po
```

```
NAME      READY   STATUS    RESTARTS   AGE
nginx-0   1/1     Running   0          5s
```

## Create IHPA with gray scale down strategy

Download [gray-strategy-sample.yaml](/examples/ihpa/gray-strategy-sample.yaml) which looks like this:

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: gray-strategy-sample
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  portraitProviders:
  - priority: 1
    static:
      replicas: 1
    type: Static
  - cron:
      crons:
      - name: cron-1
        replicas: 5
        start: 0 * * * *
        end: 10 * * * *
    priority: 2
    type: Cron
  behavior:
    scaleDown:
      grayStrategy:
        grayState: Cutoff         # GrayState is the desired state of pods that in gray stage.
        changeIntervalSeconds: 30 # ChangeIntervalSeconds is the interval time between each gray change.
        changePercent: 50         # ChangePercent is the percentage of the total change of replica numbers which is used to calculate the amount of pods to change in each gray change.
        observationSeconds: 60    # ObservationSeconds is the additional observation time after the gray change reaching 100%.
```

This IHPA contains below two portrait providers:

- A static portrait provider with priority 1 and a static replica number 1.
- A cron portrait provider with priority 2 and replica number 5 which takes effect the 0th minute to the 10th minute of every hour.

Note that the cron portrait provider's priority is **higher** than the static one, so the replica number it provided would override the one provided by the static portrait provider when it is effective.

Run following command to create the IHPA:

```shell
kubectl apply -f gray-strategy-sample.yaml
```

## Verify results

We can see that the cron portrait provider is taking effect at 0~10th minute of every hour, and the replicas of the workload are scaled up from 1 to 5:

```shell
kubectl get po -L 'kapacitystack.io/pod-state' -o wide
```

```
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          50m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          56s   10.1.5.68   docker-desktop   <none>           1/1
nginx-2   1/1     Running   0          54s   10.1.5.69   docker-desktop   <none>           1/1
nginx-3   1/1     Running   0          52s   10.1.5.70   docker-desktop   <none>           1/1
nginx-4   1/1     Running   0          50s   10.1.5.71   docker-desktop   <none>           1/1
```

The number of endpoints of the service of the workload is also changed to 5:

```shell
kubectl get ep nginx
```

```
NAME    ENDPOINTS                                            AGE
nginx   10.1.5.52:80,10.1.5.68:80,10.1.5.69:80 + 2 more...   3d3h
```

At the 10th minute, we can see that 2 pods change to Cutoff state and are removed from the endpoints of the service:

```shell
kubectl get po -L 'kapacitystack.io/pod-state' -o wide
```

```
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          51m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          63s   10.1.5.68   docker-desktop   <none>           1/1
nginx-2   1/1     Running   0          61s   10.1.5.69   docker-desktop   <none>           1/1
nginx-3   1/1     Running   0          59s   10.1.5.70   docker-desktop   <none>           0/1               Cutoff
nginx-4   1/1     Running   0          57s   10.1.5.71   docker-desktop   <none>           0/1               Cutoff
```

```shell
kubectl get ep nginx
```

```
NAME    ENDPOINTS                                AGE
nginx   10.1.5.52:80,10.1.5.68:80,10.1.5.69:80   3d3h
```

After waiting for another 30 seconds, we can see that the 4 pods change to Cutoff state and are removed from the endpoints of the service:

```shell
kubectl get po -L 'kapacitystack.io/pod-state' -o wide
```

```
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          51m   10.1.5.52   docker-desktop   <none>           1/1
nginx-1   1/1     Running   0          96s   10.1.5.68   docker-desktop   <none>           0/1               Cutoff
nginx-2   1/1     Running   0          94s   10.1.5.69   docker-desktop   <none>           0/1               Cutoff
nginx-3   1/1     Running   0          92s   10.1.5.70   docker-desktop   <none>           0/1               Cutoff
nginx-4   1/1     Running   0          90s   10.1.5.71   docker-desktop   <none>           0/1               Cutoff
```

```shell
kubectl get ep nginx
```

```
NAME    ENDPOINTS      AGE
nginx   10.1.5.52:80   3d3h
```

After waiting for another 1 minute, we can see that the replicas of the workload are finally scaled down to 1:

```shell
kubectl get po -L 'kapacitystack.io/pod-state' -o wide
```

```
NAME      READY   STATUS    RESTARTS   AGE    IP          NODE             NOMINATED NODE   READINESS GATES   POD-STATE
nginx-0   1/1     Running   0          52m    10.1.5.52   docker-desktop   <none>           1/1
```

You can also see the entire process of scaling by checking events of the IHPA:

```shell
kubectl describe ihpa gray-strategy-sample
```

```
...
Events:
  Type    Reason                Age    From             Message
  ----    ------                ----   ----             -------
  Normal  CreateReplicaProfile  3m53s  ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal  UpdateReplicaProfile  2m44s  ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 5, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  104s   ihpa_controller  update ReplicaProfile with onlineReplcas: 5 -> 3, cutoffReplicas: 0 -> 2, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  74s    ihpa_controller  update ReplicaProfile with onlineReplcas: 3 -> 1, cutoffReplicas: 2 -> 4, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  14s    ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 1, cutoffReplicas: 4 -> 0, standbyReplicas: 0 -> 0
```

## Cleanup

Run following command to cleanup all the resources:

```shell
kubectl delete -f gray-strategy-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```
