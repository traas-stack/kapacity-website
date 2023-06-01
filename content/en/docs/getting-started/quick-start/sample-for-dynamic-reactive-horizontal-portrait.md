---
title: "Example for Reactive Dynamic Scaling"
linkTitle: "Example for Reactive Dynamic Scaling"
description: "Example for Reactive Dynamic Scaling"
weight: 16

---

## Pre-requisites

- Kubernetes cluster
- Kapacity installed on the cluster
- Promethues

## Steps

### 1.Deploying Test Service

Download [nginx-statefulset.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/nginx-statefulset.yaml)
file, and execute the following command to quickly deploy an Nginx service. You can also deploy your own service, just
modify it when deploying IHPA yaml later
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

### 2.Scaling with Reactive Dynamic Portrait

Download or copy the following configuration
to [dynamic-portrait-reactive-sample.yaml](https://raw.githubusercontent.com/traas-stack/kapacity/main/examples/autoscaling/dynamic-portrait-reactive-sample.yaml)
document.

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: dynamic-portrait-reactive-sample
spec:
  paused: false
  minReplicas: 0
  maxReplicas: 10
  portraitProviders:
    - type: Dynamic
      priority: 0
      dynamic:
        portraitType: Reactive
        metrics:
          - name: metrics-scale-out
            type: Resource
            resource:
              name: cpu
              target:
                type: Utilization
                averageUtilization: 30
        algorithm:
          type: KubeHPA
  scaleTargetRef:
    kind: StatefulSet
    name: nginx
    apiVersion: apps/v1
```

Execute command to create IHPA CR

```bash
kubectl apply -f dynamic-portrait-reactive-sample.yaml
```

Check if the IHPA CR is successfully created

```bash
kubectl get ihpa

NAME                               AGE
dynamic-portrait-reactive-sample   29m
```

### 3.Pressure Test

Obtain the port number exposed by the Nginx service (32591 in the example)

```bash
kubectl get service nginx

NAME         TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
nginx        NodePort   10.111.21.74   <none>        80:32591/TCP    13m
```

Use a pressure test tool or a pressure test script to perform a pressure test on the service, such as the Apache
Benchmark (ab) for stress testing. Note that the ip and port in the url are replaced with your own.

```bash
ab -n 10000 -c 100 http://<your-node-ip>:<your-port>/index
```

### 4.Validation Results

After waiting for 3-10 minutes, you can see the specific elastic results through IHPA Events

```bash
kubectl describe ihpa dynamic-portrait-reactive-sample

Events:
  Type     Reason                             Age                From             Message
  ----     ------                             ----               ----             -------
  Warning  FailedUpdateAndFetchPortraitValue  27m                ihpa_controller  failed to update and fetch portrait value: failed to fetch portrait value: failed to get HorizontalPortrait "default/dynamic-portrait-reactive-sample-reactive": HorizontalPortrait.autoscaling.kapacity.traas.io "dynamic-portrait-reactive-sample-reactive" not found
  Warning  NoValidPortraitValue               10m (x9 over 27m)  ihpa_controller  no valid portrait value for now
  Normal   CreateReplicaProfile               9m58s              ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal   UpdateReplicaProfile               6m45s              ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 6, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile               3m15s              ihpa_controller  update ReplicaProfile with onlineReplcas: 6 -> 4, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile               2m45s              ihpa_controller  update ReplicaProfile with onlineReplcas: 4 -> 1, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

## Clean-Up

You can clean up sample-related resources by executing the following command

```bash
kubectl delete -f dynamic-portrait-reactive-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```