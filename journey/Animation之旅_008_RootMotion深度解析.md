# Animation之旅 008：Root Motion 深度解析

## 前言

在游戏开发中，角色的移动控制是一个核心问题。传统方式下，程序员通过代码直接控制角色的位移，而动画师制作的动画只负责视觉表现。但这种方式往往导致动画脚步与实际移动不匹配的"滑步"现象。

**Root Motion（根骨骼运动）** 技术应运而生——它允许动画本身驱动角色的移动，让动画师拥有更大的控制权，同时确保动画与移动的完美同步。

本篇将深入剖析 UE5 中 Root Motion 的实现原理、提取流程、应用方式以及在网络同步中的处理。

---

## 一、Root Motion 基础概念

### 1.1 什么是 Root Motion？

Root Motion 是指从动画数据中提取根骨骼（通常是 Pelvis 或 Root）的变换信息，并将其应用到角色的实际位移上。

```
传统方式:
[程序控制移动] → [角色位置变化] → [播放行走动画]
问题: 动画脚步可能与移动速度不匹配

Root Motion 方式:
[播放带位移的动画] → [提取根骨骼位移] → [应用到角色移动]
优点: 动画与移动完美同步
```

### 1.2 UAnimSequence 中的 Root Motion 配置

在 `UAnimSequence` 中，Root Motion 相关的核心属性：

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimSequence.h

/** If this is on, it will allow extracting of root motion **/
UPROPERTY(EditAnywhere, AssetRegistrySearchable, Category = RootMotion)
bool bEnableRootMotion;

/** Root Bone will be locked to that position when extracting root motion.**/
UPROPERTY(EditAnywhere, Category = RootMotion)
TEnumAsByte<ERootMotionRootLock::Type> RootMotionRootLock;

/** Force Root Bone Lock even if Root Motion is not enabled */
UPROPERTY(EditAnywhere, Category = RootMotion)
bool bForceRootLock;

/** If this is on, it will use a normalized scale value for the root motion extracted: FVector(1.0, 1.0, 1.0) **/
UPROPERTY(EditAnywhere, AssetRegistrySearchable, Category = RootMotion)
bool bUseNormalizedRootMotionScale;
```

### 1.3 Root Motion Root Lock 类型

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimEnums.h

UENUM()
namespace ERootMotionRootLock
{
    enum Type : int
    {
        /** Use reference pose root bone position. */
        RefPose,

        /** Use root bone position on first frame of animation. */
        AnimFirstFrame,

        /** FTransform::Identity. */
        Zero
    };
}
```

- **RefPose**: 使用参考姿势的根骨骼位置
- **AnimFirstFrame**: 使用动画第一帧的根骨骼位置
- **Zero**: 使用零位置（Identity）

---

## 二、Root Motion 提取流程

### 2.1 核心提取函数

UAnimSequence 提供了三个核心的 Root Motion 提取函数：

```cpp
// 提取指定时间范围内的 Root Motion
ENGINE_API virtual FTransform ExtractRootMotion(
    const FAnimExtractContext& ExtractionContext) const override final;

// 提取连续时间范围内的 Root Motion（不处理循环）
ENGINE_API virtual FTransform ExtractRootMotionFromRange(
    double StartTime, 
    double EndTime, 
    const FAnimExtractContext& ExtractionContext) const override final;

// 提取指定时间点的根骨骼变换
ENGINE_API virtual FTransform ExtractRootTrackTransform(
    const FAnimExtractContext& ExtractionContext, 
    const FBoneContainer* RequiredBones) const override final;
```

### 2.2 FAnimExtractContext 上下文

提取 Root Motion 时需要提供上下文信息：

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimationAsset.h

struct FAnimExtractContext
{
    /** Position in animation to extract pose from */
    double CurrentTime;

    /** Is root motion being extracted? */
    bool bExtractRootMotion;

    /** Delta time range required for root motion extraction **/
    FDeltaTimeRecord DeltaTimeRecord;

    /** Is the current animation asset marked as looping? **/
    bool bLooping;

