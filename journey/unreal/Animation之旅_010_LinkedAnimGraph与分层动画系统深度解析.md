# Animation之旅_010：Linked Anim Graph与分层动画系统深度解析

> 本文深入剖析UE5动画系统中的Linked Anim Graph（链接动画图）与分层动画系统，从架构设计到源码实现，带你全面理解多AnimInstance协作与动画分层的核心机制。

## 前言

在复杂的游戏项目中，单一的动画蓝图往往难以满足需求。想象以下场景：
- **模块化角色系统**：不同武器有独立的动画逻辑
- **可组合动画行为**：上半身瞄准 + 下半身移动
- **运行时动态切换**：根据装备切换不同的动画实现

这些需求催生了UE5的**Linked Anim Graph（链接动画图）**和**Anim Layer（动画层）**系统。它们允许多个AnimInstance协同工作，实现动画逻辑的模块化与动态组合。

---

## 一、系统架构概述

### 1.1 核心概念

| 概念 | 描述 | 用途 |
|------|------|------|
| **Linked Anim Graph** | 链接外部动画蓝图的整个动画图 | 完整动画逻辑复用 |
| **Linked Anim Layer** | 链接外部动画蓝图中的特定层 | 模块化动画替换 |
| **Animation Layer Interface** | 定义可替换动画层的接口 | 建立层的契约 |
| **Input Pose** | 从父图接收的输入姿态 | 层间姿态传递 |

