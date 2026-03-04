# Animation之旅_009_动画通知系统AnimNotify深度解析

## 前言

在游戏开发中，动画不仅仅是视觉表现，它往往需要与游戏逻辑紧密配合。想象一个角色挥剑的动画——我们需要在剑划过的精确时刻播放音效、生成粒子特效、进行伤害判定。这就是**动画通知系统 (AnimNotify)** 存在的意义：它是连接动画播放与游戏逻辑的核心桥梁。

本章将深入剖析 UE5 的 AnimNotify 系统，从架构设计到源码实现，带您全面理解这一关键系统。

---

## 一、AnimNotify 系统概述

### 1.1 什么是 AnimNotify？

AnimNotify 是 UE5 动画系统中的事件触发机制，允许开发者在动画播放的特定时间点执行自定义逻辑。它的典型应用场景包括：

- **音效同步**：脚步声、攻击音效、受击音效
- **视觉特效**：武器拖尾、技能粒子、环境交互效果
- **游戏逻辑**：伤害判定、碰撞检测、状态切换
- **动画控制**：蒙太奇分支、动画同步标记

### 1.2 系统架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    AnimNotify System                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────┐    ┌──────────────────┐                  │
│  │ UAnimNotify   │    │ UAnimNotifyState │                  │
│  │ (单次触发)     │    │ (持续时间)        │                  │
│  └───────┬───────┘    └────────┬─────────┘                  │
│          │                     │                             │
│          ▼                     ▼                             │
│  ┌─────────────────────────────────────────┐                │
│  │         FAnimNotifyEvent                 │                │
│  │  (通知事件数据结构)                        │                │
│  └────────────────┬────────────────────────┘                │
│                   │                                          │
│                   ▼                                          │
│  ┌─────────────────────────────────────────┐                │
│  │         FAnimNotifyQueue                 │                │
│  │  (通知队列 - 管理/过滤/调度)              │                │
│  └────────────────┬────────────────────────┘                │
│                   │                                          │
│                   ▼                                          │
│  ┌─────────────────────────────────────────┐                │
│  │         UAnimInstance                    │                │
│  │  TriggerAnimNotifies() - 触发执行         │                │
│  └─────────────────────────────────────────┘                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、核心数据结构

### 2.1 FAnimNotifyEvent - 通知事件

`FAnimNotifyEvent` 是存储在动画资产中的通知数据结构，定义在 `AnimTypes.h`：

```cpp
// Engine/Source/Runtime/Engine/Public/Animation/AnimTypes.h

USTRUCT(BlueprintType)
struct FAnimNotifyEvent : public FAnimLinkableElement
{
    GENERATED_USTRUCT_BODY()

    // 通知显示名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotifyEvent)
    FName NotifyName;

    // 单次触发的通知类实例
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=AnimNotifyEvent)
    TObjectPtr<class UAnimNotify> Notify;

    // 持续状态的通知类实例
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=AnimNotifyEvent)
    TObjectPtr<class UAnimNotifyState> NotifyStateClass;

    // 持续时间 (仅 NotifyState 有效)
    UPROPERTY()
    float Duration;

    // 触发时间偏移 (用于精确控制)
    UPROPERTY()
    float TriggerTimeOffset;

    // 结束触发时间偏移
    UPROPERTY()
    float EndTriggerTimeOffset;

    // 触发权重阈值 (低于此权重不触发)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotifyEvent)
    float TriggerWeightThreshold;

    // 蒙太奇 Tick 类型
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotifyEvent)
    TEnumAsByte<EMontageNotifyTickType::Type> MontageTickType;

    // 触发概率 (0.0 - 1.0)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotifyEvent)
    float NotifyTriggerChance;

    // 过滤类型
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotifyEvent)
    TEnumAsByte<ENotifyFilterType::Type> NotifyFilterType;

    // LOD 过滤级别
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotifyEvent)
    int32 NotifyFilterLOD;

    // 是否在 Dedicated Server 上触发
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotifyEvent)
    bool bTriggerOnDedicatedServer;

    // 是否在 Follower 上触发 (同步组)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotifyEvent)
    bool bTriggerOnFollower;

    // 轨道索引
    UPROPERTY()
    int32 TrackIndex;

    // 唯一标识符
    UPROPERTY()
    FGuid Guid;

    // 关键方法
    float GetTriggerTime() const;
    float GetEndTriggerTime() const;
    float GetDuration() const;
    void SetDuration(float NewDuration);
    bool IsBranchingPoint() const;
    FName GetNotifyEventName() const;
};
```

### 2.2 触发时间计算

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimTypes.cpp

float FAnimNotifyEvent::GetTriggerTime() const
{
    // 基础时间 + 偏移量
    return GetTime() + TriggerTimeOffset;
}

float FAnimNotifyEvent::GetEndTriggerTime() const
{
    if (!NotifyStateClass && (EndTriggerTimeOffset != 0.f))
    {
        UE_LOG(LogAnimNotify, Log, TEXT("Anim Notify %s is non state, but has an EndTriggerTimeOffset %f!"), 
               *NotifyName.ToString(), EndTriggerTimeOffset);
    }
    
    // 状态通知：开始时间 + 持续时间 + 结束偏移
    return NotifyStateClass ? (GetTriggerTime() + GetDuration() + EndTriggerTimeOffset) : GetTriggerTime();
}

