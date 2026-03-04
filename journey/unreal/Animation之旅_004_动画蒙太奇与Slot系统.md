# Animation之旅_004：动画蒙太奇 (Montage) 与 Slot 系统

> **系列目标**：通过源码级别的学习，深入理解 Unreal Engine 动画系统的设计理念与实现细节
> 
> **本篇主题**：`UAnimMontage` - 动画蒙太奇，UE 中最强大的动画播放控制工具

---

## 📚 目录

1. [什么是 AnimMontage？](#1-什么是-animmontage)
2. [类结构与继承体系](#2-类结构与继承体系)
3. [核心数据结构](#3-核心数据结构)
4. [Slot 系统详解](#4-slot-系统详解)
5. [Section 系统](#5-section-系统)
6. [Montage 播放控制](#6-montage-播放控制)
7. [FAnimMontageInstance 运行时实例](#7-fanimmontageinstance-运行时实例)
8. [BlendIn / BlendOut 混合机制](#8-blendin--blendout-混合机制)
9. [Branching Points 分支点](#9-branching-points-分支点)
10. [Root Motion 与 Montage](#10-root-motion-与-montage)
11. [动态 Montage 创建](#11-动态-montage-创建)
12. [实践：技能系统中的 Montage](#12-实践技能系统中的-montage)

---

## 1. 什么是 AnimMontage？

### 1.1 Montage 的定位

```
┌─────────────────────────────────────────────────────────────┐
│                 UE 动画播放方式对比                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  AnimSequence 直接播放                                       │
│  ├── 简单直接，但控制能力有限                                │
│  ├── 无法插入到动画图中的特定位置                            │
│  └── 无法在播放过程中跳转或循环                              │
│                                                             │
│  BlendSpace                                                  │
│  ├── 多维度混合，但无法精确控制播放流程                      │
│  └── 主要用于 locomotion 等持续动画                          │
│                                                             │
│  AnimMontage ★                                               │
│  ├── 完整的播放控制：播放、暂停、跳转、循环                  │
│  ├── 多段 Section 组合，支持运行时修改播放顺序               │
│  ├── 通过 Slot 插入到动画图的任意位置                        │
│  ├── Branching Points 实现精确的事件触发                     │
│  └── 完整的混合控制：BlendIn、BlendOut、中断                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 典型应用场景

| 场景 | 说明 |
|------|------|
| **技能释放** | 攻击、施法动画，需要精确控制和事件触发 |
| **连击系统** | 多段动画组合，运行时决定下一段播放内容 |
| **QTE 系统** | 需要在特定时间点响应输入 |
| **过场动画** | 与游戏逻辑紧密结合的角色表演 |
| **受击/死亡** | 需要打断当前动画并混合的特殊状态 |

---

## 2. 类结构与继承体系

### 2.1 继承关系

```
UObject
    │
    └── UAnimationAsset
            │
            └── UAnimSequenceBase
                    │
                    ├── UAnimSequence          // 单个动画序列
                    │
                    └── UAnimCompositeBase     // 组合动画基类
                            │
                            ├── UAnimComposite // 简单组合
                            │
                            └── UAnimMontage   // 蒙太奇 ★
                                    │
                                    ├── Sections[]        // 段落系统
                                    ├── SlotAnimTracks[]  // Slot 轨道
                                    ├── BlendIn/BlendOut  // 混合设置
                                    └── BranchingPoints   // 分支点
```

### 2.2 UAnimMontage 类定义概览

```cpp
UCLASS(config=Engine, hidecategories=(UObject, Length), MinimalAPI, BlueprintType)
class UAnimMontage : public UAnimCompositeBase
{
    GENERATED_UCLASS_BODY()

    // ========== 混合设置 ==========
    /** 混合模式：标准混合 或 惯性化混合 */
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = BlendOption)
    EMontageBlendMode BlendModeIn;
    
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = BlendOption)
    EMontageBlendMode BlendModeOut;

    /** BlendIn 混合曲线 */
    UPROPERTY(EditAnywhere, Category=BlendOption)
    FAlphaBlend BlendIn;

    /** BlendOut 混合曲线 */
    UPROPERTY(EditAnywhere, Category=BlendOption)
    FAlphaBlend BlendOut;

    /** 触发 BlendOut 的时间（相对于结束） */
    UPROPERTY(EditAnywhere, Category = BlendOption)
    float BlendOutTriggerTime;

    /** 是否自动混合出 */
    UPROPERTY(EditAnywhere, Category = BlendOption)
    bool bEnableAutoBlendOut;

    // ========== 同步组 ==========
    UPROPERTY(EditAnywhere, Category = SyncGroup)
    FName SyncGroup;

    UPROPERTY(EditAnywhere, Category = SyncGroup)
    int32 SyncSlotIndex;

    // ========== 核心数据 ==========
    /** Section 列表 - 垂直分段 */
    UPROPERTY()
    TArray<FCompositeSection> CompositeSections;
    
    /** Slot 轨道 - 水平轨道 */
    UPROPERTY()
    TArray<FSlotAnimationTrack> SlotAnimTracks;

    /** 分支点标记（运行时缓存） */
    UPROPERTY()
    TArray<FBranchingPointMarker> BranchingPointMarkers;

    // ========== Root Motion ==========
    UPROPERTY()
    bool bEnableRootMotionTranslation;

    UPROPERTY()
    bool bEnableRootMotionRotation;

    UPROPERTY()
    TEnumAsByte<ERootMotionRootLock::Type> RootMotionRootLock;

    // ========== 混合配置 ==========
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = BlendOption)
    TObjectPtr<UBlendProfile> BlendProfileIn;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = BlendOption)
    TObjectPtr<UBlendProfile> BlendProfileOut;
};
```

---

## 3. 核心数据结构

### 3.1 Slot 动画轨道

```cpp
/**
 * 每个 Slot 包含一条动画轨道
 * Montage 可以有多个 Slot，每个 Slot 对应动画图中的一个 Slot Node
 */
USTRUCT()
struct FSlotAnimationTrack
{
    GENERATED_USTRUCT_BODY()

    /** Slot 名称 - 必须与动画图中的 Slot Node 名称匹配 */
    UPROPERTY(EditAnywhere, Category=Slot)
    FName SlotName;

    /** 该 Slot 的动画轨道 */
    UPROPERTY(EditAnywhere, Category=Slot)
    FAnimTrack AnimTrack;
};
```

### 3.2 Composite Section

```cpp
/**
 * Montage 的段落结构
 * 每个 Section 定义了一个可跳转的时间点
 */
USTRUCT()
struct FCompositeSection : public FAnimLinkableElement
{
    GENERATED_USTRUCT_BODY()

    /** Section 名称 - 用于代码中引用 */
    UPROPERTY(EditAnywhere, Category=Section)
    FName SectionName;

    /** 下一个 Section 的名称（用于自动播放链） */
    UPROPERTY(VisibleAnywhere, Category=Section)
    FName NextSectionName;

    /** 元数据 */
    UPROPERTY(Category=Section, Instanced, EditAnywhere)
    TArray<TObjectPtr<class UAnimMetaData>> MetaData;
};
```

### 3.3 Branching Point Marker

```cpp
/**
 * 分支点标记 - 用于精确触发事件
 * 比普通 AnimNotify 更精确，在 Montage Advance 中同步处理
 */
USTRUCT()
struct FBranchingPointMarker
{
    GENERATED_USTRUCT_BODY()

    /** 对应的 Notify 索引 */
    UPROPERTY()
    int32 NotifyIndex;
    
    /** 触发时间 */
    UPROPERTY()
    float TriggerTime;

    /** 事件类型：Begin 或 End */
    UPROPERTY()
    TEnumAsByte<EAnimNotifyEventType::Type> NotifyEventType;
};
```

### 3.4 数据结构可视化

```
┌─────────────────────────────────────────────────────────────────┐
│                        UAnimMontage                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SlotAnimTracks[] (水平轨道)                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Slot: "UpperBody"                                        │   │
│  │ ├─[Segment0: Attack_Start]─┼─[Segment1: Attack_Loop]─┤  │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Slot: "LowerBody"                                        │   │
│  │ ├─[Segment0: Idle]─────────────────────────────────────┤  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  CompositeSections[] (垂直分段)                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │   │ Section: "Start"    │ Section: "Loop"   │ "End"    │   │
│  │   │ NextSection="Loop"  │ NextSection="Loop"│ =None    │   │
│  │   ↓                     ↓                   ↓          │   │
│  │ Time: 0.0              0.5                  1.5        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  BranchingPointMarkers[]                                         │
│  ├── [Time: 0.2, Type: Begin, NotifyIndex: 0]                   │
│  ├── [Time: 0.8, Type: End,   NotifyIndex: 0]                   │
│  └── [Time: 1.0, Type: Begin, NotifyIndex: 1]                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Slot 系统详解

### 4.1 Slot 的概念

```
┌─────────────────────────────────────────────────────────────────┐
│                      Slot 系统工作原理                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Animation Blueprint (动画图)                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │   [State Machine] ──► [Slot Node: DefaultSlot] ──► [Out] │   │
│  │         ↑                      ↑                         │   │
│  │    Base Pose              Montage 插入点                  │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  当 Montage 播放时：                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │   BasePose ──[Weight: 1-α]──┐                            │   │
│  │                              ├──► Blend ──► Final Pose   │   │
│  │   Montage  ──[Weight: α]────┘                            │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  α = Montage 的 Blend 权重 (0~1)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Slot 与 Group

```cpp
// Skeleton 中定义的 Slot 和 Group
class USkeleton
{
    // Slot 按 Group 组织
    // 同一 Group 内的 Slot 共享 Montage 播放状态
    TArray<FAnimSlotGroup> SlotGroups;
};

struct FAnimSlotGroup
{
    // Group 名称
    FName GroupName;
    
    // 该 Group 下的所有 Slot 名称
    TArray<FName> SlotNames;
};

// 默认的 Group 和 Slot
namespace FAnimSlotGroup
{
    const FName DefaultGroupName = "DefaultGroup";
    const FName DefaultSlotName = "DefaultSlot";
}
```

### 4.3 多 Slot 应用场景

```
场景：上下半身分离动画

┌──────────────────────────────────────────────────────────┐
│                   Animation Graph                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  [Locomotion] ──► [Slot: LowerBody] ──┐                  │
│                                       ├──► [Layered Blend │
│  [Locomotion] ──► [Slot: UpperBody] ──┘     Per Bone]    │
│                                             ──► [Output]  │
│                                                          │
└──────────────────────────────────────────────────────────┘

Montage 配置：
- SlotAnimTracks[0]: SlotName="UpperBody", AnimTrack=[Attack]
- SlotAnimTracks[1]: SlotName="LowerBody", AnimTrack=[Idle]

结果：上半身播放攻击动画，下半身保持移动动画
```

### 4.4 Slot 权重计算

```cpp
// AnimInstance 中的 Slot 权重追踪
struct FMontageActiveSlotTracker
{
    /** Montage 在该 Slot 中的本地权重 */
    float MontageLocalWeight;

    /** Slot 节点的全局权重 */
    float NodeGlobalWeight;

    /** 本帧是否相关（在活跃的动画图分支中） */
    bool bIsRelevantThisTick;

    /** 上一帧是否相关 */
    bool bWasRelevantOnPreviousTick;
};

// 获取 Slot 的全局权重
float UAnimInstance::GetSlotNodeGlobalWeight(const FName& SlotNodeName) const
{
    // Slot 节点的全局权重 = 该节点在动画图中的实际贡献
    return GetProxyOnGameThread<FAnimInstanceProxy>().GetSlotNodeGlobalWeight(SlotNodeName);
}

// 获取 Montage 在该 Slot 中的权重
float UAnimInstance::GetSlotMontageGlobalWeight(const FName& SlotNodeName) const
{
    // = Slot 全局权重 × Montage 本地权重
    return GetProxyOnAnyThread<FAnimInstanceProxy>().GetSlotMontageGlobalWeight(SlotNodeName);
}
```

---

## 5. Section 系统

### 5.1 Section 的作用

```
┌─────────────────────────────────────────────────────────────────┐
│                      Section 系统                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Section 将 Montage 分割成多个可跳转的段落                       │
│                                                                 │
│  Timeline:                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ [Section A]     │ [Section B]     │ [Section C]         │   │
│  │ Start: 0.0      │ Start: 1.0      │ Start: 2.0          │   │
│  │ Next: B         │ Next: B (loop)  │ Next: None (end)    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  播放流程：A → B → B → B → ... → (SetNextSection) → C → 结束    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Section 操作 API

```cpp
class UAnimInstance
{
public:
    /** 跳转到指定 Section */
    UFUNCTION(BlueprintCallable, Category="Animation|Montage")
    void Montage_JumpToSection(FName SectionName, const UAnimMontage* Montage = NULL);

    /** 跳转到 Section 的末尾 */
    UFUNCTION(BlueprintCallable, Category="Animation|Montage")
    void Montage_JumpToSectionsEnd(FName SectionName, const UAnimMontage* Montage = NULL);

    /**
     * 在运行时修改 Section 链接
     * 例如：将 Loop Section 的 NextSection 改为 End，以结束循环
     */
    UFUNCTION(BlueprintCallable, Category="Animation|Montage")
    void Montage_SetNextSection(FName SectionNameToChange, FName NextSection, 
                                const UAnimMontage* Montage = NULL);

    /** 获取当前播放的 Section 名称 */
    UFUNCTION(BlueprintPure, Category="Animation|Montage")
    FName Montage_GetCurrentSection(const UAnimMontage* Montage = NULL) const;
};
```

### 5.3 Section 实现细节

```cpp
// FAnimMontageInstance 中的 Section 管理
struct FAnimMontageInstance
{
private:
    /** 每个 Section 的下一个 Section 索引 */
    TArray<int32> NextSections;

    /** 每个 Section 的上一个 Section 索引 */
    TArray<int32> PrevSections;

public:
    /** 跳转到指定 Section */
    bool JumpToSectionName(FName const & SectionName, bool bEndOfSection = false)
    {
        const int32 SectionID = Montage->GetSectionIndex(SectionName);

        if (Montage->IsValidSectionIndex(SectionID))
        {
            FCompositeSection & CurSection = Montage->GetAnimCompositeSection(SectionID);
            
            // 计算新位置
            const float NewPosition = Montage->CalculatePos(CurSection, 
                bEndOfSection ? Montage->GetSectionLength(SectionID) - UE_KINDA_SMALL_NUMBER : 0.0f);
            
            // 设置位置
            SetPosition(NewPosition);
            OnMontagePositionChanged(SectionName);
            return true;
        }

        return false;
    }

    /** 设置 Section 的下一个 Section */
    bool SetNextSectionID(int32 const & SectionID, int32 const & NewNextSectionID)
    {
        bool const bHasValidNextSection = NextSections.IsValidIndex(SectionID);

        // 断开旧的 PrevSection 链接
        if (bHasValidNextSection && (NextSections[SectionID] != INDEX_NONE))
        {
            PrevSections[NextSections[SectionID]] = INDEX_NONE;
        }

        // 建立新的 PrevSection 链接
        if (PrevSections.IsValidIndex(NewNextSectionID))
        {
            PrevSections[NewNextSectionID] = SectionID;
        }

        // 更新 NextSection
        if (bHasValidNextSection)
        {
            NextSections[SectionID] = NewNextSectionID;
            return true;
        }

        return false;
    }
};
```

### 5.4 循环 Section 示例

```cpp
// 攻击连击实现
void AMyCharacter::StartComboAttack()
{
    // 播放 Montage
    UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
    AnimInstance->Montage_Play(ComboMontage);
    
    // Montage 结构:
    // [Start] → [Loop] → [End]
    //             ↺ (循环自己)
    
    // 默认情况下会一直循环 Loop Section
}

void AMyCharacter::FinishCombo()
{
    UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
    
    // 修改 Loop 的 NextSection 为 End，打破循环
    AnimInstance->Montage_SetNextSection(
        FName("Loop"),      // 要修改的 Section
        FName("End"),       // 新的下一个 Section
        ComboMontage
    );
    
    // 现在当 Loop 播放完毕后，会自动跳转到 End
}
```

---

## 6. Montage 播放控制

### 6.1 播放 API

```cpp
class UAnimInstance
{
public:
    /** 播放 Montage，返回动画长度（秒） */
    UFUNCTION(BlueprintCallable, Category = "Animation|Montage")
    float Montage_Play(
        UAnimMontage* MontageToPlay, 
        float InPlayRate = 1.f, 
        EMontagePlayReturnType ReturnValueType = EMontagePlayReturnType::MontageLength, 
        float InTimeToStartMontageAt = 0.f, 
        bool bStopAllMontages = true
    );

    /** 使用自定义 BlendIn 设置播放 */
    UFUNCTION(BlueprintCallable, Category = "Animation|Montage")
    float Montage_PlayWithBlendIn(
        UAnimMontage* MontageToPlay, 
        const FAlphaBlendArgs& BlendIn, 
        float InPlayRate = 1.f, 
        EMontagePlayReturnType ReturnValueType = EMontagePlayReturnType::MontageLength, 
        float InTimeToStartMontageAt = 0.f, 
        bool bStopAllMontages = true
    );

    /** 使用完整的混合设置播放 */
    UFUNCTION(BlueprintCallable, Category = "Animation|Montage")
    float Montage_PlayWithBlendSettings(
        UAnimMontage* MontageToPlay, 
        const FMontageBlendSettings& BlendInSettings, 
        float InPlayRate = 1.f, 
        EMontagePlayReturnType ReturnValueType = EMontagePlayReturnType::MontageLength, 
        float InTimeToStartMontageAt = 0.f, 
        bool bStopAllMontages = true
    );

    /** 停止 Montage */
    UFUNCTION(BlueprintCallable, Category = "Animation|Montage")
    void Montage_Stop(float InBlendOutTime, const UAnimMontage* Montage = NULL);

    /** 暂停 Montage */
    UFUNCTION(BlueprintCallable, Category = "Animation|Montage")
    void Montage_Pause(const UAnimMontage* Montage = NULL);

    /** 恢复 Montage */
    UFUNCTION(BlueprintCallable, Category = "Animation|Montage")
    void Montage_Resume(const UAnimMontage* Montage);

    /** 设置播放速率 */
    UFUNCTION(BlueprintCallable, Category="Animation|Montage")
    void Montage_SetPlayRate(const UAnimMontage* Montage, float NewPlayRate = 1.f);

    /** 设置播放位置 */
    UFUNCTION(BlueprintCallable, Category = "Animation|Montage")
    void Montage_SetPosition(const UAnimMontage* Montage, float NewPosition);
};
```

### 6.2 播放流程详解

```cpp
float UAnimInstance::Montage_PlayInternal(
    UAnimMontage* MontageToPlay, 
    const FMontageBlendSettings& BlendInSettings,
    float InPlayRate,
    EMontagePlayReturnType ReturnValueType,
    float InTimeToStartMontageAt,
    bool bStopAllMontages)
{
    if (MontageToPlay && (MontageToPlay->GetPlayLength() > 0.f) && 
        MontageToPlay->HasValidSlotSetup())
    {
        // 1. 验证骨骼匹配
        if (CurrentSkeleton && MontageToPlay->GetSkeleton())
        {
            const FName NewMontageGroupName = MontageToPlay->GetGroupName();
            
            // 2. 停止同组的其他 Montage（强制同组只有一个活跃 Montage）
            if (bStopAllMontages)
            {
                StopAllMontagesByGroupName(NewMontageGroupName, BlendInSettings);
            }

            // 3. Root Motion Montage 互斥处理
            if (MontageToPlay->bEnableRootMotionTranslation || 
                MontageToPlay->bEnableRootMotionRotation)
            {
                FAnimMontageInstance* ActiveRootMotionMontageInstance = GetRootMotionMontageInstance();
                if (ActiveRootMotionMontageInstance)
                {
                    ActiveRootMotionMontageInstance->Stop(BlendInSettings);
                }
            }

            // 4. 创建新的 MontageInstance
            FAnimMontageInstance* NewInstance = new FAnimMontageInstance(this);
            NewInstance->Initialize(MontageToPlay);
            NewInstance->Play(InPlayRate, BlendInSettings);
            NewInstance->SetPosition(FMath::Clamp(InTimeToStartMontageAt, 0.f, MontageLength));
            
            // 5. 注册到实例列表
            MontageInstances.Add(NewInstance);
            ActiveMontagesMap.Add(MontageToPlay, NewInstance);

            // 6. 设置 Root Motion 实例
            if (MontageToPlay->HasRootMotion())
            {
                RootMotionMontageInstance = NewInstance;
            }

            // 7. 触发开始事件
            OnMontageStarted.Broadcast(MontageToPlay);

            return (ReturnValueType == EMontagePlayReturnType::MontageLength) ? 
                MontageLength : 
                (MontageLength / (InPlayRate * MontageToPlay->RateScale));
        }
    }

    return 0.f;
}
```

### 6.3 播放状态查询

```cpp
class UAnimInstance
{
public:
    /** 是否有任何 Montage 正在播放 */
    UFUNCTION(BlueprintPure, Category = "Animation|Montage")
    bool IsAnyMontagePlaying() const;

    /** 指定 Montage 是否活跃（包括正在混合出的） */
    UFUNCTION(BlueprintPure, Category="Animation|Montage")
    bool Montage_IsActive(const UAnimMontage* Montage) const;

    /** 指定 Montage 是否正在播放（不包括正在混合出的） */
    UFUNCTION(BlueprintPure, Category="Animation|Montage")
    bool Montage_IsPlaying(const UAnimMontage* Montage) const;

    /** Montage 是否已停止 */
    UFUNCTION(BlueprintPure, Category = "Animation|Montage")
    bool Montage_GetIsStopped(const UAnimMontage* Montage) const;

    /** 获取当前播放位置 */
    UFUNCTION(BlueprintPure, Category = "Animation|Montage")
    float Montage_GetPosition(const UAnimMontage* Montage) const;

    /** 获取播放速率 */
    UFUNCTION(BlueprintPure, Category = "Animation|Montage")
    float Montage_GetPlayRate(const UAnimMontage* Montage) const;
};
```

---

## 7. FAnimMontageInstance 运行时实例

### 7.1 实例生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                  FAnimMontageInstance 生命周期                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [创建 & 初始化]                                                 │
│       │                                                         │
│       ▼                                                         │
│  Montage_Play() ──► Initialize() ──► Play()                     │
│       │                                                         │
│       ▼                                                         │
│  [BlendIn 阶段] ──────────────────────────────────►             │
│       │           Weight: 0 → 1                                 │
│       ▼                                                         │
│  [播放中] ◄──────────────────────────────────────────┐          │
│       │    UpdateWeight() + Advance()      循环/跳转 │          │
│       │                                              │          │
│       ├──── Section 切换 ───────────────────────────┘          │
│       │                                                         │
│       ▼                                                         │
│  [BlendOut 触发] ◄── Stop() 或 AutoBlendOut                     │
│       │           Weight: 1 → 0                                 │
│       ▼                                                         │
│  [BlendOut 完成]                                                 │
│       │                                                         │
│       ▼                                                         │
│  Terminate() ──► 清理引用 ──► delete                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 关键成员变量

```cpp
USTRUCT()
struct FAnimMontageInstance
{
    // ========== 基础状态 ==========
    UPROPERTY()
    TObjectPtr<class UAnimMontage> Montage;

    UPROPERTY()
    bool bPlaying;

    /** 唯一实例 ID */
    int32 InstanceID;

    /** 当前播放位置 */
    UPROPERTY()
    float Position;

    /** 播放速率 */
    UPROPERTY()
    float PlayRate;

    // ========== 混合状态 ==========
    UPROPERTY(transient)
    FAlphaBlend Blend;

    /** 是否被中断 */
    bool bInterrupted;

    /** 上一帧权重 */
    float PreviousWeight;

    /** 用于 Notify 的权重 */
    float NotifyWeight;

    /** 混合开始时的 Alpha 值 */
    float BlendStartAlpha;

    // ========== Section 管理 ==========
    /** 每个 Section 的下一个 Section */
    UPROPERTY()
    TArray<int32> NextSections;

    /** 每个 Section 的上一个 Section */
    UPROPERTY()
    TArray<int32> PrevSections;

    // ========== 事件委托 ==========
    FOnMontageEnded OnMontageEnded;
    FOnMontageBlendingOutStarted OnMontageBlendingOutStarted;
    FOnMontageBlendedInEnded OnMontageBlendedInEnded;
    FOnMontageSectionChanged OnMontageSectionChanged;

    // ========== 同步 ==========
    FName SyncGroupName;
    struct FAnimMontageInstance* MontageSyncLeader;
    TArray<struct FAnimMontageInstance*> MontageSyncFollowers;
};
```

### 7.3 Advance - 核心更新函数

```cpp
void FAnimMontageInstance::Advance(float DeltaTime, 
    FRootMotionMovementParams* OutRootMotionParams, bool bBlendRootMotion)
{
    if (IsValid())
    {
        if (bPlaying)
        {
            const bool bExtractRootMotion = (OutRootMotionParams != nullptr) && Montage->HasRootMotion();
            
            // 初始化时间记录
            DeltaTimeRecord.Set(Position, 0.f);

            // 子步进器处理时间推进
            MontageSubStepper.AddEvaluationTime(DeltaTime);
            
            while (bPlaying && MontageSubStepper.HasTimeRemaining())
            {
                const float PreviousSubStepPosition = Position;
                const FBranchingPointMarker* BranchingPointMarker = nullptr;
                
                // 推进位置
                EMontageSubStepResult SubStepResult = MontageSubStepper.Advance(Position, &BranchingPointMarker);

                if (SubStepResult == EMontageSubStepResult::InvalidSection ||
                    SubStepResult == EMontageSubStepResult::InvalidMontage)
                {
                    Stop(FAlphaBlend(Montage->BlendOut), false);
                    break;
                }

                // 累积移动量
                const float SubStepDeltaMove = MontageSubStepper.GetDeltaMove();
                DeltaTimeRecord.Delta += SubStepDeltaMove;

                // 自动混合出检查
                if (!IsStopped() && bEnableAutoBlendOut)
                {
                    const int32 CurrentSectionIndex = MontageSubStepper.GetCurrentSectionIndex();
                    const int32 NextSectionIndex = bPlayingForward ? 
                        NextSections[CurrentSectionIndex] : PrevSections[CurrentSectionIndex];
                    
                    // 如果是最后一个 Section
                    if (NextSectionIndex == INDEX_NONE)
                    {
                        const float PlayTimeToEnd = MontageSubStepper.GetRemainingPlayTimeToSectionEnd(Position);
                        const float BlendOutTriggerTime = (Montage->BlendOutTriggerTime >= 0) ? 
                            Montage->BlendOutTriggerTime : Montage->BlendOut.GetBlendTime();

                        if (PlayTimeToEnd <= BlendOutTriggerTime)
                        {
                            Stop(FAlphaBlend(Montage->BlendOut), false);
                        }
                    }
                }

                // 提取 Root Motion
                if (bExtractRootMotion && !IsRootMotionDisabled())
                {
                    const FTransform RootMotion = Montage->ExtractRootMotionFromTrackRange(
                        PreviousSubStepPosition, Position);
                    
                    if (bBlendRootMotion)
                    {
                        AnimInstance.Get()->QueueRootMotionBlend(
                            RootMotion, Montage->SlotAnimTracks[0].SlotName, Blend.GetBlendedValue());
                    }
                    else
                    {
                        OutRootMotionParams->Accumulate(RootMotion);
                    }
                }

                // 处理事件（通知、分支点）
                HandleEvents(PreviousSubStepPosition, Position, BranchingPointMarker);

                // Section 切换检查
                if (MontageSubStepper.HasReachedEndOfSection() && !BranchingPointMarker)
                {
                    const int32 RecentNextSectionIndex = bPlayingForward ? 
                        NextSections[CurrentSectionIndex] : PrevSections[CurrentSectionIndex];
                    
                    if (RecentNextSectionIndex != INDEX_NONE)
                    {
                        // 跳转到下一个 Section
                        float NextSectionStartTime, NextSectionEndTime;
                        Montage->GetSectionStartAndEndTime(RecentNextSectionIndex, 
                            NextSectionStartTime, NextSectionEndTime);
                        Position = bPlayingForward ? NextSectionStartTime : 
                            (NextSectionEndTime - UE_KINDA_SMALL_NUMBER);
                    }
                    else
                    {
                        // 到达结尾
                        bPlaying = false;
                        break;
                    }
                }
            }
        }
    }

    // 检查是否应该终止
    if (IsStopped() && Blend.IsComplete())
    {
        Terminate();
    }
}
```

---

## 8. BlendIn / BlendOut 混合机制

### 8.1 混合设置

```cpp
/**
 * Montage 混合设置
 */
USTRUCT(BlueprintType)
struct FMontageBlendSettings
{
    GENERATED_BODY()

    /** 混合曲线配置 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Blend")
    FAlphaBlendArgs Blend;

    /** 混合模式：标准 或 惯性化 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Blend")
    EMontageBlendMode BlendMode;

    /** 混合 Profile（骨骼混合权重配置） */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Blend")
    TObjectPtr<UBlendProfile> BlendProfile;
};

UENUM()
enum class EMontageBlendMode : uint8
{
    /** 标准权重混合 */
    Standard,
    
    /** 惯性化混合 - 需要动画图中有 Inertialization 节点 */
    Inertialization,
};
```

### 8.2 BlendIn 流程

```cpp
void FAnimMontageInstance::Play(float InPlayRate, const FMontageBlendSettings& BlendInSettings)
{
    bPlaying = true;
    PlayRate = InPlayRate;

    // 惯性化处理
    FAlphaBlendArgs BlendInArgs = BlendInSettings.Blend;
    if (AnimInstance.IsValid() && BlendInSettings.BlendMode == EMontageBlendMode::Inertialization)
    {
        FInertializationRequest Request;
        Request.Duration = BlendInArgs.BlendTime;
        Request.BlendMode = BlendInArgs.BlendOption;
        Request.CustomBlendCurve = BlendInArgs.CustomCurve;
        Request.BlendProfile = BlendInSettings.BlendProfile;

        // 请求惯性化
        AnimInstance->RequestMontageInertialization(Montage, Request);

        // 惯性化模式下立即混合
        BlendInArgs.BlendTime = 0.0f;
    }

    // 设置混合参数
    float CurrentWeight = Blend.GetBlendedValue();
    InitializeBlend(FAlphaBlend(BlendInArgs));
    BlendStartAlpha = Blend.GetAlpha();
    Blend.SetBlendTime(BlendInArgs.BlendTime * DefaultBlendTimeMultiplier);
    Blend.SetValueRange(CurrentWeight, 1.f);  // 从当前权重混合到 1.0
    bEnableAutoBlendOut = Montage->bEnableAutoBlendOut;

    ActiveBlendProfile = BlendInSettings.BlendProfile;
}
```

### 8.3 BlendOut 流程

```cpp
void FAnimMontageInstance::Stop(const FMontageBlendSettings& InBlendOutSettings, bool bInterrupt)
{
    // 标记中断状态
    if (!bInterrupted && bInterrupt)
    {
        bInterrupted = bInterrupt;
    }

    if (IsStopped() == false)
    {
        // 惯性化处理
        FAlphaBlendArgs BlendOutArgs = InBlendOutSettings.Blend;
        const bool bShouldInertialize = InBlendOutSettings.BlendMode == EMontageBlendMode::Inertialization;
        BlendOutArgs.BlendTime = bShouldInertialize ? 0.0f : BlendOutArgs.BlendTime;

        // 设置混合参数
        InitializeBlend(FAlphaBlend(BlendOutArgs));
        BlendStartAlpha = Blend.GetAlpha();
        Blend.SetDesiredValue(0.f);  // 目标权重为 0
        Blend.Update(0.0f);

        ActiveBlendProfile = InBlendOutSettings.BlendProfile;

        // 通知 AnimInstance
        AnimInstance->OnMontageInstanceStopped(*this);
        AnimInstance->QueueMontageBlendingOutEvent(
            FQueuedMontageBlendingOutEvent(Montage, bInterrupted, OnMontageBlendingOutStarted));

        // 惯性化请求
        if (bShouldInertialize)
        {
            FInertializationRequest Request;
            Request.Duration = InBlendOutSettings.Blend.BlendTime;
            Request.BlendProfile = InBlendOutSettings.BlendProfile;
            AnimInstance->RequestMontageInertialization(Montage, Request);
        }
    }

    // 混合时间为 0 则立即停止
    if (Blend.GetBlendTime() <= 0.0f)
    {
        bPlaying = false;
    }
}
```

### 8.4 权重更新

```cpp
void FAnimMontageInstance::UpdateWeight(float DeltaTime)
{
    if (IsValid())
    {
        // 保存上一帧权重
        PreviousWeight = Blend.GetBlendedValue();
        const bool bWasComplete = Blend.IsComplete();

        // 更新混合权重
        Blend.Update(DeltaTime);

        // 混合完成检测
        if (Blend.GetBlendTimeRemaining() < 0.0001f)
        {
            ActiveBlendProfile = nullptr;
        }

        // BlendedIn 事件
        if (!IsStopped() && !bWasComplete && Blend.IsComplete())
        {
            AnimInstance->QueueMontageBlendedInEvent(
                FQueuedMontageBlendedInEvent(Montage, OnMontageBlendedInEnded));
        }

        // Notify 权重取本帧和上帧的最大值
        // 确保在帧间不丢失触发
        NotifyWeight = FMath::Max(PreviousWeight, Blend.GetBlendedValue());
    }
}
```

---

## 9. Branching Points 分支点

### 9.1 什么是 Branching Point？

```
┌─────────────────────────────────────────────────────────────────┐
│              AnimNotify vs Branching Point                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  普通 AnimNotify:                                                │
│  ├── 在 TickComponent 后的 TriggerAnimNotifies 中触发            │
│  ├── 异步，可能与动画位置有微小偏差                               │
│  └── 适合：音效、特效等对时间不敏感的事件                         │
│                                                                 │
│  Branching Point:                                                │
│  ├── 在 Montage::Advance 中同步处理                              │
│  ├── 精确在指定动画位置触发                                       │
│  ├── 可以安全地修改 Montage 播放状态                              │
│  └── 适合：跳转 Section、触发连击输入窗口等                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 标记为 Branching Point

在 AnimNotify 的属性中，设置 `MontageTickType` 为 `BranchingPoint`：

```cpp
USTRUCT(BlueprintType)
struct FAnimNotifyEvent : public FAnimLinkableElement
{
    // ... 其他成员 ...

    /** Notify 的 Tick 类型 */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Trigger")
    EMontageNotifyTickType MontageTickType;
};

UENUM()
enum class EMontageNotifyTickType : uint8
{
    /** 队列模式 - 在动画更新后异步触发 */
    Queued,
    
    /** 分支点模式 - 在 Montage Advance 中同步触发 */
    BranchingPoint,
};
```

### 9.3 Branching Point 处理流程

```cpp
void FAnimMontageInstance::HandleEvents(float PreviousTrackPos, float CurrentTrackPos, 
    const FBranchingPointMarker* BranchingPointMarker)
{
    if (bInterrupted) return;

    if (AnimInstance.IsValid())
    {
        // 1. 普通 Notify 加入队列
        FAnimNotifyContext NotifyContext(TickRecord);
        Montage->GetAnimNotifiesFromDeltaPositions(PreviousTrackPos, CurrentTrackPos, NotifyContext);
        
        // 过滤掉 Branching Point 类型的 Notify
        Montage->FilterOutNotifyBranchingPoints(NotifyContext.ActiveNotifies);
        AnimInstance->NotifyQueue.AddAnimNotifies(NotifyContext.ActiveNotifies, NotifyWeight);

        // 2. 更新活跃的 State Branching Points
        UpdateActiveStateBranchingPoints(CurrentTrackPos);

        // 3. 处理当前帧的 Branching Point 事件
        if (BranchingPointMarker)
        {
            BranchingPointEventHandler(BranchingPointMarker);
        }
    }
}

void FAnimMontageInstance::BranchingPointEventHandler(const FBranchingPointMarker* BranchingPointMarker)
{
    FAnimNotifyEvent* NotifyEvent = &Montage->Notifies[BranchingPointMarker->NotifyIndex];
    
    if (NotifyEvent)
    {
        if (NotifyEvent->NotifyStateClass != nullptr)
        {
            // State Notify 的 Begin/End 处理
            if (BranchingPointMarker->NotifyEventType == EAnimNotifyEventType::Begin)
            {
                NotifyEvent->NotifyStateClass->BranchingPointNotifyBegin(BranchingPointNotifyPayload);
                ActiveStateBranchingPoints.Add(*NotifyEvent);
            }
            else
            {
                NotifyEvent->NotifyStateClass->BranchingPointNotifyEnd(BranchingPointNotifyPayload);
                ActiveStateBranchingPoints.RemoveSingleSwap(*NotifyEvent);
            }
        }
        else if (NotifyEvent->Notify != nullptr)
        {
            // 普通 Notify 的触发
            NotifyEvent->Notify->BranchingPointNotify(BranchingPointNotifyPayload);
        }
        else
        {
            // 自定义函数调用
            AnimInstance.Get()->TriggerSingleAnimNotify(NotifyEvent);
        }
    }
}
```

---

## 10. Root Motion 与 Montage

### 10.1 Montage 中的 Root Motion 设置

```cpp
class UAnimMontage
{
    /** 是否提取位移 Root Motion */
    UPROPERTY()
    bool bEnableRootMotionTranslation;

    /** 是否提取旋转 Root Motion */
    UPROPERTY()
    bool bEnableRootMotionRotation;

    /** Root 骨骼锁定模式 */
    UPROPERTY()
    TEnumAsByte<ERootMotionRootLock::Type> RootMotionRootLock;
};

namespace ERootMotionRootLock
{
    enum Type
    {
        /** 使用参考姿态的根骨骼位置 */
        RefPose,
        
        /** 使用动画第一帧的根骨骼位置 */
        AnimFirstFrame,
        
        /** 归零 */
        Zero,
    };
}
```

### 10.2 Root Motion 提取

```cpp
FTransform UAnimMontage::ExtractRootMotionFromTrackRange(
    float StartTrackPosition, float EndTrackPosition) const
{
    FRootMotionMovementParams RootMotion;

    // 从第一个 Slot 轨道提取
    if (SlotAnimTracks.Num() > 0)
    {
        const FAnimTrack& SlotAnimTrack = SlotAnimTracks[0].AnimTrack;
        ExtractRootMotionFromTrack(SlotAnimTrack, StartTrackPosition, EndTrackPosition, RootMotion);
    }

    return RootMotion.GetRootMotionTransform();
}
```

### 10.3 Root Motion 与 Character Movement 的配合

```cpp
// 在 FAnimMontageInstance::Advance 中
if (bExtractRootMotion && !IsRootMotionDisabled())
{
    const FTransform RootMotion = Montage->ExtractRootMotionFromTrackRange(
        PreviousSubStepPosition, Position);
    
    if (bBlendRootMotion)
    {
        // 延迟混合 - 等待 Slot 权重更新后再应用
        const float Weight = Blend.GetBlendedValue();
        AnimInstance.Get()->QueueRootMotionBlend(
            RootMotion, Montage->SlotAnimTracks[0].SlotName, Weight);
    }
    else
    {
        // 直接累积
        OutRootMotionParams->Accumulate(RootMotion);
    }
}
```

---

## 11. 动态 Montage 创建

### 11.1 从 AnimSequence 创建动态 Montage

```cpp
/**
 * 运行时从 AnimSequence 创建临时 Montage
 * 常用于简单的技能动画播放
 */
UFUNCTION(BlueprintCallable, Category="Animation|Montage")
UAnimMontage* UAnimInstance::PlaySlotAnimationAsDynamicMontage(
    UAnimSequenceBase* Asset, 
    FName SlotNodeName, 
    float BlendInTime = 0.25f, 
    float BlendOutTime = 0.25f, 
    float InPlayRate = 1.f, 
    int32 LoopCount = 1, 
    float BlendOutTriggerTime = -1.f, 
    float InTimeToStartMontageAt = 0.f);
```

### 11.2 动态创建实现

```cpp
UAnimMontage* UAnimMontage::CreateSlotAnimationAsDynamicMontage_WithBlendSettings(
    UAnimSequenceBase* Asset, 
    FName SlotNodeName, 
    const FMontageBlendSettings& BlendInSettings, 
    const FMontageBlendSettings& BlendOutSettings, 
    float InPlayRate, 
    int32 LoopCount, 
    float InBlendOutTriggerTime)
{
    // 验证资产
    bool bValidAsset = Asset && !Asset->IsA(UAnimMontage::StaticClass());
    if (!bValidAsset)
    {
        UE_LOG(LogAnimMontage, Warning, TEXT("Invalid input asset"));
        return nullptr;
    }

    // 创建临时 Montage
    UAnimMontage* NewMontage = NewObject<UAnimMontage>();
    NewMontage->SetSkeleton(Asset->GetSkeleton());

    // 设置 Slot 轨道
    FSlotAnimationTrack& NewTrack = NewMontage->SlotAnimTracks[0];
    NewTrack.SlotName = SlotNodeName;
    
    // 创建动画段
    FAnimSegment NewSegment;
    NewSegment.SetAnimReference(Asset, true);
    NewSegment.LoopingCount = LoopCount;
    NewMontage->SequenceLength = NewSegment.GetLength();
    NewTrack.AnimTrack.AnimSegments.Add(NewSegment);

    // 创建默认 Section
    FCompositeSection NewSection;
    NewSection.SectionName = TEXT("Default");
    NewSection.Link(Asset, Asset->GetPlayLength());
    NewSection.SetTime(0.0f);
    NewMontage->CompositeSections.Add(NewSection);

    // 设置混合参数
    NewMontage->BlendIn = FAlphaBlend(BlendInSettings.Blend);
    NewMontage->BlendModeIn = BlendInSettings.BlendMode;
    NewMontage->BlendProfileIn = BlendInSettings.BlendProfile;

    NewMontage->BlendOut = FAlphaBlend(BlendOutSettings.Blend);
    NewMontage->BlendModeOut = BlendOutSettings.BlendMode;
    NewMontage->BlendProfileOut = BlendOutSettings.BlendProfile;

    NewMontage->BlendOutTriggerTime = InBlendOutTriggerTime;

    return NewMontage;
}
```

---

## 12. 实践：技能系统中的 Montage

### 12.1 基础技能播放

```cpp
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Combat")
    UAnimMontage* AttackMontage;

    void PlayAttack()
    {
        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && AttackMontage)
        {
            // 播放 Montage
            float Duration = AnimInstance->Montage_Play(AttackMontage, 1.0f);
            
            // 绑定结束回调
            FOnMontageEnded EndDelegate;
            EndDelegate.BindUObject(this, &AMyCharacter::OnAttackEnded);
            AnimInstance->Montage_SetEndDelegate(EndDelegate, AttackMontage);
        }
    }

    void OnAttackEnded(UAnimMontage* Montage, bool bInterrupted)
    {
        if (bInterrupted)
        {
            // 动画被打断
            UE_LOG(LogTemp, Log, TEXT("Attack interrupted!"));
        }
        else
        {
            // 动画正常结束
            UE_LOG(LogTemp, Log, TEXT("Attack finished!"));
        }
    }
};
```

### 12.2 连击系统实现

```cpp
UCLASS()
class AComboCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Combat")
    UAnimMontage* ComboMontage;
    
    // Montage Sections: "Attack1" -> "Attack2" -> "Attack3" -> "End"
    // 默认链接: Attack1 -> End (不连击则直接结束)

    int32 CurrentComboCount = 0;
    bool bCanContinueCombo = false;

    void StartCombo()
    {
        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && ComboMontage)
        {
            CurrentComboCount = 1;
            AnimInstance->Montage_Play(ComboMontage);
            
            // 绑定回调
            FOnMontageEnded EndDelegate;
            EndDelegate.BindUObject(this, &AComboCharacter::OnComboEnded);
            AnimInstance->Montage_SetEndDelegate(EndDelegate, ComboMontage);
        }
    }

    // 由 AnimNotify (Branching Point) 调用
    UFUNCTION(BlueprintCallable)
    void EnableComboInput()
    {
        bCanContinueCombo = true;
    }

    UFUNCTION(BlueprintCallable)
    void DisableComboInput()
    {
        bCanContinueCombo = false;
    }

    void TryContinueCombo()
    {
        if (!bCanContinueCombo || CurrentComboCount >= 3)
            return;

        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance)
        {
            CurrentComboCount++;
            
            // 修改当前 Section 的下一个 Section
            FName CurrentSection = AnimInstance->Montage_GetCurrentSection(ComboMontage);
            FName NextSection = FName(*FString::Printf(TEXT("Attack%d"), CurrentComboCount));
            
            AnimInstance->Montage_SetNextSection(CurrentSection, NextSection, ComboMontage);
            
            bCanContinueCombo = false;
        }
    }

    void OnComboEnded(UAnimMontage* Montage, bool bInterrupted)
    {
        CurrentComboCount = 0;
        bCanContinueCombo = false;
    }
};
```

### 12.3 技能取消与队列

```cpp
UCLASS()
class ASkillCharacter : public ACharacter
{
public:
    TQueue<UAnimMontage*> SkillQueue;
    bool bIsPlayingSkill = false;

    void QueueSkill(UAnimMontage* Skill)
    {
        if (bIsPlayingSkill)
        {
            // 如果正在播放技能，加入队列
            SkillQueue.Enqueue(Skill);
        }
        else
        {
            // 立即播放
            PlaySkillMontage(Skill);
        }
    }

    void PlaySkillMontage(UAnimMontage* Skill)
    {
        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && Skill)
        {
            bIsPlayingSkill = true;
            AnimInstance->Montage_Play(Skill);
            
            FOnMontageEnded EndDelegate;
            EndDelegate.BindUObject(this, &ASkillCharacter::OnSkillEnded);
            AnimInstance->Montage_SetEndDelegate(EndDelegate, Skill);
        }
    }

    void CancelCurrentSkill()
    {
        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && bIsPlayingSkill)
        {
            // 使用快速混合出
            AnimInstance->Montage_Stop(0.1f);
        }
    }

    void OnSkillEnded(UAnimMontage* Montage, bool bInterrupted)
    {
        bIsPlayingSkill = false;
        
        // 检查队列中是否有下一个技能
        UAnimMontage* NextSkill = nullptr;
        if (SkillQueue.Dequeue(NextSkill))
        {
            PlaySkillMontage(NextSkill);
        }
    }
};
```

---

## 📊 总结

### Montage 核心要点

| 概念 | 说明 |
|------|------|
| **Slot** | 在动画图中的插入点，通过名称匹配 |
| **Section** | 可跳转的时间分段，支持运行时修改链接 |
| **Branching Point** | 精确事件触发，同步处理 |
| **BlendIn/Out** | 完整的混合控制，支持 Inertialization |
| **Root Motion** | 从 Montage 提取并应用角色位移 |

### 设计优势

1. **解耦**：动画资源与播放逻辑分离
2. **灵活**：运行时可修改播放流程
3. **精确**：Branching Point 实现帧精确事件
4. **混合**：完善的混合入/出机制

---

## 📖 下一篇预告

**Animation之旅_005：动画通知 (AnimNotify) 系统** - 我们将深入探索：

- `UAnimNotify` 与 `UAnimNotifyState` 的设计
- Notify 的触发机制与时序
- 自定义 Notify 的最佳实践
- Notify 与游戏逻辑的集成

---

## 📚 参考资源

- UE 源码路径：
  - `Engine/Source/Runtime/Engine/Classes/Animation/AnimMontage.h`
  - `Engine/Source/Runtime/Engine/Private/Animation/AnimMontage.cpp`
  - `Engine/Source/Runtime/Engine/Classes/Animation/AnimInstance.h`
- 官方文档：[Animation Montage](https://docs.unrealengine.com/5.0/en-US/animation-montage-in-unreal-engine/)

---

> **作者注**：本文档基于 UE5 源码分析，部分 API 在不同版本可能有差异。建议结合实际源码阅读。
