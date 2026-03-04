# Animation之旅 番外篇：动画压缩算法详解

> 📚 **系列导航**：本文是Animation学习之旅的番外篇，作为深入理解动画系统的"调味剂"。
>
> 🎯 **阅读目标**：深入理解 UE5 动画压缩的原理、格式和实现细节

---

## 一、为什么需要动画压缩？

### 1.1 内存压力的现实

一个典型的游戏角色可能包含：
- **100+ 骨骼**
- **每秒 30 帧**的采样率
- **每个骨骼**需要存储：
  - 位移（Translation）：3 个 float = 12 字节
  - 旋转（Rotation）：4 个 float = 16 字节（四元数）
  - 缩放（Scale）：3 个 float = 12 字节

一个 **10秒** 的动画，未压缩数据量：
```
100骨骼 × 30帧/秒 × 10秒 × (12+16+12)字节 = 1.2 MB
```

一个开放世界游戏可能有 **数千个动画资产**，如果不压缩，仅动画数据就可能占用 **数 GB 内存**！

### 1.2 压缩的三个核心指标

| 指标 | 描述 | 重要性 |
|------|------|--------|
| **压缩率** | 压缩后大小 / 原始大小 | 影响内存占用 |
| **精度损失** | 末端效应器的位移误差 | 影响动画质量 |
| **解压速度** | 运行时解码性能 | 影响帧率 |

好的压缩算法需要在这三者之间取得平衡。

---

## 二、UE5 动画数据结构

### 2.1 原始动画数据

```cpp
// 单个轨道的原始数据
USTRUCT()
struct FRawAnimSequenceTrack
{
    UPROPERTY()
    TArray<FVector3f> PosKeys;    // 位移关键帧
    
    UPROPERTY()
    TArray<FQuat4f> RotKeys;      // 旋转关键帧
    
    UPROPERTY()
    TArray<FVector3f> ScaleKeys;  // 缩放关键帧
};
```

### 2.2 压缩数据结构

```cpp
struct FCompressedAnimSequence
{
    // 轨道索引映射表
    TArray<FTrackToSkeletonMap> CompressedTrackToSkeletonMapTable;
    
    // 曲线名称索引（按 FName 排序以加速查找）
    TArray<FAnimCompressedCurveIndexedName> IndexedCurveNames;
    
    // 压缩后的字节流
    TArray<uint8> CompressedByteStream;
    
    // 压缩后的曲线数据
    TArray<uint8> CompressedCurveByteStream;
    
    // 骨骼压缩编解码器
    UAnimBoneCompressionCodec* BoneCompressionCodec;
    
    // 曲线压缩编解码器
    UAnimCurveCompressionCodec* CurveCompressionCodec;
};
```

---

## 三、压缩格式详解

### 3.1 AnimationCompressionFormat 枚举

UE5 定义了多种压缩格式，用于不同的数据类型：

```cpp
UENUM()
enum AnimationCompressionFormat : int
{
    ACF_None,                  // 无压缩，完整精度
    ACF_Float96NoW,           // 96位浮点，无W分量（旋转）
    ACF_Fixed48NoW,           // 48位定点，无W分量
    ACF_IntervalFixed32NoW,   // 32位区间定点压缩
    ACF_Fixed32NoW,           // 32位定点
    ACF_Float32NoW,           // 32位浮点
    ACF_Identity,             // 恒等变换（不存储数据）
    ACF_MAX
};
```

### 3.2 各格式详细分析

#### ACF_None（无压缩）
- **存储**：原始 float 数据
- **位移**：12 字节（3 × float32）
- **旋转**：16 字节（4 × float32）
- **用途**：需要完美精度的关键动画

#### ACF_Float96NoW（96位无W）
```cpp
struct FQuatFloat96NoW
{
    float X, Y, Z;  // 只存储 X、Y、Z
    // W 通过 sqrt(1 - X² - Y² - Z²) 运行时计算
};
```
- **旋转**：12 字节（节省 25%）
- **优点**：利用四元数单位性质，W 可重建
- **缺点**：需要运行时开方运算

