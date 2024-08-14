---
title: 源码阅读《CarlaUE5整体架构》
date: 2024-08-13 22:36:17
tags: 
  - CarlaUE5
description:
  - 总结CarlaUE5的整体架构
categories:
  - 源码阅读
---

# 前言

总结了一下CarlaUE5的整体代码架构，该总结基于以下代码：

- repo: https://github.com/carla-simulator/carla.git
- branch: `ue5-dev`
- commit: `141a8a2`

# 整体架构

<!-- <div style="width:400px; margin-left:auto; margin-right:auto;" > -->
<div style="max-width: 100%; margin: 0 auto;">
  {% asset_img carla.png 整体架构 %}
</div>

# Server架构

<!-- <div style="width:auto; margin-left:auto; margin-right:auto;" > -->
<div style="max-width: 100%; margin: 0 auto;">
  {% asset_img server.png Server架构 %}
</div>

# Client架构

<!-- <div style="width:400px; margin-left:auto; margin-right:auto;" > -->
<div style="max-width: 100%; margin: 0 auto;">
  {% asset_img client.png Client架构 %}
</div>

# PythonAPI

<!-- <div style="width:400px; margin-left:auto; margin-right:auto;" > -->
<div style="max-width: 100%; margin: 0 auto;">
  {% asset_img pythonapi.png PythonAPI架构 %}
</div>

# 一些C++知识点

## `weak_ptr`

