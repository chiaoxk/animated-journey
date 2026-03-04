# Animation之旅_006：骨骼重定向与IK系统

> 本文深入剖析UE5动画系统中**骨骼重定向（Retargeting）**和**逆向运动学（Inverse Kinematics）**的实现原理，从数学基础到源码实现，带你理解这两个动画系统中至关重要的技术。

## 一、骨骼重定向概述

### 1.1 什么是骨骼重定向？

**骨骼重定向（Skeleton Retargeting）**是将为一个骨架创建的动画数据迁移到另一个骨架上的技术。这使得动画资产可以在不同比例、不同骨骼结构的角色之间共享。

**核心挑战**：
- 骨骼比例差异（身高、四肢长度）
- 骨骼层级差异（骨骼数量、命名不同）
- 姿态参考差异（T-Pose vs A-Pose）

### 1.2 UE5重定向系统架构

UE5.6引入了全新的**IK Retargeter**系统，基于**Operation Stack（操作栈）**架构：

```
┌─────────────────────────────────────────────────────────────┐
│                    UIKRetargeter                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               Operation Stack                        │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │ FIKRetargetScaleSourceOp (缩放源姿态)        │    │   │
│  │  ├─────────────────────────────────────────────┤    │   │
│  │  │ FIKRetargetPelvisMotionOp (骨盆运动)         │    │   │
│  │  ├─────────────────────────────────────────────┤    │   │
│  │  │ FIKRetargetFKChainsOp (FK链重定向)           │    │   │
│  │  ├─────────────────────────────────────────────┤    │   │
│  │  │ FIKRetargetIKChainsOp (IK链重定向)           │    │   │
│  │  ├─────────────────────────────────────────────┤    │   │
│  │  │ FIKRetargetSpeedPlantOp (速度植入)           │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 核心类关系

```cpp
// 重定向资产
class UIKRetargeter : public UObject
{
    // 源IK Rig资产
    TObjectPtr<UIKRigDefinition> SourceIKRigAsset;
    // 目标IK Rig资产  
    TObjectPtr<UIKRigDefinition> TargetIKRigAsset;
    // 操作栈
    TArray<FInstancedStruct> RetargetOps;
    // 重定向姿态
    TMap<FName, FIKRetargetPose> SourceRetargetPoses;
    TMap<FName, FIKRetargetPose> TargetRetargetPoses;
};

// 运行时处理器
struct FIKRetargetProcessor
{
    FRetargetSkeleton SourceSkeleton;  // 源骨架数据
    FTargetSkeleton TargetSkeleton;    // 目标骨架数据
    FRetargeterBoneChains AllBoneChains;  // 骨骼链数据
    TArray<FInstancedStruct> OpStack;  // 操作栈实例
};
```

---

## 二、骨骼链映射（Chain Mapping）

### 2.1 FBoneChain - 骨骼链定义

```cpp
USTRUCT(BlueprintType)
struct FBoneChain
{
    GENERATED_BODY()

    // 链名称（如"LeftArm"、"Spine"）
    UPROPERTY(EditAnywhere)
    FName ChainName;

    // 起始骨骼
    UPROPERTY(EditAnywhere)
    FBoneReference StartBone;

    // 结束骨骼
    UPROPERTY(EditAnywhere)
    FBoneReference EndBone;

    // 关联的IK目标（可选）
    UPROPERTY(EditAnywhere)
    FName IKGoalName;
};
```

### 2.2 FResolvedBoneChain - 解析后的骨骼链

```cpp
struct FResolvedBoneChain
{
    FName ChainName;
    FName StartBone;
    FName EndBone;
    FName IKGoalName;
    
    // 链中所有骨骼的索引（从根到末端）
    TArray<int32> BoneIndices;
    
    // 参考姿态的全局变换
    TArray<FTransform> RefPoseGlobalTransforms;
    // 参考姿态的局部变换
    TArray<FTransform> RefPoseLocalTransforms;
    
    // 每个骨骼在链上的参数（0.0~1.0）
    TArray<float> Params;
    
    // 链的初始长度
    float InitialChainLength;
    
    // 链父骨骼信息
    int32 ChainParentBoneIndex;
    FTransform ChainParentInitialGlobalTransform;
    
    // 在指定参数位置获取变换
    FTransform GetTransformAtChainParam(
        const TArray<FTransform>& Transforms,
        const double& Param) const;
    
    // 获取指定参数处的拉伸比
    double GetStretchAtParam(
        const TArray<FTransform>& InitialTransforms,
        const TArray<FTransform>& CurrentTransforms,
        const double& Param) const;
};
```

### 2.3 骨骼链参数化

骨骼链的**参数化**是重定向的关键。它允许在不同骨骼数量的链之间进行映射：

```cpp
void FResolvedBoneChain::CalculateBoneParameters(FIKRigLogger& Log)
{
    Params.Reset();
    
    // 单骨骼链特殊处理
    if (RefPoseGlobalTransforms.Num() == 1)
    {
        Params.Add(1.0f);
        return;
    }

    // 计算链的总长度
    TArray<float> BoneDistances;
    InitialChainLength = 0.0f;
    BoneDistances.Add(0.0f);
    
    for (int32 i = 1; i < RefPoseGlobalTransforms.Num(); ++i)
    {
        InitialChainLength += (RefPoseGlobalTransforms[i].GetTranslation() - 
                               RefPoseGlobalTransforms[i-1].GetTranslation()).Size();
        BoneDistances.Add(InitialChainLength);
    }

    // 归一化参数：每个骨骼的位置参数 = 累积距离 / 总长度
    const float Divisor = InitialChainLength > UE_KINDA_SMALL_NUMBER ? 
                          InitialChainLength : UE_KINDA_SMALL_NUMBER;

    for (int32 i = 0; i < RefPoseGlobalTransforms.Num(); ++i)
    {
        Params.Add(BoneDistances[i] / Divisor); 
    }
}
```

**示意图**：
```
源链（3骨骼）:     目标链（5骨骼）:
  A (0.0)           1 (0.0)
  |                 |
  B (0.5)           2 (0.25)
  |                 |
  C (1.0)           3 (0.5)
                    |
                    4 (0.75)
                    |
                    5 (1.0)
                    
通过参数化，B(0.5)可以映射到3(0.5)
```

### 2.4 链到链映射

```cpp
struct FRetargetChainMapping
{
    // 源链名 -> 目标链名
    TMap<FName, FName> SourceToTargetChainMap;
    
    // 获取映射的目标链
    FName GetTargetChainForSource(const FName& SourceChainName) const
    {
        if (const FName* TargetChain = SourceToTargetChainMap.Find(SourceChainName))
        {
            return *TargetChain;
        }
        return NAME_None;
    }
    
    // 双向查找
    FName GetChainMappedTo(const FName& InChainName, 
                           ERetargetSourceOrTarget InSourceOrTarget) const;
};
```

---

## 三、重定向姿态（Retarget Pose）

### 3.1 FIKRetargetPose - 重定向姿态定义

```cpp
USTRUCT(BlueprintType)
struct FIKRetargetPose
{
    GENERATED_BODY()
    
private:
    // 骨盆的全局空间位移增量
    UPROPERTY(EditAnywhere, Category = RetargetPose)
    FVector RootTranslationOffset = FVector::ZeroVector;