#### ACF_Fixed48NoW（48位定点）
```cpp
struct FQuatFixed48NoW
{
    uint16 X, Y, Z;  // 每个分量 16 位
    // 范围映射：[-1, 1] → [0, 65535]
};
```
- **旋转**：6 字节（节省 62.5%）
- **编码方式**：
  ```cpp
  uint16 Encode(float Value) {
      return (uint16)((Value + 1.0f) * 32767.5f);
  }
  
  float Decode(uint16 Encoded) {
      return (Encoded / 32767.5f) - 1.0f;
  }
  ```

#### ACF_IntervalFixed32NoW（32位区间压缩）
```cpp
struct FQuatIntervalFixed32NoW
{
    uint32 Packed;  // 打包的 XYZ 分量
    // 格式：11:11:10 位分配
    // X: 11位, Y: 11位, Z: 10位
};
```

这是最高效的压缩格式之一：

```cpp
// 压缩流程：
// 1. 计算所有帧的边界框
float Mins[3], Ranges[3];
CalculateBounds(Keys, Mins, Ranges);

// 2. 归一化到 [0, 1] 区间
float NormX = (Value.X - Mins[0]) / Ranges[0];
float NormY = (Value.Y - Mins[1]) / Ranges[1];
float NormZ = (Value.Z - Mins[2]) / Ranges[2];

// 3. 量化到整数
uint32 PackedX = (uint32)(NormX * 2047);  // 11位
uint32 PackedY = (uint32)(NormY * 2047);  // 11位
uint32 PackedZ = (uint32)(NormZ * 1023);  // 10位

// 4. 打包
uint32 Packed = (PackedX << 21) | (PackedY << 10) | PackedZ;
```

**关键优化**：需要额外存储边界信息（Mins 和 Ranges），但对于长动画来说，这点开销可以忽略。

#### ACF_Identity（恒等变换）
- **存储**：0 字节！
- **用途**：静止骨骼或与绑定姿势相同的轨道
- **实现**：在轨道偏移表中标记为特殊值

---

## 四、压缩流程深度解析

### 4.1 整体流程

