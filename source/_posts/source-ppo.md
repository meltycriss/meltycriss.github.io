---
title: 源码阅读《PyTorch PPO》
date: 2018-10-09 11:56:11
tags: 
  - pytorch-ppo
description:
  - 基于PyTorch的PPO的源码阅读
categories:
  - 源码阅读
---

## Introduction

本文介绍的Proximal Policy Optimization (PPO)实现是基于PyTorch的，其Github地址在[这里](https://github.com/ikostrikov/pytorch-a2c-ppo-acktr)。实际上它一共实现了三个算法，包括PPO、A2C以及ACKTR。这份代码的逻辑抽象做得不错，三个算法共用了很多代码，因此看懂了PPO对于理解另外两个算法的实现有很大帮助。

这份PPO代码依赖于[OpenAI baselines](https://github.com/openai/baselines)，主要用到了其并行环境的wrapper。由于PPO和OpenAI baselines的代码一直在更新，所以最新的代码跟本文可能有所出入。**本文所述代码对应的commit id前缀**为：

- PPO：3aea397
- OpenAI baselines：f272969

下面先介绍代码的**逻辑框架**，然后再看**具体代码**。具体代码部分以一个类似于**深度优先**的方式展开。

## Logical framework

这份代码按照RL的思想，将代码分成了environment和robot（之所以不用agent这个词，主要是为了跟下面的变量名做区分）两大模块。environment部分主要是`envs`这个变量，它通过OpenAI baselines的wrapper实现了多环境并行化。robot部分分为了三个子模块，第一个是parametric policy变量`actor_critic`；第二个是用来保存trajectory的`rollouts`变量；最后一个是用于参数更新的RL algorithm变量`agent`。总结一下，其**逻辑框架**如下:

- environment
	- multiprocess wrapper: `envs`
- robot
	- parametric policy: `actor_critic`
    - trajectory: `rollouts`
	- RL algorithm: `agent`

整个**交互流程**基本上是robot从`envs`中得到一个observation，然后根据`actor_critic`作出反应，并且将交互过程记录到`rollouts`中，接着使用`agent`根据`rollouts`中的信息来更新`actor_critic`。

## Source code

### main

```py
# PPO_ROOT/main.py

def main():
    ...
    
    # 创建并行交互环境envs
    envs = [make_env(args.env_name, args.seed, i, args.log_dir, args.add_timestep)
                for i in range(args.num_processes)]
    
    if args.num_processes > 1:
        envs = SubprocVecEnv(envs)
    else:
        ...
    
    if len(envs.observation_space.shape) == 1:
        envs = VecNormalize(envs, gamma=args.gamma)
    
    ...
    
    for j in range(num_updates):
        for step in range(args.num_steps):
            # 通过rollouts获得t时刻的的信息（e.g., o_t），使用actor_critic(i.e., policy)来计算a_t
            with torch.no_grad():
                value, action, action_log_prob, states = actor_critic.act(
                        rollouts.observations[step],
                        rollouts.states[step],
                        rollouts.masks[step])
    
            ...
    
            # 使用a_t跟envs(i.e., environment)进行交互
            obs, reward, done, info = envs.step(cpu_actions)
            
            ...
    
            # 将交互信息存入rollouts中
            rollouts.insert(current_obs, states, action, action_log_prob, value, reward, masks)
    
        ...
        
        # 根据rollouts提供的信息，使用RL algorithm（i.e., PPO）对actor_critic(i.e., policy)进行参数更新
        value_loss, action_loss, dist_entropy = agent.update(rollouts)
    
        ...
```

以上代码将`main`函数的核心部分提了出来。可见，`main`函数主要有以下**核心步骤**：

1. 创建并行交互环境`envs`；
2. 通过`actor_critic`与环境进行交互；
3. 将交互信息存入`rollouts`；
4. 使用`agent`进行参数更新。

接下来分别展开介绍这四个变量。

---

### envs

由`main`函数中可见，`envs`变量通过`make_env`、`SubprocVecEnv`和`VecNormalize`得到，接下来分别介绍这三个函数。

#### make_env

```py
# PPO_ROOT/envs.py

def make_env(env_id, seed, rank, log_dir, add_timestep):
    def _thunk():
    	# 最基本的gym env
        if env_id.startswith("dm"):
        	...
        else:
            env = gym.make(env_id)
        
        ...

        # 加了一个Monitor wrapper（注意这个Monitor不是保存视频的gym.Wrappers.Monitor）来记录信息（默认是/tmp/gym/）
        if log_dir is not None:
            env = bench.Monitor(env, os.path.join(log_dir, str(rank)))

        ...

        return env

    # 返回的是一个函数而不是env
    return _thunk
```

从上述代码可以看出，`make_env`函数返回了一个函数`_thunk`，调用该函数可以得到一个gym env。（`make_env`选择返回一个函数而不直接返回一个gym env的原因我不太清楚，我推测是跟**序列化和反序列化**有关。从下面的`SubprocVecEnv`可见，`make_env`的返回结果要传给一个在另一个进程运行的函数`worker`，这个过程应该是需要做序列化和反序列化的，可能传函数比较好实现或者比较高效？）

#### SubprocVecEnv

```py
# BASELINE_ROOT/common/vec_env/subproc_vec_env.py

def worker(remote, parent_remote, env_fn_wrapper):
    parent_remote.close()
    # 实例化env，相当于_thunk()（之所以需要.x()是因为用了CloudpickleWrapper，估计是为了处理函数序列化和反序列化的一些问题）
    env = env_fn_wrapper.x()
    # 相当于一个一直在跑的env服务
    while True:
        cmd, data = remote.recv()
        if cmd == 'step':
            ob, reward, done, info = env.step(data)
            if done:
                ob = env.reset()
            remote.send((ob, reward, done, info))
        elif cmd == 'reset':
            ob = env.reset()
            remote.send(ob)
        elif cmd == 'render':
            remote.send(env.render(mode='rgb_array'))
        elif cmd == 'close':
            remote.close()
            break
        elif cmd == 'get_spaces':
            remote.send((env.observation_space, env.action_space))
        else:
            raise NotImplementedError
```

`SubprocVecEnv`首先创建不同的subprocess去运行`worker`，每一个`worker`基本上相当于一个**env服务**，每收到一个指令就对env执行相应的操作并且返回信息。`worker`的参数`env_fn_wrapper`就是`_thunk`加了一个`CloudpickleWrapper`，该`wrapper`好像是用来辅助序列化和反序列化的。

```py
# BASELINE_ROOT/common/vec_env/subproc_vec_env.py

class SubprocVecEnv(VecEnv):
    ...

    # 给subprocess发送step指令
    def step_async(self, actions):
        for remote, action in zip(self.remotes, actions):
            remote.send(('step', action))
        self.waiting = True


    # 接收并拼接subprocess返回的交互信息
    def step_wait(self):
        results = [remote.recv() for remote in self.remotes]
        self.waiting = False
        obs, rews, dones, infos = zip(*results)
        return np.stack(obs), np.stack(rews), np.stack(dones), infos

    ...

    # 这个函数在基类VecEnv中
    def step(self, actions):
        self.step_async(actions)
        return self.step_wait()
```

有了上面的subprocee，主进程中的`SubprocVecEnv`只需要**重载**`step`, `reset`等关键函数为发送相应指令给subprocess，然后将数据拼接起来，就实现了跟环境的并行交互。

#### VecNormalize

```py
# BASELINE_ROOT/common/vec_env/vec_normalize.py

class VecNormalize(VecEnvWrapper):
    def __init__(self, venv, ob=True, ret=True, clipob=10., cliprew=10., gamma=0.99, epsilon=1e-8):
        VecEnvWrapper.__init__(self, venv)
        # mean, std计算器
        self.ob_rms = RunningMeanStd(shape=self.observation_space.shape) if ob else None
        self.ret_rms = RunningMeanStd(shape=()) if ret else None

    ...

    def step_wait(self):
        obs, rews, news, infos = self.venv.step_wait()
        self.ret = self.ret * self.gamma + rews

        # 对obs做normalization（mean, std），具体代码见下面
        obs = self._obfilt(obs)
        # 对reward做normalization（不用mean只用std，而且用的是return的std）
        if self.ret_rms:
            self.ret_rms.update(self.ret)
            rews = np.clip(rews / np.sqrt(self.ret_rms.var + self.epsilon), -self.cliprew, self.cliprew)
        return obs, rews, news, infos

    def _obfilt(self, obs):
        if self.ob_rms:
            self.ob_rms.update(obs)
            obs = np.clip((obs - self.ob_rms.mean) / np.sqrt(self.ob_rms.var + self.epsilon), -self.clipob, self.clipob)
            return obs
        else:
            return obs

    ...
```

从上述代码可见，`VecNormalize`也是一个wrapper，主要功能是**对observation和reward做normalization**。

---

### actor_critic

```py
# PPO_ROOT/model.py

class Policy(nn.Module):
    ...

    def act(self, inputs, states, masks, deterministic=False):
    	# self.base集actor和critic于一身
    	# value：V(o_t)
    	# actor_features：actor网络的倒数第二层
    	# states：RNN的hidden states
        value, actor_features, states = self.base(inputs, states, masks)
        # dist最后的action分布\pi(A_t | o_t)
        dist = self.dist(actor_features)

       	...

    ...

    def evaluate_actions(self, inputs, states, masks, action):
        ...

        # value: V(o_t)
        # action_log_probs: \log(\pi(a_t | o_t))
        # dist_entropy: Entropy(\pi(A_t | o_t))
        # states：RNN的hidden states
        return value, action_log_probs, dist_entropy, states
```

表示parametric policy的`actor_critic`变量对应的类为`Policy`，基本上就是包含actor和critic的神经网络。如上所示，其主要函数为`act`和`evaluate_actions`，前者用于**求action**，后者用于从网络中**获取参数更新要用到的信息**（e.g., $V(o_t), \log\pi(a_t \vert o_t)$）。

---

### rollouts

```py
# PPO_ROOT/storage.py

class RolloutStorage(object):
    ...

    # 保存交互信息
    # current_obs: o_{t+1}
    # state: RNN中的hidden state，
    # action: a_t
    # action_log_prob: \pi(a_t | o_t)
    # value_pred: V(o_t)
    # reward: r_t
    # mask: 标识o_{t+1}时刻是否通过gym.reset()得到，是的话为0，否的话为1
    def insert(self, current_obs, state, action, action_log_prob, value_pred, reward, mask):
        ...

    ...

    # 计算return
    # 可以选择使用GAE（GAE+V）还是MC来计算return
    def compute_returns(self, next_value, use_gae, gamma, tau):
        if use_gae:
            ...
        else:
            ...

    # 将num_steps * num_processes份transitions随机分成num_mini_batch个mini_batch，每次返回一个mini_batch
    def feed_forward_generator(self, advantages, num_mini_batch):
        ...

    # 考虑到RNN的顺序有关性质，以process为单位将num_steps * num_processes份transitions随机分成num_mini_batch个mini_batch，每次返回一个mini_batch
    def recurrent_generator(self, advantages, num_mini_batch):
        ...
```

用于储存trajectory的`rollouts`变量对应的类为`RolloutStorage`，主要功能是**存和取**，其对应的成员函数分别是`insert`和`xxx_generator`。还有一个`compute_returns`函数用于辅助计算更新参数所需的return。

---

### agent

```py
# PPO_ROOT/algo/ppo.py

class PPO(object):
    ...

    def update(self, rollouts):
    	# \hat{A)_t in PPO paper（return的计算可能用的GAE，也可能用的MC）
        # 并且对advantage做normalization
        advantages = rollouts.returns[:-1] - rollouts.value_preds[:-1]
        advantages = (advantages - advantages.mean()) / (
            advantages.std() + 1e-5)

        ...

        for e in range(self.ppo_epoch):
            ...

            for sample in data_generator:
                ...

                # r_t(\theta) in the PPO paper
                ratio = torch.exp(action_log_probs - old_action_log_probs_batch)
                surr1 = ratio * adv_targ
                surr2 = torch.clamp(ratio, 1.0 - self.clip_param,
                                           1.0 + self.clip_param) * adv_targ
                # -L_t^{CLIP}(\theta) in the PPO paper
                action_loss = -torch.min(surr1, surr2).mean()

               	# L_t^{VF}(\theta) in the PPO paper
                value_loss = F.mse_loss(return_batch, values)

                self.optimizer.zero_grad()
                # -L_t^{CLIP+VF+S}(\theta) in the PPO paper（因为我们想做minization，而lr是正的，因此要加负号）
                (value_loss * self.value_loss_coef + action_loss -
                 dist_entropy * self.entropy_coef).backward()
                nn.utils.clip_grad_norm_(self.actor_critic.parameters(),
                                         self.max_grad_norm)
                self.optimizer.step()

                ...

        ...
```

表示RL algorithm的变量`agent`对应的类是`PPO`，由上可见，就是利用`rollouts`中的信息对`actor_critic`进行参数更新，更新的方式参照PPO的论文。

## Summary

至此，这份PPO代码的核心部分已经描述完毕。可见，代码的实现跟论文所说的还是有很多**细节**上的差异的，比如说对observation和reward的normalization，对return的计算，对advantage的normalization以及对gradient的clip等等。