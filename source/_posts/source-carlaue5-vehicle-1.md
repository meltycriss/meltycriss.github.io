---
title: 源码阅读《CarlaUE5 UChaosVehicleMovementComponent关键调用链》
date: 2024-08-24 22:02:58
tags: 
  - CarlaUE5
description:
  - 总结CarlaUE5车辆核心模块UChaosVehicleMovementComponent的关键调用链
categories:
  - 源码阅读
---

# 前言

初衷是希望看在CarlaUE5里面，一个`apply_control`调用是怎么一步步影响到车辆的状态的。

逻辑上是个比较直接的功能，更新车辆的输入信息，tick推帧，返回车辆更新后的状态。

就是这么个功能，在CarlaUE5里面还实现得挺复杂。

其中最关键的模块是UE5的`UChaosVehicleMovementComponent`，本文主要介绍这个模块更新车辆信息时的关键调用链，即`UChaosVehicleMovementComponent`是怎么将input传给simulator得到output的。

至于simulator的具体逻辑，以及怎么将`apply_control`中的input传给`UChaosVehicleMovementComponent`先按下不表。

下面按照核心类关系图和核心类时序图两部分展开。

# 核心类关系图

核心的类如下图所示：
- `FChaosVehicleManager`：主入口，运行在Game Thread，负责创建/提交运行在Physics Thread的async task
- `UChaosVehicleMovementComponent`：逻辑实现类，管理着车辆的输入以及状态，simulator也在这个类里面
- `FChaosVehicleManagerAsyncCallback`：async task类，由`FChaosVehicleManager`创建，会在Physics Thread被调用执行，一个task就负责所有车辆的更新
- `FChaosVehicleManagerAsyncInput/Output`：负责`FChaosVehicleManager`和`FChaosVehicleManagerAsyncCallback`的通信，里面包含所有车辆的信息
- `FChaosVehicleAsyncInput/Output`：负责`FChaosVehicleManagerAsyncCallback`与`UChaosVehicleMovementComponent`的通信，是单辆车的信息

<div style="max-width: 100%; margin: 0 auto;">
  {% asset_img class.png 核心类关系图 %}
</div>

# 核心类时序图

<div style="max-width: 100%; margin: 0 auto;">
  {% asset_img sequence.png 核心类时序图 %}
</div>

看代码的时候最难以理解的是为什么`UChaosVehicleMovementComponent`需要绑定跟`FChaosVehicleManagerAsyncCallback`通信用的`FChaosVehicleAsyncInput/Output`，这个东西的用法是：
- 将input成员变量（比如油门大小）赋值给`FChaosVehicleAsyncInput`
- `FChaosVehicleManagerAsyncCallback`通过`UChaosVehicleMovementComponent`中的simulator将`FChaosVehicleAsyncInput`转为`FChaosVehicleAsyncOutput`
- 从`FChaosVehicleAsyncOutput`获取更新后的状态赋值给output成员变量（比如引擎力矩）

我感觉完全可以抛开`FChaosVehicleAsyncInput/Output`，直接在`UChaosVehicleMovementComponent`的成员这一层里完成input到output的转换，毕竟input / output / simulator都是成员。

我现在能想到的解释是：这样子做，主视角是`FChaosVehicleManagerAsyncCallback`（知道input / output以及调用的函数），可以在这一层做一些统一的修改（比如修改传给simulator的input）。



<!-- ```mermaid
classDiagram
    direction TD
    class FChaosVehicleAsyncInput {
	    UChaosVehicleMovementComponent* Vehicle
	    Chaos::FSingleParticlePhysicsProxy* Proxy
	    FPhysicsVehicleInputs PhysicsInputs

	    virtual Simulate(args) FChaosVehicleAsyncOutput
	    virtual ApplyDeferredForces(args)
    }

    class FChaosVehicleAsyncOutput {
	    bool bValid
	    FPhysicsVehicleOutput VehicleSimOutput
    }

    class UChaosVehicleMovementComponent {
	    FChaosVehicleAsyncInput CurAsyncInput   //input of async callback task
	    FChaosVehicleAsyncOutput CurAsyncOutput //output of async callback task
	    float ThrottleInput     //physics simulation input
	    UChaosVehicleSimulation VehicleSimulationPT     //physics simulation codes
	    FPhysicsVehicleOutput PVehicleOutput	//physics simulation output
    }

    class FChaosVehicleManagerAsyncInput {
	    TArray~FChaosVehicleAsyncInput*~ VehicleInputs
    }

    class FChaosVehicleManagerAsyncOutput {
    	TArray~FChaosVehicleAsyncOutput*~ VehicleOutputs
    }

    class FChaosVehicleManagerAsyncCallback {
        FChaosVehicleManagerAsyncInput* input
        FChaosVehicleManagerAsyncOutput* output
	    ProcessInputs_Internal(int32 PhysicsStep)
	    OnPreSimulate_Internal()
    }

    class FChaosVehicleManager {
	    TArray~UChaosVehicleMovementComponent*~ Vehicles
	    FChaosVehicleManagerAsyncCallback* AsyncCallback
	    AddVehicle(UChaosVehicleMovementComponent* Vehicle)
    	Update(FPhysScene* PhysScene, float DeltaTime)
    }

    FChaosVehicleManager --> UChaosVehicleMovementComponent
    FChaosVehicleManager --> FChaosVehicleManagerAsyncCallback
    FChaosVehicleManagerAsyncCallback --> FChaosVehicleManagerAsyncInput
    FChaosVehicleManagerAsyncCallback --> FChaosVehicleManagerAsyncOutput
    FChaosVehicleManagerAsyncInput --> FChaosVehicleAsyncInput
    FChaosVehicleManagerAsyncOutput --> FChaosVehicleAsyncOutput
    UChaosVehicleMovementComponent --> FChaosVehicleAsyncInput
    UChaosVehicleMovementComponent --> FChaosVehicleAsyncOutput

```