- motivation：只有`shared_ptr`的话会出现循环引用的问题
- 具体来说，就是搞一个阉割版的，不会影响计数的“智能指针”，也就是`weak_ptr`了
- `weak_ptr`不能直接用，只能先通过`.lock()`提级（构造）为`shared_ptr`之后再用
- [这个blog](https://blog.csdn.net/qq_38410730/article/details/105903979)和[这个blog](https://blog.csdn.net/Xiejingfa/article/details/50772571)讲解得挺好的

## `atomic`

- motivation：针对变量的高效线程安全读写工具
- 相比`lock`, `mutex`这些要跟跟内核打交道（有用户态和内核态的切换开销）的多线程工具，`atomic`属于硬件指令集的工具，效率更高
- 对于`atomic<int> ai;`，需要调用`ai.load()`和`ai.store(1)`进行读写
- 又由于编译器会对代码的实际执行顺序进行优化，`atomic`提供了一些指令去控制代码的执行顺序，具体可以参考[这个blog](https://www.cnblogs.com/kekec/p/14470150.html)

## `enable_shared_from_this`

- motivation：使得成员函数使用`shared_ptr<T>(this)`成为可能。假如没有额外支持的话，成员函数直接裸使用`shared_ptr<T>(this)`会导致`this`被析构两次
- 核心要做的事情就是使得对象知道管理自己的`shared_ptr`是什么。具体来说：
    - `esft`是个模板类，我们需要继承他。这个类里有一个`weak_ptr`成员，语意是指向管理本对象的`shared_ptr`；同时会提供一个`shared_from_this`方法，去调用`weak_ptr.lock()`
    - 在调用`make_shared`的时候，会调用`dynamic_cast`来检查当前要构造的对象是否继承自`esft`，假如是，就会将其`weak_ptr`对象指向`shared_ptr`
- 关于这部分，可以参考[这个blog](https://blog.csdn.net/caoshangpa/article/details/79392878)

## `mutable`

- motivation：允许`const`成员函数修改的成员变量

## `pImpl`

- motivation
    - 完全隐藏实现细节
    - 避免修改成员变量引起头文件变化
- 具体来说，就是
    - 在类的`private`里面前向声明一个`Impl`类
    - 成员变量只保留一个指向`Impl`类的`unique_ptr`

## `compare_exchange`

- motivation：解决多线程编程时，读到的值在要用的时候可能已经被修改的问题，主要用于条件赋值，否则直接用`store`就好了
- 思路就是在用的时候再验一下货。具体来说，就是搞了一个包含比较和赋值这两个指令的原子操作，只有当`self`跟`expected`相等的时候，`self`才会被赋值为`target`
- 这个也被叫做CAS(Compare and Swap)，属于无锁编程的概念。
- 可以参考[这个blog](https://mk.woa.com/q/295415/answer/123484?kmref=vkm_push)

<!-- ```mermaid
---
title: Carla Architecture
---
flowchart LR
    PY[Python API]
    C_CPP[C++ Carla Client]
    C_RPC[C++ RPC Client]
    S_RPC[C++ RPC Server]
    S_PLUGIN[C++ Carla UE5 Plugin]
    UE[UE]
    PY --- |boost python| C_CPP
    subgraph "Client (LibCarla)"
    C_CPP --- C_RPC
    end

    C_RPC <--RPC--> S_RPC

    subgraph Server
    S_RPC --- S_PLUGIN
    S_PLUGIN --- UE
    end
```

```mermaid
---
title: Server Architecture
---
sequenceDiagram
    autonumber
    participant UE as UE
    participant GM as CarlaGameModeBase.cpp
    participant GI as CarlaGameInstance.cpp
    participant NGN as CarlaEngine.cpp <br> (Define tick logic)
    participant OBS as WorldObserver.cpp <br> (Hold actor states)
    participant EPI as CarlaEpisode.cpp <br> (Hold UE APIs)
    participant SVR as CarlaServer.cpp <br> (RPC Server)
    participant CLIENT as RPC Client

    UE ->> GM : DefaultEngine.ini
    UE ->> GI : DefaultEngine.ini
    UE ->> GM : InitGame
    GM ->> GI : NotifyInitGame
    GI ->> NGN : NotifyInitGame
    NGN ->> UE : Register OnPreTick and OnPostTick
    NGN ->> SVR : Start RPC Server
    Note over SVR: Initialize RPC Server <br> FCarlaServer::FPimpl::BindActions()
    SVR -->> NGN : return BroadcastStream
    NGN ->> OBS : Bind stream <br> WorldObserver.SetStream(BroadcastStream);

    UE ->> GM : BeginPlay
    GM ->> EPI : Episode->InitializeAtBeginPlay()
    GM ->> GI : NotifyBeginEpisode(UCarlaEpisode &)
    GI ->> NGN : NotifyBeginEpisode
    NGN ->> SVR : NotifyBeginEpisode
    SVR ->> EPI : Bind <br> Pimpl->Episode = &Episode

    loop UE Tick
        rect rgb(191, 223, 255)
            note over UE, GM : Client to Server communication
            UE ->> NGN : Call registered OnPreTick
            NGN ->> SVR : Server.RunSome(1u)
            CLIENT ->> SVR : RPC requests
            SVR ->> EPI : Call Episode's methods
        end

        rect rgb(191, 223, 255)
            note over UE, GM : Server to Client communication
            UE ->> NGN : Call registered OnPostTick
            NGN ->> OBS : WorldObserver.BroadcastTick
            OBS ->> SVR : SerializeAndSend
            SVR ->> CLIENT : Broadcast
        end
    end
```

```mermaid
---
title: Client Architecture
---
sequenceDiagram
    autonumber
    participant CLIENT as Client.h <br> (High-level Carla Client)
    participant WORLD as World.cpp <br> (Core API for user)
    participant SIM as detail/Simulator.cpp
    participant EPI as detail/Episode.cpp
    participant EPI_PXY as detail/EpisodeProxy.cpp <br> (Util Class to wrap detail/Simulator.cpp, help handling different pointer types)
    participant RPC_CLIENT as detail/Client.cpp <br> (RPC Client)
    participant SVR as RPC Server in UE

    CLIENT ->> SIM : Client():_simulator(new detail::Simulator(host, port, worker_threads)
    SIM ->> RPC_CLIENT : Simulator():_client(host, port, worker_threads)

    rect rgb(191, 223, 255)
        note right of CLIENT: Client::GetWorld() {return World(_simulator->GetCurrentEpisode())}

        CLIENT ->> SIM : _simulator->GetCurrentEpisode()
        rect rgb(200, 150, 255)
            note right of CLIENT: Simulator::GetCurrentEpisode()
            SIM ->> EPI : _episode = std::make_shared<Episode>(_client, std::weak_ptr<Simulator>(shared_from_this()));
            SIM ->> EPI : _episode->Listen();
            EPI ->> RPC_CLIENT : Register callback to update EpisodeState(snapshot): _client.SubscribeToStream
            note over RPC_CLIENT :     _pimpl->streaming_client.Subscribe
            SIM ->> EPI_PXY : epi_pxy = EpisodeProxy{shared_from_this()}
            note over EPI_PXY: EpisodeProxy():_simulator(std::move(simulator))
            SIM -->> CLIENT : return epi_pxy
        end

        CLIENT ->> WORLD : World(_simulator->GetCurrentEpisode())
        note over WORLD: World():_episode(std::move(episode))
    end

    rect rgb(191, 223, 255)   
        note over WORLD: Client to Server communication 
        note right of WORLD: e.g., World::SpawnActor() {_episode.Lock()->SpawnActor}
        WORLD ->> EPI_PXY : _episode.Lock()
        EPI_PXY -->> WORLD : return Load(_simulator) <br> Load(ptr) {return ptr.load() / ptr.lock()}
        WORLD ->> SIM : _episode.Lock().SpawnActor
        SIM ->> RPC_CLIENT : _client.SpawnActor
        RPC_CLIENT ->> SVR: _pimpl->CallAndWait<rpc::Actor>("spawn_actor", description, transform);
    end

    rect rgb(191, 223, 255)
    note over EPI: Server to Client communication
    loop
    SVR ->> RPC_CLIENT : Stream broadcast
    RPC_CLIENT ->> EPI : Call registered callback
    end
    end
```

```mermaid
---
title: Python API
---
flowchart LR
    LIBCARLA[LibCarla]
    H[PythonAPI.h]
    CPP[PythonAPI.cpp]
    CLIENT[PythonAPI\carla\src\Client.cpp]
    WORLD[PythonAPI\carla\src\World.cpp]

    LIBCARLA ---|include| H

    subgraph "Main of boost python"
    H --- CPP 
    end

    subgraph Exporter
    CPP --- CLIENT 
    CPP --- WORLD
    CPP --- ...
    end
``` -->