float FAnimNotifyEvent::GetDuration() const
{
    return NotifyStateClass ? EndLink.GetTime() - GetTime() : 0.0f;
}
```

### 2.3 FAnimNotifyQueue - 通知队列

通知队列负责收集、过滤和分发待触发的通知：

```cpp
// Engine/Source/Runtime/Engine/Public/Animation/AnimNotifyQueue.h

USTRUCT()
struct FAnimNotifyQueue
{
    GENERATED_BODY()

    // 待触发的通知列表
    TArray<FAnimNotifyEventReference> AnimNotifies;

    // 来自蒙太奇的未过滤通知 (按蒙太奇名称分组)
    TMap<FName, FAnimNotifyArray> UnfilteredMontageAnimNotifies;

    // 当前预测的 LOD 级别
    int32 PredictedLODLevel;

    // 随机流 (用于概率触发)
    FRandomStream RandomStream;

    // 世界引用 (用于网络模式判断)
    TWeakObjectPtr<UWorld> World;

    // 核心方法
    void AddAnimNotify(const FAnimNotifyEvent* Notify, const UObject* NotifySource);
    void AddAnimNotifies(bool bSrcIsLeader, const TArray<FAnimNotifyEventReference>& NewNotifies, float InstanceWeight);
    void Reset(USkeletalMeshComponent* Component);
    
private:
    bool PassesFiltering(const FAnimNotifyEvent* Notify) const;
    bool PassesChanceOfTriggering(const FAnimNotifyEvent* Event) const;
};
```

### 2.4 过滤机制实现

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimNotifyQueue.cpp

bool FAnimNotifyQueue::PassesFiltering(const FAnimNotifyEvent* Notify) const
{
    switch (Notify->NotifyFilterType)
    {
        case ENotifyFilterType::NoFiltering:
            return true;
            
        case ENotifyFilterType::LOD:
            // 只有当通知的 LOD 阈值大于当前预测 LOD 时才触发
            return Notify->NotifyFilterLOD > PredictedLODLevel;
            
        default:
            ensure(false); // 未知过滤类型
    }
    return true;
}

bool FAnimNotifyQueue::PassesChanceOfTriggering(const FAnimNotifyEvent* Event) const
{
    // NotifyState 总是触发，普通 Notify 受概率控制
    return Event->NotifyStateClass ? true : RandomStream.FRandRange(0.f, 1.f) < Event->NotifyTriggerChance;
}

void FAnimNotifyQueue::AddAnimNotifiesToDest(
    bool bSrcIsLeader, 
    const TArray<FAnimNotifyEventReference>& NewNotifies, 
    TArray<FAnimNotifyEventReference>& DestArray, 
    const float InstanceWeight)
{
    bool bIsDedicatedServer = World.IsValid() ? (World->GetNetMode() == NM_DedicatedServer) : false;
    
    for (const FAnimNotifyEventReference& NotifyRef : NewNotifies)
    {
        const FAnimNotifyEvent* Notify = NotifyRef.GetNotify();
        if (Notify && (bSrcIsLeader || Notify->bTriggerOnFollower))
        {
            // 检查服务器权限
            const bool bPassesDedicatedServerCheck = Notify->bTriggerOnDedicatedServer || !bIsDedicatedServer;
            
            // 权重阈值检查 + 过滤检查 + 概率检查
            if (bPassesDedicatedServerCheck && 
                Notify->TriggerWeightThreshold <= InstanceWeight && 
                PassesFiltering(Notify) && 
                PassesChanceOfTriggering(Notify))
            {
                // NotifyState 只添加唯一实例，防止循环动画重复触发
                Notify->NotifyStateClass ? DestArray.AddUnique(NotifyRef) : DestArray.Add(NotifyRef);
            }
        }
    }
}
```

---

## 三、UAnimNotify - 单次触发通知

### 3.1 类定义

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimNotifies/AnimNotify.h

UCLASS(abstract, Blueprintable, const, hidecategories=Object, collapsecategories, MinimalAPI)
class UAnimNotify : public UObject
{
    GENERATED_UCLASS_BODY()

    // 蓝图可实现：获取自定义通知名称
    UFUNCTION(BlueprintNativeEvent)
    ENGINE_API FString GetNotifyName() const;

    // 蓝图可实现：接收通知事件
    UFUNCTION(BlueprintImplementableEvent, meta=(AutoCreateRefTerm="EventReference"))
    ENGINE_API bool Received_Notify(
        USkeletalMeshComponent* MeshComp, 
        UAnimSequenceBase* Animation, 
        const FAnimNotifyEventReference& EventReference) const;

#if WITH_EDITORONLY_DATA
    // 编辑器中的通知颜色
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotify)
    FColor NotifyColor;

    // 是否在编辑器中触发
    UPROPERTY(EditAnywhere, BlueprintReadWrite, AdvancedDisplay, Category=AnimNotify)
    bool bShouldFireInEditor;
#endif

    // C++ 虚函数：处理通知
    ENGINE_API virtual void Notify(
        USkeletalMeshComponent* MeshComp, 
        UAnimSequenceBase* Animation, 
        const FAnimNotifyEventReference& EventReference);

    // 分支点通知 (用于蒙太奇)
    ENGINE_API virtual void BranchingPointNotify(FBranchingPointNotifyPayload& BranchingPointPayload);

    // 默认触发权重阈值
    UFUNCTION(BlueprintNativeEvent)
    ENGINE_API float GetDefaultTriggerWeightThreshold() const;

    // 是否为原生分支点
    bool bIsNativeBranchingPoint;