    // 局部空间旋转增量（骨骼名 -> 旋转四元数）
    UPROPERTY(EditAnywhere, Category = RetargetPose)
    TMap<FName, FQuat> BoneRotationOffsets;
    
    // 版本号（用于检测修改）
    int32 Version = INDEX_NONE;

public:
    FQuat GetDeltaRotationForBone(const FName BoneName) const;
    void SetDeltaRotationForBone(FName BoneName, const FQuat& RotationDelta);
    
    FVector GetRootTranslationDelta() const;
    void SetRootTranslationDelta(const FVector& TranslationDelta);
};
```

### 3.2 FResolvedRetargetPose - 解析后的重定向姿态

重定向姿态需要**解析到具体骨架**上才能使用：

```cpp
struct FResolvedRetargetPose
{
    FName Name;                          // 姿态名称
    int32 Version;                       // 版本号
    FRetargetPoseScaleWithPivot PoseScale;  // 缩放信息
    
    TArray<FTransform> LocalPose;        // 局部空间姿态
    TArray<FTransform> GlobalPose;       // 全局空间姿态
};

// 添加或更新重定向姿态
FResolvedRetargetPose& FResolvedRetargetPoseSet::AddOrUpdateRetargetPose(
    const FRetargetSkeleton& InSkeleton,
    const FName InRetargetPoseName,
    const FIKRetargetPose* InRetargetPose,
    const FName PelvisBoneName,
    const FRetargetPoseScaleWithPivot& InSourceScale)
{
    FResolvedRetargetPose& RetargetPose = FindOrAddRetargetPose(InRetargetPoseName);
    
    RetargetPose.Version = InRetargetPose->GetVersion();
    RetargetPose.PoseScale = InSourceScale;
    
    // 从骨架参考姿态初始化
    RetargetPose.LocalPose = InSkeleton.SkeletalMesh->GetRefSkeleton().GetRefBonePose();
    RetargetPose.GlobalPose = RetargetPose.LocalPose;
    
    // 转换到全局空间
    InSkeleton.UpdateGlobalTransformsBelowBone(INDEX_NONE, 
        RetargetPose.LocalPose, RetargetPose.GlobalPose);

    // 移除缩放（缩放会烘焙到位移中）
    for (int32 BoneIndex = 0; BoneIndex < InSkeleton.BoneNames.Num(); ++BoneIndex)
    {
        RetargetPose.LocalPose[BoneIndex].SetScale3D(FVector::OneVector);
        RetargetPose.GlobalPose[BoneIndex].SetScale3D(FVector::OneVector);
    }

    // 应用骨盆位移偏移
    const int32 PelvisBoneIndex = InSkeleton.FindBoneIndexByName(PelvisBoneName);
    if (PelvisBoneIndex != INDEX_NONE)
    {
        FTransform& PelvisTransform = RetargetPose.GlobalPose[PelvisBoneIndex];
        PelvisTransform.AddToTranslation(InRetargetPose->GetRootTranslationDelta());
        InSkeleton.UpdateLocalTransformOfSingleBone(PelvisBoneIndex, 
            RetargetPose.LocalPose, RetargetPose.GlobalPose);
    }

    // 应用骨骼旋转偏移
    const TArray<FTransform>& RefPoseLocal = 
        InSkeleton.SkeletalMesh->GetRefSkeleton().GetRefBonePose();
        
    for (const TTuple<FName, FQuat>& BoneDelta : InRetargetPose->GetAllDeltaRotations())
    {
        const int32 BoneIndex = InSkeleton.FindBoneIndexByName(BoneDelta.Key);
        if (BoneIndex == INDEX_NONE) continue;

        // 局部旋转 = 参考姿态旋转 * 偏移旋转
        const FQuat LocalBoneRotation = RefPoseLocal[BoneIndex].GetRotation() * BoneDelta.Value;
        RetargetPose.LocalPose[BoneIndex].SetRotation(LocalBoneRotation);
    }

    // 更新全局变换
    InSkeleton.UpdateGlobalTransformsBelowBone(INDEX_NONE, 
        RetargetPose.LocalPose, RetargetPose.GlobalPose);

    // 应用缩放
    if (RetargetPose.PoseScale.ScalePose(RetargetPose.GlobalPose))
    {
        InSkeleton.UpdateLocalTransformsBelowBone(INDEX_NONE, 
            RetargetPose.LocalPose, RetargetPose.GlobalPose);
    }

    return RetargetPose;
}
```

### 3.3 为什么需要重定向姿态？

重定向姿态解决了**参考姿态差异**的问题：

| 问题场景 | 解决方案 |
|----------|----------|
| 源用T-Pose，目标用A-Pose | 调整重定向姿态使手臂角度匹配 |
| 源角色更高，目标角色更矮 | 使用骨盆偏移调整高度差 |
| 默认站姿脚间距不同 | 调整腿部骨骼旋转 |

---

## 四、FK链重定向

### 4.1 FK重定向原理

**FK（Forward Kinematics）重定向**的核心思想是：
1. 从源骨架提取每个链的**旋转信息**
2. 根据映射关系将旋转**传递到目标链**
3. 考虑骨骼数量差异进行**插值**

### 4.2 FChainEncoderFK - FK编码器

```cpp
struct FChainEncoderFK
{
    // 当前全局变换
    TArray<FTransform> CurrentGlobalTransforms;
    // 链父骨骼的当前全局变换
    FTransform ChainParentCurrentGlobalTransform;
    // 关联的解析骨骼链
    const FResolvedBoneChain* BoneChain;
    
    // 从源姿态编码
    void EncodePose(
        const FRetargetSkeleton& SourceSkeleton,
        const TArray<int32>& SourceBoneIndices,
        const TArray<FTransform>& InSourceGlobalPose)
    {
        // 更新链父骨骼变换
        if (BoneChain->ChainParentBoneIndex != INDEX_NONE)
        {
            ChainParentCurrentGlobalTransform = 
                InSourceGlobalPose[BoneChain->ChainParentBoneIndex];
        }
        
        // 提取链中每个骨骼的当前全局变换
        CurrentGlobalTransforms.SetNum(SourceBoneIndices.Num());
        for (int32 ChainIndex = 0; ChainIndex < SourceBoneIndices.Num(); ++ChainIndex)
        {
            const int32 BoneIndex = SourceBoneIndices[ChainIndex];
            CurrentGlobalTransforms[ChainIndex] = InSourceGlobalPose[BoneIndex];
        }
    }
};
```

### 4.3 FChainDecoderFK - FK解码器

```cpp
struct FChainDecoderFK
{
    FChainDecoderCachedData CachedData;
    const FResolvedBoneChain* BoneChain;
    
