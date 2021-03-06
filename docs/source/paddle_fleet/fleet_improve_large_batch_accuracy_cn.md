# 使用LARS / LAMB 优化分布式超大batch 训练

## 简介 
在数据并行分布式训练场景中, 常使用增加GPU数量的方式来加速训练. 为了保证GPU的算力得到充分利用, 每张GPU卡上的batch size都需要足够大. 因此当GPU 数量增加时, 训练的全局batch size 也会变大. 但越大的全batch size 训练的收敛问题[[1]](https://arxiv.org/abs/1404.5997)  [[2]](https://arxiv.org/abs/1609.04836):

 * 模型最终精度损失
 * 收敛速度变慢, 需要更多的epoch 才能收敛 

LARS[[3]](https://arxiv.org/abs/1708.03888) 和 LAMB[[4]](https://arxiv.org/abs/1904.00962) 两个优化策略常用来解决上述超大batch 训练中的收敛问题. 

FleetX 实现了这两种优化策略, 并提供简单易用API 接口. 通过这两个优化策略, 我们在超大batch 场景中实现了更快的收敛速度和无损的精度, 结合FleetX 中其他的策略(e.g. [AMP](https://LINK_to_be_added)) 极大缩短的训练整体的time2train. 

中下文将通过一个简单例子介绍如何在Fleet 数据并行训练框架中使用 LARS 和LAMB, 另外给出我们使用 FleetX 实践的效果和代码.

## FleetX 效果

|resnet50 imagenet |Global batch size|epoch| top1 |
|:---:|:---:|:---:|:---:|
|[Goyal et al](https://arxiv.org/abs/1706.02677)| 8k | 90 | 76.3% |
|[LARS](https://arxiv.org/abs/1708.03888)| 32k | 90 | 72.3% |
|[FleetX: lars + amp](https://LINK_to_example_code) |16k | 60 | 75.9%|
|[FleetX: lars + amp](https://LINK_to_example_code) |32k | TBA | TBA |
|[FleetX: lars + amp](https://LINK_to_example_code) |64k | TBA | TBA |

|bert en-de |Global batch size|epoch| top1 |
|:---:|:---:|:---:|:---:|
|[FleetX: lamb + amp](https://LINK_to_example_code) |TBA | TBA | TBA |
|[FleetX: lamb + amp](https://LINK_to_example_code) |TBA | TBA | TBA |

## LARS 
我们以在单机多卡上Resent50 训练为简单例子介绍FleetX 中 lars的用法.

#### 构建模型
首先我们要导入依赖和定义模型和 data loader, 这一步和FleetX 下其他任务基本一致.

```python
import os
import fleetx as X
import paddle.fluid as fluid
import paddle.distributed.fleet.base.role_maker as role_maker
import time
import paddle.distributed.fleet as fleet

model = X.applications.Resnet50()
loader = model.load_imagenet_from_file("/pathto/ImageNet/train.txt")
```

#### 定义分布式及LARS 相关策略
LARS 优化算法的公式如下:

.. math::
    & local\_learning\_rate = learning\_rate * lars\_coeff * \\
        \\frac{||param||}{||gradient|| + lars\_weight\_decay * ||param||}
    & velocity = mu * velocity + local\_learning\_rate * (gradient + lars\_weight\_decay * param)
    & param = param - velocity

可以看到LARS 其实是在 带`weight decay` 的`momentum` 优化器的基础上加入了`local learning rate` 的逻辑, 对每一层的`learning rate` 进行了放缩. 
FleetX 将 LARS实现为一个 fleet meta optimizer, 在使用时需要设置一下几点:

1. LARS meta optimizer 的 inner optimizer 必须为 `momentum`, 并在 momentum 中定义 `mu` 和`lr` 参数.
2. 在 fleet dist_strategy 定义LARS 特有的 `lars_coeff` 参数和 `lars_weight_decay` 参数.
    * LARS 已经将 `weight decay` 包含进公式中, 用户不需要再在 optimizer中设置 `regularization`.
    * FleetX 中还提供 lars_weight_decay 过滤策略, 可以通过在`exclude_from_weight_decay` 参数加入对应layer 的 `name string`, 让这一 layer 的参数不进行 lars weight decay. (通常我们将`BN` 参数 和 `FC_bias` 从lars weight decay 中过滤)

```python
configs = X.parse_train_configs()
role = role_maker.PaddleCloudRoleMaker(is_collective=True)
fleet.init(role)
dist_strategy = fleet.DistributedStrategy()

dist_strategy.lars = True
dist_strategy.lars_configs = {
                    "lars_coeff": 0.001,
                    "lars_weight_decay": 0.0005,
                    "exclude_from_weight_decay": ['batch_norm', '.b_0']
                }

optimizer = paddle.optimizer.Momentum(learning_rate=0.01, momentum=0.9)
optimizer = fleet.distributed_optimizer(optimizer, dist_strategy)
optimizer.minimize(model.loss)
```

#### 开始训练
这一部分和FleetX 中其他任务基本相同:

```python
place = fluid.CUDAPlace(int(os.environ.get('FLAGS_selected_gpus', 0)))
exe = fluid.Executor(place)
exe.run(fluid.default_startup_program())

total_time = 0
for i, data in enumerate(data_loader()):
    start_time = time.time()
    cost_val = exe.run(paddle.static.default_main_program(),
                       feed=data,
                       fetch_list=[model.loss.name])
    end_time = time.time()
    total_time += (end_time - start_time)
    print(
        "worker_index: %d, step%d cost = %f, total time cost = %f, step per second: %f, speed: %f"
        % (fleet.worker_index(), i, cost_val[0], total_time,
           (i - 9) / total_time, 1 / (end_time - start_time))
```

### 运行训练脚本
一行启动单机多卡分布式训练：
```sh
fleetrun --gpus 0,1,2,3,4,5,6,7 resnet50_lars.py
```


## LAMB 
我们以在单机多卡上Bert 训练为简单例子介绍FleetX 中LAMB 的用法.

#### 构建模型
和LARS 中类似, 首先我们要导入依赖和定义模型和 data loader.

```python
import os
import time
import paddle
import fleetx as X
import paddle.distributed.fleet as fleet
import paddle.distributed.fleet.base.role_maker as role_maker

model = X.applications.Bert_large()

data_loader = model.load_digital_dataset_from_file(
    data_dir='./train_data',
    vocab_path='./vocab.txt',
    max_seq_len=512,
    batch_size=53,
)
```

#### 定义分布式及LARS 相关策略
LAMB 优化算法的公式如下:

..  math::
    m_t &= \\beta_1 m_{t - 1}+ (1 - \\beta_1)g_t 
    v_t &= \\beta_2 v_{t - 1}  + (1 - \\beta_2)g_t^2
    r_t &= \\frac{m_t}{\\sqrt{v_t}+\\epsilon}
    w_t &= w_{t-1} -\\eta_t \\frac{\\left \| w_{t-1}\\right \|}{\\left \| r_t + \\lambda w_{t-1}\\right \|} (r_t + \\lambda w_{t-1})

在公式中 `m` 是一阶 moment, 而`v` 是二阶moment, `\eta` 和 `\lambda` 分别是 LAMB `learning rate`  和 `weight decay rate`.

和LARS 类似, LAMB 也是在内层优化器的基础上, 套了一个`local learning rate` 的逻辑, 对每一层的`learning rate` 进行了放缩. 
FleetX 将 LAMB实现为一个 fleet meta optimizer, 在使用时需要设置一下几点:

1. LAMB meta optimizer 的 inner optimizer 必须为 `adam`, 并在 adam 中定义 学习率`lr`, 一阶 moment 的指数衰减率`beta1` 和二阶moment 的指数衰减率`beta2` 参数.
2. 在 fleet dist_strategy 定义LAMB 特有的 `lamb_weight_decay` 参数.
    * LAMB 已经将 `weight decay` 包含进公式中, 用户不需要再在 optimizer中设置 `regularization`.
    * FleetX 中还提供 lamb_weight_decay 过滤策略, 可以通过在`exclude_from_weight_decay` 参数加入对应layer 的 `name string`, 让这一 layer 的参数不进行 lars weight decay. (通常我们将`LN` 从lamb weight decay 中过滤)

```python
configs = X.parse_train_configs()
role = role_maker.PaddleCloudRoleMaker(is_collective=True)
fleet.init(role)
dist_strategy = fleet.DistributedStrategy()

dist_strategy.lamb = True
dist_strategy.lamb_configs = {
                    'lamb_weight_decay': 0.01,
                    'exclude_from_weight_decay': ['layer_norm'],
                }

optimizer = paddle.optimizer.Adam(learning_rate=0.01, beta1=0.9, beta2=0.999)
optimizer = fleet.distributed_optimizer(optimizer, dist_strategy)
optimizer.minimize(model.loss)
```

#### 开始训练
这一部分和FleetX 中其他任务基本相同:
```python
place = fluid.CUDAPlace(int(os.environ.get('FLAGS_selected_gpus', 0)))
exe = fluid.Executor(place)
exe.run(fluid.default_startup_program())

total_time = 0
for i, data in enumerate(data_loader()):
    start_time = time.time()
    cost_val = exe.run(paddle.static.default_main_program(),
                       feed=data,
                       fetch_list=[model.loss.name])
    end_time = time.time()
    total_time += (end_time - start_time)
    print(
        "worker_index: %d, step%d cost = %f, total time cost = %f, step per second: %f, speed: %f"
        % (fleet.worker_index(), i, cost_val[0], total_time,
           (i - 9) / total_time, 1 / (end_time - start_time)))
```

### 运行训练脚本
一行启动单机多卡分布式训练：
```sh
fleetrun --gpus 0,1,2,3,4,5,6,7 bert_lamb.py
```