protected:
    // Mesh 上下文 (用于 GetWorld)
    class USkeletalMeshComponent* MeshContext;
};
```

### 3.2 核心实现

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimNotify.cpp

UAnimNotify::UAnimNotify(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
    , MeshContext(NULL)
{
#if WITH_EDITORONLY_DATA
    NotifyColor = FColor(255, 200, 200, 255);
    bShouldFireInEditor = true;
#endif
    bIsNativeBranchingPoint = false;
}

void UAnimNotify::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                         const FAnimNotifyEventReference& EventReference)
{
    // 设置上下文
    USkeletalMeshComponent* PrevContext = MeshContext;
    MeshContext = MeshComp;
    
    // 调用蓝图实现
    Received_Notify(MeshComp, Animation, EventReference);
    
    // 恢复上下文
    MeshContext = PrevContext;
}

void UAnimNotify::BranchingPointNotify(FBranchingPointNotifyPayload& BranchingPointPayload)
{
    const FAnimNotifyEventReference EventReference;
    Notify(BranchingPointPayload.SkelMeshComponent, BranchingPointPayload.SequenceAsset, EventReference);
}

class UWorld* UAnimNotify::GetWorld() const
{
    // 通过 MeshContext 获取世界
    return (MeshContext ? MeshContext->GetWorld() : NULL);
}

FString UAnimNotify::GetNotifyName_Implementation() const
{
    FString NotifyName;
    
#if WITH_EDITORONLY_DATA
    if (UObject* ClassGeneratedBy = GetClass()->ClassGeneratedBy)
    {
        // 蓝图类型：使用生成类的名称
        NotifyName = ClassGeneratedBy->GetName();
    }
    else
#endif
    {
        // 原生类型：使用类名
        NotifyName = GetClass()->GetName();
    }
    
    // 移除前缀
    NotifyName.ReplaceInline(TEXT("AnimNotify_"), TEXT(""), ESearchCase::CaseSensitive);
    return NotifyName;
}

float UAnimNotify::GetDefaultTriggerWeightThreshold_Implementation() const
{
    return ZERO_ANIMWEIGHT_THRESH;
}
```

---

## 四、UAnimNotifyState - 持续状态通知

### 4.1 类定义

`UAnimNotifyState` 支持有持续时间的通知，包含三个关键回调：

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimNotifies/AnimNotifyState.h

UCLASS(abstract, editinlinenew, Blueprintable, const, hidecategories=Object, collapsecategories, MinimalAPI)
class UAnimNotifyState : public UObject
{
    GENERATED_UCLASS_BODY()

    // 蓝图可实现：获取自定义名称
    UFUNCTION(BlueprintNativeEvent)
    ENGINE_API FString GetNotifyName() const;

    // 蓝图事件：开始
    UFUNCTION(BlueprintImplementableEvent, meta=(AutoCreateRefTerm="EventReference"))
    ENGINE_API bool Received_NotifyBegin(
        USkeletalMeshComponent* MeshComp, 
        UAnimSequenceBase* Animation, 
        float TotalDuration, 
        const FAnimNotifyEventReference& EventReference) const;
    
    // 蓝图事件：Tick
    UFUNCTION(BlueprintImplementableEvent, meta=(AutoCreateRefTerm="EventReference"))
    ENGINE_API bool Received_NotifyTick(
        USkeletalMeshComponent* MeshComp, 
        UAnimSequenceBase* Animation, 
        float FrameDeltaTime, 
        const FAnimNotifyEventReference& EventReference) const;

    // 蓝图事件：结束
    UFUNCTION(BlueprintImplementableEvent, meta=(AutoCreateRefTerm="EventReference"))
    ENGINE_API bool Received_NotifyEnd(
        USkeletalMeshComponent* MeshComp, 
        UAnimSequenceBase* Animation, 
        const FAnimNotifyEventReference& EventReference) const;

#if WITH_EDITORONLY_DATA
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=AnimNotify)
    FColor NotifyColor;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, AdvancedDisplay, Category=AnimNotify)
    bool bShouldFireInEditor;