    // 解码姿态到目标链
    void DecodePose(
        const FIKRetargetPelvisMotionOp* PelvisMotionOp,
        const FRetargetFKChainSettings& Settings,
        const TArray<int32>& TargetBoneIndices,
        FChainEncoderFK& SourceChain,
        const TArray<FTransform>& InSourceGlobalPose,
        const FRetargetSkeleton& SourceSkeleton,
        const FTargetSkeleton& TargetSkeleton,
        TArray<FTransform>& InOutGlobalPose);

private:
    // 重定向旋转
    FQuat RetargetRotation(
        const int32 InChainIndex,
        const FRetargetFKChainSettings& Settings,
        const TArray<int32>& TargetBoneIndices,
        const FTargetSkeleton& TargetSkeleton,
        const TArray<FTransform>& InOutGlobalPose,
        FChainEncoderFK& SourceChain,
        FTransform& OutSourceCurrentTransform,
        FTransform& OutSourceInitialTransform) const;

    // 重定向位置
    FVector RetargetPosition(...) const;
    
    // 重定向缩放
    FVector RetargetScale(...) const;
};
```

### 4.4 旋转重定向模式

```cpp
UENUM(BlueprintType)
enum class EFKChainRotationMode : uint8
{
    None = 5,              // 不重定向旋转
    Interpolated = 0,      // 插值模式（默认）
    OneToOne = 1,          // 一对一映射
    OneToOneReversed = 2,  // 反向一对一映射
    MatchChain = 3,        // 匹配链形状
    MatchScaledChain = 4,  // 匹配缩放后的链形状
    CopyLocal = 6,         // 直接复制局部旋转
};
```

### 4.5 插值模式实现

**Interpolated模式**是最常用的FK重定向模式：

```cpp
FQuat FChainDecoderFK::RetargetRotation(
    const int32 InChainIndex,
    const FRetargetFKChainSettings& Settings,
    const TArray<int32>& TargetBoneIndices,
    const FTargetSkeleton& TargetSkeleton,
    const TArray<FTransform>& InOutGlobalPose,
    FChainEncoderFK& SourceChain,
    FTransform& OutSourceCurrentTransform,
    FTransform& OutSourceInitialTransform) const
{
    switch (Settings.RotationMode)
    {
    case EFKChainRotationMode::Interpolated:
    {
        // 获取目标骨骼在链上的参数位置
        const float TargetParam = BoneChain->Params[InChainIndex];
        
        // 在源链上找到对应参数位置的变换
        OutSourceCurrentTransform = SourceChain.BoneChain->GetTransformAtChainParam(
            SourceChain.CurrentGlobalTransforms, TargetParam);
        OutSourceInitialTransform = SourceChain.BoneChain->GetTransformAtChainParam(
            SourceChain.BoneChain->RefPoseGlobalTransforms, TargetParam);
        
        // 计算源链的旋转变化
        const FQuat SourceCurrentRotation = OutSourceCurrentTransform.GetRotation();
        const FQuat SourceInitialRotation = OutSourceInitialTransform.GetRotation();
        const FQuat SourceDeltaRotation = SourceCurrentRotation * SourceInitialRotation.Inverse();
        
        // 获取目标链的初始旋转
        const FQuat TargetInitialRotation = BoneChain->RefPoseGlobalTransforms[InChainIndex].GetRotation();
        
        // 应用旋转变化到目标
        return SourceDeltaRotation * TargetInitialRotation;
    }
    
    case EFKChainRotationMode::OneToOne:
    {
        // 直接一对一映射（用于骨骼数量相同的链）
        int32 SourceChainIndex = FMath::Clamp(InChainIndex, 0, 
            SourceChain.BoneChain->BoneIndices.Num() - 1);
        // ...
    }
    
    // ... 其他模式
    }
}
```

### 4.6 位置重定向模式

```cpp
UENUM(BlueprintType)
enum class EFKChainTranslationMode : uint8
{
    None,                           // 不重定向位置
    GloballyScaled,                 // 全局缩放
    Absolute,                       // 绝对位置
    StretchBoneLengthUniformly,     // 均匀拉伸骨骼长度
    StretchBoneLengthNonUniformly,  // 非均匀拉伸
    OrientAndScale,                 // 朝向和缩放
};
```

---

## 五、骨盆运动重定向

### 5.1 骨盆运动的重要性

骨盆是角色运动的**核心**：
- 决定角色的整体高度
- 影响所有四肢链的起点
- 包含根运动数据

### 5.2 FIKRetargetPelvisMotionOp

```cpp
USTRUCT(BlueprintType, meta = (DisplayName = "Pelvis Motion"))
struct FIKRetargetPelvisMotionOp : public FIKRetargetOpBase
{
    GENERATED_BODY()
    
    // 骨盆运动设置
    UPROPERTY()
    FIKRetargetPelvisMotionSettings Settings;
    
    // 初始化
    virtual bool Initialize(
        const FIKRetargetProcessor& InProcessor,
        const FRetargetSkeleton& InSourceSkeleton,
        const FTargetSkeleton& InTargetSkeleton,
        const FIKRetargetOpBase* InParentOp,
        FIKRigLogger& InLog) override;
    
    // 执行
    virtual void Run(
        FIKRetargetProcessor& InProcessor,
        const double InDeltaTime,
        const TArray<FTransform>& InSourceGlobalPose,
        TArray<FTransform>& OutTargetGlobalPose) override;
    
    // 获取骨盆骨骼名称
    FName GetPelvisBoneName(ERetargetSourceOrTarget SourceOrTarget) const;
    
private:
    int32 SourcePelvisIndex = INDEX_NONE;
    int32 TargetPelvisIndex = INDEX_NONE;
    float PelvisHeightRatio = 1.0f;  // 高度缩放比
};
```

### 5.3 骨盆高度缩放计算

```cpp
bool FIKRetargetPelvisMotionOp::Initialize(...)
{
    // 获取源/目标骨盆在参考姿态中的高度
    const FVector& SourcePelvisPosition = 
        SourceSkeleton.RetargetPoses.GetGlobalRetargetPose()[SourcePelvisIndex].GetTranslation();
    const FVector& TargetPelvisPosition = 
        TargetSkeleton.RetargetPoses.GetGlobalRetargetPose()[TargetPelvisIndex].GetTranslation();
    
    // 计算高度比例
    const float SourcePelvisHeight = SourcePelvisPosition.Z;
    const float TargetPelvisHeight = TargetPelvisPosition.Z;
    
    if (FMath::Abs(SourcePelvisHeight) > KINDA_SMALL_NUMBER)
    {
        PelvisHeightRatio = TargetPelvisHeight / SourcePelvisHeight;
    }
    
    return true;
}
```

### 5.4 骨盆运动传递

```cpp
void FIKRetargetPelvisMotionOp::Run(
    FIKRetargetProcessor& InProcessor,
    const double InDeltaTime,
    const TArray<FTransform>& InSourceGlobalPose,
    TArray<FTransform>& OutTargetGlobalPose)
{
    const FRetargetSkeleton& SourceSkeleton = InProcessor.GetSkeleton(ERetargetSourceOrTarget::Source);
    FTargetSkeleton& TargetSkeleton = InProcessor.GetTargetSkeleton();
    
    // 获取源骨盆的当前和初始变换
    const FTransform& SourceCurrentPelvisTransform = InSourceGlobalPose[SourcePelvisIndex];
    const FTransform& SourceInitialPelvisTransform = 
        SourceSkeleton.RetargetPoses.GetGlobalRetargetPose()[SourcePelvisIndex];
    
    // 获取目标骨盆的初始变换
    const FTransform& TargetInitialPelvisTransform = 
        TargetSkeleton.RetargetPoses.GetGlobalRetargetPose()[TargetPelvisIndex];
    
    // 计算源骨盆的位移增量
    FVector SourceTranslationDelta = SourceCurrentPelvisTransform.GetTranslation() - 
                                     SourceInitialPelvisTransform.GetTranslation();
    
    // 应用高度缩放
    SourceTranslationDelta *= PelvisHeightRatio;
    
    // 计算源骨盆的旋转增量
    FQuat SourceRotationDelta = SourceCurrentPelvisTransform.GetRotation() * 
                                 SourceInitialPelvisTransform.GetRotation().Inverse();
    
    // 构建目标骨盆变换
    FTransform NewTargetPelvisTransform;
    NewTargetPelvisTransform.SetTranslation(
        TargetInitialPelvisTransform.GetTranslation() + SourceTranslationDelta);
    NewTargetPelvisTransform.SetRotation(
        SourceRotationDelta * TargetInitialPelvisTransform.GetRotation());
    NewTargetPelvisTransform.SetScale3D(FVector::OneVector);
    
    // 更新目标姿态
    OutTargetGlobalPose[TargetPelvisIndex] = NewTargetPelvisTransform;
    
    // 更新子骨骼
    TargetSkeleton.UpdateGlobalTransformsBelowBone(
        TargetPelvisIndex, TargetSkeleton.InputLocalPose, OutTargetGlobalPose);
}
```

---

## 六、IK链重定向

### 6.1 IK重定向原理

IK链重定向在FK重定向之后运行，用于确保**末端执行器位置正确**：

```
1. FK重定向复制链的形状
      ↓