    // ...其他成员
};
```

### 2.3 提取流程详解

以 `ExtractRootMotionFromRange` 为例，展示完整的提取流程：

```cpp
FTransform UAnimSequence::ExtractRootMotionFromRange(
    double StartTime, 
    double EndTime, 
    const FAnimExtractContext& ExtractionContext) const
{
    // 1. 检查是否启用 Root Motion
    if (!bEnableRootMotion)
    {
        return FTransform::Identity;
    }

    // 2. 计算时间范围
    const double ClampedStart = FMath::Clamp(StartTime, 0.0, GetPlayLength());
    const double ClampedEnd = FMath::Clamp(EndTime, 0.0, GetPlayLength());

    // 3. 提取起始和结束位置的根骨骼变换
    FAnimExtractContext StartContext(ClampedStart);
    FAnimExtractContext EndContext(ClampedEnd);
    
    FTransform StartTransform = ExtractRootTrackTransform(StartContext, nullptr);
    FTransform EndTransform = ExtractRootTrackTransform(EndContext, nullptr);

    // 4. 计算相对变换（Root Motion Delta）
    FTransform DeltaTransform = EndTransform.GetRelativeTransform(StartTransform);

    return DeltaTransform;
}
```

### 2.4 从压缩数据中提取

当动画数据被压缩后，需要从压缩数据中解压提取：

```cpp
void UAnimSequence::GetBoneTransform(
    FTransform& OutAtom, 
    FSkeletonPoseBoneIndex BoneIndex, 
    const FAnimExtractContext& ExtractionContext, 
    bool bUseRawData) const
{
    // 获取压缩数据
    const FCompressedAnimSequence& CompressedData = GetCompressedData(ExtractionContext);
    
    if (CompressedData.IsValid(this) && !bUseRawData)
    {
        // 从压缩数据解压骨骼变换
        FAnimSequenceDecompressionContext DecompContext(
            ExtractionContext.CurrentTime,
            GetPlayLength(),
            Interpolation,
            CompressedData
        );
        
        // 解压特定骨骼的变换
        DecompressBone(OutAtom, BoneIndex, DecompContext);
    }
    else
    {
        // 使用原始数据
        GetBoneTransformFromRawData(OutAtom, BoneIndex, ExtractionContext);
    }
}
```

---

## 三、FRootMotionMovementParams - Root Motion 累积

### 3.1 核心数据结构

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimationAsset.h

USTRUCT()
struct FRootMotionMovementParams
{
    GENERATED_USTRUCT_BODY()

    UPROPERTY()
    bool bHasRootMotion;

    UPROPERTY()
    float BlendWeight;

private:
    UPROPERTY()
    FTransform RootMotionTransform;

public:
    // 设置 Root Motion
    void Set(const FTransform& InTransform)
    {
        bHasRootMotion = true;
        RootMotionTransform = InTransform;
        RootMotionTransform.SetScale3D(RootMotionScale);
        BlendWeight = 1.f;
    }

    // 累积 Root Motion（顺序很重要）
    void Accumulate(const FTransform& InTransform)
    {
        if (!bHasRootMotion)
        {
            Set(InTransform);
        }
        else
        {
            // 注意：先旋转后平移的顺序
            RootMotionTransform = InTransform * RootMotionTransform;
            RootMotionTransform.SetScale3D(RootMotionScale);
        }
    }

    // 带权重的累积（用于混合）
    void AccumulateWithBlend(const FTransform& InTransform, float InBlendWeight)
    {
        const ScalarRegister VBlendWeight(InBlendWeight);
        if (bHasRootMotion)
        {
            RootMotionTransform.AccumulateWithShortestRotation(InTransform, VBlendWeight);
            RootMotionTransform.SetScale3D(RootMotionScale);
            BlendWeight += InBlendWeight;
        }
        else
        {
            Set(InTransform * VBlendWeight);
            BlendWeight = InBlendWeight;
        }
    }
};
```

### 3.2 动画播放时的 Root Motion 累积

在动画 Tick 过程中累积 Root Motion：

```cpp
// FAnimAssetTickContext 中的 Root Motion 处理
struct FAnimAssetTickContext
{
    // Root Motion 累积结果
    FRootMotionMovementParams RootMotionMovementParams;

    // Root Motion 模式
    ERootMotionMode::Type RootMotionMode;
    
    // ...
};
```

---

## 四、Root Motion Mode - 应用模式

### 4.1 ERootMotionMode 枚举

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimEnums.h

