---
title: "Reactive Scaling"
weight: 2

---

## Before you begin

You need to have a Kubernetes cluster with Kapacity and Prometheus installed.

## Run sample workload

Download [nginx-statefulset.yaml](/examples/workload/nginx-statefulset.yaml) and run following command to run an NGINX workload.

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

## Create IHPA with dynamic reactive portrait provider

Download [dynamic-reactive-portrait-sample.yaml](/examples/ihpa/dynamic-reactive-portrait-sample.yaml) which looks like this:

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: dynamic-reactive-portrait-sample
spec:
  minReplicas: 1
  maxReplicas: 10
  portraitProviders:
  - type: Dynamic
    priority: 1
    dynamic:
      portraitType: Reactive
      metrics:
      - type: Resource
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

Run following command to create the IHPA:

```shell
kubectl apply -f dynamic-reactive-portrait-sample.yaml
```

## Increase the load

Run following command to get the ClusterIP and port of the NGINX service：

```shell
kubectl get svc nginx
```

```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
nginx        ClusterIP   10.111.21.74   <none>        80/TCP     13m
```

Start a different pod to act as a client which will send requests to the NGINX service infinitely with the service ip and port replaced by the value got in previous step:

```shell
# Run this in a separate terminal so that the load generation continues and you can carry on with the rest of the steps
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://<service-ip>:<service-port>; done"
```

After several minutes, you can see that the workload is scaled up by checking events of the IHPA:

```shell
kubectl describe ihpa dynamic-reactive-portrait-sample
```

```
...
Events:
  Type    Reason                Age    From             Message
  ----    ------                ----   ----             -------
  Normal  CreateReplicaProfile  6m58s  ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal  UpdateReplicaProfile  3m45s  ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 6, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

## Stop generating load

In the terminal where you created the Pod that runs a `busybox` image, terminate the load generation by typing `<Ctrl> + C`.

After several minutes, you can see that the workload is scaled down by checking events of the IHPA:

```shell
kubectl describe ihpa dynamic-reactive-portrait-sample
```

```
...
Events:
  Type    Reason                Age    From             Message
  ----    ------                ----   ----             -------
  Normal  CreateReplicaProfile  9m58s  ihpa_controller  create ReplicaProfile with onlineReplcas: 1, cutoffReplicas: 0, standbyReplicas: 0
  Normal  UpdateReplicaProfile  6m45s  ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 6, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  3m15s  ihpa_controller  update ReplicaProfile with onlineReplcas: 6 -> 4, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal  UpdateReplicaProfile  2m45s  ihpa_controller  update ReplicaProfile with onlineReplcas: 4 -> 1, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

## Cleanup

Run following command to cleanup all the resources:

```shell
kubectl delete -f dynamic-reactive-portrait-sample.yaml 
kubectl delete -f nginx-statefulset.yaml 
```