2. IK重定向调整末端位置
      ↓
3. 反向解算整个链以到达目标
```

### 6.2 FIKRetargetIKChainsOp

```cpp
USTRUCT(BlueprintType, meta = (DisplayName = "IK Chains"))
struct FIKRetargetIKChainsOp : public FIKRetargetOpBase
{
    GENERATED_BODY()
    
    UPROPERTY()
    FIKRetargetIKChainsOpSettings Settings;
    
    // 运行IK链重定向
    virtual void Run(
        FIKRetargetProcessor& InProcessor,
        const double InDeltaTime,
        const TArray<FTransform>& InSourceGlobalPose,
        TArray<FTransform>& OutTargetGlobalPose) override;
        
private:
    // IK解算器实例
    TMap<FName, TSharedPtr<FIKRigProcessor>> IKProcessors;
};
```

### 6.3 IK目标位置计算

```cpp
void FIKRetargetIKChainsOp::Run(...)
{
    // 对于每个启用IK的链
    for (const FRetargetIKChainSettings& ChainSetting : Settings.ChainsToRetarget)
    {
        if (!ChainSetting.EnableIK) continue;
        
        // 获取源链末端当前位置
        const FResolvedBoneChain* SourceChain = 
            AllBoneChains.GetResolvedBoneChainByName(ChainSetting.SourceChainName, 
                ERetargetSourceOrTarget::Source);
        
        const int32 SourceEndBoneIndex = SourceChain->BoneIndices.Last();
        const FTransform& SourceEndCurrentTransform = InSourceGlobalPose[SourceEndBoneIndex];
        const FTransform& SourceEndInitialTransform = 
            SourceChain->RefPoseGlobalTransforms.Last();
        
        // 计算源链末端的位移
        FVector SourceEndDelta = SourceEndCurrentTransform.GetTranslation() - 
                                  SourceEndInitialTransform.GetTranslation();
        
        // 缩放到目标比例
        const float LimbRatio = TargetChain->InitialChainLength / SourceChain->InitialChainLength;
        SourceEndDelta *= LimbRatio;
        
        // 计算目标IK目标位置
        const FTransform& TargetEndInitialTransform = 
            TargetChain->RefPoseGlobalTransforms.Last();
        FVector IKTargetPosition = TargetEndInitialTransform.GetTranslation() + SourceEndDelta;
        
        // 设置IK目标
        FIKRigGoal& Goal = GoalContainer.SetIKGoal(FIKRigGoal(
            ChainSetting.IKGoalName,
            IKTargetPosition,
            TargetEndInitialTransform.GetRotation()));
        
        // 运行IK解算器
        IKProcessor->Solve(OutTargetGlobalPose, GoalContainer);
    }
}
```

---

## 七、逆向运动学（IK）系统

### 7.1 IK系统概述

UE5提供了多种IK解算器：

| 解算器 | 用途 | 特点 |
|--------|------|------|
| Two Bone IK | 双骨骼链（手臂、腿） | 解析解，高效 |
| FABRIK | 多骨骼链 | 迭代式，灵活 |
| CCDIK | 多骨骼链 | 旋转限制支持 |
| Full Body IK | 全身IK | 整体求解 |

### 7.2 Two Bone IK原理

**Two Bone IK**是最常用的IK解算器，用于三关节链（如肩膀-肘-手）：

```cpp
namespace AnimationCore
{
    void SolveTwoBoneIK(
        const FVector& RootPos,      // 根位置（肩膀）
        const FVector& JointPos,     // 关节位置（肘）
        const FVector& EndPos,       // 末端位置（手）
        const FVector& JointTarget,  // 关节目标方向（肘朝向）
        const FVector& Effector,     // 末端执行器目标位置
        FVector& OutJointPos,        // 输出：新的关节位置
        FVector& OutEndPos,          // 输出：新的末端位置
        double UpperLimbLength,      // 上肢长度
        double LowerLimbLength,      // 下肢长度
        bool bAllowStretching,       // 是否允许拉伸
        double StartStretchRatio,    // 开始拉伸的比例
        double MaxStretchScale);     // 最大拉伸倍数
}
```

### 7.3 Two Bone IK数学原理

```cpp
void SolveTwoBoneIK(
    const FVector& RootPos, const FVector& JointPos, const FVector& EndPos,
    const FVector& JointTarget, const FVector& Effector,
    FVector& OutJointPos, FVector& OutEndPos,
    double UpperLimbLength, double LowerLimbLength,
    bool bAllowStretching, double StartStretchRatio, double MaxStretchScale)
{
    // 目标方向和距离
    FVector DesiredPos = Effector;
    FVector DesiredDelta = DesiredPos - RootPos;
    double DesiredLength = DesiredDelta.Size();
    
    // 最大可达距离
    double MaxLimbLength = LowerLimbLength + UpperLimbLength;

    // 处理DesiredPos与RootPos重合的情况
    FVector DesiredDir;
    if (DesiredLength < DOUBLE_KINDA_SMALL_NUMBER)
    {
        DesiredLength = DOUBLE_KINDA_SMALL_NUMBER;
        DesiredDir = FVector(1, 0, 0);
    }
    else
    {
        DesiredDir = DesiredDelta.GetSafeNormal();
    }

    // 计算关节弯曲平面
    FVector JointTargetDelta = JointTarget - RootPos;
    FVector JointPlaneNormal, JointBendDir;
    
    if (JointTargetDelta.SizeSquared() < FMath::Square(DOUBLE_KINDA_SMALL_NUMBER))
    {
        // 关节目标与根重合，使用默认方向
        JointBendDir = FVector(0, 1, 0);
        JointPlaneNormal = FVector(0, 0, 1);
    }
    else
    {
        JointPlaneNormal = DesiredDir ^ JointTargetDelta;
        
        if (JointPlaneNormal.SizeSquared() < FMath::Square(DOUBLE_KINDA_SMALL_NUMBER))
        {
            // 共线情况
            DesiredDir.FindBestAxisVectors(JointPlaneNormal, JointBendDir);
        }
        else
        {
            JointPlaneNormal.Normalize();
            JointBendDir = JointTargetDelta - ((JointTargetDelta | DesiredDir) * DesiredDir);
            JointBendDir.Normalize();
        }
    }

    // 处理拉伸
    if (bAllowStretching)
    {
        const double ScaleRange = MaxStretchScale - StartStretchRatio;
        if (ScaleRange > DOUBLE_KINDA_SMALL_NUMBER && MaxLimbLength > DOUBLE_KINDA_SMALL_NUMBER)
        {
            const double ReachRatio = DesiredLength / MaxLimbLength;
            const double ScalingFactor = (MaxStretchScale - 1.0) * 
                FMath::Clamp((ReachRatio - StartStretchRatio) / ScaleRange, 0.0, 1.0);
            
            if (ScalingFactor > DOUBLE_KINDA_SMALL_NUMBER)
            {
                LowerLimbLength *= (1.0 + ScalingFactor);
                UpperLimbLength *= (1.0 + ScalingFactor);
                MaxLimbLength *= (1.0 + ScalingFactor);
            }
        }
    }

    OutEndPos = DesiredPos;
    OutJointPos = JointPos;

    // 目标超出可达范围 - 完全伸展
    if (DesiredLength >= MaxLimbLength)
    {
        OutEndPos = RootPos + (MaxLimbLength * DesiredDir);
        OutJointPos = RootPos + (UpperLimbLength * DesiredDir);
    }
    else
    {
        // 使用余弦定理计算关节角度
        // 三角形边长: a = UpperLimbLength, b = LowerLimbLength, c = DesiredLength
        const double TwoAB = 2.0 * UpperLimbLength * DesiredLength;
        const double CosAngle = (TwoAB != 0.0) ? 
            ((UpperLimbLength*UpperLimbLength) + (DesiredLength*DesiredLength) - 
             (LowerLimbLength*LowerLimbLength)) / TwoAB : 0.0;

        const bool bReverseUpperBone = (CosAngle < 0.0);

        // 计算关节角度
        const double Angle = FMath::Acos(CosAngle);

        // 关节到根-目标连线的垂直距离
        const double JointLineDist = UpperLimbLength * FMath::Sin(Angle);

        // 关节在根-目标连线上的投影距离
        const double ProjJointDistSqr = (UpperLimbLength*UpperLimbLength) - (JointLineDist*JointLineDist);
        double ProjJointDist = (ProjJointDistSqr > 0.0) ? FMath::Sqrt(ProjJointDistSqr) : 0.0;
        
        if (bReverseUpperBone)
        {
            ProjJointDist *= -1.f;
        }

        // 计算新的关节位置
        OutJointPos = RootPos + (ProjJointDist * DesiredDir) + (JointLineDist * JointBendDir);
    }
}
```

**几何图解**：
```
           JointTarget (肘朝向目标)
                *
               /
              /
    Root ----+---- DesiredDir ----> Effector
   (肩膀)    |                      (目标)
             |
          JointBendDir
             |
             * Joint (新肘位置)