UENUM()
namespace ERootMotionMode
{
    enum Type : int
    {
        /** Leave root motion in animation. */
        NoRootMotionExtraction,

        /** Extract root motion but do not apply it. */
        IgnoreRootMotion,

        /** Root motion is taken from all animations contributing to the final pose,
         *  not suitable for network multiplayer setups. */
        RootMotionFromEverything,

        /** Root motion is only taken from montages, 
         *  suitable for network multiplayer setups. */
        RootMotionFromMontagesOnly,
    };
}
```

### 4.2 各模式详解

| 模式 | 描述 | 适用场景 |
|------|------|----------|
| NoRootMotionExtraction | 不提取 Root Motion，保留在动画中 | 不需要 Root Motion 的动画 |
| IgnoreRootMotion | 提取但不应用 | 需要获取但不使用的场景 |
| RootMotionFromEverything | 从所有动画提取 | 单机游戏，所有动画都需要 Root Motion |
| RootMotionFromMontagesOnly | 仅从 Montage 提取 | 网络游戏，便于同步控制 |

### 4.3 为什么网络游戏推荐 MontagesOnly？

```
网络游戏中的挑战：
1. 普通动画播放时机在客户端和服务器可能不同步
2. 动画混合状态难以精确复制
3. Root Motion 累积的微小差异会导致位置偏移

Montage 的优势：
1. Montage 有明确的开始/结束点
2. 可以通过 RPC 精确同步 Montage 的播放
3. Server 可以作为权威来源
4. 容易进行位置校正
```

---

## 五、Montage 中的 Root Motion

### 5.1 Montage Root Motion 提取

Montage 由多个动画片段组成，需要遍历片段提取 Root Motion：

```cpp
// Engine/Source/Runtime/Engine/Private/Animation/AnimCompositeBase.cpp

void UAnimCompositeBase::ExtractRootMotionFromTrack(
    const FAnimTrack& SlotAnimTrack, 
    float StartTrackPosition, 
    float EndTrackPosition, 
    const FAnimExtractContext& Context,
    FRootMotionMovementParams& RootMotion) const
{
    // 获取时间范围内的所有 Root Motion 步骤
    TArray<FRootMotionExtractionStep> RootMotionExtractionSteps;
    SlotAnimTrack.GetRootMotionExtractionStepsForTrackRange(
        RootMotionExtractionSteps, 
        StartTrackPosition, 
        EndTrackPosition);

    // 按顺序累积 Root Motion
    for (int32 StepIndex = 0; StepIndex < RootMotionExtractionSteps.Num(); StepIndex++)
    {
        const FRootMotionExtractionStep& CurrentStep = RootMotionExtractionSteps[StepIndex];
        
        if (CurrentStep.AnimSequence->bEnableRootMotion)
        {
            FTransform DeltaTransform = CurrentStep.AnimSequence->ExtractRootMotionFromRange(
                CurrentStep.StartPosition, 
                CurrentStep.EndPosition, 
                Context);
                
            RootMotion.Accumulate(DeltaTransform);
        }
    }
}
```

### 5.2 FRootMotionExtractionStep

```cpp
struct FRootMotionExtractionStep
{
    UAnimSequence* AnimSequence;
    float StartPosition;  // 动画内的开始位置
    float EndPosition;    // 动画内的结束位置
};
```

### 5.3 处理循环动画

当 Montage 中的动画设置了循环时：

```cpp
void FAnimSegment::GetRootMotionExtractionStepsForTrackRange(
    TArray<FRootMotionExtractionStep>& RootMotionExtractionSteps, 
    const float StartTrackPosition, 
    const float EndTrackPosition) const
{
    // ...
    
    for(int32 IterationsLeft = FMath::Max(LoopingCount, 1); 
        ((IterationsLeft > 0) && (TrackTimeToGo > 0.f)); 
        --IterationsLeft)
    {
        // 计算到动画结束点的时间
        const float TrackTimeToAnimEndPoint = (AnimEndPoint - AnimStartPosition) / ValidPlayRate;

        if(FMath::Abs(TrackTimeToGo) < FMath::Abs(TrackTimeToAnimEndPoint))
        {
            // 还没到动画结束，添加这段
            const float AnimEndPosition = (TrackTimeToGo * PlayRate) + AnimStartPosition;
            RootMotionExtractionSteps.Add(FRootMotionExtractionStep(
                AnimSequence, AnimStartPosition, AnimEndPosition));
            break;
        }
        else
        {
            // 到达动画结束点，添加到结束的这段
            RootMotionExtractionSteps.Add(FRootMotionExtractionStep(
                AnimSequence, AnimStartPosition, AnimEndPoint));

            // 重置到动画开始，准备下一次循环
            TrackTimeToGo -= TrackTimeToAnimEndPoint;
            AnimStartPosition = ResetStartPosition;
        }
    }
}
```

---

## 六、FRootMotionSource - 代码驱动的 Root Motion

### 6.1 Root Motion Source 概述

除了动画驱动的 Root Motion，UE5 还支持代码驱动的 Root Motion：

```cpp
// Engine/Source/Runtime/Engine/Classes/GameFramework/RootMotionSource.h