#endif

    // C++ 虚函数
    ENGINE_API virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                        float TotalDuration, const FAnimNotifyEventReference& EventReference);
    ENGINE_API virtual void NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                       float FrameDeltaTime, const FAnimNotifyEventReference& EventReference);
    ENGINE_API virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                      const FAnimNotifyEventReference& EventReference);

    // 分支点版本
    ENGINE_API virtual void BranchingPointNotifyBegin(FBranchingPointNotifyPayload& BranchingPointPayload);
    ENGINE_API virtual void BranchingPointNotifyTick(FBranchingPointNotifyPayload& BranchingPointPayload, float FrameDeltaTime);
    ENGINE_API virtual void BranchingPointNotifyEnd(FBranchingPointNotifyPayload& BranchingPointPayload);

    bool bIsNativeBranchingPoint;
};
```

### 4.2 核心实现

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimNotifyState.cpp

UAnimNotifyState::UAnimNotifyState(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
#if WITH_EDITORONLY_DATA
    NotifyColor = FColor(200, 200, 255, 255);
    bShouldFireInEditor = true;
#endif
    bIsNativeBranchingPoint = false;
}

void UAnimNotifyState::NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                   float TotalDuration, const FAnimNotifyEventReference& EventReference)
{
    Received_NotifyBegin(MeshComp, Animation, TotalDuration, EventReference);
}

void UAnimNotifyState::NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                  float FrameDeltaTime, const FAnimNotifyEventReference& EventReference)
{
    Received_NotifyTick(MeshComp, Animation, FrameDeltaTime, EventReference);
}

void UAnimNotifyState::NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                 const FAnimNotifyEventReference& EventReference)
{
    Received_NotifyEnd(MeshComp, Animation, EventReference);
}

void UAnimNotifyState::BranchingPointNotifyEnd(FBranchingPointNotifyPayload& BranchingPointPayload)
{
    FAnimNotifyEventReference EventReference;
    
    // 添加结束上下文数据
    if (BranchingPointPayload.bReachedEnd)
    {
        EventReference.AddContextData<UE::Anim::FAnimNotifyEndDataContext>(true);
    }
    
    NotifyEnd(BranchingPointPayload.SkelMeshComponent, BranchingPointPayload.SequenceAsset, EventReference);
}
```

---

## 五、通知触发流程

### 5.1 UAnimInstance::TriggerAnimNotifies

这是整个通知系统的核心调度函数：

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimInstance.cpp