```

### 7.4 FAnimNode_TwoBoneIK

```cpp
USTRUCT(BlueprintInternalUseOnly)
struct FAnimNode_TwoBoneIK : public FAnimNode_SkeletalControlBase
{
    GENERATED_USTRUCT_BODY()
    
    // IK骨骼（末端骨骼，如hand_r）
    UPROPERTY(EditAnywhere, Category=IK)
    FBoneReference IKBone;
    
    // 末端执行器位置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Effector, meta=(PinShownByDefault))
    FVector EffectorLocation;
    
    // 末端执行器目标（可以是骨骼或Socket）
    UPROPERTY(EditAnywhere, Category=Effector)
    FBoneSocketTarget EffectorTarget;
    
    // 关节目标位置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=JointTarget, meta=(PinShownByDefault))
    FVector JointTargetLocation;
    
    // 关节目标
    UPROPERTY(EditAnywhere, Category=JointTarget)
    FBoneSocketTarget JointTarget;
    
    // 拉伸设置
    UPROPERTY(EditAnywhere, Category=IK)
    uint8 bAllowStretching:1;
    
    UPROPERTY(EditAnywhere, Category=IK)
    double StartStretchRatio;
    
    UPROPERTY(EditAnywhere, Category=IK)
    double MaxStretchScale;
    
    // 是否使用末端执行器旋转
    UPROPERTY(EditAnywhere, Category=IK)
    uint8 bTakeRotationFromEffectorSpace : 1;
    
    // 是否保持末端骨骼的相对旋转
    UPROPERTY(EditAnywhere, Category=IK)
    uint8 bMaintainEffectorRelRot : 1;
    
    // 是否允许扭曲
    UPROPERTY(EditAnywhere, Category=IK)
    uint8 bAllowTwist : 1;
    
    // 扭曲轴
    UPROPERTY(EditAnywhere, Category=IK)
    FAxis TwistAxis;
    
    // 缓存的骨骼索引
    FCompactPoseBoneIndex CachedUpperLimbIndex;
    FCompactPoseBoneIndex CachedLowerLimbIndex;
    
