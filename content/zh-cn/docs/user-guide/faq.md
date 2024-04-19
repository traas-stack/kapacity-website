---
title: "常见问题"
weight: 4
---

## IHPA

### 预测式算法任务执行报错 `replicas estimation failed`

该报错是由于「流量、容量与副本数关联建模」算法没有产出可用的模型导致，可以尝试下面的方法解决此问题：

- 通过调整算法任务参数 `--re-history-len` 增大历史数据长度。
- 结合报错返回的详细模型评估信息，通过调整算法任务参数 `--re-min-correlation-allowed` 与 `--re-max-mse-allowed` 适当放宽模型的验证要求。但需要注意，如果放宽的值和默认值差距过大，模型的准确性将很难得到保证。