void UAnimInstance::TriggerAnimNotifies(float DeltaSeconds)
{
    SCOPE_CYCLE_COUNTER(STAT_AnimTriggerAnimNotifies);
    USkeletalMeshComponent* SkelMeshComp = GetSkelMeshComponent();

    // 1. 准备新的活跃状态数组
    TArray<FAnimNotifyEvent> NewActiveAnimNotifyState;
    NewActiveAnimNotifyState.Reserve(NotifyQueue.AnimNotifies.Num());
    
    TArray<FAnimNotifyEventReference> NewActiveAnimNotifyEventReference;
    NewActiveAnimNotifyEventReference.Reserve(NotifyQueue.AnimNotifies.Num());

    // 2. 收集新激活的 NotifyState
    TArray<const FAnimNotifyEvent*> NotifyStateBeginEvent;
    TArray<const FAnimNotifyEventReference*> NotifyStateBeginEventReference;

    // 3. 遍历队列中的所有通知
    for (int32 Index = 0; Index < NotifyQueue.AnimNotifies.Num(); Index++)
    {
        if (const FAnimNotifyEvent* AnimNotifyEvent = NotifyQueue.AnimNotifies[Index].GetNotify())
        {
            // 3.1 处理 AnimNotifyState
            if (AnimNotifyEvent->NotifyStateClass)
            {
                int32 ExistingItemIndex = INDEX_NONE;
                
                // 检查是否已在活跃列表中
                if (ActiveAnimNotifyState.Find(*AnimNotifyEvent, ExistingItemIndex))
                {
                    // 已存在，从旧列表移除 (将被加入新列表)
                    ActiveAnimNotifyState.RemoveAtSwap(ExistingItemIndex, EAllowShrinking::No);
                    ActiveAnimNotifyEventReference.RemoveAtSwap(ExistingItemIndex, EAllowShrinking::No);
                }
                else
                {
                    // 新激活的状态，记录以便后续调用 NotifyBegin
                    NotifyStateBeginEvent.Add(AnimNotifyEvent);
                    NotifyStateBeginEventReference.Add(&NotifyQueue.AnimNotifies[Index]);
                }
                
                // 添加到新的活跃列表
                NewActiveAnimNotifyState.Add(*AnimNotifyEvent);
                FAnimNotifyEventReference& EventRef = NewActiveAnimNotifyEventReference.Add_GetRef(NotifyQueue.AnimNotifies[Index]);
                EventRef.SetNotify(&NewActiveAnimNotifyState.Top());
                continue;
            }

            // 3.2 触发普通 AnimNotify
            TriggerSingleAnimNotify(NotifyQueue.AnimNotifies[Index]);
        }
    }

    // 4. 处理结束的 NotifyState (在旧列表中但不在新列表中)
    for (int32 Index = 0; Index < ActiveAnimNotifyState.Num(); ++Index)
    {
        const FAnimNotifyEvent& AnimNotifyEvent = ActiveAnimNotifyState[Index];
        const FAnimNotifyEventReference& EventReference = ActiveAnimNotifyEventReference[Index];
        
        if (AnimNotifyEvent.NotifyStateClass && ShouldTriggerAnimNotifyState(AnimNotifyEvent.NotifyStateClass))
        {
#if WITH_EDITOR
            if (!SkelMeshComp->IsA<UDebugSkelMeshComponent>() || AnimNotifyEvent.NotifyStateClass->ShouldFireInEditor())
#endif
            {
                TRACE_ANIM_NOTIFY(this, AnimNotifyEvent, End);
                AnimNotifyEvent.NotifyStateClass->NotifyEnd(SkelMeshComp, 
                    Cast<UAnimSequenceBase>(AnimNotifyEvent.NotifyStateClass->GetOuter()), EventReference);
            }
        }
        
        // 安全检查：NotifyEnd 可能触发了 Actor 销毁
        if (ActiveAnimNotifyState.IsValidIndex(Index) == false)
        {
            ensureMsgf(false, TEXT("ActiveAnimNotifyState invalidated by NotifyEnd"));
            return;
        }
    }

    // 5. 触发新激活的 NotifyState 的 NotifyBegin
    for (int32 Index = 0; Index < NotifyStateBeginEvent.Num(); Index++)
    {
        const FAnimNotifyEvent* AnimNotifyEvent = NotifyStateBeginEvent[Index];
        const FAnimNotifyEventReference* AnimNotifyEventReference = NotifyStateBeginEventReference[Index];
        
        if (ShouldTriggerAnimNotifyState(AnimNotifyEvent->NotifyStateClass))
        {
#if WITH_EDITOR
            if (!SkelMeshComp->IsA<UDebugSkelMeshComponent>() || AnimNotifyEvent->NotifyStateClass->ShouldFireInEditor())
#endif
            {
                TRACE_ANIM_NOTIFY(this, *AnimNotifyEvent, Begin);
                AnimNotifyEvent->NotifyStateClass->NotifyBegin(SkelMeshComp, 
                    Cast<UAnimSequenceBase>(AnimNotifyEvent->NotifyStateClass->GetOuter()), 
                    AnimNotifyEvent->GetDuration(), *AnimNotifyEventReference);
            }
        }
    }

    // 6. 切换活跃数组
    ActiveAnimNotifyState = MoveTemp(NewActiveAnimNotifyState);
    ActiveAnimNotifyEventReference = MoveTemp(NewActiveAnimNotifyEventReference);
    
    // 7. Tick 所有活跃的 NotifyState
    for (int32 Index = 0; Index < ActiveAnimNotifyState.Num(); Index++)
    {
        const FAnimNotifyEvent& AnimNotifyEvent = ActiveAnimNotifyState[Index];
        const FAnimNotifyEventReference& EventReference = ActiveAnimNotifyEventReference[Index];
        
        if (ShouldTriggerAnimNotifyState(AnimNotifyEvent.NotifyStateClass))
        {
#if WITH_EDITOR
            if (!SkelMeshComp->IsA<UDebugSkelMeshComponent>() || AnimNotifyEvent.NotifyStateClass->ShouldFireInEditor())
#endif
            {
                TRACE_ANIM_NOTIFY(this, AnimNotifyEvent, Tick);
                AnimNotifyEvent.NotifyStateClass->NotifyTick(SkelMeshComp, 
                    Cast<UAnimSequenceBase>(AnimNotifyEvent.NotifyStateClass->GetOuter()), 
                    DeltaSeconds, EventReference);
            }
        }
    }
}
```

### 5.2 触发单个通知

```cpp
void UAnimInstance::TriggerSingleAnimNotify(FAnimNotifyEventReference& EventReference)
{
    const FAnimNotifyEvent* AnimNotifyEvent = EventReference.GetNotify();
    
    // 只处理非状态通知
    if (AnimNotifyEvent && (AnimNotifyEvent->NotifyStateClass == NULL))
    {
        // 1. 尝试自定义处理
        if (HandleNotify(*AnimNotifyEvent))
        {
            return;
        }

        // 2. 有通知类实例的情况
        if (AnimNotifyEvent->Notify != nullptr)
        {
#if WITH_EDITOR
            if (!GetSkelMeshComponent()->IsA<UDebugSkelMeshComponent>() || 
                AnimNotifyEvent->Notify->ShouldFireInEditor())
#endif
            {
                TRACE_ANIM_NOTIFY(this, *AnimNotifyEvent, Event);
                AnimNotifyEvent->Notify->Notify(GetSkelMeshComponent(), 
                    Cast<UAnimSequenceBase>(AnimNotifyEvent->Notify->GetOuter()), EventReference);
            }
        }
        // 3. 自定义事件 (基于名称的蓝图事件)
        else if (AnimNotifyEvent->NotifyName != NAME_None)
        {
            // 生成函数名: AnimNotify_XXX
            FName FuncName = AnimNotifyEvent->GetNotifyEventName(EventReference.GetMirrorDataTable());
            
            auto NotifyAnimInstance = [this, AnimNotifyEvent, FuncName](UAnimInstance* InAnimInstance)
            {
                check(InAnimInstance);
                
                if (InAnimInstance == this || InAnimInstance->bReceiveNotifiesFromLinkedInstances)
                {
                    TRACE_ANIM_NOTIFY(this, *AnimNotifyEvent, Event);
                    
                    UFunction* Function = InAnimInstance->FindFunction(FuncName);
                    if (Function)
                    {
                        if (Function->NumParms == 0)
                        {
                            InAnimInstance->ProcessEvent(Function, nullptr);
                        }
                        else if ((Function->NumParms == 1) && 
                                 (CastField<FObjectProperty>(Function->PropertyLink) != nullptr))
                        {
                            struct FAnimNotifierHandler_Parms
                            {
                                UAnimNotify* Notify;
                            };
                            FAnimNotifierHandler_Parms Parms;
                            Parms.Notify = AnimNotifyEvent->Notify;
                            InAnimInstance->ProcessEvent(Function, &Parms);
                        }
                        else
                        {
                            UE_LOG(LogAnimNotify, Warning, 
                                TEXT("Anim notifier named %s, but the parameter number does not match"), 
                                *FuncName.ToString());
                        }
                    }
                }
            };
            
            // 检查外部处理器
            if (FSimpleMulticastDelegate* ExistingDelegate = ExternalNotifyHandlers.Find(FuncName))
            {
                ExistingDelegate->Broadcast();
            }

            // 传播到链接实例
            if (bPropagateNotifiesToLinkedInstances)
            {
                GetSkelMeshComponent()->ForEachAnimInstance(NotifyAnimInstance);
            }
            else
            {
                NotifyAnimInstance(this);
            }
        }
    }
}
```

---

## 六、蒙太奇与分支点

### 6.1 蒙太奇通知 Tick 类型

```cpp
// Engine/Source/Runtime/Engine/Public/Animation/AnimTypes.h