### 1.2 架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SkeletalMeshComponent                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │               Main AnimInstance (AnimScriptInstance)         │   │
│  │  ┌─────────────────────────────────────────────────────────┐│   │
│  │  │                     AnimGraph                            ││   │
│  │  │  ┌───────────────────┐   ┌───────────────────────────┐  ││   │
│  │  │  │FAnimNode_Linked   │   │FAnimNode_LinkedAnimLayer  │  ││   │
│  │  │  │AnimGraph          │   │(Layer: UpperBody)         │  ││   │
│  │  │  │InstanceClass: ABP_│   │ Interface: IAnim_Weapon   │  ││   │
│  │  │  │ Locomotion        │   │ InstanceClass: ABP_Rifle  │  ││   │
│  │  │  └─────────┬─────────┘   └─────────────┬─────────────┘  ││   │
│  │  └────────────│───────────────────────────│────────────────┘│   │
│  └───────────────│───────────────────────────│─────────────────┘   │
│                  │                           │                      │
│                  ▼                           ▼                      │
│  ┌─────────────────────────┐   ┌─────────────────────────────────┐ │
│  │ Linked AnimInstance #1  │   │   Linked AnimInstance #2        │ │
│  │ (ABP_Locomotion)        │   │   (ABP_Rifle - Layer Instance)  │ │
│  │ ┌─────────────────────┐ │   │ ┌─────────────────────────────┐ │ │
│  │ │ AnimGraph           │ │   │ │ Layer: UpperBody            │ │ │
│  │ │ ┌─────────────────┐ │ │   │ │ ┌─────────────────────────┐ │ │ │
│  │ │ │LinkedInputPose  │ │ │   │ │ │LinkedInputPose          │ │ │ │
│  │ │ │(receives pose   │ │ │   │ │ │(receives base pose)     │ │ │ │
│  │ │ │ from parent)    │ │ │   │ │ └────────────┬────────────┘ │ │ │
│  │ │ └─────────────────┘ │ │   │ │              │              │ │ │
│  │ └─────────────────────┘ │   │ │     ┌───────▼────────┐     │ │ │
│  └─────────────────────────┘   │ │     │ Animation Logic │     │ │ │
│                                 │ │     └───────┬────────┘     │ │ │
│                                 │ │             │ Output Pose  │ │ │
│                                 │ └─────────────│──────────────┘ │ │
│                                 └───────────────│────────────────┘ │
│                                                 │                  │
│  ┌──────────────────────────────────────────────▼────────────────┐ │
│  │                    LinkedInstances Array                       │ │
│  │    [Linked AnimInstance #1, Linked AnimInstance #2, ...]      │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 类继承关系

```
FAnimNode_CustomProperty (属性传递基类)
    └── FAnimNode_LinkedAnimGraph (链接完整动画图)
            └── FAnimNode_LinkedAnimLayer (链接动画层)

UAnimInstance
    ├── 主动画实例 (AnimScriptInstance)
    └── 链接动画实例 (LinkedInstances[])
```

---

## 二、核心数据结构

### 2.1 FAnimNode_LinkedAnimGraph - 链接动画图节点

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_LinkedAnimGraph.h

USTRUCT(BlueprintInternalUseOnly)
struct FAnimNode_LinkedAnimGraph : public FAnimNode_CustomProperty
{
    GENERATED_BODY()

public:
    /** 输入姿态链接数组，用于向子图传递姿态 */
    UPROPERTY()
    TArray<FPoseLink> InputPoses;

    /** 输入姿态名称列表，与InputPoses一一对应 */
    UPROPERTY()
    TArray<FName> InputPoseNames;

    /** 要实例化的动画蓝图类 */
    UPROPERTY(EditAnywhere, Category = Settings)
    TSubclassOf<UAnimInstance> InstanceClass;

    /** 链接实例的根节点 */
    FAnimNode_Base* LinkedRoot;

    /** 当前节点在父图中的索引 */
    int32 NodeIndex;

    /** 缓存的链接节点索引 */
    int32 CachedLinkedNodeIndex;

protected:
    /** 待处理的惯性混合淡出时长 */
    float PendingBlendOutDuration;

    /** 惯性混合淡出的混合配置 */
    UPROPERTY(Transient)
    TObjectPtr<const UBlendProfile> PendingBlendOutProfile;

    /** 待处理的惯性混合淡入时长 */
    float PendingBlendInDuration;

    /** 惯性混合淡入的混合配置 */
    UPROPERTY(Transient)
    TObjectPtr<const UBlendProfile> PendingBlendInProfile;

public:
    /** 是否从链接实例接收命名通知 */
    UPROPERTY(EditAnywhere, Category = Settings)
    uint8 bReceiveNotifiesFromLinkedInstances : 1;

    /** 是否向链接实例传播命名通知 */
    UPROPERTY(EditAnywhere, Category = Settings)
    uint8 bPropagateNotifiesToLinkedInstances : 1;
};
```

### 2.2 FAnimNode_LinkedAnimLayer - 链接动画层节点

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_LinkedAnimLayer.h

USTRUCT(BlueprintInternalUseOnly)
struct FAnimNode_LinkedAnimLayer : public FAnimNode_LinkedAnimGraph
{
    GENERATED_BODY()

public:
    /** 动画层接口类（定义可替换的层契约） */
    UPROPERTY()
    TSubclassOf<UAnimLayerInterface> Interface;

    /** 层的名称（对应接口中定义的函数名） */
    UPROPERTY()
    FName Layer;

    // FAnimNode_LinkedAnimGraph重写
    virtual FName GetDynamicLinkFunctionName() const override { return Layer; }
    virtual UAnimInstance* GetDynamicLinkTarget(UAnimInstance* InOwningAnimInstance) const override;
};
```

### 2.3 FAnimNode_LinkedInputPose - 链接输入姿态节点

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_LinkedInputPose.h

USTRUCT()
struct FAnimNode_LinkedInputPose : public FAnimNode_Base
{
    GENERATED_BODY()

    /** 默认输入姿态名称 */
    static ENGINE_API const FName DefaultInputPoseName;  // "InPose"

    /** 此输入姿态的名称，用于标识图的输入 */
    UPROPERTY(EditAnywhere, Category = "Inputs", meta = (NeverAsPin))
    FName Name;

    /** 此节点所在的图名称，由编译器填充 */
    UPROPERTY()
    FName Graph;

    /** 输入姿态链接，可动态链接到其他图 */
    UPROPERTY()
    FPoseLink InputPose;

    /** 
     * 如果未动态链接，此缓存数据将在处理前由调用方填充
     * 存储从父图传入的姿态数据
     */
    FCompactHeapPose CachedInputPose;
    FBlendedHeapCurve CachedInputCurve;
    UE::Anim::FHeapAttributeContainer CachedAttributes;

    /** 缓存输入姿态是否已初始化 */
    uint8 bIsCachedInputPoseInitialized : 1;

    /** 输出是否连接到图的根节点 */
    UPROPERTY(meta = (BlueprintCompilerGeneratedDefaults))
    uint8 bIsOutputLinked : 1;

    /** 外部图中当前链接节点的索引 */
    int32 OuterGraphNodeIndex;

private:
    /** 动态链接时使用的代理 */
    FAnimInstanceProxy* InputProxy;

public:
    /** 由链接实例节点调用，动态链接到外部图 */
    ENGINE_API void DynamicLink(FAnimInstanceProxy* InInputProxy, 
                                 FPoseLinkBase* InPoseLink, 
                                 int32 InOuterGraphNodeIndex);

    /** 由链接实例节点调用，解除动态链接 */
    ENGINE_API void DynamicUnlink();
};
```

### 2.4 FLinkedAnimLayerInstanceData - 链接层实例数据

```cpp
// Engine/Source/Runtime/Engine/Public/Animation/AnimSubsystem_SharedLinkedAnimLayers.h

USTRUCT()
struct FLinkedAnimLayerInstanceData
{
    GENERATED_USTRUCT_BODY()

    /** 动画实例指针 */
    UPROPERTY(Transient)
    TObjectPtr<UAnimInstance> Instance;

    /** 是否为持久实例（即使取消链接也保留） */
    bool IsPersistent() const { return bIsPersistent; }

    /** 标记函数为已链接 */
    void AddLinkedFunction(FName Function, UAnimInstance* AnimInstance);
    
    /** 取消函数链接标记 */
    void RemoveLinkedFunction(FName Function);

    /** 返回当前链接的函数映射 */
    const TMap<FName, TWeakObjectPtr<UAnimInstance>>& GetLinkedFunctions() const;

private:
    /** 已链接函数到动画实例的映射 */
    UPROPERTY(Transient)
    TMap<FName, TWeakObjectPtr<UAnimInstance>> LinkedFunctions;

    bool bIsPersistent = false;
};
```

### 2.5 FAnimSubsystem_SharedLinkedAnimLayers - 共享链接层子系统

```cpp
// Engine/Source/Runtime/Engine/Public/Animation/AnimSubsystem_SharedLinkedAnimLayers.h

USTRUCT()
struct FAnimSubsystem_SharedLinkedAnimLayers : public FAnimSubsystemInstance
{
    GENERATED_BODY()

    /** 从骨骼网格组件获取子系统 */
    static ENGINE_API FAnimSubsystem_SharedLinkedAnimLayers* GetFromMesh(
        USkeletalMeshComponent* SkelMesh);

    /** 清除所有链接层数据 */
    ENGINE_API void Reset();

    /** 检查给定动画实例是否为共享实例 */
    bool IsSharedInstance(const UAnimInstance* AnimInstance);

    /** 添加持久化动画层类 */
    void AddPersistentAnimLayerClass(TSubclassOf<UAnimInstance> AnimInstanceClass);

    /** 移除持久化动画层类 */
    ENGINE_API void RemovePersistentAnimLayerClass(TSubclassOf<UAnimInstance> AnimInstanceClass);

    /** 添加链接函数，返回要绑定的实例 */
    ENGINE_API UAnimInstance* AddLinkedFunction(UAnimInstance* OwningInstance, 
        TSubclassOf<UAnimInstance> AnimClass, FName Function, bool& bIsNewInstance);

    /** 移除链接函数，如果实例不再使用且非持久化则清理 */
    ENGINE_API void RemoveLinkedFunction(UAnimInstance* AnimInstance, FName Function);

private:
    /** 类数据数组 */
    UPROPERTY(Transient)
    TArray<FLinkedAnimLayerClassData> ClassesData;

    /** 应保持活动状态的动画实例类（即使取消链接） */
    UPROPERTY(Transient)
    TArray<TSubclassOf<UAnimInstance>> PersistentClasses;
};
```

---

## 三、Linked Anim Graph实现详解

### 3.1 初始化流程

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimNode_LinkedAnimGraph.cpp

void FAnimNode_LinkedAnimGraph::OnInitializeAnimInstance(
    const FAnimInstanceProxy* InProxy, 
    const UAnimInstance* InAnimInstance)
{
#if WITH_EDITORONLY_DATA
    SourceInstance = const_cast<UAnimInstance*>(InAnimInstance);
#endif
    
    UAnimInstance* InstanceToRun = GetTargetInstance<UAnimInstance>();

    // 如果有有效的实例类
    if (*InstanceClass)
    {
        // 重新初始化链接的动画实例
        ReinitializeLinkedAnimInstance(InAnimInstance);
    }
    else if (InstanceToRun)
    {
        // 有实例但没有实例类，需要清理
        TeardownInstance(InAnimInstance);
    }

    if (InstanceToRun)
    {
        // 初始化代理对象
        InstanceToRun->GetProxyOnAnyThread<FAnimInstanceProxy>().InitializeObjects(InstanceToRun);
    }
}
```

### 3.2 重新初始化链接实例

```cpp
void FAnimNode_LinkedAnimGraph::ReinitializeLinkedAnimInstance(
    const UAnimInstance* InOwningAnimInstance, 
    UAnimInstance* InNewAnimInstance)
{
    UAnimInstance* InstanceToRun = GetTargetInstance<UAnimInstance>();

    // 保存之前的动画蓝图类（用于混合请求）
    IAnimClassInterface* PriorAnimBPClass = InstanceToRun ? 
        IAnimClassInterface::GetFromClass(InstanceToRun->GetClass()) : nullptr;

    // 完全重新初始化，销毁旧实例
    TeardownInstance(InOwningAnimInstance);

    if (*InstanceClass || InNewAnimInstance)
    {
        USkeletalMeshComponent* MeshComp = InOwningAnimInstance->GetSkelMeshComponent();
        check(MeshComp);

        // 创建或使用传入的新实例
        InstanceToRun = InNewAnimInstance ? 
            InNewAnimInstance : NewObject<UAnimInstance>(MeshComp, InstanceClass);
        
        if (InNewAnimInstance == nullptr)
        {
            // 标记为由链接动画图创建（用于内存管理）
            InstanceToRun->bCreatedByLinkedAnimGraph = true;
            InstanceToRun->bPropagateNotifiesToLinkedInstances = bPropagateNotifiesToLinkedInstances;
            InstanceToRun->bReceiveNotifiesFromLinkedInstances = bReceiveNotifiesFromLinkedInstances;
        }

        SetTargetInstance(InstanceToRun);

        // 在调用InitializeAnimation之前链接，以便传播调用到链接输入姿态
        DynamicLink(const_cast<UAnimInstance*>(InOwningAnimInstance));

        if (InNewAnimInstance == nullptr)
        {
            // 初始化新实例
            InstanceToRun->InitializeAnimation();

            if (MeshComp->HasBegunPlay())
            {
                InstanceToRun->NativeBeginPlay();
                InstanceToRun->BlueprintBeginPlay();
            }

            // 添加到链接实例数组
            MeshComp->GetLinkedAnimInstances().Add(InstanceToRun);
        }

        // 初始化属性映射
        InitializeProperties(InOwningAnimInstance, InstanceToRun->GetClass());

        IAnimClassInterface* NewAnimBPClass = IAnimClassInterface::GetFromClass(InstanceToRun->GetClass());

        // 请求惯性混合
        RequestBlend(PriorAnimBPClass, NewAnimBPClass);
    }
    else
    {
        RequestBlend(PriorAnimBPClass, nullptr);
    }
}
```

### 3.3 动态链接机制

```cpp
void FAnimNode_LinkedAnimGraph::DynamicLink(UAnimInstance* InOwningAnimInstance)
{
    CachedLinkedNodeIndex = INDEX_NONE;

    UAnimInstance* LinkTargetInstance = GetDynamicLinkTarget(InOwningAnimInstance);
    if (LinkTargetInstance)
    {
        IAnimClassInterface* SubAnimBlueprintClass = 
            IAnimClassInterface::GetFromClass(LinkTargetInstance->GetClass());
        
        if (SubAnimBlueprintClass)
        {
            FAnimInstanceProxy* NonConstProxy = 
                &InOwningAnimInstance->GetProxyOnAnyThread<FAnimInstanceProxy>();
            const FName FunctionToLink = GetDynamicLinkFunctionName();

            // 遍历动画蓝图函数
            const TArray<FAnimBlueprintFunction>& AnimBlueprintFunctions = 
                SubAnimBlueprintClass->GetAnimBlueprintFunctions();
            
            for (int32 FunctionIndex = 0; FunctionIndex < AnimBlueprintFunctions.Num(); ++FunctionIndex)
            {
                const FAnimBlueprintFunction& AnimBlueprintFunction = 
                    AnimBlueprintFunctions[FunctionIndex];

                // 找到匹配的函数
                if (AnimBlueprintFunction.Name == FunctionToLink)
                {
                    CachedLinkedNodeIndex = AnimBlueprintFunction.OutputPoseNodeIndex;

                    // 链接输入姿态
                    for (int32 InputPoseIndex = 0; 
                         InputPoseIndex < AnimBlueprintFunction.InputPoseNames.Num() && 
                         InputPoseIndex < InputPoses.Num(); 
                         ++InputPoseIndex)
                    {
                        // 尝试重新链接
                        FAnimationInitializeContext Context(NonConstProxy);
                        InputPoses[InputPoseIndex].AttemptRelink(Context);

                        int32 InputPropertyIndex = FindFunctionInputIndex(
                            AnimBlueprintFunction, 
                            AnimBlueprintFunction.InputPoseNames[InputPoseIndex]);
                        
                        if (InputPropertyIndex != INDEX_NONE && 
                            AnimBlueprintFunction.InputPoseNodeProperties[InputPropertyIndex])
                        {
                            // 获取LinkedInputPose节点并动态链接
                            FAnimNode_LinkedInputPose* LinkedInputPoseNode = 
                                AnimBlueprintFunction.InputPoseNodeProperties[InputPropertyIndex]
                                    ->ContainerPtrToValuePtr<FAnimNode_LinkedInputPose>(LinkTargetInstance);
                            
                            check(LinkedInputPoseNode->Name == 
                                  AnimBlueprintFunction.InputPoseNames[InputPoseIndex]);
                            
                            LinkedInputPoseNode->DynamicLink(
                                NonConstProxy, &InputPoses[InputPoseIndex], NodeIndex);
                        }
                    }

                    // 链接根节点
                    if (AnimBlueprintFunction.OutputPoseNodeProperty)
                    {
                        LinkedRoot = AnimBlueprintFunction.OutputPoseNodeProperty
                            ->ContainerPtrToValuePtr<FAnimNode_Root>(LinkTargetInstance);
                    }

                    break;
                }
            }
        }
    }
}
```

### 3.4 Update流程

```cpp
void FAnimNode_LinkedAnimGraph::Update_AnyThread(const FAnimationUpdateContext& InContext)
{
    // 执行暴露输入的更新
    GetEvaluateGraphExposedInputs().Execute(InContext);

    UAnimInstance* InstanceToRun = GetTargetInstance<UAnimInstance>();
    if (InstanceToRun && LinkedRoot)
    {
        FAnimInstanceProxy& Proxy = InstanceToRun->GetProxyOnAnyThread<FAnimInstanceProxy>();
        
        // 同步更新计数器
        Proxy.UpdateCounter.SynchronizeWith(InContext.AnimInstanceProxy->UpdateCounter);

        // 传播输入属性
        PropagateInputProperties(InContext.AnimInstanceProxy->GetAnimInstanceObject());

        // 创建新的更新上下文并执行更新
        FAnimationUpdateContext NewContext = InContext.WithOtherProxy(&Proxy);
        NewContext.SetNodeId(INDEX_NONE);
        NewContext.SetNodeId(CachedLinkedNodeIndex);
        
        Proxy.UpdateAnimation_WithRoot(NewContext, LinkedRoot, GetDynamicLinkFunctionName());
    }
    else if (InputPoses.Num() > 0)
    {
        // 如果没有有效实例，需要向下传播以确保后续节点正确更新
        InputPoses[0].Update(InContext);
    }

    // 处理待处理的惯性混合请求
    if (PendingBlendOutDuration >= 0.0f || PendingBlendInDuration >= 0.0f)
    {
        UE::Anim::IInertializationRequester* InertializationRequester = 
            InContext.GetMessage<UE::Anim::IInertializationRequester>();
        
        if (InertializationRequester)
        {
            if (PendingBlendOutDuration >= 0.0f)
            {
                FInertializationRequest Request;
                Request.Duration = PendingBlendOutDuration;
                Request.BlendProfile = PendingBlendOutProfile;
                InertializationRequester->RequestInertialization(Request);
            }

            if (PendingBlendInDuration >= 0.0f)
            {
                FInertializationRequest Request;
                Request.Duration = PendingBlendInDuration;
                Request.BlendProfile = PendingBlendInProfile;
                InertializationRequester->RequestInertialization(Request);
            }
        }
    }
    
    // 重置待处理的混合参数
    PendingBlendOutDuration = -1.0f;
    PendingBlendOutProfile = nullptr;
    PendingBlendInDuration = -1.0f;
    PendingBlendInProfile = nullptr;
}
```

### 3.5 Evaluate流程

```cpp
void FAnimNode_LinkedAnimGraph::Evaluate_AnyThread(FPoseContext& Output)
{
#if ANIMNODE_STATS_VERBOSE
    // 记录链接图名称用于性能分析
    FScopeCycleCounter LinkedAnimGraphNameCycleCounter(StatID);
#endif

    UAnimInstance* InstanceToRun = GetTargetInstance<UAnimInstance>();
    if (InstanceToRun && LinkedRoot)
    {
        // 保存当前代理以便递归后恢复
        FAnimInstanceProxy& OldProxy = *Output.AnimInstanceProxy;

        FAnimInstanceProxy& Proxy = InstanceToRun->GetProxyOnAnyThread<FAnimInstanceProxy>();
        
        // 同步评估计数器
        Proxy.EvaluationCounter.SynchronizeWith(Output.AnimInstanceProxy->EvaluationCounter);
        
        // 切换到链接实例的骨骼容器
        Output.Pose.SetBoneContainer(&Proxy.GetRequiredBones());
        Output.AnimInstanceProxy = &Proxy;
        Output.SetNodeId(INDEX_NONE);
        Output.SetNodeId(CachedLinkedNodeIndex);

        // 执行动画蓝图评估
        Proxy.EvaluateAnimation_WithRoot(Output, LinkedRoot);

        // 恢复原始代理和所需骨骼
        Output.AnimInstanceProxy = &OldProxy;
        Output.Pose.SetBoneContainer(&OldProxy.GetRequiredBones());
    }
    else if (InputPoses.Num() > 0)
    {
        // 无有效实例时，传播到下游确保正确评估
        InputPoses[0].Evaluate(Output);
    }
    else
    {
        // 最后退路：重置到参考姿态
        Output.ResetToRefPose();
    }
}
```

---

## 四、Linked Anim Layer实现详解

### 4.1 Layer与LinkedAnimGraph的区别

| 特性 | Linked Anim Graph | Linked Anim Layer |
|------|-------------------|-------------------|
| 链接粒度 | 整个AnimGraph | 特定的层函数 |
| 接口契约 | 无 | Animation Layer Interface |
| 自链接 | 不支持 | 支持（链接自身实现） |
| 共享实例 | 每个节点独立实例 | 支持跨节点共享实例 |
| 动态切换 | 切换整个类 | 切换层实现 |

### 4.2 Layer初始化

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimNode_LinkedAnimLayer.cpp

void FAnimNode_LinkedAnimLayer::OnInitializeAnimInstance(
    const FAnimInstanceProxy* InProxy, 
    const UAnimInstance* InAnimInstance)
{
#if WITH_EDITORONLY_DATA
    SourceInstance = const_cast<UAnimInstance*>(InAnimInstance);
#endif
    
    check(SourcePropertyNames.Num() == DestPropertyNames.Num());
    
    // 初始化源属性
    InitializeSourceProperties(InAnimInstance);

    // 只有在运行"self" layer时才在这里初始化
    // 使用外部实例的层需要由拥有的动画实例初始化（可能共享链接实例）
    if (Interface.Get() == nullptr || InstanceClass.Get() == nullptr)
    {
        InitializeSelfLayer(InAnimInstance);
    }
}
```

### 4.3 Self Layer初始化

当Layer没有指定外部接口时，它作为"自链接"运行，使用自身的实现：

```cpp
void FAnimNode_LinkedAnimLayer::InitializeSelfLayer(const UAnimInstance* SelfAnimInstance)
{
    UAnimInstance* CurrentTarget = GetTargetInstance<UAnimInstance>();

    IAnimClassInterface* PriorAnimBPClass = CurrentTarget ? 
        IAnimClassInterface::GetFromClass(CurrentTarget->GetClass()) : nullptr;

    USkeletalMeshComponent* MeshComp = SelfAnimInstance->GetSkelMeshComponent();
    check(MeshComp);

    if (LinkedRoot)
    {
        DynamicUnlink(const_cast<UAnimInstance*>(SelfAnimInstance));
    }

    // 从动态外部切换到内部，销毁旧实例
    if (CurrentTarget && CurrentTarget != SelfAnimInstance)
    {
        if (CanTeardownLinkedInstance(CurrentTarget))
        {
            CurrentTarget->UninitializeAnimation();
            MeshComp->GetLinkedAnimInstances().Remove(CurrentTarget);
            CurrentTarget->MarkAsGarbage();
            CurrentTarget = nullptr;
        }
    }

    // 设置目标为自身
    SetTargetInstance(const_cast<UAnimInstance*>(SelfAnimInstance));

    // 链接到自身
    DynamicLink(const_cast<UAnimInstance*>(SelfAnimInstance));

    UClass* SelfClass = SelfAnimInstance->GetClass();
    InitializeProperties(SelfAnimInstance, SelfClass);

    IAnimClassInterface* NewAnimBPClass = IAnimClassInterface::GetFromClass(SelfClass);

    // 只有实例变更时才请求混合
    if (CurrentTarget != GetTargetInstance<UAnimInstance>())
    {
        RequestBlend(PriorAnimBPClass, NewAnimBPClass);
    }
}
```

### 4.4 设置链接层实例

```cpp
void FAnimNode_LinkedAnimLayer::SetLinkedLayerInstance(
    const UAnimInstance* InOwningAnimInstance, 
    UAnimInstance* InNewLinkedInstance)
{
    UAnimInstance* PreviousTargetInstance = GetTargetInstance<UAnimInstance>();

    // 重置为self-layer（如果适用）
    if ((Interface.Get() == nullptr || InstanceClass.Get() == nullptr) && 
        (InNewLinkedInstance == nullptr))
    {
        InitializeSelfLayer(InOwningAnimInstance);
    }
    else
    {
        ReinitializeLinkedAnimInstance(InOwningAnimInstance, InNewLinkedInstance);
    }

    // 清理共享层数据
    if (PreviousTargetInstance != GetTargetInstance<UAnimInstance>())
    {
        CleanupSharedLinkedLayersData(InOwningAnimInstance, PreviousTargetInstance);
    }

#if WITH_EDITOR
    OnInstanceChangedEvent.Broadcast();
#endif
}
```

### 4.5 获取动态链接目标

```cpp
UAnimInstance* FAnimNode_LinkedAnimLayer::GetDynamicLinkTarget(
    UAnimInstance* InOwningAnimInstance) const
{
    if (Interface.Get())
    {
        // 有接口时返回目标实例
        return GetTargetInstance<UAnimInstance>();
    }
    else
    {
        // 无接口时返回拥有者实例（自链接）
        return InOwningAnimInstance;
    }
}
```

---

## 五、LinkedInstances管理

### 5.1 SkeletalMeshComponent中的LinkedInstances

```cpp
// Engine/Source/Runtime/Engine/Classes/Components/SkeletalMeshComponent.h

class USkeletalMeshComponent : public USkinnedMeshComponent
{
private:
    /** 运行中的链接动画实例数组 */
    UPROPERTY(transient)
    TArray<TObjectPtr<UAnimInstance>> LinkedInstances;

public:
    /** 获取链接实例数组（可修改） */
    TArray<TObjectPtr<UAnimInstance>>& GetLinkedAnimInstances() { return LinkedInstances; }
    
    /** 获取链接实例数组（只读） */
    const TArray<TObjectPtr<UAnimInstance>>& GetLinkedAnimInstances() const { return LinkedInstances; }

    /** 遍历所有动画实例（主实例 + 链接实例 + 后处理实例） */
    template<typename Func>
    void ForEachAnimInstance(Func InFunc)
    {
        if (AnimScriptInstance)
        {
            InFunc(AnimScriptInstance);
        }

        for (UAnimInstance* LinkedInstance : LinkedInstances)
        {
            InFunc(LinkedInstance);
        }

        if (PostProcessAnimInstance)
        {
            InFunc(PostProcessAnimInstance);
        }
    }
};
```

### 5.2 初始化分组层

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimInstance.cpp

void UAnimInstance::InitializeGroupedLayers(bool bInDeferRootNodeInitialization)
{
    IAnimClassInterface* AnimBlueprintClass = IAnimClassInterface::GetFromClass(GetClass());
    if (AnimBlueprintClass == nullptr)
    {
        return;
    }

    // 获取共享链接层子系统
    FAnimSubsystem_SharedLinkedAnimLayers* SharedLinkedAnimLayers = 
        FindSubsystem<FAnimSubsystem_SharedLinkedAnimLayers>();
    
    if (SharedLinkedAnimLayers == nullptr)
    {
        return;
    }

    // 遍历所有链接动画层节点属性
    for (FStructProperty* LayerNodeProperty : AnimBlueprintClass->GetLinkedAnimLayerNodeProperties())
    {
        FAnimNode_LinkedAnimLayer* LinkedAnimLayerNode = 
            LayerNodeProperty->ContainerPtrToValuePtr<FAnimNode_LinkedAnimLayer>(this);

        // 跳过没有实例类或接口的节点
        if (*LinkedAnimLayerNode->InstanceClass == nullptr || 
            LinkedAnimLayerNode->Interface.Get() == nullptr)
        {
            continue;
        }

        // 通过共享系统添加链接函数
        bool bIsNewInstance = false;
        UAnimInstance* TargetInstance = SharedLinkedAnimLayers->AddLinkedFunction(
            this, 
            LinkedAnimLayerNode->InstanceClass, 
            LinkedAnimLayerNode->GetDynamicLinkFunctionName(), 
            bIsNewInstance);

        if (TargetInstance)
        {
            LinkedAnimLayerNode->SetTargetInstance(TargetInstance);
            LinkedAnimLayerNode->DynamicLink(this);

            if (bIsNewInstance)
            {
                LinkedAnimLayerNode->InitializeProperties(this, TargetInstance->GetClass());
                
                if (!bInDeferRootNodeInitialization)
                {
                    FAnimationInitializeContext InitContext(&GetProxyOnAnyThread<FAnimInstanceProxy>());
                    LinkedAnimLayerNode->InitializeSubGraph_AnyThread(InitContext);
                }
            }
        }
    }
}
```

### 5.3 共享链接实例查找与创建

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimSubsystem_SharedLinkedAnimLayers.cpp

UAnimInstance* FLinkedAnimLayerClassData::FindOrAddInstanceForLinking(
    UAnimInstance* OwningInstance, 
    FName Function, 
    bool& bIsNewInstance)
{
    USkeletalMeshComponent* Mesh = OwningInstance->GetSkelMeshComponent();

    // 遍历现有实例数据
    for (FLinkedAnimLayerInstanceData& LayerInstanceData : InstancesData)
    {
        // 检查函数是否已被链接
        if (!LayerInstanceData.GetLinkedFunctions().Contains(Function))
        {
            // 重新添加持久实例
            if (LayerInstanceData.IsPersistent() && LayerInstanceData.GetLinkedFunctions().Num() == 0)
            {
                LayerInstanceData.Instance->RecalcRequiredBones();
                check(!Mesh->GetLinkedAnimInstances().Contains(LayerInstanceData.Instance));
                Mesh->GetLinkedAnimInstances().Add(LayerInstanceData.Instance);
            }

            bIsNewInstance = false;
            LayerInstanceData.AddLinkedFunction(Function, LayerInstanceData.Instance);
            return LayerInstanceData.Instance;
        }
    }

    // 创建新实例
    bIsNewInstance = true;
    UAnimInstance* NewAnimInstance = NewObject<UAnimInstance>(Mesh, Class);
    NewAnimInstance->bCreatedByLinkedAnimGraph = true;
    NewAnimInstance->InitializeAnimation();

    if (Mesh->HasBegunPlay())
    {
        NewAnimInstance->NativeBeginPlay();
        NewAnimInstance->BlueprintBeginPlay();
    }
    
    FLinkedAnimLayerInstanceData& NewInstanceData = AddInstance(NewAnimInstance);
    NewInstanceData.AddLinkedFunction(Function, NewAnimInstance);
    OwningInstance->GetSkelMeshComponent()->GetLinkedAnimInstances().Add(NewAnimInstance);

    return NewAnimInstance;
}
```

---

## 六、LinkedInputPose实现详解

### 6.1 动态链接

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimNode_LinkedInputPose.cpp

void FAnimNode_LinkedInputPose::DynamicLink(
    FAnimInstanceProxy* InInputProxy, 
    FPoseLinkBase* InPoseLink, 
    int32 InOuterGraphNodeIndex)
{
    check(GIsReinstancing || InputProxy == nullptr);  // 必须先解除链接再重新链接

    InputProxy = InInputProxy;
    InputPose.SetDynamicLinkNode(InPoseLink);
    OuterGraphNodeIndex = InOuterGraphNodeIndex;
}

void FAnimNode_LinkedInputPose::DynamicUnlink()
{
    check(GIsReinstancing || InputProxy != nullptr);  // 必须已链接才能解除

    InputProxy = nullptr;
    InputPose.SetDynamicLinkNode(nullptr);
    OuterGraphNodeIndex = INDEX_NONE;
}
```

### 6.2 Update与Evaluate

```cpp
void FAnimNode_LinkedInputPose::Update_AnyThread(const FAnimationUpdateContext& Context)
{
    if (InputProxy)
    {
        // 使用外部代理更新输入姿态
        FAnimationUpdateContext InputContext = Context.WithOtherProxy(InputProxy);
        InputContext.SetNodeId(OuterGraphNodeIndex);
        InputPose.Update(InputContext);
    }
}

void FAnimNode_LinkedInputPose::Evaluate_AnyThread(FPoseContext& Output)
{
    if (InputProxy)
    {
        // 保存当前代理
        FAnimInstanceProxy& OldProxy = *Output.AnimInstanceProxy;

        // 切换到外部代理
        Output.AnimInstanceProxy = InputProxy;
        Output.Pose.SetBoneContainer(&InputProxy->GetRequiredBones());
        Output.SetNodeId(INDEX_NONE);
        Output.SetNodeId(OuterGraphNodeIndex);
        
        // 评估输入姿态
        InputPose.Evaluate(Output);

        // 恢复原始代理
        Output.AnimInstanceProxy = &OldProxy;
        Output.Pose.SetBoneContainer(&OldProxy.GetRequiredBones());
    }
    else if (CachedInputPose.IsValid() && 
             ensure(Output.Pose.GetNumBones() == CachedInputPose.GetNumBones()) && 
             bIsCachedInputPoseInitialized)
    {
        // 使用缓存的输入姿态
        Output.Pose.CopyBonesFrom(CachedInputPose);
        Output.Curve.CopyFrom(CachedInputCurve);
        Output.CustomAttributes.CopyFrom(CachedAttributes);
    }
    else
    {
        // 回退到参考姿态
        Output.ResetToRefPose();
    }
}
```

---

## 七、实际应用场景

### 7.1 场景一：模块化武器动画

```cpp
// 武器动画接口定义
UINTERFACE(MinimalAPI, Blueprintable)
class UAnimLayerInterface_Weapon : public UInterface
{
    GENERATED_BODY()
};

class IAnimLayerInterface_Weapon
{
    GENERATED_BODY()

public:
    // 上半身武器层
    UFUNCTION(BlueprintCallable, Category = "Animation Layers")
    virtual void UpperBodyWeaponLayer(FAnimUpdateContext& UpdateContext, 
        FPoseLink& InputPose, FPoseLink& OutputPose) = 0;

    // 瞄准偏移层
    UFUNCTION(BlueprintCallable, Category = "Animation Layers")
    virtual void AimOffsetLayer(FAnimUpdateContext& UpdateContext, 
        FPoseLink& InputPose, FPoseLink& OutputPose) = 0;
};
```

**使用方式**：
1. 创建实现接口的动画蓝图（如`ABP_Rifle`, `ABP_Pistol`）
2. 主动画蓝图中使用`Linked Anim Layer`节点
3. 运行时通过`LinkAnimClassLayers`切换武器动画

### 7.2 场景二：分层骨骼动画

```
主动画图:
┌─────────────────────────────────────────────────────────┐
│  Locomotion State Machine                                │
│  ┌─────────────┐   ┌─────────────┐   ┌───────────────┐  │
│  │ Idle State  │──▶│ Walk State  │──▶│ Run State     │  │
│  └─────────────┘   └─────────────┘   └───────────────┘  │
└──────────────────────────┬──────────────────────────────┘
                           │ 下半身姿态
                           ▼
┌─────────────────────────────────────────────────────────┐
│  Layered Blend Per Bone                                  │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Base Layer: 下半身姿态                             │  │
│  ├───────────────────────────────────────────────────┤  │
│  │ Blend Layer: 上半身 (从spine_01开始)              │◀─┼── Linked Anim Layer (UpperBody)
│  └───────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
                     最终输出姿态
```

### 7.3 场景三：运行时动态切换

```cpp
// 在角色蓝图中切换武器动画层
UFUNCTION(BlueprintCallable, Category = "Weapon")
void EquipWeapon(EWeaponType NewWeaponType)
{
    USkeletalMeshComponent* MeshComp = GetMesh();
    UAnimInstance* AnimInstance = MeshComp->GetAnimInstance();

    // 获取对应武器的动画蓝图类
    TSubclassOf<UAnimInstance> WeaponAnimClass = GetWeaponAnimClass(NewWeaponType);

    // 链接新的武器动画层
    if (WeaponAnimClass)
    {
        AnimInstance->LinkAnimClassLayers(WeaponAnimClass);
    }
    else
    {
        AnimInstance->UnlinkAnimClassLayers(CurrentWeaponAnimClass);
    }

    CurrentWeaponAnimClass = WeaponAnimClass;
}
```

---

## 八、通知传播机制

### 8.1 通知传播配置

```cpp
// FAnimNode_LinkedAnimGraph中
UPROPERTY(EditAnywhere, Category = Settings)
uint8 bReceiveNotifiesFromLinkedInstances : 1;  // 接收来自链接实例的通知

UPROPERTY(EditAnywhere, Category = Settings)
uint8 bPropagateNotifiesToLinkedInstances : 1;  // 向链接实例传播通知
```

### 8.2 通知传播流程

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimInstance.cpp

void UAnimInstance::TriggerSingleAnimNotify(FAnimNotifyEventReference& EventReference)
{
    const FAnimNotifyEvent* AnimNotifyEvent = EventReference.GetNotify();
    
    if (AnimNotifyEvent && (AnimNotifyEvent->NotifyStateClass == NULL))
    {
        // ... 处理自定义事件
        if (AnimNotifyEvent->NotifyName != NAME_None)
        {
            FName FuncName = AnimNotifyEvent->GetNotifyEventName(EventReference.GetMirrorDataTable());
            
            auto NotifyAnimInstance = [this, AnimNotifyEvent, FuncName](UAnimInstance* InAnimInstance)
            {
                check(InAnimInstance);
                
                // 检查是否应该传播到此实例
                if (InAnimInstance == this || InAnimInstance->bReceiveNotifiesFromLinkedInstances)
                {
                    UFunction* Function = InAnimInstance->FindFunction(FuncName);
                    if (Function)
                    {
                        InAnimInstance->ProcessEvent(Function, nullptr);
                    }
                }
            };
            
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

## 九、性能优化建议

### 9.1 实例共享

使用`FAnimSubsystem_SharedLinkedAnimLayers`实现实例共享：

```cpp
// 多个Layer节点可以共享同一个动画实例
// 当它们使用相同的InstanceClass且不同时需要同一个函数时

// 添加持久化类（即使取消链接也保留实例）
SharedLinkedAnimLayers->AddPersistentAnimLayerClass(ABP_Rifle::StaticClass());
```

**优势**：
- 减少实例创建销毁开销
- 降低内存使用
- 维持状态连续性

### 9.2 惯性混合配置

```cpp
// 在动画蓝图中配置混合选项
UPROPERTY(EditAnywhere, Category = "Blend Options")
FAnimGraphBlendOptions BlendOptions;

struct FAnimGraphBlendOptions
{
    float BlendInTime = 0.2f;    // 淡入时间
    float BlendOutTime = 0.2f;   // 淡出时间
    TObjectPtr<UBlendProfile> BlendInProfile;   // 混合配置
    TObjectPtr<UBlendProfile> BlendOutProfile;
};
```

### 9.3 骨骼优化

```cpp
// 只在需要时重新计算所需骨骼
void UAnimInstance::RecalcRequiredBones()
{
    USkeletalMeshComponent* SkelMeshComp = GetSkelMeshComponent();
    
    if (SkelMeshComp && SkelMeshComp->GetSkeletalMeshAsset())
    {
        RequiredBones.InitializeTo(
            SkelMeshComp->GetSkeletalMeshAsset()->GetRefSkeleton(),
            SkelMeshComp->GetPredictedLODLevel(),
            true,  // 包含虚拟骨骼
            GetCurrentSkeleton());
    }
}
```

### 9.4 避免不必要的链接操作

```cpp
// 检查是否需要重新链接
if (CurrentInstanceClass != NewInstanceClass)
{
    // 只有类变更时才重新链接
    SetAnimClass(NewInstanceClass, OwningInstance);
}
```

---

## 十、调试技巧

### 10.1 动画蓝图调试器

在编辑器中：
1. 选择骨骼网格组件
2. 打开动画蓝图调试器
3. 查看链接实例的状态和数据流

### 10.2 运行时查询

```cpp
// 获取所有链接实例
const TArray<TObjectPtr<UAnimInstance>>& LinkedInstances = 
    MeshComp->GetLinkedAnimInstances();

for (UAnimInstance* LinkedInstance : LinkedInstances)
{
    UE_LOG(LogAnimation, Log, TEXT("Linked Instance: %s"), 
        *LinkedInstance->GetClass()->GetName());
}

// 检查特定层的状态
if (UAnimInstance* MainInstance = MeshComp->GetAnimInstance())
{
    UAnimInstance* LayerInstance = MainInstance->GetLinkedAnimLayerInstanceByClass(
        UAnimLayerInterface_Weapon::StaticClass());
    
    if (LayerInstance)
    {
        UE_LOG(LogAnimation, Log, TEXT("Weapon Layer: %s"), 
            *LayerInstance->GetClass()->GetName());
    }
}
```

### 10.3 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 层不执行 | 未正确链接 | 检查接口实现和层名称 |
| 姿态跳变 | 缺少惯性混合 | 配置BlendOptions |
| 通知不触发 | 传播设置错误 | 检查bReceive/bPropagate标志 |
| 内存泄漏 | 实例未清理 | 确保正确调用Unlink |

---

## 十一、总结

### 11.1 关键要点

1. **Linked Anim Graph**：用于复用完整的动画图逻辑
2. **Linked Anim Layer**：用于模块化替换特定动画层
3. **Animation Layer Interface**：定义层的契约和输入/输出
4. **LinkedInputPose**：实现层间的姿态传递
5. **SharedLinkedAnimLayers**：优化共享实例管理

### 11.2 最佳实践

1. **模块化设计**：将复杂动画逻辑分解为独立的层
2. **接口优先**：使用Animation Layer Interface定义清晰的契约
3. **性能意识**：合理使用实例共享和持久化
4. **惯性混合**：配置适当的混合参数确保平滑过渡
5. **调试友好**：利用编辑器工具和日志进行问题排查

### 11.3 扩展阅读

- UE5官方文档：Animation Layers
- Epic Games技术博客：Modular Animation Systems
- GDC演讲：Building Scalable Animation Systems

---

## 参考源码路径

```
Engine/Source/Runtime/Engine/Classes/Animation/
├── AnimNode_LinkedAnimGraph.h
├── AnimNode_LinkedAnimLayer.h  
├── AnimNode_LinkedInputPose.h
├── AnimSubsystem_SharedLinkedAnimLayers.h
└── AnimLayerInterface.h

Engine/Source/Runtime/Engine/Private/Animation/
├── AnimNode_LinkedAnimGraph.cpp
├── AnimNode_LinkedAnimLayer.cpp
├── AnimNode_LinkedInputPose.cpp
└── AnimSubsystem_SharedLinkedAnimLayers.cpp

Engine/Source/Editor/AnimGraph/
├── Public/AnimGraphNode_LinkedAnimGraph.h
├── Public/AnimGraphNode_LinkedAnimLayer.h
├── Private/AnimGraphNode_LinkedAnimGraph.cpp
└── Private/AnimGraphNode_LinkedAnimLayer.cpp
```