    // 评估骨骼控制
    virtual void EvaluateSkeletalControl_AnyThread(
        FComponentSpacePoseContext& Output, 
        TArray<FBoneTransform>& OutBoneTransforms) override;
};
```

### 7.5 TwoBoneIK评估实现

```cpp
void FAnimNode_TwoBoneIK::EvaluateSkeletalControl_AnyThread(
    FComponentSpacePoseContext& Output, 
    TArray<FBoneTransform>& OutBoneTransforms)
{
    const FBoneContainer& BoneContainer = Output.Pose.GetPose().GetBoneContainer();
    
    // 获取三个关节的索引
    FCompactPoseBoneIndex IKBoneCompactPoseIndex = IKBone.GetCompactPoseIndex(BoneContainer);
    
    // 获取当前位置
    const FTransform& EndBoneLocalTransform = Output.Pose.GetLocalSpaceTransform(IKBoneCompactPoseIndex);
    const FTransform& LowerLimbCSTransform = Output.Pose.GetComponentSpaceTransform(CachedLowerLimbIndex);
    const FTransform& UpperLimbCSTransform = Output.Pose.GetComponentSpaceTransform(CachedUpperLimbIndex);
    FTransform EndBoneCSTransform = Output.Pose.GetComponentSpaceTransform(IKBoneCompactPoseIndex);
    
    // 计算肢体长度
    const double UpperLimbLength = FVector::Dist(
        UpperLimbCSTransform.GetLocation(), LowerLimbCSTransform.GetLocation());
    const double LowerLimbLength = FVector::Dist(
        LowerLimbCSTransform.GetLocation(), EndBoneCSTransform.GetLocation());
    
    // 获取目标位置
    FTransform EffectorTransform = GetTargetTransform(
        Output.AnimInstanceProxy->GetComponentTransform(),
        Output.Pose, EffectorTarget, EffectorLocationSpace, EffectorLocation);
    
    FVector DesiredPos = EffectorTransform.GetLocation();
    
    // 获取关节目标位置
    FTransform JointTargetTransform = GetTargetTransform(
        Output.AnimInstanceProxy->GetComponentTransform(),
        Output.Pose, JointTarget, JointTargetLocationSpace, JointTargetLocation);
    
    FVector JointTargetPos = JointTargetTransform.GetLocation();
    
    // 执行IK求解
    FVector NewJointPos, NewEndPos;
    AnimationCore::SolveTwoBoneIK(
        UpperLimbCSTransform.GetLocation(),
        LowerLimbCSTransform.GetLocation(),
        EndBoneCSTransform.GetLocation(),
        JointTargetPos,
        DesiredPos,
        NewJointPos,
        NewEndPos,
        UpperLimbLength,
        LowerLimbLength,
        bAllowStretching,
        StartStretchRatio,
        MaxStretchScale);
    
    // 更新上臂变换
    {
        FVector OldDir = (LowerLimbCSTransform.GetLocation() - 
                          UpperLimbCSTransform.GetLocation()).GetSafeNormal();
        FVector NewDir = (NewJointPos - UpperLimbCSTransform.GetLocation()).GetSafeNormal();
        FQuat DeltaRotation = FQuat::FindBetweenNormals(OldDir, NewDir);
        
        FTransform NewUpperLimbTransform = UpperLimbCSTransform;
        NewUpperLimbTransform.SetRotation(DeltaRotation * UpperLimbCSTransform.GetRotation());
        OutBoneTransforms.Add(FBoneTransform(CachedUpperLimbIndex, NewUpperLimbTransform));
    }
    
    // 更新前臂变换
    {
        FVector OldDir = (EndBoneCSTransform.GetLocation() - 
                          LowerLimbCSTransform.GetLocation()).GetSafeNormal();
        FVector NewDir = (NewEndPos - NewJointPos).GetSafeNormal();
        FQuat DeltaRotation = FQuat::FindBetweenNormals(OldDir, NewDir);
        
        FTransform NewLowerLimbTransform = LowerLimbCSTransform;
        NewLowerLimbTransform.SetRotation(DeltaRotation * LowerLimbCSTransform.GetRotation());
        NewLowerLimbTransform.SetTranslation(NewJointPos);
        OutBoneTransforms.Add(FBoneTransform(CachedLowerLimbIndex, NewLowerLimbTransform));
    }
    
    // 更新末端骨骼变换
    {
        FTransform NewEndBoneTransform = EndBoneCSTransform;
        NewEndBoneTransform.SetTranslation(NewEndPos);
        
        if (bTakeRotationFromEffectorSpace)
        {
            NewEndBoneTransform.SetRotation(EffectorTransform.GetRotation());
        }
        
        OutBoneTransforms.Add(FBoneTransform(IKBoneCompactPoseIndex, NewEndBoneTransform));
    }
}
```

---

## 八、FABRIK算法

### 8.1 FABRIK原理

**FABRIK（Forward And Backward Reaching Inverse Kinematics）**是一种迭代式IK算法：

1. **前向阶段**：从末端向根移动，使每个关节指向子关节
2. **后向阶段**：从根向末端移动，使每个关节指向父关节
3. **迭代**：重复直到收敛或达到最大迭代次数

### 8.2 FABRIK数据结构

```cpp
USTRUCT()
struct FFABRIKChainLink
{
    GENERATED_USTRUCT_BODY()

    // 骨骼在组件空间中的位置
    FVector Position;
    
    // 到父骨骼的距离
    double Length;
    
    // 骨架中的骨骼索引
    int32 BoneIndex;
    
    // 输出变换数组中的索引
    int32 TransformIndex;
    
    // 默认的到父骨骼方向
    FVector DefaultDirToParent;
    
    // 零长度子骨骼（会继承父变换）
    TArray<int32> ChildZeroLengthTransformIndices;
};
```

### 8.3 FABRIK核心算法

```cpp
namespace AnimationCore
{
    bool SolveFabrik(
        TArray<FFABRIKChainLink>& InOutChain,
        const FVector& TargetPosition,
        double MaximumReach,
        double Precision,
        int32 MaxIterations)
    {
        bool bBoneLocationUpdated = false;
        
        double RootToTargetDistSq = FVector::DistSquared(InOutChain[0].Position, TargetPosition);
        int32 NumChainLinks = InOutChain.Num();

        // 如果目标超出可达范围，完全伸展
        if (RootToTargetDistSq > FMath::Square(MaximumReach))
        {
            for (int32 LinkIndex = 1; LinkIndex < NumChainLinks; LinkIndex++)
            {
                FFABRIKChainLink& ParentLink = InOutChain[LinkIndex - 1];
                FFABRIKChainLink& CurrentLink = InOutChain[LinkIndex];
                
                // 沿目标方向伸展
                CurrentLink.Position = ParentLink.Position + 
                    (TargetPosition - ParentLink.Position).GetUnsafeNormal() * CurrentLink.Length;
            }
            bBoneLocationUpdated = true;
        }
        else
        {
            // 目标在可达范围内
            int32 TipBoneLinkIndex = NumChainLinks - 1;
            
            // 检查末端是否已经足够接近目标
            double Slop = FVector::Dist(InOutChain[TipBoneLinkIndex].Position, TargetPosition);
            
            if (Slop > Precision)
            {
                // 将末端放到目标位置
                InOutChain[TipBoneLinkIndex].Position = TargetPosition;

                int32 IterationCount = 0;
                while ((Slop > Precision) && (IterationCount++ < MaxIterations))
                {
                    // ========== 前向阶段（从末端向根） ==========
                    for (int32 LinkIndex = TipBoneLinkIndex - 1; LinkIndex > 0; LinkIndex--)
                    {
                        FFABRIKChainLink& CurrentLink = InOutChain[LinkIndex];
                        FFABRIKChainLink& ChildLink = InOutChain[LinkIndex + 1];

                        // 将当前关节放置在正确距离
                        CurrentLink.Position = ChildLink.Position + 
                            (CurrentLink.Position - ChildLink.Position).GetUnsafeNormal() * 
                            ChildLink.Length;
                    }

                    // ========== 后向阶段（从根向末端） ==========
                    for (int32 LinkIndex = 1; LinkIndex < TipBoneLinkIndex; LinkIndex++)
                    {
                        FFABRIKChainLink& ParentLink = InOutChain[LinkIndex - 1];
                        FFABRIKChainLink& CurrentLink = InOutChain[LinkIndex];

                        // 将当前关节放置在正确距离
                        CurrentLink.Position = ParentLink.Position + 
                            (CurrentLink.Position - ParentLink.Position).GetUnsafeNormal() * 
                            CurrentLink.Length;
                    }

                    // 重新检查误差
                    Slop = FMath::Abs(InOutChain[TipBoneLinkIndex].Length - 
                        FVector::Dist(InOutChain[TipBoneLinkIndex - 1].Position, TargetPosition));
                }

                // 最终放置末端骨骼
                {
                    FFABRIKChainLink& ParentLink = InOutChain[TipBoneLinkIndex - 1];
                    FFABRIKChainLink& CurrentLink = InOutChain[TipBoneLinkIndex];

                    CurrentLink.Position = ParentLink.Position + 
                        (CurrentLink.Position - ParentLink.Position).GetUnsafeNormal() * 
                        CurrentLink.Length;
                }

                bBoneLocationUpdated = true;
            }
        }

        return bBoneLocationUpdated;
    }
}
```

### 8.4 FABRIK图解

```
初始状态:                   迭代过程:
    Root                        Root (固定)
     |                           |
     1                           1' ← 后向调整
     |                           |
     2                           2' ← 后向调整
     |                           |
     3                           3' ← 前向调整
     |                           |
    End                         End (目标)
     