UENUM()
namespace EMontageNotifyTickType
{
    enum Type : int
    {
        // 队列模式：通知在评估阶段结束后触发 (更快)
        // 不适合需要改变 Section 或蒙太奇位置的情况
        Queued,
        
        // 分支点模式：通知在遇到时立即触发 (较慢)
        // 适合需要改变 Section 或蒙太奇位置的情况
        BranchingPoint,
    };
}
```

### 6.2 判断是否为分支点

```cpp
bool FAnimNotifyEvent::IsBranchingPoint() const
{
    return GetLinkedMontage() && 
           ((MontageTickType == EMontageNotifyTickType::BranchingPoint) || 
            (Notify && Notify->bIsNativeBranchingPoint) || 
            (NotifyStateClass && NotifyStateClass->bIsNativeBranchingPoint));
}
```

### 6.3 分支点负载结构

```cpp
USTRUCT()
struct FBranchingPointNotifyPayload
{
    GENERATED_USTRUCT_BODY()

    USkeletalMeshComponent* SkelMeshComponent;
    UAnimSequenceBase* SequenceAsset;
    FAnimNotifyEvent* NotifyEvent;
    int32 MontageInstanceID;
    bool bReachedEnd = false;

    FBranchingPointNotifyPayload()
        : SkelMeshComponent(nullptr)
        , SequenceAsset(nullptr)
        , NotifyEvent(nullptr)
        , MontageInstanceID(INDEX_NONE)
    {}

    FBranchingPointNotifyPayload(USkeletalMeshComponent* InSkelMeshComponent, 
                                  UAnimSequenceBase* InSequenceAsset, 
                                  FAnimNotifyEvent* InNotifyEvent, 
                                  int32 InMontageInstanceID, 
                                  bool bInReachedEnd = false)
        : SkelMeshComponent(InSkelMeshComponent)
        , SequenceAsset(InSequenceAsset)
        , NotifyEvent(InNotifyEvent)
        , MontageInstanceID(InMontageInstanceID)
        , bReachedEnd(bInReachedEnd)
    {}
};
```

---

## 七、同步标记系统 (Sync Markers)

### 7.1 FAnimSyncMarker 结构

```cpp
USTRUCT(BlueprintType)
struct FAnimSyncMarker
{
    GENERATED_USTRUCT_BODY()

    // 标记名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=SyncMarker)
    FName MarkerName;

    // 时间点
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=SyncMarker)
    float Time;

    // 轨道索引
    UPROPERTY()
    int32 TrackIndex;

    // 唯一标识符
    UPROPERTY()
    FGuid Guid;
};
```

### 7.2 同步标记的作用

同步标记用于动画同步，特别是在以下场景：

- **多角色同步**：确保多个角色的动画在关键点同步
- **动画混合**：在混合空间中使用标记进行精确同步
- **Leader-Follower 模式**：一个动画领导，其他动画跟随

---

## 八、内置通知类型示例

### 8.1 UAnimNotifyState_Trail (拖尾效果)

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimNotifyState_Trail.cpp

UCLASS(Blueprintable, meta=(DisplayName="Trail"))
class UAnimNotifyState_Trail : public UAnimNotifyState
{
    GENERATED_UCLASS_BODY()

public:
    // 粒子系统模板
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Trail)
    TObjectPtr<UParticleSystem> PSTemplate;

    // 第一个 Socket 名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Trail)
    FName FirstSocketName;

    // 第二个 Socket 名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Trail)
    FName SecondSocketName;

    // 宽度缩放模式
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Trail)
    TEnumAsByte<ETrailWidthMode> WidthScaleMode;

    // 宽度曲线名称 (从动画曲线读取)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Trail)
    FName WidthScaleCurve;

    // 是否在结束时销毁
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Trail)
    bool bDestroyAtEnd;

    // 是否回收生成的系统
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Trail)
    bool bRecycleSpawnedSystems;

    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                            float TotalDuration, const FAnimNotifyEventReference& EventReference) override;
    virtual void NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                           float FrameDeltaTime, const FAnimNotifyEventReference& EventReference) override;
    virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                          const FAnimNotifyEventReference& EventReference) override;
};
```

### 8.2 UAnimNotifyState_TimedNiagaraEffect (Niagara 特效)