note for FChaosVehicleManager "Main entry, run on game thread"
note for UChaosVehicleMovementComponent "Class including input / actual physics / output"
note for FChaosVehicleManagerAsyncCallback "AsyncCallback task, run on physics thread"
note for FChaosVehicleManagerAsyncInput "General input for AsyncCallback task"
note for FChaosVehicleAsyncInput "Actual input for MovementComponent task"

```mermaid
sequenceDiagram
    autonumber
    participant MGR as ChaosVehicleManager.cpp
    participant CB as ChaosVehicleManagerAsyncCallback.cpp
    participant MC as ChaosVehicleMovementComponent.cpp

    rect rgb(191, 223, 255)
    loop Game thread
        rect rgb(200, 150, 255)
        note right of MGR: FChaosVehicleManager::Update

        MGR ->> MC : Vehicle->Update(DeltaTime) // PASS ANY INUTS TO THE PHYSICS THREAD SIMULATION IN HERE
        note over MC: update FPhysicsVehicleInputs PhysicsInputs <br> in FChaosVehicleAsyncInput* CurAsyncInput <br> from MovementComponent's members (e.g., ThrottleInput)
        end

        rect rgb(200, 150, 255)
        note right of MGR: FChaosVehicleManager::ScenePreTick
        MGR ->> MC : Vehicles[i]->PreTickGT(DeltaTime)
        end

        rect rgb(200, 150, 255)
        note right of MGR: FChaosVehicleManager::ParallelUpdateVehicles

	    MGR ->> CB : FChaosVehicleManagerAsyncInput* AsyncInput = <br> AsyncCallback->GetProducerInputData_External()
        MGR ->> CB : FChaosVehicleManagerAsyncOutput LatestOutput ~= <br> AsyncCallback->PopOutputData_External()
        MGR ->> MC : AsyncInput->VehicleInputs.Add(Vehicle->SetCurrentAsyncInputOutput(LatestOutput, ...))
        note over MC : bind CurAsyncInput / CurAsyncOutput to AsyncCallBack's Input / Output

	    MGR ->> MC : Vehicle->ParallelUpdate(DeltaSeconds) // Access the async output data from the Physics Thread 
        note over MC: update MovementComponent's member <br> FPhysicsVehicleOutput PVehicleOutput <br> from VehicleSimOutput <br> in FChaosVehicleAsyncOutput* CurAsyncOutput (e.g., EngineTorque)
        end
    end
    end

    
    rect rgb(191, 223, 255)
    loop Physics thread
        rect rgb(200, 150, 255)
        note right of CB : FChaosVehicleManagerAsyncCallback::ProcessInputs_Internal
        CB ->> MC : const FChaosVehicleManagerAsyncInput* AsyncInput = GetConsumerInput_Internal() <br> for (const TUniquePtr<FChaosVehicleAsyncInput>& VehicleInput : AsyncInput->VehicleInputs) <br> UChaosVehicleSimulation* VehicleSim = VehicleInput->Vehicle->VehicleSimulationPT.Get() <br> VehicleSim->VehicleInputs = VehicleInput->PhysicsInputs.NetworkInputs.VehicleInputs
        end

        rect rgb(200, 150, 255)
        note right of CB : FChaosVehicleManagerAsyncCallback::OnPreSimulate_Internal
        CB ->> MC : const FChaosVehicleManagerAsyncInput* Input = GetConsumerInput_Internal() <br> FChaosVehicleManagerAsyncOutput& Output = GetProducerOutputData_Internal() <br> const TArray<TUniquePtr<FChaosVehicleAsyncInput>>& InputVehiclesBatch = Input->VehicleInputs <br> TArray<TUniquePtr<FChaosVehicleAsyncOutput>>& OutputVehiclesBatch = Output.VehicleOutputs <br> const FChaosVehicleAsyncInput& VehicleInput = *InputVehiclesBatch[Idx] <br> OutputVehiclesBatch[Idx] = VehicleInput.Simulate(World, DeltaTime, SimTime, bWake)
        note over MC : VehicleSimulationPT->TickVehicle(args) 
        CB ->> MC : VehicleInput->ApplyDeferredForces(Handle) 
        note over MC : VehicleSimulationPT->ApplyDeferredForces(RigidHandle)
        end
    end
    end
``` -->