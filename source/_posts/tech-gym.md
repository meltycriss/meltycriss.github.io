---
title: 技术总结《OpenAI Gym》
date: 2018-03-26 12:01:12
tags:
	- Gym
categories:
	- 技术总结
description:
	- 关于OpenAI Gym的一些技术总结
---

本文首先介绍Gym的**核心函数调用链**，然后介绍如何**创建自定义的Gym环境**，最后给出一些使用Gym过程中碰到的**问题及其解决方案**。

## Gym核心函数调用链

一般来说，使用Gym的代码如下：

```python
# main.py

import gym

def choose_action(o):
	...

env = gym.make('CartPole-v0')
o = env.reset()
while True:
	a = choose_action(o)
	o_, r, done, info = env.step(a)
	o = o_
	if done:
		break
```

可见，关键的函数有：

- `env = gym.make('CartPole-v0')`
- `env.reset()`
- `env.step(a)`


---

我们先关注`env.reset()`和`env.step(a)`。这两个函数是超类`Env`的成员函数，`Env`的相关代码如下：

```python
# gym/core.py

class Env(object):
	...

    # Override in ALL subclasses
    def _step(self, action): raise NotImplementedError
    def _reset(self): raise NotImplementedError

    ...

    def step(self, action):
        return self._step(action)

    def reset(self):
        return self._reset()

    ...
```

可以看到这两个函数依赖于子类的`_reset(self)`和`_step(self, action)`实现，子类`CartPoleEnv`的相关代码如下：

```python
# gym/envs/classic_control/CartPole.py

class CartPoleEnv(gym.Env):
	...

    def _step(self, action):
        ...

    def _reset(self):
        ...

    ...
```

综上，`env.reset()`和`env.step(a)`实际上是调用子类的`_reset(self)`和`_step(self, action)`。

---

下面我们关注`gym.make('CartPole-v0')`，它的实现如下：


```python
# gym/envs/registration.py

# Have a global registry
registry = EnvRegistry()

...

def make(id):
    return registry.make(id)
```

可以看到`gym.make`依赖于类`EnvRegistry`的成员函数`make`，`EnvRegistry`的相关代码如下：

```python
# gym/envs/registration.py

class EnvRegistry(object):
    def __init__(self):
    	# 注册表
    	# key:	环境名称（e.g., 'CartPole-v0'）
    	# value:类型为EnvSpec，可以暂时理解为环境
        self.env_specs = {}

    def make(self, id):
        ...

        # 根据环境名称，通过成员函数找到对应的环境
        spec = self.spec(id)
        # 实例化环境
        env = spec.make()

        ...

        return env

    ...

    def spec(self, id):
        ...

    ...
```

可见类`EnvRegistry`的成员函数`make`依赖于类`EnvSpec`的成员函数`make`，`EnvSpec`的相关代码如下：

```python
# gym/envs/registration.py

def load(name):
    ...

# EnvSpec与Env之间的关系类似于说明商品规格的订单与商品之间的关系，
# 下面用一个例子来说明：
# 假设你网购看中了一款衣服，那么你会挑选该款衣服的颜色、码数，然后再下单。
# 在这个例子里面，那款衣服就是Env，而说明该款衣服颜色、码数的订单就是EnvSpec。
# 这就是为什么EnvRegistry.make(self, id)中，在得到spec之后还要再spec.make()，
# 因为EnvSpec并不是Env，正如订单不是衣服。
class EnvSpec(object):
    def __init__(self, id, entry_point=None, ...):
    	self.id = id
        ...
        self._entry_point = entry_point
        ...

    def make(self):
        ...

        # 动态加载环境类
        # 相当于以下代码
        # from self._entry_point import classA
        # cls = classA
        cls = load(self._entry_point)
        # 实例化环境
        env = cls(**self._kwargs)

        ...

        return env

    ...
```

至此，我们对Gym的核心函数调用链有了一个基本的了解：

- `gym.make(id)`：通过`EnvRegistry`中的注册表找到对应的`EnvSpec`，`EnvSpec`根据`entry_point`动态`import`对应的`Env`，并将其实例化；
- `env.reset()`和`env.step(a)`：子类的`_reset(self)`和`_step(self, action)`。

## 创建自定义环境

对Gym的核心函数调用链有了基本了解后，我们知道创建自定义环境的关键有两个：

- 第一个是搭建自己的`Env`子类`FooEnv`；
- 第二个是注册`FooEnv`（i.e., 将`FooEnv`添加到`registry.env_specs`中），使得`gym.make(id)`可以找到`FooEnv`。