前向阶段: End→3→2→1→Root
后向阶段: Root→1→2→3→End
```

---

## 九、CCDIK算法

### 9.1 CCDIK原理

**CCDIK（Cyclic Coordinate Descent IK）**通过逐个旋转关节来使末端接近目标：

1. 从末端关节开始（或从根开始）
2. 旋转当前关节，使末端尽可能接近目标
3. 移动到下一个关节，重复
4. 循环迭代直到收敛

### 9.2 CCDIK数据结构

```cpp
USTRUCT()
struct FCCDIKChainLink
{
    GENERATED_USTRUCT_BODY()

    // 当前全局变换
    FTransform Transform;
    
    // 局部变换（相对于父骨骼）
    FTransform LocalTransform;
    
    // 输出数组中的索引
    int32 TransformIndex;
    
    // 零长度子骨骼索引
    TArray<int32> ChildZeroLengthTransformIndices;
};
```

### 9.3 CCDIK核心算法

```cpp
namespace AnimationCore
{
    bool SolveCCDIK(
        TArray<FCCDIKChainLink>& InOutChain,
        const FVector& TargetPos,
        float Precision,
        int32 MaxIterations,
        bool bStartFromTail,
        bool bEnableRotationLimit,
        const TArray<float>& RotationLimitPerJoints)
    {
        bool bBoneLocationUpdated = false;
        int32 NumChainLinks = InOutChain.Num();
        int32 TipIndex = NumChainLinks - 1;
        
        FVector TipPosition = InOutChain[TipIndex].Transform.GetLocation();
        float TipToTargetDist = FVector::Dist(TipPosition, TargetPos);
        
        if (TipToTargetDist <= Precision)
        {
            return false; // 已经足够接近
        }
        
        for (int32 Iteration = 0; Iteration < MaxIterations; ++Iteration)
        {
            // 确定遍历顺序
            int32 StartIndex = bStartFromTail ? TipIndex - 1 : 0;
            int32 EndIndex = bStartFromTail ? 0 : TipIndex - 1;
            int32 Direction = bStartFromTail ? -1 : 1;
            
            for (int32 LinkIndex = StartIndex; 
                 (bStartFromTail ? LinkIndex >= EndIndex : LinkIndex <= EndIndex); 
                 LinkIndex += Direction)
            {
                FCCDIKChainLink& CurrentLink = InOutChain[LinkIndex];
                
                // 当前关节位置
                FVector CurrentPosition = CurrentLink.Transform.GetLocation();
                
                // 从当前关节到末端的向量
                FVector ToTip = TipPosition - CurrentPosition;
                ToTip.Normalize();
                
                // 从当前关节到目标的向量
                FVector ToTarget = TargetPos - CurrentPosition;
                ToTarget.Normalize();
                
                // 计算需要的旋转
                FQuat DeltaRotation = FQuat::FindBetweenNormals(ToTip, ToTarget);
                
                // 应用旋转限制
                if (bEnableRotationLimit && RotationLimitPerJoints.IsValidIndex(LinkIndex))
                {
                    float LimitAngle = FMath::DegreesToRadians(RotationLimitPerJoints[LinkIndex]);
                    
                    FVector Axis;
                    float Angle;
                    DeltaRotation.ToAxisAndAngle(Axis, Angle);
                    
                    // 限制旋转角度
                    Angle = FMath::Clamp(Angle, -LimitAngle, LimitAngle);
                    DeltaRotation = FQuat(Axis, Angle);
                }
                
                // 应用旋转
                FQuat NewRotation = DeltaRotation * CurrentLink.Transform.GetRotation();
                NewRotation.Normalize();
                CurrentLink.Transform.SetRotation(NewRotation);
                
                // 更新后续所有骨骼的变换
                for (int32 UpdateIndex = LinkIndex + 1; UpdateIndex < NumChainLinks; ++UpdateIndex)
                {
                    FCCDIKChainLink& ParentLink = InOutChain[UpdateIndex - 1];
                    FCCDIKChainLink& ChildLink = InOutChain[UpdateIndex];
                    
                    ChildLink.Transform = ChildLink.LocalTransform * ParentLink.Transform;
                }
                
                // 更新末端位置
                TipPosition = InOutChain[TipIndex].Transform.GetLocation();
                
                bBoneLocationUpdated = true;
            }
            
            // 检查是否已收敛
            TipToTargetDist = FVector::Dist(TipPosition, TargetPos);
            if (TipToTargetDist <= Precision)
            {
                break;
            }
        }
        
        return bBoneLocationUpdated;
    }
}
```

### 9.4 CCDIK与FABRIK对比

| 特性 | CCDIK | FABRIK |
|------|-------|--------|
| 迭代方式 | 逐关节旋转 | 前向/后向位置调整 |
| 旋转限制 | 原生支持 | 需要额外处理 |
| 收敛速度 | 较慢 | 较快 |
| 适用场景 | 需要关节限制 | 无限制或后处理限制 |
| 物理准确性 | 更符合真实关节 | 可能产生不自然姿态 |

---

## 十、IK Rig系统

### 10.1 UIKRigDefinition

```cpp
UCLASS(BlueprintType)
class UIKRigDefinition : public UObject
{
    GENERATED_BODY()
    
    // 预览骨架网格
    UPROPERTY(EditAnywhere, Category = "Skeleton")
    TSoftObjectPtr<USkeletalMesh> PreviewSkeletalMesh;
    
    // IK解算器列表
    UPROPERTY()
    TArray<FInstancedStruct> Solvers;
    
    // 重定向骨骼链
    UPROPERTY()
    TArray<FBoneChain> RetargetChains;
    
    // IK目标
    UPROPERTY()
    TArray<UIKRigEffectorGoal*> Goals;
    
    // 骨盆骨骼名称
    UPROPERTY()
    FName PelvisBoneName;
    
