---
title: "Train Time Series Forecasting Model"
weight: 1
---

{{% pageinfo color="primary" %}}
Note: Most contents of this page are translated by AI from [Chinese](/zh-cn/docs/user-guide/algorithm/train-tsf-model/). It would be very appreciated if you could help us improve by editing this page (click the corresponding link on the right side).
{{% /pageinfo %}}

## Before you begin

This document will guide you on how to train the time series prediction deep learning model used by Kapacity.

Before getting started, please make sure that [Conda](https://docs.conda.io/en/latest/) has been installed in your environment.

## Install dependencies

Execute the following command to download the code of the Kapacity algorithm version you are using, install algorithm dependencies, and activate the algorithm runtime environment:

```shell
git clone --depth 1 -b algorithm-<your-kapacity-algorithm-version> https://github.com/traas-stack/kapacity.git
cd kapacity/algorithm
conda env create -f environment.yml
conda activate kapacity
```

## Prepare configuration file

The training script will read parameters related to dataset fetching and model training from an external configuration file, so we need to prepare this configuration file in advance. You can download this example configuration file [tsf-model-train-config.yaml](/examples/algorithm/tsf-model-train-config.yaml) and modify its content as needed. The content of this file is as follows:

```yaml
targets:
- workloadNamespace: default
  workloadRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: nginx
  historyLength: 24H
  metrics:
  - name: qps
    type: Object
    object:
      metric:
        name: nginx_ingress_controller_requests_rate
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: nginx-server
freq: 10min
predictionLength: 3
contextLength: 7
hyperParams:
  learningRate: 0.001
  epochs: 100
  batchSize: 32
```

Here is an explanation of each field:

- `targets`: List of workloads to be trained and their metric information (the algorithm supports training multiple metrics of multiple workloads in one model, but it also makes the model larger), if you wish to manually prepare the dataset instead of automatically fetching it by the algorithm, you can leave this field blank.
  - `workloadNamespace`: The namespace where the workload to be trained resides.
  - `workloadRef`: Reference identification of the workload object to be trained.
  - `historyLength`: The length of metrics history to fetch as a dataset, supporting `min` (minutes), `H` (hours), `D` (days) three time units. Generally, it is recommended to cover at least two complete cycles of periodic metrics.
  - `metrics`: List of metrics to be trained for this workload, with the same format as the metrics field of IHPA. Note that you must set a different `name` for each metric to distinguish different metrics under the same workload in the same model.
- `freq`: The precision of the model, that is, the unit of the parameters `predictionLength` and `contextLength` below. The currently supported values are `1min`, `10min`, `1H`, `1D`. Note that this parameter does not affect the model size.
- `predictionLength`: The number of prediction points in the model, and the final prediction length is `predictionLength * freq`. The larger this parameter is set, the larger the model will be. Generally, it is not recommended to set it too large, because the further the prediction time point, the lower the accuracy of the prediction.
- `contextLength`: The number of historical points referenced by the model during prediction (inference), and the final reference history length is `contextLength * freq`. The larger this parameter is set, the larger the model will be.
- `hyperParams`: Hyperparameters of the deep learning model, generally do not need to be adjusted.

## Train model

Execute the following command to train the model, note to replace the related parameters with the actual values:

```shell
python kapacity/timeseries/forecasting/train.py \
  --config-file=<your-config-file> \
  --model-save-path=<your-model-save-path> \
  --dataset-file=<your-datset-file> \
  --metrics-server-addr=<your-metrics-server-addr> \
  --dataloader-num-workers=<your-dataloader-num-workers>
```

Here is an explanation of each parameter:

- `config-file`: The address of the configuration file prepared in the previous step.
- `model-save-path`: The directory address where the model is to be saved.
- `dataset-file`: The address of the dataset file prepared manually. And `metrics-server-addr` parameter choose one to fill in.
- `metrics-server-addr`: The address of the metric server for automatically fetching the dataset, that is, the address of the Kapacity gRPC service that can be accessed. And `dataset-file` parameter choose one to fill in.
- `dataloader-num-workers`: The number of subprocesses for loading the dataset, generally recommended to be set to the number of CPUs on the machine, if set to `0`, only the main process will be used for loading.

{{% alert title="Tips on how to access the Kapacity gRPC service" %}}

By default, Kapacity's gRPC service will be exposed as a ClusterIP Service, you can use the following command to view its ClusterIP and port:

```shell
kubectl get svc -n kapacity-system kapacity-grpc-service
```

```
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kapacity-grpc-service   ClusterIP   192.168.38.172   <none>        9090/TCP   5m
```

If you are running the training script in the cluster, you can directly use this ClusterIP and port as `metrics-server-addr`.

If you are not running the training script in the cluster, you can use the following command to access this Service in the cluster from your local machine through kubectl's port forwarding function, note to replace `<local-port>` with a spare port on your local machine:

```shell
kubectl port-forward -n kapacity-system svc/kapacity-grpc-service <local-port>:9090
```

Then, you can use `localhost:<local-port>` as `metrics-server-addr`.

{{% /alert %}}

After execution, you will see training logs similar to the following, please wait until the training is completed (the command exits normally):

```
2023-10-18 20:05:07,757 - INFO: Epoch: 1 cost time: 55.25944399833679
2023-10-18 20:05:07,774 - INFO: Epoch: 1, Steps: 6 | Train Loss: 188888896.6564227
Validation loss decreased (inf --> 188888896.656423).  Saving model ...
2023-10-18 20:05:51,157 - INFO: Epoch: 2 cost time: 43.38192820549011
2023-10-18 20:05:51,158 - INFO: Epoch: 2, Steps: 6 | Train Loss: 212027786.7585510
EarlyStopping counter: 1 out of 15
2023-10-18 20:06:30,055 - INFO: Epoch: 3 cost time: 38.89493203163147
2023-10-18 20:06:30,060 - INFO: Epoch: 3, Steps: 6 | Train Loss: 226666666.7703293
```

After the training is completed, you can see the trained model and its associated files in the `model-save-path` directory:

```
-rw-r--r-- 1 admin  staff   316K Oct 18 20:18 checkpoint.pth
-rw-r--r-- 1 admin  staff   287B Oct 18 20:04 estimator_config.yaml
-rw-r--r-- 1 admin  staff    29B Oct 18 20:04 item2id.json
```