[官方文档](https://github.com/openai/gym/tree/master/gym/envs)推荐的自定义环境目录结构如下：

```shell
gym-foo/
  README.md
  setup.py 			#将gym_foo这个package加到系统环境变量中
  gym_foo/ 			#核心部分
    __init__.py 	#注册FooEnv
    envs/
      __init__.py
      foo_env.py 	#实现FooEnv
```

实现`FooEnv`没什么特别的，就是根据自己的需求，实现`_step(self, action)`、`_reset(self)`等函数。

值得一提的是注册`FooEnv`，我们**无需自己实现**注册环境的代码，因为Gym已经有**现成的注册环境API**，我们只需要调用该API即可。在我们的自定义环境中，负责注册FooEnv的文件为`gym-foo/gym_foo/__init__.py`，它的内容如下：


```python
# gym-foo/gym_foo/__init__.py

from gym.envs.registration import register

register(
    id='foo-v0', 	# 环境名
    entry_point='gym_foo.envs:FooEnv', 	# 环境类，之后就根据这个路径动态import环境
)
```

可见，注册的关键是`register`函数，而`register`函数的实现如下：

```python
# gym/envs/registration.py

# Have a global registry
registry = EnvRegistry()

# Gym的注册环境API
def register(id, **kwargs):
    return registry.register(id, **kwargs)

def make(id):
    return registry.make(id)
```

可以看到`register`的实现依赖于类`EnvRegistry`的成员函数`register`，其相关代码如下：

```python
# gym/envs/registration.py

class EnvRegistry(object):
	...

    def register(self, id, **kwargs):
        ...
        # 将FooEnv对应的“订单”写到“注册表”上
        self.env_specs[id] = EnvSpec(id, **kwargs)

```

综上，我们可以通过API函数`register`注册自定义的环境`FooEnv`。

## 注意事项

### server render

假如你通过ssh连接server，在server上运行（i.e., `python main.py`）以下代码（关键点在使用`env.render()`保存录像）：
```python
# main.py

import gym
from gym import wrappers

env = gym.make('CartPole-v0')
env = wrappers.Monitor(env, 'video')
for i_episode in range(20):
    observation = env.reset()
    for t in range(100):
        env.render()
        action = env.action_space.sample()
        observation, reward, done, info = env.step(action)
        if done:
            break
```
那么你会得到一个报错，报错的信息大概是`pyglet.canvas.xlib.NoSuchDisplayException: Cannot connect to "None"`。

原因大概是`env.render()`**需要图形界面**（就是弹出来的那个框框），而当你使用ssh连接server时是没有图形界面的。因此我们需要一个**虚拟的图形界面**，而`xvfb-run`就是一个提供虚拟图形界面的工具。

所以我们需要使用`xvfb-run -a -s "-screen 0 1400x900x24 +extension RANDR" -- python main.py`来运行我们的代码。

一般来说，运行上述指令是会报错的，报错的信息大概是`pyglet requires an X server with GLX`，主要原因在于**显卡驱动以及cuda的安装有问题**，没有加`--no-opengl`的flag。解决方案可以参考[这里](https://gist.github.com/8enmann/931ec2a9dc45fde871d2139a7d1f2d78)和[这里](https://davidsanwald.github.io/2016/11/13/building-tensorflow-with-gpu-support.html )

### 保存每一段episode的录像

`wrappers.Monitor`默认不会保存所有episode的录像，但我们可以通过以下代码来设置保存所有episode的录像：

```python
env = wrappers.Monitor(env, 'video', video_callable=lambda episode_id: True)
```

### 动态修改episode的最大step

`env._max_episode_steps = xxx`。注意，这仅当env的类型为`TimeLimit`时可用。

### 关于wrapper

- 相同的两个wrapper不能叠加（e.g., `Monitor`不可以和`Monitor`叠加，但是`Monitor`可以和`TimeLimit`叠加），否则会报double wrapper的错。
- 在注册`FooEnv`时，加不加`max_episode_steps=xxx`会影响返回的`Env`的类型。假如加了，返回的是`TimeLimit`类型的wrapper；假如不加，返回的就是裸的`FooEnv`。
- `Monitor`里面有两个`recorder`，一个是`stat_recorder`，用于保存数据（reward之类的）；另一个是`video_recorder`，用于录像。`Monitor`会在每一次调用`env.reset`和`env.step`之后调用`render`

### 屏蔽log信息

```python
# main.py

import logging

# suppress INFO level logging 'Making new env: ...'
logging.getLogger('gym.envs.registration').setLevel(logging.WARNING)
# suppress INFO level logging 'Starting new video recorder writing to ...'
logging.getLogger('gym.monitoring.video_recorder').setLevel(logging.WARNING)
# suppress INFO level logging 'Creating monitor directory ...'
logging.getLogger('gym.wrappers.monitoring').setLevel(logging.WARNING)

```