USTRUCT()
struct FRootMotionSource
{
    GENERATED_USTRUCT_BODY()

    /** Priority of this source relative to other sources */
    UPROPERTY()
    uint16 Priority;

    /** ID local to this client or server instance */
    UPROPERTY()
    uint16 LocalID;

    /** Instance name for finding the source later */
    UPROPERTY()
    FName InstanceName;

    /** Time elapsed so far for this source */
    UPROPERTY()
    float CurrentTime;

    /** The length of this root motion - < 0 for infinite */
    UPROPERTY()
    float Duration;

    /** Accumulation mode (override or additive) */
    UPROPERTY()
    ERootMotionAccumulateMode AccumulateMode;

    /** True when contributing local space accumulation */
    UPROPERTY()
    bool bInLocalSpace;

    /** Root Motion generated by this Source */
    UPROPERTY(NotReplicated)
    FRootMotionMovementParams RootMotionParams;

    // 准备 Root Motion（每帧调用）
    virtual void PrepareRootMotion(
        float SimulationTime,
        float MovementTickTime,
        const ACharacter& Character, 
        const UCharacterMovementComponent& MoveComponent);
};
```

### 6.2 内置的 Root Motion Source 类型

#### 6.2.1 FRootMotionSource_ConstantForce

持续施加固定方向的力：

```cpp
USTRUCT()
struct FRootMotionSource_ConstantForce : public FRootMotionSource
{
    UPROPERTY()
    FVector Force;

    UPROPERTY()
    TObjectPtr<UCurveFloat> StrengthOverTime;

    virtual void PrepareRootMotion(...) override
    {
        FTransform NewTransform(Force);

        // 根据时间曲线调整强度
        if (StrengthOverTime)
        {
            const float TimeValue = Duration > 0.f 
                ? FMath::Clamp(GetTime() / Duration, 0.f, 1.f) 
                : GetTime();
            const float TimeFactor = StrengthOverTime->GetFloatValue(TimeValue);
            NewTransform.ScaleTranslation(TimeFactor);
        }

        // 根据时间比例缩放
        const float Multiplier = (MovementTickTime > UE_SMALL_NUMBER) 
            ? (SimulationTime / MovementTickTime) 
            : 1.f;
        NewTransform.ScaleTranslation(Multiplier);

        RootMotionParams.Set(NewTransform);
        SetTime(GetTime() + SimulationTime);
    }
};
```

#### 6.2.2 FRootMotionSource_MoveToForce

移动到指定位置：

```cpp
USTRUCT()
struct FRootMotionSource_MoveToForce : public FRootMotionSource
{
    UPROPERTY()
    FVector StartLocation;

    UPROPERTY()
    FVector TargetLocation;

    UPROPERTY()
    bool bRestrictSpeedToExpected;

    UPROPERTY()
    TObjectPtr<UCurveVector> PathOffsetCurve;  // 路径偏移曲线