```
┌─────────────────────────────────────────────────────────────┐
│                    动画压缩流程                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │ 原始数据  │───▶│  预处理阶段   │───▶│   压缩编码阶段   │   │
│  └──────────┘    └──────────────┘    └─────────────────┘   │
│                         │                      │             │
│                         ▼                      ▼             │
│               ┌──────────────────┐   ┌─────────────────┐   │
│               │ • 移除冗余关键帧   │   │ • 选择压缩格式   │   │
│               │ • 四元数最短路径   │   │ • 量化编码数据   │   │
│               │ • 轨道剔除        │   │ • 构建索引表    │   │
│               └──────────────────┘   └─────────────────┘   │
│                                                              │
│                          ▼                                   │
│                ┌─────────────────┐                          │
│                │   误差评估阶段   │                          │
│                └─────────────────┘                          │
│                         │                                    │
│                         ▼                                    │
│               ┌──────────────────┐                          │
│               │ • 末端效应器误差   │                          │
│               │ • 迭代优化参数     │                          │
│               └──────────────────┘                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 预处理：移除冗余关键帧

```cpp
// 检测静止轨道
bool Compression::CompressRawAnimSequenceTrack(
    FRawAnimSequenceTrack& RawTrack, 
    int32 NumberOfKeys,
    float MaxPosDiff,   // 位移阈值
    float MaxAngleDiff, // 角度阈值
    float MaxScaleDiff) // 缩放阈值
{
    bool bRemovedKeys = false;
    
    // 检查位移是否全部相同
    if (RawTrack.PosKeys.Num() > 1)
    {
        FVector3f FirstPos = RawTrack.PosKeys[0];
        bool bAllSame = true;
        
        for (int32 i = 1; i < RawTrack.PosKeys.Num(); ++i)
        {
            if ((FirstPos - RawTrack.PosKeys[i]).SizeSquared() > 
                FMath::Square(MaxPosDiff))
            {
                bAllSame = false;
                break;
            }
        }
        
        // 如果全部相同，保留第一帧即可
        if (bAllSame)
        {
            RawTrack.PosKeys.RemoveAt(1, RawTrack.PosKeys.Num() - 1);
            bRemovedKeys = true;
        }
    }
    
    // 类似处理旋转和缩放...
    return bRemovedKeys;
}
```

### 4.3 四元数最短路径优化

四元数有个特性：`q` 和 `-q` 代表相同的旋转。为了确保插值时走最短路径：

```cpp
void UAnimCompress::PrecalculateShortestQuaternionRoutes(
    TArray<FRotationTrack>& RotationData)
{
    for (FRotationTrack& Track : RotationData)
    {
        for (int32 i = 1; i < Track.RotKeys.Num(); ++i)
        {
            const FQuat4f& R0 = Track.RotKeys[i-1];
            FQuat4f& R1 = Track.RotKeys[i];
            
            // 如果点积为负，翻转 R1
            if ((R0 | R1) < 0.0f)
            {
                R1 = -R1;  // q 和 -q 代表相同旋转
            }
        }
    }
}
```

### 4.4 位压缩编码

```cpp
void UAnimCompress::BitwiseCompressAnimationTracks(
    const FCompressibleAnimData& AnimData,
    FCompressibleAnimDataResult& OutResult,
    AnimationCompressionFormat TransFormat,
    AnimationCompressionFormat RotFormat,
    AnimationCompressionFormat ScaleFormat,
    const TArray<FTranslationTrack>& TranslationData,
    const TArray<FRotationTrack>& RotationData,
    const TArray<FScaleTrack>& ScaleData)
{
    FUECompressedAnimDataMutable& CompData = 
        static_cast<FUECompressedAnimDataMutable&>(*OutResult.AnimData);
    
    const int32 NumTracks = RotationData.Num();
    
    // 初始化轨道偏移表
    // 格式：[Trans.Offset, Trans.NumKeys, Rot.Offset, Rot.NumKeys] × NumTracks
    CompData.CompressedTrackOffsets.SetNumUninitialized(NumTracks * 4);
    
    for (int32 TrackIndex = 0; TrackIndex < NumTracks; ++TrackIndex)
    {
        // === 压缩位移数据 ===
        const FTranslationTrack& Trans = TranslationData[TrackIndex];
        const int32 TransOffset = CompData.CompressedByteStream.Num();
        const int32 NumTransKeys = Trans.PosKeys.Num();
        
        // 记录偏移和关键帧数
        CompData.CompressedTrackOffsets[TrackIndex * 4 + 0] = TransOffset;
        CompData.CompressedTrackOffsets[TrackIndex * 4 + 1] = NumTransKeys;
        
        if (NumTransKeys > 1)
        {
            // 多帧情况：计算边界框
            FBox3f Bounds(Trans.PosKeys);
            float Mins[3] = { Bounds.Min.X, Bounds.Min.Y, Bounds.Min.Z };
            float Ranges[3] = { 
                Bounds.Max.X - Bounds.Min.X,
                Bounds.Max.Y - Bounds.Min.Y,
                Bounds.Max.Z - Bounds.Min.Z 
            };
            
            // 写入边界信息（用于 IntervalFixed32 格式）
            if (TransFormat == ACF_IntervalFixed32NoW)
            {
                WriteToStream(CompData.CompressedByteStream, Mins, 12);
                WriteToStream(CompData.CompressedByteStream, Ranges, 12);
            }
            
            // 写入所有关键帧
            for (const FVector3f& Key : Trans.PosKeys)
            {
                PackVectorToStream(CompData.CompressedByteStream, 
                                   TransFormat, Key, Mins, Ranges);
            }
        }
        else if (NumTransKeys == 1)
        {
            // 单帧：直接写入未压缩数据
            WriteToStream(CompData.CompressedByteStream, 
                         &Trans.PosKeys[0], sizeof(FVector3f));
        }
        
        // 4字节对齐
        PadByteStream(CompData.CompressedByteStream, 4, AnimationPadSentinel);
        
        // === 类似处理旋转和缩放 ===
        // ...
    }
}
```

---

## 五、误差评估系统

### 5.1 末端效应器误差

压缩误差不是简单地比较原始值和压缩值的差异，而是关注**末端效应器**（手、脚、头部等）在世界空间中的位移：

```cpp
// 误差统计结构
struct FAnimationErrorStats
{
    float AverageError;   // 平均误差（厘米）
    float MaxError;       // 最大误差
    float MaxErrorTime;   // 最大误差出现的时间
    int32 MaxErrorBone;   // 最大误差的骨骼索引
};
```

### 5.2 误差计算流程

```cpp
void FAnimationUtils::ComputeCompressionError(
    const FCompressibleAnimData& AnimData,
    FCompressibleAnimDataResult& CompressedData,
    AnimationErrorStats& ErrorStats)
{
    // 为每个关键帧计算误差
    for (int32 KeyIndex = 0; KeyIndex < NumKeys; ++KeyIndex)
    {
        float Time = FrameRate.AsSeconds(KeyIndex);
        
        // 获取原始姿势（世界空间）
        for (int32 BoneIndex = 0; BoneIndex < NumBones; ++BoneIndex)
        {
            ExtractTransformFromRawTrack(RawTransforms[BoneIndex], ...);
            
            // 解压缩数据
            Codec->DecompressBone(DecompContext, TrackIndex, 
                                  CompressedTransforms[BoneIndex]);
            
            // 转换到世界空间（级联父骨骼变换）
            if (BoneIndex > 0)
            {
                int32 ParentIndex = GetParentIndex(BoneIndex);
                RawTransforms[BoneIndex] *= RawTransforms[ParentIndex];
                CompressedTransforms[BoneIndex] *= CompressedTransforms[ParentIndex];
            }
            
            // 只检查末端效应器
            if (BoneData[BoneIndex].IsEndEffector())
            {
                // 添加虚拟骨骼来放大旋转误差的影响
                const float DummyBoneLength = BoneData[BoneIndex].bKeyEndEffector 
                    ? END_EFFECTOR_DUMMY_BONE_LENGTH_SOCKET 
                    : END_EFFECTOR_DUMMY_BONE_LENGTH;
                    
                FTransform DummyBone(FQuat::Identity, 
                                     FVector(DummyBoneLength, 0, 0));
                RawTransforms[BoneIndex] = DummyBone * RawTransforms[BoneIndex];
                CompressedTransforms[BoneIndex] = DummyBone * CompressedTransforms[BoneIndex];
                
                // 计算末端位置差异
                float Error = (RawTransforms[BoneIndex].GetLocation() - 
                              CompressedTransforms[BoneIndex].GetLocation()).Size();
                
                ErrorStats.Update(Error, BoneIndex, Time);
            }
        }
    }
}
```

### 5.3 虚拟骨骼技巧

为什么要添加"虚拟骨骼"？

因为旋转误差在关节处可能很小，但会随着到末端的距离放大。通过添加一个固定长度的虚拟骨骼，可以更真实地评估旋转误差对最终位置的影响。

```
           原始                     压缩后
            ●                         ●
           /                         / ← 微小角度差异
          /                         /
    关节 ●──────────────● 末端   ●──────────────● ← 较大位置差异
         │                        │
         │ 虚拟骨骼               │
         │                        │
         ●                        ● ← 误差被放大
