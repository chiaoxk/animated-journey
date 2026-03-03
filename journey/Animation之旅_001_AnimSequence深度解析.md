# Animation之旅_001：UAnimSequence 深度解析

> **系列目标**：通过源码级别的学习，深入理解 Unreal Engine 动画系统的设计理念与实现细节
> 
> **本篇主题**：`UAnimSequence` - 动画序列，UE 动画系统的核心数据载体

---

## 📚 目录

1. [设计背景与目的](#1-设计背景与目的)
2. [类继承体系](#2-类继承体系)
3. [核心数据结构](#3-核心数据结构)
4. [关键函数详解](#4-关键函数详解)
5. [压缩系统](#5-压缩系统)
6. [Additive 动画机制](#6-additive-动画机制)
7. [Root Motion 系统](#7-root-motion-系统)
8. [运行时数据流](#8-运行时数据流)
9. [设计模式与架构思考](#9-设计模式与架构思考)
10. [实践建议](#10-实践建议)

---

## 1. 设计背景与目的

### 1.1 为什么需要 AnimSequence？

在游戏引擎中，动画数据的管理面临以下核心挑战：

| 挑战 | 解决方案 |
|------|----------|
| **数据量庞大** | 压缩算法、LOD 系统 |
| **实时采样性能** | 预计算、缓存机制 |
| **内存占用** | 流式加载、平台特定优化 |
| **多平台支持** | 派生数据缓存 (DDC) |
| **动画混合** | 标准化 Pose 输出接口 |

`UAnimSequence` 正是为了解决这些问题而设计的**动画资产容器**。

### 1.2 核心职责

```
┌─────────────────────────────────────────────────────────────┐
│                     UAnimSequence 职责                       │
├─────────────────────────────────────────────────────────────┤
│  📦 数据存储    │ 存储骨骼动画轨道、曲线数据、通知事件        │
│  🔄 数据转换    │ 原始数据 ↔ 压缩数据 的双向转换              │
│  ⏱️ 时间采样    │ 根据时间点采样骨骼姿态 (Pose)               │
│  ➕ 叠加支持    │ Additive 动画的基准姿态管理                  │
│  🦶 根运动提取  │ Root Motion 的计算与提取                    │
│  📊 曲线求值    │ Morph Target、Material 曲线的实时求值       │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 类继承体系

### 2.1 继承关系图

```
UObject
    │
    └── UAnimationAsset          // 动画资产基类
            │
            ├── USkeleton 引用    // 关联的骨骼定义
            ├── MetaData 支持     // 元数据扩展
            └── PreviewMesh       // 预览网格
                    │
                    └── UAnimSequenceBase    // 可播放动画基类
                            │
                            ├── Notifies[]   // 动画通知数组
                            ├── RateScale    // 播放速率
                            ├── bLoop        // 循环标记
                            └── DataModel    // 数据模型 (编辑器)
                                    │
                                    └── UAnimSequence    // 动画序列
                                            │
                                            ├── 压缩数据
                                            ├── Additive 设置
                                            ├── Root Motion 设置
                                            └── 重定向设置
```

### 2.2 各层职责划分

| 类 | 职责 | 关键成员 |
|----|------|----------|
| `UAnimationAsset` | 资产通用接口 | `Skeleton`, `MetaData`, `PreviewMesh` |
| `UAnimSequenceBase` | 可播放动画通用逻辑 | `Notifies`, `SequenceLength`, `RateScale` |
| `UAnimSequence` | 具体动画序列实现 | `CompressedData`, `AdditiveAnimType`, `RootMotion` |

### 2.3 为什么要这样分层？

```cpp
// UAnimationAsset - 所有动画资产共享的能力
class UAnimationAsset : public UObject
{
    USkeleton* Skeleton;           // 所有动画都需要骨骼
    TArray<UAnimMetaData*> MetaData; // 可扩展的元数据
};

// UAnimSequenceBase - 可"播放"的动画共享能力
class UAnimSequenceBase : public UAnimationAsset
{
    TArray<FAnimNotifyEvent> Notifies;  // 播放时触发的事件
    float SequenceLength;                // 有时间长度的概念
    
    // 纯虚函数 - 子类必须实现
    virtual void GetAnimationPose(...) PURE_VIRTUAL;
};

// UAnimSequence - 具体的动画序列
class UAnimSequence : public UAnimSequenceBase
{
    // 实现 GetAnimationPose
    virtual void GetAnimationPose(...) override;
    
    // 特有的压缩、Additive、RootMotion 功能
    FCompressedAnimSequence CompressedData;
    EAdditiveAnimationType AdditiveAnimType;
};
```

**设计意图**：
- **单一职责**：每层只关注自己的职责
- **开闭原则**：新增动画类型只需继承，不修改基类
- **接口隔离**：使用者只需知道 `UAnimSequenceBase` 接口

---

## 3. 核心数据结构

### 3.1 原始动画轨道 (Raw Animation Track)

```cpp
/**
 * 单个骨骼的原始动画数据
 * 每个数组要么包含 NumKeys 个元素，要么只有 1 个元素（常量压缩）
 */
USTRUCT(BlueprintType)
struct FRawAnimSequenceTrack
{
    GENERATED_USTRUCT_BODY()

    /** 位置关键帧 */
    UPROPERTY()
    TArray<FVector3f> PosKeys;

    /** 旋转关键帧 (四元数) */
    UPROPERTY()
    TArray<FQuat4f> RotKeys;

    /** 缩放关键帧 */
    UPROPERTY()
    TArray<FVector3f> ScaleKeys;
};
```

**数据布局示意**：

```
时间轴:     0.0s    0.1s    0.2s    0.3s    0.4s
            │       │       │       │       │
PosKeys:   [P0]    [P1]    [P2]    [P3]    [P4]
RotKeys:   [R0]    [R1]    [R2]    [R3]    [R4]
ScaleKeys: [S0]    [S1]    [S2]    [S3]    [S4]

// 常量轨道优化：如果所有帧相同，只存储 1 个值
PosKeys:   [P0]  // 只有 1 个元素，表示整个动画位置不变
```

### 3.2 轨道到骨骼映射

```cpp
/**
 * 将压缩数据中的轨道索引映射到骨骼树索引
 * 这是因为压缩后的轨道数量可能与原始骨骼数量不同
 */
USTRUCT()
struct FTrackToSkeletonMap
{
    GENERATED_USTRUCT_BODY()

    // 骨骼树中的索引
    UPROPERTY()
    int32 BoneTreeIndex;
};
```

**为什么需要映射？**

```
原始骨骼 (100 根):      压缩后轨道 (80 条):
┌──────────────┐        ┌──────────────┐
│ Bone 0       │───────▶│ Track 0      │  BoneTreeIndex = 0
│ Bone 1       │───────▶│ Track 1      │  BoneTreeIndex = 1
│ Bone 2 (静态)│        │ (被移除)      │  
│ Bone 3       │───────▶│ Track 2      │  BoneTreeIndex = 3
│ ...          │        │ ...          │
└──────────────┘        └──────────────┘

// 静态骨骼（整个动画不动）的轨道被移除以节省空间
// 需要映射表来还原正确的骨骼索引
```

### 3.3 压缩动画数据

```cpp
struct FCompressedAnimSequence
{
    /** 轨道到骨骼的映射表 */
    TArray<FTrackToSkeletonMap> CompressedTrackToSkeletonMapTable;

    /** 曲线名称索引表 */
    TArray<FAnimCompressedCurveIndexedName> IndexedCurveNames;

    /** 压缩后的字节流 - 核心数据 */
    TArray<uint8> CompressedByteStream;

    /** 压缩曲线数据流 */
    TArray<uint8> CompressedCurveByteStream;

    /** 骨骼压缩编解码器 */
    UAnimBoneCompressionCodec* BoneCompressionCodec;

    /** 曲线压缩编解码器 */
    UAnimCurveCompressionCodec* CurveCompressionCodec;
};
```

### 3.4 动画提取上下文

```cpp
/**
 * 采样动画时需要的所有上下文信息
 */
struct FAnimExtractContext
{
    /** 当前采样时间点 */
    double CurrentTime;

    /** 是否提取根运动 */
    bool bExtractRootMotion;

    /** 增量时间记录（用于根运动） */
    FDeltaTimeRecord DeltaTimeRecord;

    /** 动画是否循环 */
    bool bLooping;

    /** 姿势曲线值（用于 Pose Asset） */
    TArray<FPoseCurve> PoseCurves;

    /** 骨骼需求标记（优化用） */
    TArray<bool> BonesRequired;

    /** 插值类型覆盖 */
    TOptional<EAnimInterpolationType> InterpolationOverride;
};
```

---

## 4. 关键函数详解

### 4.1 GetAnimationPose - 核心采样函数

```cpp
/**
 * 获取指定时间点的动画姿态
 * 这是动画系统最核心的函数，所有动画播放最终都会调用它
 */
virtual void GetAnimationPose(
    FAnimationPoseData& OutAnimationPoseData, 
    const FAnimExtractContext& ExtractionContext
) const override;
```

**调用流程**：

```
GetAnimationPose()
    │
    ├── 检查是否是 Additive 动画
    │       │
    │       ├── AAT_None: 调用 GetBonePose()
    │       │
    │       ├── AAT_LocalSpaceBase: 调用 GetBonePose_Additive()
    │       │
    │       └── AAT_RotationOffsetMeshSpace: 调用 GetBonePose_AdditiveMeshRotationOnly()
    │
    └── 返回采样后的 Pose
```

### 4.2 GetBonePose - 骨骼姿态采样

```cpp
/**
 * 获取骨骼变换数据
 * 
 * @param OutAnimationPoseData  输出的姿态数据
 * @param ExtractionContext     提取上下文（时间、循环等）
 * @param bForceUseRawData      是否强制使用原始数据（编辑器用）
 */
void GetBonePose(
    FAnimationPoseData& OutAnimationPoseData, 
    const FAnimExtractContext& ExtractionContext, 
    bool bForceUseRawData = false
) const;
```

**内部实现逻辑**（简化版）：

```cpp
void UAnimSequence::GetBonePose(
    FAnimationPoseData& OutAnimationPoseData, 
    const FAnimExtractContext& ExtractionContext, 
    bool bForceUseRawData) const
{
    // 1. 决定使用压缩数据还是原始数据
    bool bUseRawData = ShouldUseRawDataForPoseExtraction(...);
    
    // 2. 获取压缩数据（线程安全）
    auto ScopedCompressedData = GetCompressedData(ExtractionContext);
    
    // 3. 解压缩姿态
    UE::Anim::Decompression::DecompressPose(
        OutAnimationPoseData.GetPose(),
        ScopedCompressedData.Get(),
        ExtractionContext,
        GetSkeleton(),
        GetPlayLength(),
        Interpolation,
        ...
    );
    
    // 4. 应用重定向（如果需要）
    for (int32 BoneIndex : RequiredBones)
    {
        RetargetBoneTransform(
            OutPose[BoneIndex],
            SkeletonBoneIndex,
            BoneIndex,
            RequiredBones,
            bIsBakedAdditive
        );
    }
    
    // 5. 采样曲线数据
    EvaluateCurveData(OutAnimationPoseData.GetCurve(), ExtractionContext);
    
    // 6. 评估属性曲线
    EvaluateAttributes(OutAnimationPoseData, ExtractionContext, bUseRawData);
}
```

### 4.3 GetBoneTransform - 单骨骼采样

```cpp
/**
 * 获取单个骨骼在指定时间的变换
 * 用于需要查询特定骨骼的场景
 */
void GetBoneTransform(
    FTransform& OutAtom,                    // 输出变换
    FSkeletonPoseBoneIndex BoneIndex,       // 骨骼索引
    const FAnimExtractContext& ExtractionContext,  // 上下文
    bool bUseRawData                        // 是否使用原始数据
) const;
```

### 4.4 EvaluateCurveData - 曲线数据采样

```cpp
/**
 * 在指定时间点评估所有曲线数据
 * 曲线用于驱动 Morph Target、材质参数等
 */
virtual void EvaluateCurveData(
    FBlendedCurve& OutCurve,
    const FAnimExtractContext& AnimExtractContext,
    bool bForceUseRawData = false
) const override;

/**
 * 评估单条曲线
 */
virtual float EvaluateCurveData(
    FName CurveName,
    const FAnimExtractContext& AnimExtractContext,
    bool bForceUseRawData = false
) const override;
```

---

## 5. 压缩系统

### 5.1 压缩的必要性

```
原始动画数据 (30fps, 100骨骼, 10秒):
─────────────────────────────────────
位置: 300帧 × 100骨骼 × 12字节 = 360,000 字节
旋转: 300帧 × 100骨骼 × 16字节 = 480,000 字节
缩放: 300帧 × 100骨骼 × 12字节 = 360,000 字节
─────────────────────────────────────
总计: ~1.2 MB (单个 10 秒动画!)

游戏中可能有数百个动画 → 数百 MB 内存占用
∴ 压缩是必须的
```

### 5.2 压缩格式枚举

```cpp
/**
 * 动画关键帧格式
 */
UENUM()
enum AnimationKeyFormat : int
{
    AKF_ConstantKeyLerp,      // 常量关键帧线性插值
    AKF_VariableKeyLerp,      // 可变关键帧线性插值
    AKF_PerTrackCompression,  // 逐轨道压缩（最灵活）
    AKF_MAX,
};

/**
 * 压缩格式
 */
enum AnimationCompressionFormat
{
    ACF_None,           // 无压缩
    ACF_Float96NoW,     // 96位浮点（旋转不带 W）
    ACF_Fixed48NoW,     // 48位定点
    ACF_IntervalFixed32NoW,  // 32位区间定点
    ACF_Fixed32NoW,     // 32位定点
    ACF_Float32NoW,     // 32位浮点
    ACF_Identity,       // 恒等变换（不存储）
};
```

### 5.3 压缩数据结构

```cpp
/**
 * 压缩骨骼动画数据的基础结构
 */
template <template <typename> class ContainerTypeMakerTemplate>
struct FCompressedAnimDataBase
{
    /**
     * 轨道偏移数组 - 定位每个轨道的数据位置
     * 布局: [Trans0.Offset, Trans0.NumKeys, Rot0.Offset, Rot0.NumKeys, ...]
     */
    typename ContainerTypeMakerTemplate<int32>::Type CompressedTrackOffsets;

    /**
     * 缩放偏移数据
     */
    FCompressedOffsetDataBase<...> CompressedScaleOffsets;

    /**
     * 压缩后的字节流 - 实际的动画数据
     */
    typename ContainerTypeMakerTemplate<uint8>::Type CompressedByteStream;

    /** 编解码器指针 */
    AnimEncoding* TranslationCodec;
    AnimEncoding* RotationCodec;
    AnimEncoding* ScaleCodec;

    /** 压缩格式 */
    AnimationCompressionFormat TranslationCompressionFormat;
    AnimationCompressionFormat RotationCompressionFormat;
    AnimationCompressionFormat ScaleCompressionFormat;
};
```

### 5.4 压缩流程（编辑器时）

```
┌─────────────────────────────────────────────────────────────┐
│                      动画压缩流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ Raw Data    │───▶│ Compression  │───▶│ DDC Cache    │   │
│  │ (原始数据)  │    │ Codec        │    │ (派生数据)   │   │
│  └─────────────┘    └──────────────┘    └──────────────┘   │
│        │                   │                    │           │
│        ▼                   ▼                    ▼           │
│  FRawAnimSequenceTrack   压缩算法选择      FCompressedAnimSequence │
│  - PosKeys[]         - 骨骼压缩设置       - CompressedByteStream  │
│  - RotKeys[]         - 曲线压缩设置       - CompressedCurveByteStream │
│  - ScaleKeys[]       - 误差阈值           - Codec 引用            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.5 解压缩（运行时）

```cpp
namespace UE::Anim::Decompression
{
    /**
     * 解压缩姿态数据
     */
    void DecompressPose(
        FCompactPose& OutPose,
        const FCompressedAnimSequence& CompressedData,
        const FAnimExtractContext& ExtractionContext,
        USkeleton* Skeleton,
        float SequenceLength,
        EAnimInterpolationType Interpolation,
        bool bIsBakedAdditive,
        ...
    );
}
```

---

## 6. Additive 动画机制

### 6.1 Additive 类型

```cpp
UENUM()
enum EAdditiveAnimationType : int
{
    /** 非 Additive 动画 */
    AAT_None,
    
    /** 本地空间 Additive - 最常用 */
    AAT_LocalSpaceBase,
    
    /** 网格空间旋转偏移 - 用于 Aim Offset */
    AAT_RotationOffsetMeshSpace,
};
```

### 6.2 Additive 基准姿态类型

```cpp
UENUM()
enum EAdditiveBasePoseType : int
{
    /** 使用骨骼的参考姿态 */
    ABPT_RefPose,
    
    /** 使用指定动画的指定帧 */
    ABPT_AnimScaled,
    
    /** 使用指定动画的第一帧 */
    ABPT_AnimFrame,
    
    /** 自定义 */
    ABPT_Custom,
};
```

### 6.3 Additive 动画原理

```
Full Pose 动画:
┌────────────────────────────────────┐
│  关键帧直接存储完整的骨骼变换       │
│  Pose = AnimationPose              │
└────────────────────────────────────┘

Additive 动画:
┌────────────────────────────────────┐
│  关键帧存储相对于基准姿态的差值     │
│  Delta = AnimationPose - BasePose  │
│  FinalPose = CurrentPose + Delta   │
└────────────────────────────────────┘
```

**代码实现**：

```cpp
// 获取 Additive 姿态
void UAnimSequence::GetBonePose_Additive(
    FAnimationPoseData& OutAnimationPoseData, 
    const FAnimExtractContext& ExtractionContext
) const
{
    // 1. 获取当前动画的姿态
    GetBonePose(OutAnimationPoseData, ExtractionContext, false);
    
    // 2. 获取基准姿态
    FAnimationPoseData BasePoseData;
    GetAdditiveBasePose(BasePoseData, ExtractionContext);
    
    // 3. 计算差值: Delta = AnimPose - BasePose
    for (int32 BoneIndex : RequiredBones)
    {
        OutPose[BoneIndex] = OutPose[BoneIndex].GetRelativeTransform(BasePose[BoneIndex]);
    }
}
```

### 6.4 Additive 动画的应用场景

| 场景 | 类型 | 说明 |
|------|------|------|
| 受击反应 | LocalSpace | 叠加在任何动画上的小幅度抖动 |
| 呼吸动画 | LocalSpace | 周期性的微小起伏 |
| Aim Offset | MeshSpace | 瞄准方向的上半身偏移 |
| 表情动画 | LocalSpace | 面部表情叠加 |

---

## 7. Root Motion 系统

### 7.1 Root Motion 的作用

```
无 Root Motion:
┌─────────────────────────────────────┐
│  角色位移由 MovementComponent 控制   │
│  动画只播放原地动作                  │
│  可能出现脚滑现象                    │
└─────────────────────────────────────┘

有 Root Motion:
┌─────────────────────────────────────┐
│  角色位移从动画根骨骼提取            │
│  动画师完全控制角色移动              │
│  脚步与地面完美匹配                  │
└─────────────────────────────────────┘
```

### 7.2 Root Motion 相关属性

```cpp
class UAnimSequence
{
    /** 是否启用根运动提取 */
    UPROPERTY(EditAnywhere, Category = RootMotion)
    bool bEnableRootMotion;

    /** 根骨骼锁定方式 */
    UPROPERTY(EditAnywhere, Category = RootMotion)
    TEnumAsByte<ERootMotionRootLock::Type> RootMotionRootLock;
    
    /** 是否强制锁定根骨骼 */
    UPROPERTY(EditAnywhere, Category = RootMotion)
    bool bForceRootLock;

    /** 是否使用归一化的根运动缩放 */
    UPROPERTY(EditAnywhere, Category = RootMotion)
    bool bUseNormalizedRootMotionScale;
};
```

### 7.3 Root Motion 锁定类型

```cpp
namespace ERootMotionRootLock
{
    enum Type
    {
        /** 使用参考姿态的根骨骼位置 */
        RefPose,
        
        /** 使用动画第一帧的根骨骼位置 */
        AnimFirstFrame,
        
        /** 归零（原点） */
        Zero,
    };
}
```

### 7.4 Root Motion 提取函数

```cpp
/**
 * 从动画中提取根运动变换
 */
virtual FTransform ExtractRootMotion(
    const FAnimExtractContext& ExtractionContext
) const override;

/**
 * 从时间范围内提取根运动（不支持循环）
 */
virtual FTransform ExtractRootMotionFromRange(
    double StartTime, 
    double EndTime, 
    const FAnimExtractContext& ExtractionContext
) const override;

/**
 * 提取指定时间点的根骨骼变换
 */
virtual FTransform ExtractRootTrackTransform(
    const FAnimExtractContext& ExtractionContext, 
    const FBoneContainer* RequiredBones
) const override;
```

### 7.5 Root Motion 提取流程

```
┌─────────────────────────────────────────────────────────────┐
│                   Root Motion 提取流程                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 获取前一帧根骨骼变换: PrevRootTransform                  │
│                    ↓                                        │
│  2. 获取当前帧根骨骼变换: CurrRootTransform                  │
│                    ↓                                        │
│  3. 计算增量: Delta = CurrRootTransform - PrevRootTransform │
│                    ↓                                        │
│  4. 应用到 Character Movement                               │
│                                                             │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐    │
│  │ Frame N-1  │ ──▶  │ Frame N    │ ──▶  │ Delta      │    │
│  │ Root Pos   │      │ Root Pos   │      │ Movement   │    │
│  └────────────┘      └────────────┘      └────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. 运行时数据流

### 8.1 动画更新流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      一帧动画更新流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SkeletalMeshComponent::TickComponent()                         │
│         │                                                       │
│         ▼                                                       │
│  AnimInstance::UpdateAnimation()                                │
│         │                                                       │
│         ├──▶ 更新动画图表 (Animation Blueprint)                  │
│         │         │                                             │
│         │         ▼                                             │
│         │    各 AnimNode::Update()                              │
│         │         │                                             │
│         │         ▼                                             │
│         │    FAnimTickRecord 生成                                │
│         │                                                       │
│         ▼                                                       │
│  UAnimSequence::TickAssetPlayer()                               │
│         │                                                       │
│         ├──▶ 时间推进 & 同步                                     │
│         │                                                       │
│         ▼                                                       │
│  AnimInstance::ParallelEvaluateAnimation()                      │
│         │                                                       │
│         ▼                                                       │
│  UAnimSequence::GetAnimationPose()                              │
│         │                                                       │
│         ├──▶ GetBonePose() / GetBonePose_Additive()             │
│         │         │                                             │
│         │         ▼                                             │
│         │    DecompressPose() - 解压缩                           │
│         │                                                       │
│         ├──▶ EvaluateCurveData() - 曲线采样                      │
│         │                                                       │
│         └──▶ EvaluateAttributes() - 属性采样                     │
│                                                                 │
│         ▼                                                       │
│  FCompactPose (最终姿态)                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 多线程考虑

```cpp
// UAnimSequence 使用共享递归互斥锁保护压缩数据
mutable UE::FSharedRecursiveMutex SharedCompressedDataMutex;

// 获取压缩数据时使用 RAII 模式加锁
struct FScopedCompressedAnimSequence
{
protected:
    FScopedCompressedAnimSequence(const UAnimSequence* InAnimSequence, ...)
    {
#if WITH_EDITOR
        // 编辑器下需要加锁，因为数据可能被修改
        SharedLock = MakeUnique<UE::TSharedLock<UE::FSharedRecursiveMutex>>(
            InAnimSequence->SharedCompressedDataMutex
        );
#endif
    }
};

// 使用示例
auto ScopedData = AnimSequence->GetCompressedData();
// ScopedData 在作用域内保持锁，离开作用域自动释放
```

---

## 9. 设计模式与架构思考

### 9.1 使用的设计模式

| 模式 | 应用 | 说明 |
|------|------|------|
| **策略模式** | 压缩编解码器 | `UAnimBoneCompressionCodec` 可替换 |
| **模板方法** | `GetAnimationPose` | 定义骨架，子步骤可重写 |
| **享元模式** | `USkeleton` 共享 | 多个动画共享同一骨骼定义 |
| **工厂模式** | DDC 数据创建 | 按平台创建不同压缩数据 |
| **RAII** | 压缩数据访问 | `FScopedCompressedAnimSequence` |

### 9.2 为什么压缩器是可替换的？

```cpp
// 抽象基类定义接口
class UAnimBoneCompressionCodec : public UObject
{
    virtual void DecompressPose(...) PURE_VIRTUAL;
    virtual void DecompressBone(...) PURE_VIRTUAL;
};

// 不同的压缩实现
class UAnimCompress_RemoveLinearKeys : public UAnimBoneCompressionCodec;
class UAnimCompress_PerTrackCompression : public UAnimBoneCompressionCodec;
class UAnimCompress_Automatic : public UAnimBoneCompressionCodec;
```

**好处**：
- 可以根据动画特性选择最优压缩算法
- 可以为不同平台使用不同压缩策略
- 新算法可以通过插件添加，无需修改引擎

### 9.3 DDC (Derived Data Cache) 设计

```
┌──────────────────────────────────────────────────────────────┐
│                        DDC 架构                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   源数据 (Raw)              派生数据 (Compressed)             │
│  ┌─────────────┐           ┌─────────────────┐              │
│  │ AnimSequence│ ──Hash──▶ │ DDC Key         │              │
│  │ + Settings  │           │ (平台+设置哈希) │              │
│  └─────────────┘           └────────┬────────┘              │
│                                     │                        │
│                                     ▼                        │
│                            ┌─────────────────┐              │
│                            │  DDC Storage    │              │
│                            │ ┌─────────────┐ │              │
│                            │ │ PC Data     │ │              │
│                            │ │ PS5 Data    │ │              │
│                            │ │ Xbox Data   │ │              │
│                            │ └─────────────┘ │              │
│                            └─────────────────┘              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**设计意图**：
- 避免重复压缩相同数据
- 支持多平台交叉编译
- 编辑器实时预览与打包数据分离

---

## 10. 实践建议

### 10.1 性能优化建议

| 建议 | 原因 |
|------|------|
| 使用 `BonesRequired` 数组 | 跳过不需要的骨骼采样 |
| 合理设置压缩误差阈值 | 平衡质量和大小 |
| 使用 Additive 而非完整动画 | 节省内存，提高混合效果 |
| 考虑帧剥离 (Frame Stripping) | 移动平台节省内存 |

### 10.2 调试技巧

```cpp
// 1. 查看动画大小
int32 Size = AnimSequence->GetApproxCompressedSize();

// 2. 强制使用原始数据（排查压缩问题）
AnimSequence->GetBonePose(PoseData, Context, true /* bForceUseRawData */);

// 3. 查看压缩信息
auto ScopedData = AnimSequence->GetCompressedData();
UE_LOG(LogAnimation, Log, TEXT("%s"), *ScopedData.Get().GetDebugString());
```

### 10.3 常见问题排查

| 问题 | 可能原因 | 排查方法 |
|------|----------|----------|
| 动画看起来"抖动" | 压缩误差过大 | 增大 `CompressionErrorThresholdScale` |
| Additive 动画无效果 | 基准姿态设置错误 | 检查 `RefPoseSeq` 和 `RefPoseType` |
| Root Motion 不工作 | 未启用或锁定设置 | 检查 `bEnableRootMotion` |
| 内存占用过大 | 未使用压缩或曲线过多 | 检查压缩设置，精简曲线 |

---

## 📖 下一篇预告

**Animation之旅_002：动画蓝图与状态机** - 我们将深入探索：

- `UAnimInstance` - 动画蓝图运行时
- `FAnimNode_Base` - 动画节点基类
- `UAnimStateMachine` - 状态机实现
- 动画混合的数学原理

---

## 📚 参考资源

- UE 源码路径：`Engine/Source/Runtime/Engine/Classes/Animation/`
- 官方文档：[Animation System Overview](https://docs.unrealengine.com/5.0/en-US/animation-system-overview-in-unreal-engine/)
- Animation Insights：编辑器内置的动画调试工具

---

> **作者注**：本文档基于 UE5 源码分析，部分 API 在不同版本可能有差异。建议结合实际源码阅读。