    virtual void PrepareRootMotion(...) override
    {
        if (Duration > UE_SMALL_NUMBER && MovementTickTime > UE_SMALL_NUMBER)
        {
            const float MoveFraction = (GetTime() + SimulationTime) / Duration;

            // 计算当前目标位置
            FVector CurrentTargetLocation = FMath::Lerp<FVector, float>(
                StartLocation, TargetLocation, MoveFraction);
            CurrentTargetLocation += GetPathOffsetInWorldSpace(MoveFraction);

            const FVector CurrentLocation = Character.GetActorLocation();

            // 计算需要的速度
            FVector Force = (CurrentTargetLocation - CurrentLocation) / MovementTickTime;

            // 可选：限制速度不超过预期
            if (bRestrictSpeedToExpected && !Force.IsNearlyZero())
            {
                // ... 速度限制逻辑
            }

            FTransform NewTransform(Force);
            RootMotionParams.Set(NewTransform);
        }

        SetTime(GetTime() + SimulationTime);
    }
};
```

#### 6.2.3 FRootMotionSource_JumpForce

跳跃力：

```cpp
USTRUCT()
struct FRootMotionSource_JumpForce : public FRootMotionSource
{
    UPROPERTY()
    FRotator Rotation;

    UPROPERTY()
    float Distance;

    UPROPERTY()
    float Height;

    UPROPERTY()
    bool bDisableTimeout;  // 是否在落地时才结束

    UPROPERTY()
    TObjectPtr<UCurveVector> PathOffsetCurve;

    UPROPERTY()
    TObjectPtr<UCurveFloat> TimeMappingCurve;
};
```

### 6.3 FRootMotionSourceGroup - Source 管理

```cpp
USTRUCT()
struct FRootMotionSourceGroup
{
    /** Root Motion Sources currently applied */
    TArray<TSharedPtr<FRootMotionSource>> RootMotionSources;

    /** Root Motion Sources to be added next frame */
    TArray<TSharedPtr<FRootMotionSource>> PendingAddRootMotionSources;

    /** Whether this group has additive sources */
    uint8 bHasAdditiveSources:1;

    /** Whether this group has override sources */
    uint8 bHasOverrideSources:1;

    // 应用一个 Root Motion Source
    uint16 ApplyRootMotionSource(TSharedPtr<FRootMotionSource> SourcePtr);

    // 准备所有 Root Motion
    void PrepareRootMotion(float DeltaTime, const ACharacter& Character, 
        const UCharacterMovementComponent& InMoveComponent, bool bForcePrepareAll = false);

    // 累积 Override 类型的 Root Motion 到速度
    void AccumulateOverrideRootMotionVelocity(float DeltaTime, 
        const ACharacter& Character, 
        const UCharacterMovementComponent& MoveComponent, 
        FVector& InOutVelocity) const;

    // 累积 Additive 类型的 Root Motion 到速度
    void AccumulateAdditiveRootMotionVelocity(float DeltaTime, 
        const ACharacter& Character, 
        const UCharacterMovementComponent& MoveComponent, 
        FVector& InOutVelocity) const;
};
```

### 6.4 累积模式

```cpp
UENUM()
enum class ERootMotionAccumulateMode : uint8
{
    // 直接设置速度（覆盖）
    Override = 0, 
    // 叠加到现有速度上
    Additive = 1
};
```

---

## 七、Motion Warping - Root Motion 扭曲

### 7.1 什么是 Motion Warping？

Motion Warping 允许在运行时动态调整 Root Motion，使角色能够精确到达目标位置：

```
应用场景：
- 攻击动画需要精确击中目标
- 翻越障碍需要精确到达目标点
- 开门动画需要手部精确接触门把手
```

### 7.2 UMotionWarpingComponent

```cpp
// Engine/Plugins/Animation/MotionWarping/Source/MotionWarping/Private/MotionWarpingComponent.cpp

UCLASS()
class UMotionWarpingComponent : public UActorComponent
{
    // Warp 目标列表
    TArray<FMotionWarpingTarget> WarpTargets;

    // Root Motion 修改器列表
    TArray<URootMotionModifier*> Modifiers;

    // 处理 Root Motion（在转换为世界空间之前）
    FTransform ProcessRootMotionPreConvertToWorld(
        const FTransform& InRootMotion, 
        float DeltaSeconds,
        const FMotionWarpingUpdateContext* InContext);
};
```

### 7.3 FMotionWarpingTarget

```cpp
USTRUCT(BlueprintType)
struct FMotionWarpingTarget
{
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FName Name;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector Location;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FRotator Rotation;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TWeakObjectPtr<USceneComponent> Component;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bFollowComponent;
};
```

### 7.4 URootMotionModifier

```cpp
UCLASS()
class URootMotionModifier : public UObject
{
    // 动画引用
    UPROPERTY()
    TWeakObjectPtr<const UAnimSequenceBase> Animation;

