---
title: "Time series forecasting model training"
weight: 2
---

## Prerequisites

- kapacity 0.2+
- anaconda 23.5+
- python 3.9+

## Dependencies

Enter the kapacity project directory, run following commands to install the dependent packages, and activate the
environment.

```shell
cd kapacity
conda env create -f algorithm/environment.yml
conda activate kapacity
```

## Configs

The execution of training script depend on dataset and model parameters from the config file, which needs to be set by
users in advance. You can download the config file [train-config.yaml](/examples/algorithm/train-config.yaml) to the
local directory and edit related
parameters.

```yaml
targets:
- workloadNamespace: kapacity-test
  workloadName: nginx
  historyLength: 48H
  metrics:
  - type: Object
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

in the configs：

- targets: workload information list to be trained
    * workloadNamespace: Namespace of workload be trained
    * workloadName: Name of workload to be trained
    * historyLength: The length of time to collect the training dataset, The time unit can be "min", "H", or "D",
      indicating "minute", "hour", or "day".
    * metrics: Indicates how to collect dataset
- freq: The time frequency of the model. The frequency currently supported is “1H”，“10min”，“1min”，“D”.
- predictionLength: Predictive time series length
- contextLength：The length of the context time when predicting
- hyperParams：Hyper parameters
    * learningRate：Learning rate during training
    * epochs：The number of rounds of training
    * batchSize：One batch training data size

## Training model

### Parameters

- dataset-file: path of training dataset file, if set, will load dataset from this file instead of fetching from metrics
  server
- config-file：path of training config file
- metrics-server-addr：address of the gRPC metrics provider server
- model-save-path：dir path where the model and its related files would be saved in

### Run

If you fetch metrics from kapacity in an external k8s cluster, you can use port-forward function of kubectl to allow
local access to the remote grpc service (note local-port is replaced with the real port).

```shell
kubectl port-forward -n kapacity-system  svc/kapacity-grpc-service <local-port>:9090
```

Run the python script for model training, taking care to replace the script parameter values to the correct values.

```shell
python kapacity/timeseries/forecasting/train.py \
   --config-file=<config-file> \
   --metrics-server-addr=<metrics-server-addr> \
   --model-save-path=<model-save-path>
```

After running the script, you will see the following training log and wait for the script to complete training (training
time depends on the dataset size and parameters).

```shell
2023-10-18 20:05:07,757 - INFO: Epoch: 1 cost time: 55.25944399833679
2023-10-18 20:05:07,774 - INFO: Epoch: 1, Steps: 6 | Train Loss: 188888896.6564227
Validation loss decreased (inf --> 188888896.656423).  Saving model ...
2023-10-18 20:05:51,157 - INFO: Epoch: 2 cost time: 43.38192820549011
2023-10-18 20:05:51,158 - INFO: Epoch: 2, Steps: 6 | Train Loss: 212027786.7585510
EarlyStopping counter: 1 out of 15
2023-10-18 20:06:30,055 - INFO: Epoch: 3 cost time: 38.89493203163147
2023-10-18 20:06:30,060 - INFO: Epoch: 3, Steps: 6 | Train Loss: 226666666.7703293
```

After the training is completed, the following model files will be generated in the **model-save-path** directory.

```shell
-rw-r--r--@ 1 admin  staff   316K Oct 18 20:18 checkpoint.pth
-rw-r--r--@ 1 admin  staff   287B Oct 18 20:04 estimator_config.yaml
-rw-r--r--@ 1 admin  staff    29B Oct 18 20:04 item2id.json
```