```cpp
// 来自 Niagara 插件
UCLASS(Blueprintable, meta=(DisplayName="Timed Niagara Effect"))
class UAnimNotifyState_TimedNiagaraEffect : public UAnimNotifyState
{
    GENERATED_UCLASS_BODY()

    // Niagara 系统模板
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=NiagaraSystem)
    TObjectPtr<UNiagaraSystem> Template;

    // 附着的 Socket 名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=NiagaraSystem)
    FName SocketName;

    // 位置偏移
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=NiagaraSystem)
    FVector LocationOffset;

    // 旋转偏移
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=NiagaraSystem)
    FRotator RotationOffset;

    // 缩放
    UPROPERTY(EditAnywhere, Category=NiagaraSystem)
    FVector Scale = FVector(1.0f);

    // 是否应用动画速率缩放作为时间膨胀
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=NiagaraSystem)
    bool bApplyRateScaleAsTimeDilation = false;

    // 是否在结束时立即销毁
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=NiagaraSystem)
    bool bDestroyAtEnd;

    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                            float TotalDuration, const FAnimNotifyEventReference& EventReference) override;
    virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                          const FAnimNotifyEventReference& EventReference) override;
};
```

---

## 九、创建自定义 AnimNotify

### 9.1 C++ 自定义单次通知

```cpp
// MyAnimNotify_PlaySound.h
UCLASS()
class MYGAME_API UAnimNotify_PlaySound : public UAnimNotify
{
    GENERATED_BODY()

public:
    UAnimNotify_PlaySound();

    // 音效资源
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="AnimNotify")
    TObjectPtr<USoundBase> Sound;

    // 音量倍率
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="AnimNotify", meta=(ClampMin="0.0"))
    float VolumeMultiplier = 1.0f;

    // 是否附着到骨骼
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="AnimNotify")
    bool bAttachToMesh = true;

    // 附着的骨骼名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="AnimNotify", meta=(EditCondition="bAttachToMesh"))
    FName AttachName;

    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                       const FAnimNotifyEventReference& EventReference) override;
    
    virtual FString GetNotifyName_Implementation() const override;
};

// MyAnimNotify_PlaySound.cpp
UAnimNotify_PlaySound::UAnimNotify_PlaySound()
{
#if WITH_EDITORONLY_DATA
    NotifyColor = FColor(196, 142, 255, 255);
#endif
}

void UAnimNotify_PlaySound::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                    const FAnimNotifyEventReference& EventReference)
{
    Super::Notify(MeshComp, Animation, EventReference);

    if (Sound && MeshComp)
    {
        if (bAttachToMesh)
        {
            UGameplayStatics::SpawnSoundAttached(Sound, MeshComp, AttachName, 
                FVector::ZeroVector, EAttachLocation::SnapToTarget, 
                true, VolumeMultiplier);
        }
        else
        {
            UGameplayStatics::PlaySoundAtLocation(MeshComp->GetWorld(), Sound, 
                MeshComp->GetComponentLocation(), VolumeMultiplier);
        }
    }
}

FString UAnimNotify_PlaySound::GetNotifyName_Implementation() const
{
    if (Sound)
    {
        return FString::Printf(TEXT("Play Sound: %s"), *Sound->GetName());
    }
    return TEXT("Play Sound");
}
```

### 9.2 C++ 自定义状态通知

```cpp
// MyAnimNotifyState_MeleeWindow.h
UCLASS()
class MYGAME_API UAnimNotifyState_MeleeWindow : public UAnimNotifyState
{
    GENERATED_BODY()

public:
    UAnimNotifyState_MeleeWindow();

    // 伤害值
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Melee")
    float Damage = 10.0f;

    // 命中 Socket 名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Melee")
    FName HitSocketName;

    // 检测半径
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Melee")
    float TraceRadius = 30.0f;

    // 已命中的 Actor 列表 (防止重复伤害)
    TSet<AActor*> HitActors;

    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                            float TotalDuration, const FAnimNotifyEventReference& EventReference) override;
    virtual void NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                           float FrameDeltaTime, const FAnimNotifyEventReference& EventReference) override;
    virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                          const FAnimNotifyEventReference& EventReference) override;
    
    virtual FString GetNotifyName_Implementation() const override;
};

// MyAnimNotifyState_MeleeWindow.cpp
UAnimNotifyState_MeleeWindow::UAnimNotifyState_MeleeWindow()
{
#if WITH_EDITORONLY_DATA
    NotifyColor = FColor(255, 100, 100, 255);
#endif
}

void UAnimNotifyState_MeleeWindow::NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                                float TotalDuration, const FAnimNotifyEventReference& EventReference)
{
    Super::NotifyBegin(MeshComp, Animation, TotalDuration, EventReference);
    
    // 清空命中列表
    HitActors.Empty();
}

void UAnimNotifyState_MeleeWindow::NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                               float FrameDeltaTime, const FAnimNotifyEventReference& EventReference)
{
    Super::NotifyTick(MeshComp, Animation, FrameDeltaTime, EventReference);

    if (!MeshComp) return;

    // 获取 Socket 位置
    FVector SocketLocation = MeshComp->GetSocketLocation(HitSocketName);
    AActor* OwnerActor = MeshComp->GetOwner();

    // 执行球形检测
    TArray<FHitResult> HitResults;
    FCollisionShape SphereShape = FCollisionShape::MakeSphere(TraceRadius);
    
    MeshComp->GetWorld()->SweepMultiByChannel(
        HitResults,
        SocketLocation,
        SocketLocation,
        FQuat::Identity,
        ECC_Pawn,
        SphereShape);

    // 处理命中
    for (const FHitResult& Hit : HitResults)
    {
        AActor* HitActor = Hit.GetActor();
        
        // 跳过自己和已命中的 Actor
        if (HitActor == OwnerActor || HitActors.Contains(HitActor))
        {
            continue;
        }

        // 记录命中
        HitActors.Add(HitActor);

        // 应用伤害
        UGameplayStatics::ApplyDamage(HitActor, Damage, nullptr, OwnerActor, nullptr);
    }
}

void UAnimNotifyState_MeleeWindow::NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, 
                                              const FAnimNotifyEventReference& EventReference)
{
    Super::NotifyEnd(MeshComp, Animation, EventReference);
    
    // 清空命中列表
    HitActors.Empty();
}

FString UAnimNotifyState_MeleeWindow::GetNotifyName_Implementation() const
{
    return FString::Printf(TEXT("Melee Window (Damage: %.1f)"), Damage);
}
```