    // 开始/结束时间
    UPROPERTY()
    float StartTime;

    UPROPERTY()
    float EndTime;

    // 处理 Root Motion
    virtual FTransform ProcessRootMotion(
        const FTransform& InRootMotion, 
        float DeltaSeconds) 
    { 
        return InRootMotion; 
    }
};
```

---

## 八、Root Motion 在角色移动组件中的应用

### 8.1 CharacterMovementComponent 中的处理

```cpp
void UCharacterMovementComponent::PerformMovement(float DeltaTime)
{
    // 1. 准备 Root Motion Sources
    CurrentRootMotion.PrepareRootMotion(DeltaTime, *CharacterOwner, *this);

    // 2. 判断是否有 Root Motion
    const bool bHasRootMotionFromAnimation = HasAnimRootMotion();
    const bool bHasRootMotionFromSources = HasRootMotionSources();

    // 3. 处理动画 Root Motion
    if (bHasRootMotionFromAnimation)
    {
        // 从动画实例获取 Root Motion
        FRootMotionMovementParams RootMotionParams = RootMotionParams;
        
        if (RootMotionParams.bHasRootMotion)
        {
            // 转换到世界空间
            FTransform WorldSpaceRootMotion = ConvertLocalRootMotionToWorld(
                RootMotionParams.GetRootMotionTransform());
            
            // 应用到速度
            Velocity = WorldSpaceRootMotion.GetTranslation() / DeltaTime;
        }
    }

    // 4. 处理 Root Motion Sources
    if (bHasRootMotionFromSources)
    {
        // 先应用 Override 类型
        if (CurrentRootMotion.HasOverrideVelocity())
        {
            CurrentRootMotion.AccumulateOverrideRootMotionVelocity(
                DeltaTime, *CharacterOwner, *this, Velocity);
        }

        // 再应用 Additive 类型
        if (CurrentRootMotion.HasAdditiveVelocity())
        {
            CurrentRootMotion.AccumulateAdditiveRootMotionVelocity(
                DeltaTime, *CharacterOwner, *this, Velocity);
        }
    }

    // 5. 执行实际移动
    // ...
}
```

### 8.2 本地空间到世界空间的转换

```cpp
FTransform UCharacterMovementComponent::ConvertLocalRootMotionToWorld(
    const FTransform& LocalRootMotion) const
{
    const FTransform& ActorTransform = CharacterOwner->GetActorTransform();
    
    // 旋转平移向量到世界空间
    FVector WorldTranslation = ActorTransform.TransformVector(
        LocalRootMotion.GetTranslation());
    
    // 组合旋转
    FQuat WorldRotation = ActorTransform.GetRotation() * LocalRootMotion.GetRotation();
    
    return FTransform(WorldRotation, WorldTranslation);
}
```

---

## 九、网络同步中的 Root Motion

### 9.1 Montage Root Motion 的网络同步

由于 Montage 有明确的播放控制，可以精确同步：

```cpp
// Server 端播放 Montage
void ACharacter::PlayMontageOnServer(UAnimMontage* Montage)
{
    // 1. 在服务器播放
    PlayAnimMontage(Montage);
    
    // 2. 通知所有客户端
    MulticastPlayMontage(Montage);
}

// 客户端接收
void ACharacter::MulticastPlayMontage_Implementation(UAnimMontage* Montage)
{
    // 在客户端播放相同的 Montage
    PlayAnimMontage(Montage);
}
```

### 9.2 Root Motion Source 的网络同步

Root Motion Sources 需要特殊处理来保持同步：

```cpp
// FRootMotionSourceGroup 的网络序列化
bool FRootMotionSourceGroup::NetSerialize(FArchive& Ar, UPackageMap* Map, 
    bool& bOutSuccess, uint8 MaxNumRootMotionSourcesToSerialize)
{
    // 序列化 Root Motion Sources 数组
    NetSerializeRMSArray(Ar, Map, bOutSuccess, RootMotionSources, 
        MaxNumRootMotionSourcesToSerialize);
    
    // 序列化待添加的 Sources
    NetSerializeRMSArray(Ar, Map, bOutSuccess, PendingAddRootMotionSources,
        MaxNumRootMotionSourcesToSerialize);
    
    return true;
}
```

### 9.3 Server 校正

当客户端和服务器出现位置偏差时：

```cpp
// 服务器发送校正
void UCharacterMovementComponent::ServerMoveHandleClientError(...)
{
    // 比较客户端和服务器的位置
    if (FVector::Dist(ClientLoc, ServerLoc) > MaxClientLocationError)
    {
        // 发送校正数据，包括 Root Motion Source 状态
        ClientAckGoodMove_Implementation(ClientData);
    }
}
```

---

## 十、Root Motion 的调试

### 10.1 控制台变量

```cpp
// 调试 Root Motion
static TAutoConsoleVariable<int32> CVarDebugRootMotionSources(
    TEXT("p.RootMotion.Debug"),
    0,
    TEXT("Whether to draw root motion source debug information.\n")
    TEXT("0: Disable, 1: Enable"),
    ECVF_Cheat);