```

---

## 六、ACL 插件：革命性的压缩方案

### 6.1 什么是 ACL？

**Animation Compression Library (ACL)** 是由 Nicholas Frechette 开发的开源动画压缩库，已被 Epic 集成为官方插件。

### 6.2 ACL 的优势

| 指标 | UE 内置 | ACL |
|------|---------|-----|
| 压缩比 | ~10:1 | ~15:1 |
| 压缩速度 | 较慢 | 快 5-10x |
| 解压速度 | 标准 | 快 2-3x |
| 误差控制 | 启发式 | 精确误差边界 |

### 6.3 ACL 的核心技术

#### 1. 自适应压缩
ACL 会为每个轨道选择最佳的压缩格式，而不是对所有轨道使用相同格式：

```cpp
// ACL 内部会分析每个轨道的特征
// - 范围（Range）
// - 变化率（Variance）
// - 重要性（距离末端效应器的距离）
// 然后选择最佳的量化级别
```

#### 2. 误差度量系统
ACL 使用基于球壳的误差度量：

```cpp
struct FErrorMetric
{
    float ShellDistance;  // 虚拟顶点到骨骼的距离
    
    float CalculateError(const FTransform& Raw, const FTransform& Compressed)
    {
        // 在骨骼周围放置虚拟顶点
        // 计算变换后的最大位移
        FVector ShellPoint(ShellDistance, 0, 0);
        
        FVector RawPoint = Raw.TransformPosition(ShellPoint);
        FVector CompressedPoint = Compressed.TransformPosition(ShellPoint);
        
        return (RawPoint - CompressedPoint).Size();
    }
};
```

#### 3. 数据库压缩
ACL 支持将多个动画压缩到共享数据库中：

```cpp
UCLASS()
class UAnimationCompressionLibraryDatabase : public UObject
{
    // 多个动画共享字典
    // 可以流式加载不同质量级别
    // 支持运行时质量切换
    
