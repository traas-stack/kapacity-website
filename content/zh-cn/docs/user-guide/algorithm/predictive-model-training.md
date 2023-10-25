---
title: "时序预测模型训练"
weight: 2
---

## 环境准备

- kapacity 0.2+
- anaconda 23.5+
- python 3.9+

## 安装依赖

进入 kapacity 项目目录，执行以下命令安装脚本依赖的包，并激活环境

```shell
cd kapacity
conda env create -f algorithm/environment.yml
conda activate kapacity
```

## 配置文件

训练脚本执行时依赖从配置文件获取数据集和模型的相关参数，需要用户提前设置好，可以下载配置文件 [train-config.yaml](/examples/algorithm/train-config.yaml)
到本地，编辑相关参数。

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

示例中：

- targets：待训练的 workload 信息列表
    * workloadNamespace：待训练 workload 的命名空间
    * workloadName：待训练 workload 的名称
    * historyLength：待采集训练数据集的时间长度，支持 "min"、"H"、"D" 时间单位，表示"分钟"、"小时"、"天"。
    * metrics：表示如何从监控系统采集指标数据集，支持多指标数据
- freq：表示模型的时间频率，目前支持的频率为，"1H"、"10min"、"1min"、"D"
- predictionLength：表示预测时间长度
- contextLength：预测时上下文时间长度
- hyperParams：超参数
    * learningRate：训练过程中学习率
    * epochs：训练迭代的轮数
    * batchSize：一个batch训练数据大小

## 训练模型

### 脚本参数

- dataset-file：表示训练数据集csv文件地址，支持用户自己获取训练数据集，优先级高于从 metric 服务端获取。
- config-file：表示训练配置文件地址，用于调整模型相关参数
- metrics-server-addr：metric 服务端 grpc 地址，可以通过 kapacity service 获取
- model-save-path：模型存储目录

### 脚本执行

如果是从外部 k8s 集群里的 kapacity 获取数据的话，可以通过 kubectl 的端口转发功能让本地可以访问远端 kapacity 的 grpc
服务（注意 local-port 替换为真实的端口号）。

```shell
kubectl port-forward -n kapacity-system  svc/kapacity-grpc-service <local-port>:9090
```

执行 python 脚本进行模型训练，注意替换脚本参数值为正确的值。

```shell
python kapacity/timeseries/forecasting/train.py \
   --config-file=<config-file> \
   --metrics-server-addr=<metrics-server-addr> \
   --model-save-path=<model-save-path>
```

脚本执行后会看到以下训练日志，等待脚本训练完成（训练时间跟数据集和参数有关）

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

训练完成后会在 **model-save-path** 目录下生成以下模型文件。

```shell
-rw-r--r--@ 1 admin  staff   316K Oct 18 20:18 checkpoint.pth
-rw-r--r--@ 1 admin  staff   287B Oct 18 20:04 estimator_config.yaml
-rw-r--r--@ 1 admin  staff    29B Oct 18 20:04 item2id.json
```
