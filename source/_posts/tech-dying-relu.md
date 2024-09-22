---
title: 技术总结《PyTorch排查Dying ReLU问题》
date: 2024-09-22 21:35:33
tags:
    - Debug
    - Dying ReLU
categories:
	- 技术总结
description:
	- 一种在PyTorch上排查Dying ReLU的方法
---

# 现象

最近碰到一个比较神奇的问题，RL训练一个很简单的任务死活训不出来。

从指标上看，最诡异的点在于动作头的entropy一直很高，基本就处于均匀采样的状态（`logN`）。

# 猜想

尝试把entropy loss关掉，以及加大learning rate都无济于事。

最后怀疑是Dying ReLU的原因，就尝试把整个网络所有ReLU的输出打印出来。

# 验证

实现的思路大概就是给所有ReLU加上一个hook，让调用ReLU的forward时调用这个hook，具体实现见下面代码：

```py
# Map: ReLU layer name -> dead_rate
stats = {}

def make_hook(name):
    # 计算dead_rate，并将结果写入stats
    def hook(model, input, output):
        key = f"dead_rate/{name}".replace('.', '-')
        dead_rate = (output <= 0).sum().float() / output.nelement()
        stats[key] = dead_rate
    return hook

# 给网络中所有ReLU挂上hook
for name, module in network.named_modules():
    if (isinstance(module, (nn.ReLU, nn.LeakyReLU))):
        module.register_forward_hook(make_hook(name))

# 后续每个training step打印stats数据或者写入tensorboard
```

将`dead_rate`打印出来发现确实是Dying ReLU了，最后一层ReLU的`dead_rate`蹭蹭蹭地干到了99%以上。

# 应对

尝试过Leaky ReLU效果不太理想，最后选择了牺牲效率，通过调低learning rate来解决。

# 参考

- [主要参考](https://discuss.pytorch.org/t/checking-whether-network-is-dead-pytorch/21266)
- [一个挺有意思的思路](https://discuss.pytorch.org/t/splitting-neurons-best-practice/110910/2)