    void SetVisualFidelity(ACLVisualFidelity Fidelity);
    // Highest: 完整质量
    // Medium:  移除部分关键帧
    // Lowest:  最小内存占用
};
```

### 6.4 使用 ACL

```cpp
// 在项目设置中启用 ACL
// Project Settings → Animation → Default Bone Compression Settings

// 或者在动画资产上直接指定
UAnimSequence* AnimSeq = ...;
AnimSeq->BoneCompressionSettings = LoadObject<UAnimBoneCompressionSettings>(
    nullptr, TEXT("/ACLPlugin/ACL_Settings")
);
```

---

## 七、解压缩：运行时的关键路径

### 7.1 解压上下文

```cpp
struct FAnimSequenceDecompressionContext
{
    // 采样帧率
    FFrameRate SamplingFrameRate;
    
    // 总帧数
    int32 NumSampledKeys;
    
    // 插值模式
    EAnimInterpolationType Interpolation;
    
    // 当前采样时间
    float Time;
    
    // 压缩数据引用
    const ICompressedAnimData& CompressedAnimData;
    
    void Seek(float InTime);
};
```

### 7.2 解压流程

```cpp
void DecompressPose(
    FCompactPose& OutPose,
    const FCompressedAnimSequence& CompressedData,
    const FAnimExtractContext& ExtractionContext,
    FAnimSequenceDecompressionContext& DecompContext)
{
    // 1. 设置采样时间
    DecompContext.Seek(ExtractionContext.CurrentTime);
    
    // 2. 构建轨道对
    BoneTrackArray RotationPairs, TranslationPairs, ScalePairs;
    BuildTrackPairs(CompressedData, OutPose.GetBoneContainer(),
                    RotationPairs, TranslationPairs, ScalePairs);
    
    // 3. 调用编解码器解压
    CompressedData.BoneCompressionCodec->DecompressPose(
        DecompContext,
        RotationPairs,
        TranslationPairs,
        ScalePairs,
        OutPose.GetMutableBones()
    );
    
    // 4. 处理重定向
    if (RequireRetargeting)
    {
        ApplyRetargeting(OutPose, ...);
    }
    
    // 5. 处理根运动
    if (ExtractionContext.bExtractRootMotion)
    {
        ResetRootBoneForRootMotion(OutPose[0], ...);
    }
}
```

### 7.3 插值计算

```cpp
// 线性插值（最常用）
FTransform InterpolateLinear(
    const FTransform& A, 
    const FTransform& B, 
    float Alpha)
{
    return FTransform(
        FQuat::Slerp(A.GetRotation(), B.GetRotation(), Alpha),
        FMath::Lerp(A.GetTranslation(), B.GetTranslation(), Alpha),
        FMath::Lerp(A.GetScale3D(), B.GetScale3D(), Alpha)
    );
}

// 阶梯插值（某些特效动画需要）
FTransform InterpolateStep(
    const FTransform& A, 
    const FTransform& B, 
    float Alpha)
{
    return Alpha < 1.0f ? A : B;
}
```

---

## 八、性能优化技巧

### 8.1 批量解压

UE5 支持 SoA（Structure of Arrays）布局以提升 SIMD 性能：

```cpp
// 传统 AoS 布局
struct FTransform { FQuat Rotation; FVector Translation; FVector Scale; };
TArray<FTransform> Poses;

