---
title: "训练时序预测模型"
weight: 1
---

## 准备开始

本文档将指导您如何训练 Kapacity 所使用的时序预测深度学习算法的模型。

在开始之前，请确保您的环境已经安装了 [Conda](https://docs.conda.io/en/latest/)。

## 安装依赖

执行以下命令以下载你所使用的 Kapacity 算法版本的代码、安装算法依赖并激活算法运行环境：

```shell
git clone --depth 1 -b algorithm-<your-kapacity-algorithm-version> https://github.com/traas-stack/kapacity.git
cd kapacity/algorithm
conda env create -f environment.yml
conda activate kapacity
```

## 准备配置文件

训练脚本会从一个外部配置文件读取数据集拉取和模型训练的相关参数，因此我们需要提前准备好该配置文件。你可以下载这份示例配置文件 [tsf-model-train-config.yaml](/examples/algorithm/tsf-model-train-config.yaml) 并按需修改其内容，该文件的内容如下所示：

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

下面解释其中各字段的含义：

- `targets`：待训练的工作负载及其指标信息列表（算法支持训练多个工作负载的多个指标到一个模型，但也会使模型变得更大），如果你希望手动准备数据集而非算法自动拉取，可不填写该字段。
  - `workloadNamespace`：待训练工作负载所在命名空间。
  - `workloadRef`：待训练工作负载对象的引用标识。
  - `historyLength`：拉取作为数据集的指标历史长度，支持 `min`（分钟）、`H`（小时）、`D`（天）三个时间单位。一般建议至少覆盖周期性指标的两个完整周期。
  - `metrics`：该工作负载待训练的指标列表，格式同 IHPA 的指标字段。注意必须为每个指标设置不同的 `name` 以在同一个模型中的同一个工作负载下区分不同的指标。
- `freq`：模型的精度，即下面 `predictionLength` 和 `contextLength` 参数的单位。目前支持的值为 `1min`、`10min`、`1H`、`1D`。注意该参数不会影响模型大小。
- `predictionLength`：模型的预测点数，最终预测长度为 `predictionLength * freq`。该参数设置地越大，模型也会越大。一般不建议设置得过大，因为预测的时间点越久远，预测的准确度也会越低。
- `contextLength`：模型在预测（推理）时参考的历史点数，最终参考的历史长度为 `contextLength * freq`。该参数设置地越大，模型也会越大。
- `hyperParams`：深度学习模型的超参数，一般不需要调整。

## 训练模型

执行以下命令进行模型训练，注意将相关参数替换为真实值：

```shell
python kapacity/timeseries/forecasting/train.py \
  --config-file=<your-config-file> \
  --model-save-path=<your-model-save-path> \
  --dataset-file=<your-datset-file> \
  --metrics-server-addr=<your-metrics-server-addr> \
  --dataloader-num-workers=<your-dataloader-num-workers>
```

下面解释各参数的含义：

- `config-file`：上一步准备的配置文件的地址。
- `model-save-path`：模型要保存到的目录地址。
- `dataset-file`：手动准备的数据集文件的地址。和 `metrics-server-addr` 参数二选一填写。
- `metrics-server-addr`：用于自动拉取数据集的指标服务器的地址，即可访问的 Kapacity gRPC 服务的地址。和 `dataset-file` 参数二选一填写。
- `dataloader-num-workers`：用于加载数据集的子进程数量，一般建议设置为本机 CPU 核数，如果设置为 `0` 则仅使用主进程加载。

{{% alert title="关于如何访问 Kapacity gRPC 服务的贴士" %}}

默认情况下，Kapacity 的 gRPC 服务会暴露为一个 ClusterIP Service，你可以使用如下命令查看其 ClusterIP 和端口：

```shell
kubectl get svc -n kapacity-system kapacity-grpc-service
```

```
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kapacity-grpc-service   ClusterIP   192.168.38.172   <none>        9090/TCP   5m
```

如果你是在集群中运行训练脚本，则可以直接使用该 ClusterIP 和端口作为 `metrics-server-addr`。

如果你不在集群中运行训练脚本，则可以使用如下命令通过 kubectl 的端口转发功能来从本机访问集群中的这个 Service，注意将 `<local-port>` 替换为本地的一个闲置端口：

```shell
kubectl port-forward -n kapacity-system svc/kapacity-grpc-service <local-port>:9090
```

随后，你可以使用 `localhost:<local-port>` 作为 `metrics-server-addr`。

{{% /alert %}}

执行后你将看到类似下面内容的训练日志，请等待直到训练完成（命令正常退出）：

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

训练完成后你可以在 `model-save-path` 目录下看到训练好的模型及其附属文件：

```
-rw-r--r-- 1 admin  staff   316K Oct 18 20:18 checkpoint.pth
-rw-r--r-- 1 admin  staff   287B Oct 18 20:04 estimator_config.yaml
-rw-r--r-- 1 admin  staff    29B Oct 18 20:04 item2id.json
```
