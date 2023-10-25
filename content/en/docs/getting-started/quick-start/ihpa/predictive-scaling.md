---
title: "Predictive Scaling"
weight: 3

---

## Before you begin

You need to have a Kubernetes cluster with Kapacity, Prometheus installed.

## Install Ingress Controller

Install Ingress-Nginx Controller by referring to
the <a href="https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/" target="_blank">documentation</a>, and
modify the configuration for scraping the metrics of the Ingress-Nginx Controller, please verify the result on the
Prometheus Dashboard.

## Add Ingress custom metrics

Add ingress custom metric **nginx_ingress_controller_requests_rate** to metrics config, and then restart kapacity

```shell
kubectl edit cm kapacity-config -nkapacity-system
```

```yaml
kind: ConfigMap
apiVersion: v1
...
data:
  prometheus-metrics-config.yaml: |
    # Add nginx ingress metric
    rules:
    - seriesQuery: '{__name__=~"^nginx_ingress_controller_requests.*",namespace!=""}'
      seriesFilters: []
      resources:
        template: "<<.Resource>>"
      name:
        matches: ""
        as: "nginx_ingress_controller_requests_rate"
      metricsQuery: "round(sum(rate(<<.Series>>{<<.LabelMatchers>>}[3m])) by (<<.GroupBy>>), 0.001)"
    externalRules:
      ...
```

## Run sample workload

1. Download [nginx-statefulset.yaml](/examples/workload/nginx-statefulset.yaml) and run following command to run an
   NGINX workload.

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

Periodic flow diagram:

<img src="/images/en/workload-metrics.png" width="80%"/>

2. Download [nginx-ingress.yaml](/examples/workload/nginx-ingress.yaml) and run following command to create kubernetes
   ingress:

```shell
kubectl apply -f nginx-ingress.yaml
```

Check if the ingress is ok:

```shell
kubectl get ingress
```

```shell
NAME           CLASS   HOSTS                  ADDRESS     PORTS   AGE
nginx-server   nginx   nginx.example.com      localhost   80      2d
```

3. Run the script to simulate the business request and generate the dataset required for training.

Download [test-client.yaml](/examples/workload/test-client.yaml), Start a different Pod as a client and make requests to
the NGINX service periodically (1 cycle per hour) through the
script, simulating a periodic business load:

```shell
kubectl apply -f test-client.yaml
```

Due to the large dataset required for training, it is recommended to run the model for at least 24 hours before
training.

## Training model

Training the model according
to [Time series forecasting model training](/docs/user-guide/algorithm/predictive-model-training), Save the model files
to a separate directory.

```shell
-rw-r--r--@ 1 admin  staff   316K Oct 11 17:29 checkpoint.pth
-rw-r--r--@ 1 admin  staff   286B Oct 11 17:30 estimator_config.yaml
-rw-r--r--@ 1 admin  staff    29B Oct 11 17:30 item2id.json
```

Save the predictive model to configmap

```shell
kubectl -n <your-namespace> create configmap kapacity-tsf-model --from-file=<your-model-path>
```

**your-namespace** and **your-model-path** need to be replaced with their own namespaces and model directories.

## Create IHPA with dynamic predictive portrait provider

Download [dynamic-predictive-portrait-sample.yaml]() file, edit the yaml file, where **algorithm-image-id**
needs to be replaced with the real algorithm image ID, as follows:

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: dynamic-predictive-portrait-sample
spec:
  maxReplicas: 10
  minReplicas: 1
  portraitProviders:
  - dynamic:
      algorithm:
        externalJob:
          job:
            type: CronJob
            cronJob:
              template:
                spec:
                  schedule: "0/30 * * * *"
                  jobTemplate:
                    spec:
                      template:
                        spec:
                          containers:
                          - name: algorithm
                            image: <algorithm-image-id>
                            imagePullPolicy: IfNotPresent
                            args:
                            - --tsf-model-path=/opt/kapacity/timeseries/forecasting/model
                            - --tsf-freq=10min
                            - --re-history-len=48H
                            - --re-time-delta-hours=8
                            - --scaling-freq=10min
                            - --re-test-dataset-size-in-seconds=86400
                            volumeMounts:
                            - name: model
                              mountPath: /opt/kapacity/timeseries/forecasting/model
                              readOnly: true
                          volumes:
                          - name: model
                            configMap:
                              name: kapacity-tsf-model
                          restartPolicy: OnFailure
          resultSource:
            type: ConfigMap
        type: ExternalJob
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            averageUtilization: 30
            type: Utilization
      - type: External
        external:
          metric:
            name: ready_pods_count
          target:
            type: NA
      - type: Object
        name: qps
        object:
          describedObject:
            apiVersion: networking.k8s.io/v1
            kind: Ingress
            name: nginx-server
          metric:
            name: nginx_ingress_controller_requests_rate
          target:
            type: NA
      portraitType: Predictive
    priority: 10
    type: Dynamic
  scaleMode: Auto
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nginx
```

In the prediction algorithm, we specify the metric type and order:

- The first is a resource metric. Used to query the historical resource data (cpu and mem).
- The second is a replica metric. Used to query the historical replica data of ready pods.
- The third and others are traffic metrics. Used to query historical traffic data (rpc, msg, pv, etc.).

Use **kubectl** command to create IHPA CR

```shell
kubectl apply -f dynamic-predictive-portrait-sample.yaml
```

## Verify results

1. The IHPA portrait provider creates a CronJob and starts the predictive algorithm job.

```shell
kubectl get cronjob
```

```shell
NAME                                 SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
dynamic-predictive-portrait-sample   0/30 * * * *   False     1        26m             2d1h
```

```shell
kubectl get job
```

```shell
NAME                                          COMPLETIONS   DURATION   AGE
dynamic-predictive-portrait-sample-28286564   1/1           16s        28m
```

2. When the predictive algorithm job is completed, the predictive result is written back to configmap

```shell
kubectl get cm dynamic-predictive-portrait-sample-result -oyaml
```

You can see that the predictive result contains a time series of three points.

```yaml
apiVersion: v1
data:
  expireTime: "2023-10-13T13:00:00.00Z"
  timeSeries: '{"1697200200": 3, "1697200800": 2, "1697201400": 1}'
  type: TimeSeries
kind: ConfigMap
metadata:
  name: dynamic-predictive-portrait-sample-result
  ownerReferences:
  - apiVersion: autoscaling.kapacitystack.io/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: HorizontalPortrait
    name: dynamic-predictive-portrait-sample
    uid: 8e0ea542-6e7d-4529-83dc-efcfdec8ec6e
```

3. Perform replica changes based on the predictive result.

```shell
 kubectl describe ihpa dynamic-predictive-portrait-sample
```

```shell
Events:
  Type     Reason                Age                 From             Message
  ----     ------                ----                ----             -------
  Warning  NoValidPortraitValue  29m (x10 over 85m)  ihpa_controller  no valid portrait value for now
  Normal   UpdateReplicaProfile  25m                 ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 3, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  15m                 ihpa_controller  update ReplicaProfile with onlineReplcas: 3 -> 2, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  5m9s                ihpa_controller  update ReplicaProfile with onlineReplcas: 2 -> 1, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0

```

## Cleanup

Run following command to cleanup all the resources:

```shell
kubectl delete -f dynamic-predictive-portrait-sample.yaml
kubectl delete -f test-client.yaml
kubectl delete -f nginx-ingress.yaml
kubectl delete -f nginx-statefulset.yaml 
```