// SoA 布局（更好的缓存局部性）
struct FPoseSoA {
    TArray<FQuat> Rotations;
    TArray<FVector> Translations;
    TArray<FVector> Scales;
};
```

### 8.2 关键帧剥离

对于不重要的动画，可以在烘焙时移除部分帧：

```cpp
// 移除偶数帧（保留奇数帧）
void StripFramesEven(TArray<FVector3f>& Keys, int32 NumFrames)
{
    if (Keys.Num() > 1)
    {
        for (int32 Dst = 1, Src = 2; Src < NumFrames; ++Dst, Src += 2)
        {
            Keys[Dst] = Keys[Src];
        }
        
        int32 NewSize = (NumFrames - 1) / 2 + 1;
        Keys.SetNum(NewSize);
    }
}
```

### 8.3 专用服务器优化

在专用服务器上，可以剥离除根运动外的所有数据：

```cpp
UENUM()
enum class EStripAnimDataOnDedicatedServerSettings : uint8
{
    UseProjectSetting,                    // 使用项目设置
    StripAnimDataOnDedicatedServer,       // 强制剥离
    DoNotStripAnimDataOnDedicatedServer   // 强制保留
};
```

---

## 九、调试与分析

### 9.1 控制台命令

```bash
# 列出所有编解码器的使用情况
ACL.ListCodecs

# 列出所有动画序列的压缩详情
ACL.ListAnimSequences

# 设置 ACL 数据库的视觉质量
ACL.SetDatabaseVisualFidelity Highest|Medium|Lowest
```

### 9.2 压缩统计输出

```cpp
// 启用压缩统计
#if WITH_EDITOR
UE::Anim::Compression::FAnimationCompressionMemorySummaryScope CompressionScope;
// 压缩完成后会输出详细统计
#endif
```

输出示例：
```
Compressed 128 Animation(s)
Pre Compression:  Raw: 156.32 MB - Compressed: 45.21 MB (Ratio: 3.46)
Post Compression: Raw: 156.32 MB - Compressed: 12.87 MB (Ratio: 12.15)

Top 10 Worst Bone Errors:
1) 0.342 in Animation Idle_Combat, Bone: hand_r (#45), at Time 2.100
2) 0.298 in Animation Run_Forward, Bone: foot_l (#62), at Time 0.433
...
```

---

## 十、最佳实践

### 10.1 压缩设置建议

| 动画类型 | 推荐设置 | 原因 |
|---------|---------|------|
| 角色主要动画 | ACL Default | 平衡质量和大小 |
| 面部动画 | ACL Safe | 需要高精度 |
| 背景NPC | ACL + 帧剥离 | 可接受较大误差 |
| 过场动画 | 无压缩或ACL Safe | 艺术家要求精确 |

### 10.2 常见问题排查

**Q: 压缩后动画抖动？**
- 检查误差阈值是否过大
- 考虑使用 ACL Safe 编解码器
- 检查骨骼是否设置了正确的 Shell Distance

**Q: 加载时间过长？**
- 启用异步压缩
- 使用 DDC（Derived Data Cache）
- 考虑使用 ACL Database 共享数据

**Q: 内存占用过高？**
- 审计不必要的动画资产
- 使用帧剥离
- 在专用服务器上剥离动画数据

---

## 总结

动画压缩是一个在**精度、内存和性能**之间权衡的艺术。UE5 提供了灵活的压缩系统：

1. **多种内置格式**：从无损到高度压缩
2. **智能误差评估**：基于末端效应器的精确误差测量
3. **ACL 集成**：业界领先的压缩比和解压性能
4. **可扩展架构**：支持自定义编解码器

理解这些原理，可以帮助你：
- 为不同类型的动画选择合适的压缩策略
- 诊断和解决压缩相关的质量问题
- 优化游戏的内存占用和加载时间

---

## 参考资源

- [UE5 Animation Compression Documentation](https://docs.unrealengine.com/5.0/en-US/animation-compression-in-unreal-engine/)
- [ACL GitHub Repository](https://github.com/nfrechette/acl)
- [GDC 2017: Animation Compression](https://www.gdcvault.com/play/1024463/Animation-Compression)

---

> 📝 **作者注**：本文基于 UE5.x 源码分析，部分 API 可能在后续版本中变化。如有疑问，请以官方文档为准。
