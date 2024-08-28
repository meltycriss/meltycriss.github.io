---
title: 源码阅读《CarlaUE5 创建车辆（SpawnActor）和控制车辆（ApplyControl）》
date: 2024-08-28 23:49:16
tags: 
  - CarlaUE5
description:
  - 总结CarlaUE5里面，关于创建车辆（SpawnActor）和控制车辆（ApplyControl）的相关代码
categories:
  - 源码阅读
---

# 前言

书接上回，（{% post_link source-carlaue5-vehicle-1 源码阅读《CarlaUE5 UChaosVehicleMovementComponent关键调用链》 %}）总结完`MovementComponent`是怎么被调用的，相当于把核心部分的调用总结了，本文就把头尾都补充一下：
- 头：
    - 哪里控制可以创建什么车辆
    - 当client起`spawn_actor`请求时，车辆是怎么创建出来的
- 尾：
    - 当client发起`apply_control`请求时，控制指令是怎么驱动车辆的
    - client又是如何拿到更新后的车辆信息的

加上本文的内容之后，关于CarlaUE5的整体框架总结应该算是比较完整了：
- 关于构建客户端：{% post_link tech-windows-ue5-carla-build 技术总结《在Win11上构建UE5版本的Carla》 %}
- 关于整体框架：{% post_link source-carlaue5-architecture 源码阅读《CarlaUE5整体架构》 %}
- 关于控制移动的`MovementComponent`：{% post_link source-carlaue5-vehicle-1 源码阅读《CarlaUE5 UChaosVehicleMovementComponent关键调用链》 %}
- 关于创建车辆以及控制车辆：本文

下面就先介绍头，再介绍尾：

# 创建车辆`SpawnActor`

这一部分比较有意思的地方在于有不少东西是通过UE Editor的UI界面去控制的（标有blueprint的模块）

<div style="max-width: 100%; margin: 0 auto;">
  {% asset_img spawn_actor.png 创建车辆 %}
</div>

# 控制车辆`ApplyControl`

这一部分比较有意思的地方在于车辆（`UpdatedPrimitive`）是直接在simulator被更新的，我一开始还以为simulator只负责计算，实际更新会在别的地方

<div style="max-width: 100%; margin: 0 auto;">
  {% asset_img apply_control.png 控制车辆 %}
</div>


<!-- ```mermaid
sequenceDiagram
    autonumber
    participant OBS as WorldObserver.cpp
    participant CAR as CarlaWheeledVehicle.cpp
    participant MC as DefaultMovementComponent.cpp
    participant MGR as ChaosVehicleManager.cpp
    participant VMC as UChaosVehicleMovementComponent
    participant CB as ChaosVehicleManagerAsyncCallback.cpp
    participant SIM as UChaosVehicleSimulation

    note over CAR : In constructor
    CAR ->> MC : UDefaultMovementComponent::<br>CreateDefaultMovementComponent(this)
    note over CAR : In ApplyVehicleControl (called by CarlaServer.cpp): <br> InputControl.Control = Control;
    note over CAR : In FlushVehicleControl (called by WheeledVehicleAIController.cpp)
    CAR ->> MC : BaseMovementComponent->ProcessControl(<br>InputControl.Control)

    MC ->> VMC : MovementComponent->SetThrottleInput(Control.Throttle)
    VMC ->> CB : Update : PASS ANY INUTS TO <br>THE PHYSICS THREAD SIMULATION
    CB ->> SIM : TickVehicle(UWorld* WorldIn, <br>float DeltaTime, <br>const FChaosVehicleAsyncInput& InputData, <br>FChaosVehicleAsyncOutput& OutputData, <br>Chaos::FRigidBodyHandle_Internal* Handle)
    note over CB : IMPORTANT KNOWLEDGE!!!: <br> 1. In WheeledVehiclePawn.cpp: <br> VehicleMovementComponent->UpdatedComponent = Mesh (chassis of vehicle) <br> 2. UpdatedPrimitive ~= UpdatedComponent ~= RootComponent <br> 3. Chaos::FRigidBodyHandle_Internal* Handle <br> = UpdatedPrimitive->GetBodyInstance() <br> ->ActorHandle->GetPhysicsThreadAPI()

    note over SIM : state + input -- physics dynamics -> forces + output
    CB ->> SIM : ApplyDeferredForces(Chaos::FRigidBodyHandle_Internal* Handle)
    note over SIM : Apply computed forces to rigid bodies (Chaos::FRigidBodyHandle_Internal*)
    VMC ->> CB : ParallelUpdate : Access the async output data <br>from the Physics Thread
    MGR ->> VMC : PreTickGT
    note over VMC : Update FVehicleState	VehicleState from FBodyInstance* (sth like Chaos::FRigidBodyHandle_Internal*)
    
    OBS ->> CAR : BroadcastTick <br> FWorldObserver_Serialize
    note over CAR : Get vehicle states from RootComponent or members
```

```mermaid
sequenceDiagram
    autonumber
    participant SVR as CarlaServer.cpp
    participant ED as UE Editor
    participant GM as CarlaGameMode (blueprint)
    participant VF as VehicleFactory (blueprint)
    participant LC as BP_Lincoln2024 (blueprint)
    participant GMB as CarlaGameModeBase.cpp
    participant EPI as CarlaEpisode.cpp
    participant AD as ActorDispatcher.cpp

    GM ->> GMB : Derive from
    ED ->> GM : Set ActorFactories <br> (include VehicleFactory)
    ED ->> VF : Set Vehicles
    VF ->> LC : Add concrete vehicle
    note over LC : AIControllerClass is set to <br> WheeledVehicleAIController

    note right of GMB : ACarlaGameModeBase::InitGame <br> SpawnActorFactories
    GMB ->> EPI : Episode->RegisterActorFactory(*Factory)
    EPI ->> AD : ActorDispatcher->Bind(ActorFactory)
    note over AD : Bind(Definition, ActorFactory.SpawnActor) <br> SpawnFunctions.Emplace(Functor)

    note over SVR : spawn_actor RPC request
    SVR ->> EPI : Episode->SpawnActorWithInfo(Transform, std::move(Description))
    EPI ->> AD : ActorDispatcher->SpawnActor(<br>LocalTransform, thisActorDescription, DesiredId)
    note over AD : FActorSpawnResult Result = <br> SpawnFunctions[Description.UId - 1](Transform, Description)
``` -->