    // 获取重定向链
    const TArray<FBoneChain>& GetRetargetChains() const { return RetargetChains; }
};
```

### 10.2 FIKRigGoal - IK目标

```cpp
USTRUCT(BlueprintType)
struct FIKRigGoal
{
    GENERATED_BODY()
    
    // 目标名称
    UPROPERTY(EditAnywhere)
    FName Name;
    
    // 关联的骨骼名称
    UPROPERTY(EditAnywhere)
    FName BoneName;
    
    // 目标位置
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector Position;
    
    // 目标旋转
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FQuat Rotation;
    
    // 位置Alpha（0-1）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta = (ClampMin = "0.0", ClampMax = "1.0"))
    double PositionAlpha = 1.0;
    
    // 旋转Alpha（0-1）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta = (ClampMin = "0.0", ClampMax = "1.0"))
    double RotationAlpha = 1.0;
    
    // 位置空间
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EIKRigGoalSpace PositionSpace = EIKRigGoalSpace::Component;
    
    // 旋转空间
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EIKRigGoalSpace RotationSpace = EIKRigGoalSpace::Component;
};
```

### 10.3 FIKRigProcessor - IK Rig处理器

```cpp
struct FIKRigProcessor
{
    // 初始化
    void Initialize(const UIKRigDefinition* IKRigAsset);
    
    // 解算
    void Solve(
        TArray<FTransform>& InOutGlobalPose,
        const FIKRigGoalContainer& Goals);
    
private:
    // 骨架数据
    FIKRigSkeleton Skeleton;
    
    // 解算器实例
    TArray<TSharedPtr<FIKRigSolverBase>> Solvers;
    
    // 目标数据
    TArray<FIKRigGoal> Goals;
};
```

---

## 十一、实际应用示例

### 11.1 设置IK Retargeter

```cpp
// 创建重定向器
UIKRetargeter* Retargeter = NewObject<UIKRetargeter>();

// 设置源和目标IK Rig
Retargeter->SetSourceIKRig(SourceIKRig);
Retargeter->SetTargetIKRig(TargetIKRig);

// 添加链映射
Retargeter->AddChainMapping(TEXT("LeftArm"), TEXT("LeftArm"));
Retargeter->AddChainMapping(TEXT("RightArm"), TEXT("RightArm"));
Retargeter->AddChainMapping(TEXT("Spine"), TEXT("Spine"));
```

### 11.2 运行时重定向

```cpp
void UMyAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);
    
    if (!Processor.IsInitialized())
    {
        Processor.Initialize(SourceMesh, TargetMesh, RetargeterAsset, Profile, false);
    }
    
    // 获取源姿态
    TArray<FTransform> SourcePose = GetSourcePose();
    
    // 应用源缩放
    Processor.ApplySourceScaleToPose(SourcePose);
    
    // 运行重定向
    TArray<FTransform>& OutputPose = Processor.RunRetargeter(
        SourcePose, Profile, DeltaSeconds, LODLevel);
    
    // 应用到骨架
    ApplyPoseToSkeleton(OutputPose);
}
```

### 11.3 使用Two Bone IK实现脚部IK

```cpp
// 在动画蓝图中配置TwoBoneIK节点
FAnimNode_TwoBoneIK FootIK;
FootIK.IKBone = FBoneReference(TEXT("foot_r"));
FootIK.EffectorLocation = GetFootTargetLocation();  // 地面检测位置
FootIK.JointTargetLocation = GetKneeTargetLocation();  // 膝盖朝向
FootIK.bAllowStretching = true;
FootIK.StartStretchRatio = 0.9f;
FootIK.MaxStretchScale = 1.15f;
```

---

## 十二、性能优化建议

### 12.1 重定向优化

| 优化点 | 建议 |
|--------|------|
| 链数量 | 只映射必要的链 |
| IK解算 | 非必要时禁用IK |
| LOD | 使用LOD阈值禁用复杂操作 |
| 缓存 | 利用三角形索引缓存 |

### 12.2 IK优化

| 优化点 | 建议 |
|--------|------|
| 迭代次数 | FABRIK/CCDIK使用合理迭代次数（4-16） |
| 精度 | 根据需求调整精度阈值 |
| 解算器选择 | 双骨骼链优先使用Two Bone IK |
| 更新频率 | 非关键IK可降低更新频率 |

---

## 十三、调试技巧

### 13.1 可视化调试

```cpp
// 在编辑器中启用调试绘制
UPROPERTY(EditAnywhere, Category = Debug)
bool bDebugDraw = true;

// 绘制IK链
void DrawIKDebug(FPrimitiveDrawInterface* PDI)
{
    // 绘制骨骼链
    for (int32 i = 0; i < Chain.Num() - 1; ++i)
    {
        PDI->DrawLine(Chain[i].Position, Chain[i+1].Position, 
            FLinearColor::Green, SDPG_Foreground);
    }
    
    // 绘制目标位置
    DrawDebugSphere(World, TargetLocation, 5.f, 8, FColor::Red);
}
```

### 13.2 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 肢体伸展过度 | 目标超出范围 | 启用拉伸限制 |
| 关节弯曲方向错误 | JointTarget设置不当 | 调整关节目标位置 |
| 动画抖动 | 迭代不收敛 | 增加迭代次数或精度 |
| 姿态不匹配 | 重定向姿态差异 | 调整重定向姿态 |

---

## 十四、总结

### 14.1 重定向核心流程

```
1. 源动画姿态输入
      ↓
2. 源缩放 (ScaleSourceOp)
      ↓
3. 骨盆运动传递 (PelvisMotionOp)
      ↓
4. FK链形状复制 (FKChainsOp)
      ↓
5. IK链末端调整 (IKChainsOp)
      ↓
6. 输出目标姿态
```

### 14.2 IK解算器选择指南

| 场景 | 推荐解算器 |
|------|------------|
| 手臂/腿（双骨骼） | Two Bone IK |
| 脊柱/尾巴 | FABRIK |
| 需要关节限制 | CCDIK |
| 全身协调 | Full Body IK |

### 14.3 关键技术点

| 技术点 | 核心算法/数据结构 |
|--------|-------------------|
| 链映射 | 参数化 (0-1) 插值 |
| FK重定向 | 旋转差值传递 |
| IK重定向 | 末端位置缩放 |
| Two Bone IK | 余弦定理求解 |
| FABRIK | 前向/后向迭代 |
| CCDIK | 循环坐标下降 |

---

## 参考资料

1. UE5源码: `Engine/Plugins/Animation/IKRig/`
2. UE5源码: `Engine/Source/Runtime/AnimationCore/`
3. UE5源码: `Engine/Source/Runtime/AnimGraphRuntime/`
4. [FABRIK论文](http://andreasaristidou.com/publications/FABRIK.pdf)
5. [Two Bone IK原理](https://theorangeduck.com/page/simple-two-joint)
6. [UE5 IK Retargeter文档](https://docs.unrealengine.com/5.0/en-US/ik-retargeter-in-unreal-engine/)

---

> 下一篇预告：**Animation之旅_007：Control Rig与程序化动画** - 将深入探讨Control Rig的架构设计，以及如何使用Control Rig实现程序化动画效果。
