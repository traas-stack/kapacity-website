---
title: "FAQ"
weight: 4
---

{{% pageinfo color="primary" %}}
Note: Most contents of this page are translated by AI from [Chinese](/zh-cn/docs/user-guide/faq/). It would be very appreciated if you could help us improve by editing this page (click the corresponding link on the right side).
{{% /pageinfo %}}

## IHPA

### Predictive algorithm job failed with error `replicas estimation failed`

The error is due to the failure of the "Traffic, Capacity and Replicas Relationship Modeling" algorithm to produce a usable model, the following methods can be attempted to solve this issue:

- Try increasing the length of historical data by adjusting the algorithm job parameter `--re-history-len`.
- With the detailed model evaluation information returned with the error, try relaxing the model validation requirements by adjusting the algorithm job parameters `--re-min-correlation-allowed` and `--re-max-mse-allowed`. However, be aware that if the relaxed values differ too much from the default values, the accuracy of the model may be hard to guarantee.
