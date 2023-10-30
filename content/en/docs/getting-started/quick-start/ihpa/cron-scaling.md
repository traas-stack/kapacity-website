---
title: "Cron Scaling"
weight: 1

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

## Create IHPA with cron portrait provider

Download [cron-portrait-sample.yaml](/examples/ihpa/cron-portrait-sample.yaml) which looks like this:

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: cron-portrait-sample
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  portraitProviders:
  - type: Cron
    priority: 1
    cron:
      crons:
      - name: cron-1
        start: 0 * * * *
        end: 10 * * * *
        replicas: 1
      - name: cron-2
        start: 10 * * * *
        end: 20 * * * *
        replicas: 2
      - name: cron-3
        start: 20 * * * *
        end: 30 * * * *
        replicas: 3
      - name: cron-4
        start: 30 * * * *
        end: 40 * * * *
        replicas: 4
      - name: cron-5
        start: 40 * * * *
        end: 50 * * * *
        replicas: 5
```

Run following command to create the IHPA:

```shell
kubectl apply -f cron-portrait-sample.yaml
```

## Verify results

You can see that the replica number of the workload is changing dynamically accroding to our configration by checking the events of the IHPA:

```shell
kubectl describe ihpa cron-portrait-sample
```

```
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

You can also verify it by directly watching the replica number of the workload.

{{% alert title="Note" %}}
You can see a `NoValidPortraitValue` event because the cron expressions configured do not cover this period of time, and the replica number of the workload will remain unchanged at this time.
{{% /alert %}}

## Cleanup

Run following command to cleanup all the resources:

```shell
kubectl delete -f cron-portrait-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```