### 9.3 蓝图自定义通知

在蓝图中创建自定义通知也非常简单：

1. 创建新蓝图，父类选择 `AnimNotify` 或 `AnimNotifyState`
2. 重写 `Received_Notify` 或 `Received_NotifyBegin/Tick/End`
3. 在动画编辑器中将通知拖入时间轴

---

## 十、性能优化建议

### 10.1 使用过滤机制

```cpp
// 在编辑器中设置 LOD 过滤
// 远距离角色不触发复杂的通知
AnimNotifyEvent.NotifyFilterType = ENotifyFilterType::LOD;
AnimNotifyEvent.NotifyFilterLOD = 2; // 只在 LOD 0, 1 时触发
```

### 10.2 概率触发

```cpp
// 设置触发概率，减少不必要的事件
AnimNotifyEvent.NotifyTriggerChance = 0.5f; // 50% 概率触发
```

### 10.3 权重阈值

```cpp
// 只在动画权重足够高时触发
AnimNotifyEvent.TriggerWeightThreshold = 0.5f; // 权重低于 50% 不触发
```

### 10.4 网络优化

```cpp
// Dedicated Server 上禁用纯客户端通知
AnimNotifyEvent.bTriggerOnDedicatedServer = false;
```

### 10.5 分支点与队列模式

- **队列模式 (Queued)**：适合不需要改变动画流程的通知，性能更好
- **分支点模式 (BranchingPoint)**：适合需要立即响应并可能改变动画的情况

---

## 十一、调试技巧

### 11.1 使用 TRACE_ANIM_NOTIFY

UE5 提供了动画追踪宏用于调试：

```cpp
TRACE_ANIM_NOTIFY(this, AnimNotifyEvent, Begin); // 追踪开始
TRACE_ANIM_NOTIFY(this, AnimNotifyEvent, Tick);  // 追踪 Tick
TRACE_ANIM_NOTIFY(this, AnimNotifyEvent, End);   // 追踪结束
TRACE_ANIM_NOTIFY(this, AnimNotifyEvent, Event); // 追踪事件
```

### 11.2 日志输出

```cpp
// 使用专门的日志类别
DEFINE_LOG_CATEGORY(LogAnimNotify);

UE_LOG(LogAnimNotify, Log, TEXT("Notify %s triggered at time %f"), 
       *NotifyName.ToString(), GetTriggerTime());
```

### 11.3 编辑器可视化

```cpp
#if WITH_EDITOR
virtual void DrawInEditor(FPrimitiveDrawInterface* PDI, USkeletalMeshComponent* MeshComp, 
                          const UAnimSequenceBase* Animation, const FAnimNotifyEvent& NotifyEvent) const override
{
    // 在编辑器中绘制调试信息
    FVector Location = MeshComp->GetSocketLocation(SocketName);
    DrawDebugSphere(MeshComp->GetWorld(), Location, TraceRadius, 12, FColor::Red, false, -1.0f);
}
#endif
```

---

## 十二、总结

AnimNotify 系统是 UE5 动画系统与游戏逻辑交互的核心机制。通过本章的学习，我们深入了解了：

1. **核心数据结构**：`FAnimNotifyEvent`、`FAnimNotifyQueue`、`FAnimNotifyEventReference`
2. **两种通知类型**：`UAnimNotify` (单次) 和 `UAnimNotifyState` (持续)
3. **触发流程**：从队列收集到 `TriggerAnimNotifies()` 的完整调度过程
4. **蒙太奇集成**：分支点与队列模式的区别与应用
5. **同步标记**：用于多动画同步的 `FAnimSyncMarker`
6. **过滤机制**：LOD、权重、概率、网络等多维度过滤
7. **自定义实现**：C++ 和蓝图两种创建自定义通知的方法
8. **性能优化**：各种减少不必要通知触发的策略

掌握 AnimNotify 系统，将使您能够创建更加生动、响应性更强的游戏动画体验。在下一章中，我们将探讨动画系统的多线程评估与性能优化策略。

---

## 参考源码路径

- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNotifies/AnimNotify.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNotifies/AnimNotifyState.h`
- `Engine/Source/Runtime/Engine/Public/Animation/AnimTypes.h`
- `Engine/Source/Runtime/Engine/Public/Animation/AnimNotifyQueue.h`
- `Engine/Source/Runtime/Engine/Private/Animation/AnimInstance.cpp`
- `Engine/Source/Runtime/Engine/Private/Animation/AnimNotify.cpp`
- `Engine/Source/Runtime/Engine/Private/Animation/AnimNotifyState.cpp`
- `Engine/Source/Runtime/Engine/Private/Animation/AnimNotifyQueue.cpp`
