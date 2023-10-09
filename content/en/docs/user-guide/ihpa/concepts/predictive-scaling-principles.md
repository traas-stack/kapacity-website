---
title: "Principles of Predictive Scaling"
weight: 2
---

{{% pageinfo color="primary" %}}
Note: Most contents of this page are translated by AI from [Chinese](/zh-cn/docs/user-guide/ihpa/concepts/predictive-scaling-principles/). It would be very appreciated if you could help us improve by editing this page (click the corresponding link on the right side).
{{% /pageinfo %}}

## Advantages of Predictive Scaling

<img src="/images/en/reactive-vs-predictive.png" width="1000"/>

By the comparison of above figure, we can conclude several advantages of predictive scaling over reactive scaling:

- Predictive scaling can respond to traffic changes in advance
- Predictive scaling can control resource levels more stably
- Predictive scaling has higher accuracy and can use resources more effectively

## Traffic-Driven Replicas Prediction

In this section, we will introduce in detail the design concept and working principle of IHPA's "traffic-driven replicas prediction" algorithm.

### Why Traffic-Driven

For online applications, capacity (resource) indicators (such as CPU utilization) are strongly correlated with traffic, i.e., traffic variations drive changes in capacity indicators. Predicting capacity through traffic, rather than directly predicting capacity indicators, has the following advantages:

- Traffic indicators are the most upstream indicators, which change before capacity indicators, and respond quickly.
- Capacity indicators are easily disturbed by a variety of factors (such as the application's own code issues, host performance, etc.), while traffic indicators are only directly related to application characteristics (such as user usage habits), making them easier to predict over time.

### Modeling the Relationship between Traffic, Capacity and Replicas

In order to convert the replica count prediction problem into a traffic prediction problem, we designed a *Linear-Residual Model* to find the association function between traffic, capacity, and replica count, as shown in the following figure:

<img src="/images/en/linear-residual-model.png" width="800"/>

In this model, we set the resource utilization rate as the target indicator, because controlling the resource level of the application is our ultimate goal of using elastic scaling, which is the most intuitive.

However, unlike the reactive scaling algorithm of Kubernetes HPA, although we set the resource utilization rate as the target indicator, this algorithm will not only consider this indicator, but will take historical traffic (supporting multiple lines), historical resource utilization, and historical replica count as inputs. These indicators will first go through a linear model, which can learn the linear association between the three, and get the association function in the above figure; then, they will go through a residual model with other information (currently only includes time information), which will correct the association function after considering other information, and can learn the complex non-linear association between traffic, capacity and replica count.

Here is a simple example to illustrate the main function of the residual model: Suppose an online application executes an internal timed task every Sunday morning, which brings additional CPU resource consumption, but it has no association with the external traffic handled by the application. At this time, this feature cannot be learned through the linear model alone. After introducing the residual model, this model can learn this feature based on time information, so at the time of Sunday morning, we give the same traffic and replica count as other times, and the function it gives will output higher CPU consumption, which is in line with the actual situation.

In the current algorithm implementation, we use ElasticNet as the linear model and LightGBM as the residual model. They are both traditional machine learning algorithms, not strongly dependent on GPU, have lower usage overhead compared to deep learning algorithms, and can also achieve good results. Of course, you can also replace the specific implementation of these models according to your own needs, and you are welcome to provide implementations that you think are superior in certain scenarios.

After using this model to get the association function, we can convert the replica count prediction problem into a traffic prediction problem: knowing the target resource utilization rate, just input the predicted traffic, and you can get the predicted replica count (which can maintain the target average resource utilization rate under the predicted traffic).

#### Model Details

{{% alert %}}
If you're not interested in the specific implementation details of the model, you can skip this section.
{{% /alert %}}

Denote workflow information (including traffics, resource usage, replicas) as $k$, target indicator (resource usage) as $y$, and meta information (including time, etc.) as $\omega$. We first use a linear model to characterize the skeleton of target indicator: $$\hat y_l = L(x)$$ Then we calculate the error of the linear model: $$e = y - \hat y$$ Next, we combine meta information with the error of the linear model, and mature it by a residual model: $$\hat e = R(\hat y,\omega)$$ Finally, we get the estimation of $y$ as: $$\hat y_r = \hat y_l + \hat e$$ where $L$ and $R$ could be any linear and residual model.

### Time Series Forecasting of Traffic

We designed a deep learning model called *Swish Net for Time Series Forecasting* to predict traffic indicators over time. This model is specifically optimized for the use case of IHPA and has the following two main characteristics:

- **Lightweight**：The model has a relatively simple structure, which results in a smaller model size and lower training cost. For example, for a single traffic prediction, predicting 12 future points from 12 historical points (which is a 2-hour long prediction at 10-minute accuracy), the trained model size is less than 1 MiB. Under PC-level CPU training, an epoch only takes about 1 minute, and training can be completed in about 1-2 hours.
- **Better performance on production traffics forecasting**：We compared this model with other common deep learning time series forecasting models using a production traffic dataset. The results show that this model outperforms other models in the task of production traffic time series forecasting, as shown in the following table:

|         | MAE   | RMSE   |
|---------|-------|--------|
| DeepAR  | 1.734 | 31.315 |
| N-BEATS | 1.851 | 41.681 |
| ours    | 1.597 | 28.732 |

#### Model Details

{{% alert %}}
If you're not interested in the specific implementation details of the model, you can skip this section.
{{% /alert %}}

<img src="/images/en/swish-net-tsf-model.png" width="600"/>

Assuming that the historical traffic $y_{1:T,i}$ is known, the future real traffic is $y_{T+1:T+\tau,i}$, the predicted traffic is $\hat y_{T+1:T+\tau,i}$, and the category of the traffic (such as App) is $i$. The traffic time series will have cyclical, trend, and autoregressive features. We design the following modules to capture these characteristics, and aggregate information to make predictions for the future.

1. The Embedding Layer of the model projects category information and time information into high-dimensional vectors. The category information expresses the differences between different sequences, and the time information can express the cyclicity of the time series: $$V_i = Embed(i)$$ $$V_t = Embed(t)$$

2. The cross product of the model's time and category features with historical traffic can further extract different differential and cyclical features of different sequences: $$\tilde V_i = V_i \odot y_{1:T}$$ $$\tilde V_t = V_t \odot y_{1:T}$$

3. The difference feature between the next time step and the previous time step of the traffic time series can eliminate the trend and better express the periodicity of the time series. The trend feature is included in the original sequence: $$\tilde y_{1:T} = y_{2:T} - y_{1:T-1}$$

4. The input and network structure of the Multilayer Perception layer model are expressed as follows: $$in = concate(V_i,V_t,\tilde V_i,\tilde V_t,Embed(i),Embed(t),\tilde y_{1:T},y_{1:T})$$ $$\hat y_{T+1:T+\tau,i} = MLP(in)$$ The multilayer time network summarizes the information of the above feature modules and predicts the time series of future time steps.

5. The loss function of the model is MSE: $$loss = \sum_{i,t}(y_{i,t}-\hat y_{i,t})^2$$

### Full Algorithm Workflow

Finally, we can link the above two models with the relevant data sources to get the complete workflow of the IHPA predictive autoscaling algorithm, as shown in the following diagram:

<img src="/images/en/predictive-algo-workflow.png" width="1000"/>

Legend:

- The blue line represents the offline workflow, which requires a large amount of data & GPU & a longer execution time, and has a lower execution frequency.
- The yellow line represents the online workflow, which requires a moderate amount of data & CPU & a shorter execution time, and has a higher execution frequency.

Note:

- The "app" here is an abstract concept, representing the smallest unit of autoscaling, typically a specific workload such as Deployment, etc.
- The "all traffics" here does not represent all monitorable traffic in the monitoring system, but represents the component traffic that is significantly positively correlated with the target resource usage indicator, which can be selected as needed according to the specific application scenario.