// 调试绘制持续时间
static TAutoConsoleVariable<float> CVarDebugRootMotionSourcesLifetime(
    TEXT("p.RootMotion.DebugSourceLifeTime"),
    6.f,
    TEXT("How long a visualized root motion source persists.\n"),
    ECVF_Cheat);
```

### 10.2 在编辑器中调试

```cpp
// 打印 Root Motion 调试信息
#if ROOT_MOTION_DEBUG
void RootMotionSourceDebug::PrintOnScreen(
    const ACharacter& InCharacter, 
    const FString& InString)
{
    if (InCharacter.IsPlayerControlled())
    {
        const FString AdjustedDebugString = FString::Printf(
            TEXT("[%" UINT64_FMT "] [%s] %s"), 
            (uint64)GFrameCounter, 
            *InCharacter.GetName(), 
            *InString);

        GEngine->AddOnScreenDebugMessage(
            INDEX_NONE, 0.f, FColor::Green, 
            AdjustedDebugString, false, FVector2D::UnitVector * 1.5f);
    }
}
#endif
```

---

## 十一、最佳实践

### 11.1 动画制作建议

1. **确保根骨骼移动平滑**
   - 避免根骨骼的突然跳跃
   - 使用曲线平滑过渡

2. **考虑不同播放速率**
   - Root Motion 会随播放速率缩放
   - 确保在不同速率下移动仍然合理

3. **标记清晰的动画区间**
   - 使用 Notify 标记 Root Motion 窗口
   - 便于 Motion Warping 使用

### 11.2 程序设计建议

1. **网络游戏使用 MontagesOnly 模式**
   ```cpp
   // 在 AnimInstance 中设置
   GetAnimInstanceProxy()->SetRootMotionMode(ERootMotionMode::RootMotionFromMontagesOnly);
   ```

2. **合理使用 Root Motion Source**
   - Override 用于完全控制移动
   - Additive 用于叠加效果

3. **Motion Warping 配置**
   - 只在必要时启用
   - 合理设置 Warp 窗口

### 11.3 性能考虑

1. **压缩数据的使用**
   - 运行时使用压缩数据
   - 编辑器可使用原始数据进行调试

2. **避免每帧创建 Root Motion Source**
   - 复用现有 Source
   - 使用对象池

---

## 十二、总结

Root Motion 是 UE5 动画系统中的重要组成部分，它实现了：

1. **动画驱动移动**：从动画数据中提取根骨骼变换
2. **代码驱动移动**：通过 Root Motion Source 实现程序化控制
3. **运行时调整**：Motion Warping 允许动态修正目标位置
4. **网络同步**：支持多人游戏中的同步需求

理解 Root Motion 的工作原理对于制作高质量的角色动画至关重要。无论是简单的行走动画，还是复杂的攻击连招，都可以借助 Root Motion 实现动画与移动的完美统一。

---

## 参考资源

- [UE5 官方文档 - Root Motion](https://docs.unrealengine.com/5.0/en-US/root-motion-in-unreal-engine/)
- [Motion Warping 插件文档](https://docs.unrealengine.com/5.0/en-US/motion-warping-in-unreal-engine/)
- 源码路径：
  - `Engine/Source/Runtime/Engine/Classes/Animation/AnimSequence.h`
  - `Engine/Source/Runtime/Engine/Classes/GameFramework/RootMotionSource.h`
  - `Engine/Plugins/Animation/MotionWarping/`
