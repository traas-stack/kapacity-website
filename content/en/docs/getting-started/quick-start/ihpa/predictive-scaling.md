---
title: "Predictive Scaling"
weight: 3

---

{{% pageinfo color="primary" %}}
Note: Most contents of this page are translated by AI from [Chinese](/zh-cn/docs/getting-started/quick-start/ihpa/predictive-scaling/). It would be very appreciated if you could help us improve by editing this page (click the corresponding link on the right side).
{{% /pageinfo %}}

## Before you begin

You need to have a Kubernetes cluster with Kapacity and Prometheus installed.

Make sure your Kubernetes cluster has a working DNS (like [CoreDNS](https://coredns.io/)) to resolve Service domain names. If not, you need to adjust the configuration of Kapacity as follows:

Use the following command to view the ClusterIP and port of the Kapacity gRPC Server:

```shell
kubectl get svc -n kapacity-system kapacity-grpc-service
```

```
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kapacity-grpc-service   ClusterIP   192.168.38.172   <none>        9090/TCP   5m
```

Use the following command to update the configuration of Kapacity, where the parameters related to the Kapacity gRPC Server address are the values viewed in the previous step:

```shell
helm upgrade \
  kapacity-manager kapacity/kapacity-manager \
  --namespace kapacity-system \
  --reuse-values \
  --set algorithmJob.defaultMetricsServerAddr=<kapacity-grpc-server-clusterip>:<kapacity-grpc-server-port> 
```

## Install and configure Ingress NGINX Controller

Kapacity IHPA's predictive scaling uses the "[Traffic-Driven Replicas Prediction](/docs/user-guide/ihpa/concepts/predictive-scaling-principles/#traffic-driven-replicas-prediction)" algorithm, so we need at least one traffic metric to use predictive scaling. Here we use Ingress NGINX as an example of workload ingress traffic.

If your Kubernetes cluster does not yet have Ingress NGINX Controller, please refer to the [official documentation](https://kubernetes.github.io/ingress-nginx/deploy/) for installation.

After the installation is complete, follow [this document](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/) to configure to ensure that Prometheus can collect the metrics of Ingress NGINX.

## Configure Kapacity to recognize Ingress NGINX metrics

Use the following command to add the metrics of Ingress NGINX in the custom Prometheus metrics configuration of Kapacity:

```shell
kubectl edit cm -n kapacity-system kapacity-config
```

```yaml
apiVersion: v1
data:
  prometheus-metrics-config.yaml: |
    resourceRules:
      ...
    # add Ingress NGINX metrics in rules
    rules:
    - seriesQuery: '{__name__="nginx_ingress_controller_requests"}'
      metricsQuery: round(sum(irate(<<.Series>>{<<.LabelMatchers>>}[3m])) by (<<.GroupBy>>), 0.001)
      name:
        as: nginx_ingress_controller_requests_rate
      resources:
        template: <<.Resource>>
        # note: uncomment the overrides field below if your Prometheus is installed with Prometheus Operator
        # overrides:
        #   exported_namespace:
        #     resource: namespace
    externalRules:
      ...
kind: ConfigMap
...
```

As you can see, this configuration is fully compatible with the [Prometheus Adapter configuration](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md), more background information can be found in [this user guide](/docs/user-guide/custom-metrics).

Then, use the following command to restart Kapacity Manager to load the latest configuration:

```shell
kubectl rollout restart -n kapacity-system deploy/kapacity-manager
```

{{% alert title="Notice" %}}
Kapacity Manager requires some time to sync the custom metric configuration from Prometheus after startup. If there are many metrics within Prometheus, it might take a considerable amount of time (usually several minutes). You can determine whether the synchronization is complete by checking the standard output logs of the Kapacity Manager Pod for the message `metrics relisted successfully`. Please wait for Kapacity Manager to finish synchronizing the custom metrics before proceeding with subsequent steps.
{{% /alert %}}

## Run sample workload

1. Download [nginx-statefulset.yaml](/examples/workload/nginx-statefulset.yaml) and run following command to run an NGINX workload:

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

2. Download [nginx-ingress.yaml](/examples/workload/nginx-ingress.yaml) and run following command to create an Ingress for the NGINX workload:

```shell
kubectl apply -f nginx-ingress.yaml
```

Verify that Ingress was created successfully and record the ADDRESS of Ingress:

```shell
kubectl get ing
```

```
NAME           CLASS   HOSTS               ADDRESS           PORTS   AGE
nginx-server   nginx   nginx.example.com   139.224.120.211   80      2d
```

3. Download [periodic-client.yaml](/examples/workload/periodic-client.yaml)，replace `<nginx-ingress-address>` with the ADDRESS of the Ingress recorded in the previous step, and then execute the following command to create a client Pod that sends requests to the NGINX service on a periodic basis (with 1 hour as 1 cycle):

```shell
kubectl apply -f periodic-client.yaml
```

It will generate periodic traffic as shown in the following figure:

<img src="/images/en/periodic-traffic-metric.png" width="1000"/>

Since the algorithm needs a certain amount of data for learning, it is recommended to run for at least 24 hours before proceeding to the next step.

## Train time series forecasting model

Please refer to the [user guide](/docs/user-guide/algorithm/train-tsf-model/) to use [this configuration](/examples/algorithm/tsf-model-train-config.yaml) to complete the training of the time series prediction model, and then execute the following command to save the model and its accessory files as a ConfigMap for subsequent algorithm tasks to use, replace `<model-save-path>` with the actual model saving directory path:

```shell
kubectl create cm -n kapacity-system example-model --from-file=<model-save-path>
```

{{% alert title="Note" %}}
In actual use, we may get a larger model, in this case, we recommend storing model files on [persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) rather than a ConfigMap.
{{% /alert %}}

## Create IHPA with dynamic predictive portrait provider

Download [dynamic-predictive-portrait-sample.yaml](/examples/ihpa/dynamic-predictive-portrait-sample.yaml) which looks like this:

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: IntelligentHorizontalPodAutoscaler
metadata:
  name: predictive-sample
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  portraitProviders:
  - type: Dynamic
    priority: 1
    dynamic:
      portraitType: Predictive
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: AverageValue
            averageValue: 1m
      - type: Pods
        pods:
          metric:
            name: kube_pod_status_ready
          target:
            type: NA
      - name: qps
        type: Object
        object:
          describedObject:
            apiVersion: networking.k8s.io/v1
            kind: Ingress
            name: nginx-server
          metric:
            name: nginx_ingress_controller_requests_rate
          target:
            type: NA
      algorithm:
        type: ExternalJob
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
                            args:
                            - --tsf-model-path=/opt/kapacity/timeseries/forecasting/model
                            - --re-history-len=24H
                            - --re-time-delta-hours=8
                            - --re-test-dataset-size-in-seconds=3600
                            - --scaling-freq=10min
                            volumeMounts:
                            - name: model
                              mountPath: /opt/kapacity/timeseries/forecasting/model
                              readOnly: true
                          volumes:
                          - name: model
                            configMap:
                              name: example-model
                          restartPolicy: OnFailure
          resultSource:
            type: ConfigMap
```

Please replace the value of the algorithm parameter `--re-time-delta-hours` with the UTC offset value of your time zone, such as 8 for UTC+8 time zone, -7 for UTC-7 time zone.

Let's look at the metrics first. In the "Traffic-driven replica prediction" algorithm, we need multiple types of metrics to jointly drive the algorithm, so we have agreed on the following metric configuration specification:

- The first metric should be configured as the target resource metric of this workload, so the type can only be `Resource` or `ContainerResource`. It specifies the target resource level we expect IHPA to help us maintain.
- The second metric should be configured as the online replica count metric of this workload. The algorithm will use this metric to query the historical Ready Pod (i.e., the Pod carrying traffic) count of this workload. The type of this metric can only be `Pods`. It will be aggregated on the workload dimension by regular matching of the Pod name for the query. Kapacity defaults to the `kube_pod_status_ready` metric based on kube-state-metrics for direct use. Note that since this metric is only used for historical query, we do not need to specify a target value for it. Therefore, we write a placeholder `NA` for its `target` `type`.
- The third and subsequent metrics should be configured as traffic metrics (such as QPS, RPC, etc.) that are positively correlated with the target resource metric of this workload. The algorithm will perform time series prediction on these metrics, and then based on the historical resource level and replica count, it will give a predicted replica count that can meet the target resource level in the future. These metrics can be of any type other than `Resource` and `ContainerResource`, but **note that you must set the same `name` for these metrics as set during training**. Similarly, these metrics are also only used for historical queries, so you do not need to set target values.

Then let's look at the algorithm parameters. Here is a brief explanation of the functions of a few key parameters. More information can be referred to the flags description of the algorithm script itself:

- `--re-history-len`: This parameter specifies the historical length of the replica recommendation algorithm learning, it is generally recommended to cover at least two behavioral cycles of the application.
- `--re-time-delta-hours`: This parameter specifies the UTC offset value of the time zone where the application is located, the replica number recommendation algorithm needs to sense the time zone information to learn the time features.
- `--re-test-dataset-size-in-seconds`: This parameter specifies the test set size of the replica recommendation algorithm learning, default is one day (86400), only when the historical length is less than one day do you need to shorten it, such as setting it as one hour (3600) in this example.
- `--scaling-freq`: This parameter specifies the accuracy of the final replica number prediction result of the algorithm, that is, the maximum frequency of actual scaling, so it cannot be shorter than the original prediction accuracy of the timeseries forecasting algorithm (the `freq` parameter used in timeseries forecasting model training). The algorithm will resample the original prediction result by the maximum value according to the given accuracy and output it. For example, if this parameter is set to 1 hour, the algorithm will finally give the maximum number of replicas needed by this workload every hour, and finally this workload will scale up and down at most once an hour.

Execute the following command to create this IHPA:

```shell
kubectl apply -f dynamic-predictive-portrait-sample.yaml
```

## Verify results

1. Verify that IHPA automatically created a CronJob to run the algorithm task, and the last task ran successfully:

```shell
kubectl get cj -n kapacity-system
```

```
NAME                                   SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
default-predictive-sample-predictive   0/30 * * * *   False     1        26m             2d1h
```

```shell
kubectl get job -n kapacity-system
```

```
NAME                                            COMPLETIONS   DURATION   AGE
default-predictive-sample-predictive-28286564   1/1           16s        28m
```

2. Verify that the algorithm result was successfully written to the predictive horizontal portrait of IHPA:

```shell
kubectl get hp predictive-sample-predictive -o yaml
```

```yaml
apiVersion: autoscaling.kapacitystack.io/v1alpha1
kind: HorizontalPortrait
metadata:
  name: predictive-sample-predictive
  namespace: default
  ...
spec:
  ...
status:
  conditions:
  - lastTransitionTime: "2023-10-25T11:00:00Z"
    message: portrait has been successfully generated
    observedGeneration: 1
    reason: SucceededGeneratePortrait
    status: "True"
    type: PortraitGenerated
  portraitData:
    expireTime: "2023-10-25T11:30:00Z"
    timeSeries:
      timeSeries:
      - replicas: 4
        timestamp: 1698231600
      - replicas: 3
        timestamp: 1698232200
      - replicas: 2
        timestamp: 1698232800
    type: TimeSeries
```

3. Verify that IHPA has adjusted the replica count of the workload according to the prediction result of the algorithm:

```shell
kubectl describe ihpa predictive-sample
```

```
...
Events:
  Type     Reason                Age                 From             Message
  ----     ------                ----                ----             -------
  Warning  NoValidPortraitValue  29m (x10 over 85m)  ihpa_controller  no valid portrait value for now
  Normal   UpdateReplicaProfile  25m                 ihpa_controller  update ReplicaProfile with onlineReplcas: 1 -> 4, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  15m                 ihpa_controller  update ReplicaProfile with onlineReplcas: 4 -> 3, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
  Normal   UpdateReplicaProfile  5m9s                ihpa_controller  update ReplicaProfile with onlineReplcas: 3 -> 2, cutoffReplicas: 0 -> 0, standbyReplicas: 0 -> 0
```

## Cleanup

Run following command to cleanup all the resources:

```shell
kubectl delete -f dynamic-predictive-portrait-sample.yaml
kubectl delete -f periodic-client.yaml
kubectl delete -f nginx-ingress.yaml
kubectl delete -f nginx-statefulset.yaml
```
