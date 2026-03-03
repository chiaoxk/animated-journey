# Animation之旅_007：Control Rig与程序化动画

## 前言

在前几篇文章中，我们探讨了动画序列、动画蓝图、BlendSpace以及IK系统。然而，这些系统主要依赖于预制的动画资源。如果我们需要在运行时动态生成或修改动画，比如让角色精确地拾取桌面上的物品、根据地形调整脚部位置，或者实现复杂的面部表情系统，传统方法就显得力不从心了。

**Control Rig** 正是 Epic Games 为解决这类问题而推出的程序化动画解决方案。它是UE5动画系统的核心组件之一，基于强大的 **RigVM（Rig Virtual Machine）** 构建，提供了可视化的骨骼控制能力。

## 一、Control Rig 架构概览

### 1.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      Control Rig System                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │ URigHierarchy│<-->│ UControlRig │<-->│ URigVM              │  │
│  │  骨骼层级管理 │    │  控制器核心  │    │ 虚拟机执行引擎      │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│         │                  │                      │             │
│         v                  v                      v             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │FRigElement  │    │FRigUnit     │    │FRigVMExecuteContext │  │
│  │ Bone/Control│    │ 操作节点     │    │ 执行上下文           │  │
│  │ Null/Curve  │    │             │    │                     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心类关系

```cpp
// 核心类层次结构
UControlRig : public URigVMHost
    │
    ├── URigHierarchy*  Hierarchy      // 骨骼层级管理
    ├── URigVM*         VM             // RigVM 虚拟机实例
    └── FControlRigExecuteContext      // 执行上下文

URigHierarchy : public UObject
    │
    ├── TArray<FRigBaseElement*>       // 所有元素
    ├── TMap<FRigElementKey, int32>    // 元素索引映射
    └── URigHierarchyController*       // 层级控制器
```

## 二、URigHierarchy 深度解析

`URigHierarchy` 是 Control Rig 的骨骼层级管理核心，它定义了所有可操控的元素。

### 2.1 元素类型体系

```cpp
// Engine\Plugins\Animation\ControlRig\Source\ControlRig\Public\Rigs\RigHierarchy.h

// 元素类型枚举
UENUM()
enum class ERigElementType : uint8
{
    None        = 0,
    Bone        = 1,        // 骨骼元素
    Null        = 2,        // 空元素（辅助定位）
    Control     = 4,        // 控制器元素
    Curve       = 8,        // 曲线元素
    Reference   = 16,       // 引用元素
    Connector   = 32,       // 连接器（模块化Rig）
    Socket      = 64,       // 插槽元素
    All         = Bone | Null | Control | Curve | Reference | Connector | Socket
};

// 元素基类
struct FRigBaseElement
{
    FRigElementKey Key;           // 元素唯一标识
    int32 Index;                  // 在数组中的索引
    int32 SubIndex;               // 子类型索引
    bool bSelected;               // 选中状态
    int32 MetadataStorageIndex;   // 元数据存储索引
    
    virtual ERigElementType GetElementType() const = 0;
};
```

### 2.2 变换元素（Transform Elements）

```cpp
// 可变换元素基类
struct FRigTransformElement : public FRigBaseElement
{
    // 姿态存储 - 包含初始和当前变换
    struct FElementPoseStorage
    {
        struct FPoseStorageByState
        {
            FElementTransformStorage Local;   // 本地空间变换
            FElementTransformStorage Global;  // 全局空间变换
        };
        FPoseStorageByState Initial;    // 初始姿态
        FPoseStorageByState Current;    // 当前姿态
    } PoseStorage;
    
    // 脏标记存储
    struct FElementDirtyState { ... } PoseDirtyState;
    
    // 需要更新的子元素列表
    TArray<FRigTransformElement*> ElementsToDirty;
};
```

### 2.3 控制器元素详解

控制器（Control）是 Control Rig 中最重要的交互元素：

```cpp
// 控制器类型
UENUM()
enum class ERigControlType : uint8
{
    Bool,           // 布尔开关
    Float,          // 单浮点数
    Integer,        // 整数
    Vector2D,       // 2D向量
    Position,       // 3D位置
    Scale,          // 缩放
    Rotator,        // 旋转器
    Transform,      // 完整变换
    TransformNoScale,   // 无缩放变换
    EulerTransform,     // 欧拉变换
};

// 控制器元素
struct FRigControlElement : public FRigTransformElement
{
    UPROPERTY()
    FRigControlSettings Settings;    // 控制器设置
    
    // 偏移变换存储
    FElementPoseStorage OffsetStorage;
    FElementDirtyState OffsetDirtyState;
    
    // 形状变换存储（用于视口显示）
    FElementPoseStorage ShapeStorage;
    FElementDirtyState ShapeDirtyState;
    
    // 优选欧拉角（避免万向锁）
    FElementEulerAnglesStorage PreferredEulerAngles;
};

// 控制器设置
USTRUCT(BlueprintType)
struct FRigControlSettings
{
    UPROPERTY()
    ERigControlType ControlType;     // 控制器类型
    
    UPROPERTY()
    FName DisplayName;               // 显示名称
    
    UPROPERTY()
    bool bAnimatable;                // 是否可动画化
    
    UPROPERTY()
    bool bDrawLimits;                // 是否绘制限制
    
    UPROPERTY()
    FRigControlLimitEnabled LimitEnabled;  // 限制启用状态
    
    UPROPERTY()
    FRigControlValue MinimumValue;   // 最小值限制
    
    UPROPERTY()
    FRigControlValue MaximumValue;   // 最大值限制
    
    UPROPERTY()
    ERigControlVisibility ShapeVisibility;  // 形状可见性
};
```

### 2.4 层级操作 API

```cpp
// URigHierarchy 核心操作方法

// 元素访问
template<typename T>
T* Find(const FRigElementKey& InKey);

template<typename T>
TArray<T*> GetElementsOfType(bool bTraverse = false) const;

// 变换操作
FTransform GetGlobalTransform(const FRigElementKey& InKey) const;
void SetGlobalTransform(const FRigElementKey& InKey, const FTransform& InTransform);

FTransform GetLocalTransform(const FRigElementKey& InKey) const;
void SetLocalTransform(const FRigElementKey& InKey, const FTransform& InTransform);

// 控制器值操作
template<typename T>
T GetControlValue(const FRigElementKey& InKey) const;

template<typename T>
void SetControlValue(const FRigElementKey& InKey, const T& InValue);

// 父子关系
bool SetParent(const FRigElementKey& InChild, const FRigElementKey& InParent);
FRigElementKey GetParent(const FRigElementKey& InChild) const;
TArray<FRigElementKey> GetChildren(const FRigElementKey& InParent) const;

// 姿态管理
void ResetPoseToInitial(ERigElementType InTypeFilter = ERigElementType::All);
FRigPose GetPose(bool bInitial = false) const;
void SetPose(const FRigPose& InPose);
```

## 三、RigVM 虚拟机系统

RigVM 是 Control Rig 的执行引擎，它编译并执行 Rig Graph 中定义的逻辑。

### 3.1 RigVM 架构

```cpp
// Engine\Plugins\Runtime\RigVM\Source\RigVM\Public\RigVMCore\RigVM.h

UCLASS()
class URigVM : public UObject
{
    // 字节码存储
    FRigVMByteCode ByteCodeStorage;
    
    // 函数名称列表
    TArray<FName> FunctionNamesStorage;
    
    // 函数指针数组
    TArray<const FRigVMFunction*> FunctionsStorage;
    
    // 内存存储
    FRigVMMemoryStorageStruct LiteralMemoryStorage;   // 字面量内存
    FRigVMMemoryStorageStruct DefaultWorkMemoryStorage;   // 工作内存
    FRigVMMemoryStorageStruct DefaultDebugMemoryStorage;  // 调试内存
    
    // 指令数组
    FRigVMInstructionArray Instructions;
    
    // 参数
    TArray<FRigVMParameter> Parameters;
    
public:
    // 执行入口
    ERigVMExecuteResult Execute(
        FRigVMExtendedExecuteContext& Context,
        const FName& InEntryName = NAME_None
    );
    
    // 添加函数
    int32 AddRigVMFunction(const FString& InFunctionName);
    
    // 内存管理
    FRigVMMemoryStorageStruct* GetMemoryByType(
        FRigVMExtendedExecuteContext& Context,
        ERigVMMemoryType InMemoryType
    );
};
```

### 3.2 字节码与操作码

```cpp
// Engine\Plugins\Runtime\RigVM\Source\RigVM\Public\RigVMCore\RigVMByteCode.h

// 操作码枚举
UENUM()
enum class ERigVMOpCode : uint8
{
    Execute,            // 执行函数
    Zero,               // 清零寄存器
    BoolFalse,          // 设置 false
    BoolTrue,           // 设置 true
    Copy,               // 复制数据
    Increment,          // 递增
    Decrement,          // 递减
    Equals,             // 相等比较
    NotEquals,          // 不等比较
    JumpAbsolute,       // 绝对跳转
    JumpForward,        // 向前跳转
    JumpBackward,       // 向后跳转
    JumpAbsoluteIf,     // 条件绝对跳转
    JumpForwardIf,      // 条件向前跳转
    JumpBackwardIf,     // 条件向后跳转
    Exit,               // 退出执行
    BeginBlock,         // 开始内存块
    EndBlock,           // 结束内存块
    InvokeEntry,        // 调用入口
    JumpToBranch,       // 跳转到分支
    RunInstructions,    // 懒执行指令集
    SetupTraits,        // 设置特征
    // ... 数组操作（已废弃）
};

// 执行操作
USTRUCT()
struct FRigVMExecuteOp : public FRigVMBaseOp
{
    uint16 FunctionIndex;      // 函数索引
    uint16 ArgumentCount;      // 参数数量
    uint16 FirstPredicateIndex;// 首个谓词索引
    uint16 PredicateCount;     // 谓词数量
};
```

### 3.3 执行上下文

```cpp
// Engine\Plugins\Runtime\RigVM\Source\RigVM\Public\RigVMCore\RigVMExecuteContext.h

USTRUCT(BlueprintType)
struct FRigVMExecuteContext : public FRigVMExecutePin
{
    FName EventName;            // 当前事件名
    FName FunctionName;         // 当前函数名
    int32 InstructionIndex;     // 当前指令索引
    uint32 NumExecutions;       // 执行次数
    double DeltaTime;           // 帧时间
    double AbsoluteTime;        // 绝对时间
    double FramesPerSecond;     // 帧率
    
    FRigVMRuntimeSettings RuntimeSettings;  // 运行时设置
    FRigVMNameCache* NameCache;             // 名称缓存
    FRigVMDrawInterface* DrawInterfacePtr;  // 绘制接口
    FRigVMDrawContainer* DrawContainerPtr;  // 绘制容器
    
    FTransform ToWorldSpaceTransform;       // 到世界空间的变换
    const UObject* OwningObject;            // 拥有者对象
    const AActor* OwningActor;              // 拥有者Actor
    const UWorld* World;                    // 世界指针
    
    TArray<FRigVMTraitScope> Traits;        // 特征作用域
};

// Control Rig 专用执行上下文
USTRUCT(BlueprintType)
struct FControlRigExecuteContext : public FRigVMExecuteContext
{
    FRigUnitContext UnitContext;    // 单元上下文
    URigHierarchy* Hierarchy;       // 骨骼层级
    UControlRig* ControlRig;        // Control Rig 引用
    
    TArray<const UAssetUserData*> AssetUserData;  // 资源用户数据
    
    // 模块化 Rig 支持
    const FRigModuleInstance* GetRigModuleInstance() const;
    const FString& GetRigModulePrefix() const;
};
```

## 四、FRigUnit 节点系统

RigUnit 是 Control Rig Graph 中的基本执行单元。

### 4.1 RigUnit 基类

```cpp
// Engine\Plugins\Animation\ControlRig\Source\ControlRig\Public\Units\RigUnit.h

/** 所有 RigUnit 的基类 */
USTRUCT(BlueprintType, meta=(Abstract, NodeColor="0.1 0.1 0.1", 
    ExecuteContext="FControlRigExecuteContext"))
struct FRigUnit : public FRigVMStruct
{
    GENERATED_BODY()
    
    // 确定指定 Pin 的空间
    virtual FRigElementKey DetermineSpaceForPin(
        const FString& InPinPath, 
        void* InUserContext
    ) const { return FRigElementKey(); }
    
    // 确定指定 Pin 的偏移变换
    virtual FTransform DetermineOffsetTransformForPin(
        const FString& InPinPath, 
        void* InUserContext
    ) const { return FTransform::Identity; }
    
    // 执行方法名
    static FName GetMethodName() { return FRigVMStruct::ExecuteName; }
    
#if WITH_EDITOR
    // 直接操控支持
    virtual bool GetDirectManipulationTargets(...) const;
    virtual bool UpdateHierarchyForDirectManipulation(...);
    virtual bool UpdateDirectManipulationFromHierarchy(...);
#endif
};

/** 可变 RigUnit 基类（可修改数据） */
USTRUCT(BlueprintType, meta=(Abstract))
struct FRigUnitMutable : public FRigUnit
{
    GENERATED_BODY()
    
    // 执行引脚（用于串联执行流）
    UPROPERTY(DisplayName="Execute", Transient, meta=(Input, Output))
    FRigVMExecutePin ExecutePin;
};
```

### 4.2 常用内置 RigUnit

#### 变换操作节点

```cpp
// 获取变换
USTRUCT(meta=(DisplayName="Get Transform"))
struct FRigUnit_GetTransform : public FRigUnit
{
    RIGVM_METHOD()
    virtual void Execute() override;
    
    UPROPERTY(meta=(Input))
    FRigElementKey Item;
    
    UPROPERTY(meta=(Input))
    ERigVMTransformSpace Space = ERigVMTransformSpace::GlobalSpace;
    
    UPROPERTY(meta=(Output))
    FTransform Transform;
};

// 设置变换
USTRUCT(meta=(DisplayName="Set Transform"))
struct FRigUnit_SetTransform : public FRigUnitMutable
{
    RIGVM_METHOD()
    virtual void Execute() override;
    
    UPROPERTY(meta=(Input))
    FRigElementKey Item;
    
    UPROPERTY(meta=(Input))
    ERigVMTransformSpace Space = ERigVMTransformSpace::GlobalSpace;
    
    UPROPERTY(meta=(Input))
    FTransform Transform;
    
    UPROPERTY(meta=(Input))
    float Weight = 1.f;
    
    UPROPERTY(meta=(Input))
    bool bPropagateToChildren = true;
};
```

#### IK 解算节点

```cpp
// Two Bone IK 解算
USTRUCT(meta=(DisplayName="Two Bone IK"))
struct FRigUnit_TwoBoneIKSimple : public FRigUnitMutable
{
    RIGVM_METHOD()
    virtual void Execute() override;
    
    UPROPERTY(meta=(Input))
    FRigElementKey BoneA;    // 上臂/大腿
    
    UPROPERTY(meta=(Input))
    FRigElementKey BoneB;    // 前臂/小腿
    
    UPROPERTY(meta=(Input))
    FRigElementKey EffectorBone;  // 手/脚
    
    UPROPERTY(meta=(Input))
    FTransform Effector;     // 目标变换
    
    UPROPERTY(meta=(Input))
    FVector PrimaryAxis = FVector(1.f, 0.f, 0.f);
    
    UPROPERTY(meta=(Input))
    FVector SecondaryAxis = FVector(0.f, 1.f, 0.f);
    
    UPROPERTY(meta=(Input))
    float PoleVectorKind;    // 极向量类型
    
    UPROPERTY(meta=(Input))
    FVector PoleVector;      // 极向量
    
    UPROPERTY(meta=(Input))
    bool bEnableStretch = false;
    
    UPROPERTY(meta=(Input))
    float StretchMaximum = 1.25f;
};

// FABRIK 解算
USTRUCT(meta=(DisplayName="FABRIK"))
struct FRigUnit_FABRIK : public FRigUnitMutable
{
    RIGVM_METHOD()
    virtual void Execute() override;
    
    UPROPERTY(meta=(Input))
    FRigElementKey RootBone;
    
    UPROPERTY(meta=(Input))
    FRigElementKey EffectorBone;
    
    UPROPERTY(meta=(Input))
    FTransform EffectorTransform;
    
    UPROPERTY(meta=(Input))
    float Precision = 0.1f;
    
    UPROPERTY(meta=(Input))
    int32 MaxIterations = 10;
    
    UPROPERTY(meta=(Input))
    bool bPropagateToChildren = true;
};
```

#### 控制器操作节点

```cpp
// 获取控制器值
USTRUCT(meta=(DisplayName="Get Control Float"))
struct FRigUnit_GetControlFloat : public FRigUnit
{
    RIGVM_METHOD()
    virtual void Execute() override;
    
    UPROPERTY(meta=(Input))
    FName Control;
    
    UPROPERTY(meta=(Output))
    float FloatValue;
    
    UPROPERTY(meta=(Output))
    float Minimum;
    
    UPROPERTY(meta=(Output))
    float Maximum;
};

// 设置控制器值
USTRUCT(meta=(DisplayName="Set Control Transform"))
struct FRigUnit_SetControlTransform : public FRigUnitMutable
{
    RIGVM_METHOD()
    virtual void Execute() override;
    
    UPROPERTY(meta=(Input))
    FName Control;
    
    UPROPERTY(meta=(Input))
    FTransform Transform;
    
    UPROPERTY(meta=(Input))
    ERigVMTransformSpace Space = ERigVMTransformSpace::GlobalSpace;
};
```

### 4.3 自定义 RigUnit

创建自定义节点：

```cpp
// MyProject/Source/MyRigUnits/MyCustomRigUnit.h

#pragma once
#include "Units/RigUnit.h"
#include "MyCustomRigUnit.generated.h"

/**
 * 自定义 RigUnit 示例：计算两点间的中点
 */
USTRUCT(meta=(DisplayName="Calculate Midpoint", Category="My Custom"))
struct MYPROJECT_API FRigUnit_CalculateMidpoint : public FRigUnit
{
    GENERATED_BODY()
    
    // 执行逻辑
    RIGVM_METHOD()
    virtual void Execute() override
    {
        // 获取两个骨骼的世界位置
        if (ExecuteContext.Hierarchy)
        {
            FTransform TransformA = ExecuteContext.Hierarchy->GetGlobalTransform(BoneA);
            FTransform TransformB = ExecuteContext.Hierarchy->GetGlobalTransform(BoneB);
            
            // 计算中点
            FVector LocationA = TransformA.GetLocation();
            FVector LocationB = TransformB.GetLocation();
            
            Midpoint = (LocationA + LocationB) * 0.5f;
            
            // 插值旋转
            FQuat RotA = TransformA.GetRotation();
            FQuat RotB = TransformB.GetRotation();
            MidpointRotation = FQuat::Slerp(RotA, RotB, 0.5f).Rotator();
        }
    }
    
    // 输入参数
    UPROPERTY(meta=(Input, ExpandByDefault))
    FRigElementKey BoneA;
    
    UPROPERTY(meta=(Input, ExpandByDefault))
    FRigElementKey BoneB;
    
    // 输出参数
    UPROPERTY(meta=(Output))
    FVector Midpoint;
    
    UPROPERTY(meta=(Output))
    FRotator MidpointRotation;
};

/**
 * 自定义可变 RigUnit：应用噪声到骨骼
 */
USTRUCT(meta=(DisplayName="Apply Noise to Bone", Category="My Custom"))
struct MYPROJECT_API FRigUnit_ApplyNoiseToBone : public FRigUnitMutable
{
    GENERATED_BODY()
    
    RIGVM_METHOD()
    virtual void Execute() override
    {
        if (!ExecuteContext.Hierarchy)
            return;
            
        // 获取当前变换
        FTransform CurrentTransform = ExecuteContext.Hierarchy->GetGlobalTransform(Bone);
        
        // 生成 Perlin 噪声
        float Time = ExecuteContext.GetAbsoluteTime();
        float NoiseX = FMath::PerlinNoise1D(Time * Frequency) * Amplitude;
        float NoiseY = FMath::PerlinNoise1D(Time * Frequency + 100.f) * Amplitude;
        float NoiseZ = FMath::PerlinNoise1D(Time * Frequency + 200.f) * Amplitude;
        
        // 应用噪声偏移
        FVector NoiseOffset(NoiseX, NoiseY, NoiseZ);
        FVector NewLocation = CurrentTransform.GetLocation() + NoiseOffset;
        
        CurrentTransform.SetLocation(NewLocation);
        
        // 设置新变换
        ExecuteContext.Hierarchy->SetGlobalTransform(Bone, CurrentTransform);
    }
    
    UPROPERTY(meta=(Input, ExpandByDefault))
    FRigElementKey Bone;
    
    UPROPERTY(meta=(Input))
    float Amplitude = 1.0f;
    
    UPROPERTY(meta=(Input))
    float Frequency = 1.0f;
};
```

## 五、UControlRig 核心类

### 5.1 类定义

```cpp
// Engine\Plugins\Animation\ControlRig\Source\ControlRig\Public\ControlRig.h

UCLASS(Blueprintable, Abstract)
class CONTROLRIG_API UControlRig : public URigVMHost
{
    GENERATED_UCLASS_BODY()
    
public:
    // 骨骼层级
    UPROPERTY()
    TObjectPtr<URigHierarchy> DynamicHierarchy;
    
    // 层级访问器
    URigHierarchy* GetHierarchy();
    const URigHierarchy* GetHierarchy() const;
    
    // 执行
    virtual bool Execute(const FName& InEventName) override;
    
    // 初始化
    virtual void Initialize(bool bRequestInit = true) override;
    
    // 事件
    void ExecuteForwardSolve(const FName& InEventName);
    void ExecuteBackwardsSolve(const FName& InEventName);
    void ExecuteConstruction(const FName& InEventName);
    
    // 控制器操作
    template<typename T>
    T GetControlValue(const FName& InControlName) const;
    
    template<typename T>
    void SetControlValue(const FName& InControlName, const T& InValue);
    
    // 数据绑定
    void SetBoneInitialTransformsFromSkeletalMesh(USkeletalMesh* InSkeletalMesh);
    void SetBoneInitialTransformsFromRefSkeleton(const FReferenceSkeleton& InReferenceSkeleton);
    
    // 预览网格
    UPROPERTY()
    TSoftObjectPtr<USkeletalMesh> PreviewSkeletalMesh;
    
    // 事件委托
    FControlRigExecuteEvent InitializedEvent;
    FControlRigExecuteEvent PreConstructionEvent;
    FControlRigExecuteEvent PostConstructionEvent;
    FControlRigExecuteEvent PreForwardsSolveEvent;
    FControlRigExecuteEvent PostForwardsSolveEvent;
    
protected:
    // 层级初始化
    virtual void InitializeFromCDO() override;
    void CreateRigControlsForCurveContainer();
    
    // 执行上下文
    FControlRigExecuteContext ExecuteContext;
    
    // 对象绑定
    TSharedPtr<IControlRigObjectBinding> ObjectBinding;
};
```

### 5.2 执行流程

```cpp
bool UControlRig::Execute(const FName& InEventName)
{
    // 1. 检查是否可以执行
    if (!CanExecute())
    {
        return false;
    }
    
    // 2. 设置执行上下文
    ExecuteContext.Hierarchy = GetHierarchy();
    ExecuteContext.ControlRig = this;
    ExecuteContext.SetEventName(InEventName);
    ExecuteContext.SetDeltaTime(DeltaTime);
    ExecuteContext.SetAbsoluteTime(AbsoluteTime);
    
    // 3. 调用 Pre 事件
    PreForwardsSolveEvent.Broadcast(this, InEventName);
    
    // 4. 执行 VM
    if (VM)
    {
        VM->Execute(GetRigVMExtendedExecuteContext(), InEventName);
    }
    
    // 5. 调用 Post 事件
    PostForwardsSolveEvent.Broadcast(this, InEventName);
    
    // 6. 传播变换到绑定对象
    if (ObjectBinding.IsValid())
    {
        ObjectBinding->PropagateTransforms();
    }
    
    return true;
}
```

## 六、Control Rig 在动画蓝图中的使用

### 6.1 动画蓝图节点

```cpp
// Control Rig 动画节点
USTRUCT(BlueprintInternalUseOnly)
struct CONTROLRIG_API FAnimNode_ControlRig : public FAnimNode_CustomProperty
{
    GENERATED_BODY()
    
    // Control Rig 类
    UPROPERTY(EditAnywhere, Category=ControlRig)
    TSubclassOf<UControlRig> ControlRigClass;
    
    // Control Rig 实例
    UPROPERTY(Transient)
    TObjectPtr<UControlRig> ControlRig;
    
    // 混合权重
    UPROPERTY(EditAnywhere, Category=Settings, meta=(PinShownByDefault))
    float Alpha = 1.0f;
    
    // 执行
    virtual void Evaluate_AnyThread(FPoseContext& Output) override
    {
        DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(Evaluate_AnyThread)
        
        // 从输入姿态初始化骨骼
        if (ControlRig && ControlRig->GetHierarchy())
        {
            // 转换骨骼数据
            TransferInputPose(Output.Pose, ControlRig->GetHierarchy());
            
            // 执行 Control Rig
            ControlRig->SetDeltaTime(Output.AnimInstanceProxy->GetDeltaSeconds());
            ControlRig->Execute(FRigUnit_BeginExecution::EventName);
            
            // 转换输出
            TransferOutputPose(ControlRig->GetHierarchy(), Output.Pose, Alpha);
        }
        else
        {
            // 直接输出输入姿态
            FAnimNode_Base::Evaluate_AnyThread(Output);
        }
    }
    
protected:
    void TransferInputPose(const FCompactPose& InPose, URigHierarchy* OutHierarchy);
    void TransferOutputPose(URigHierarchy* InHierarchy, FCompactPose& OutPose, float InAlpha);
};
```

### 6.2 蓝图使用示例

```cpp
// 在动画蓝图中配置 Control Rig 节点

// 1. AnimGraph 中添加 Control Rig 节点
// 2. 设置 ControlRigClass 为你的 Control Rig 蓝图
// 3. 连接输入姿态和输出姿态

// EventGraph 中可以这样控制：
void UMyAnimInstance::BlueprintUpdateAnimation(float DeltaSeconds)
{
    if (UControlRig* CR = GetControlRig())
    {
        // 设置控制器值
        CR->SetControlValue<FVector>(TEXT("IK_Foot_L"), LeftFootTarget);
        CR->SetControlValue<FVector>(TEXT("IK_Foot_R"), RightFootTarget);
        
        // 设置控制器旋转
        CR->SetControlValue<FRotator>(TEXT("Head_Rotation"), HeadRotation);
    }
}
```

## 七、实战案例：程序化脚部 IK

### 7.1 场景设置

```cpp
// FootIKControlRig.h

UCLASS(Blueprintable)
class UFootIKControlRig : public UControlRig
{
    GENERATED_BODY()
    
public:
    // 事件名称
    static const FName FootIKEventName;
    
    virtual void Initialize(bool bRequestInit) override
    {
        Super::Initialize(bRequestInit);
        
        // 设置初始骨骼层级
        if (URigHierarchy* H = GetHierarchy())
        {
            // 这些通常在 Rig Graph 中配置
            // 这里展示编程方式
        }
    }
    
    // 脚部 IK 目标设置
    UFUNCTION(BlueprintCallable, Category="Foot IK")
    void SetFootIKTargets(
        const FVector& LeftFootLocation,
        const FVector& RightFootLocation,
        const FRotator& LeftFootRotation,
        const FRotator& RightFootRotation
    );
    
    // 获取脚部信息
    UFUNCTION(BlueprintPure, Category="Foot IK")
    void GetFootInfo(
        FVector& OutLeftFootLocation,
        FVector& OutRightFootLocation
    ) const;
};
```

### 7.2 地面检测组件

```cpp
// GroundTraceComponent.h

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class UGroundTraceComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Ground Trace")
    float TraceDistance = 100.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Ground Trace")
    float FootOffset = 5.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Ground Trace")
    TEnumAsByte<ECollisionChannel> TraceChannel = ECC_Visibility;
    
    // 执行地面追踪
    UFUNCTION(BlueprintCallable, Category="Ground Trace")
    bool TraceGround(
        const FVector& FootLocation,
        FVector& OutHitLocation,
        FVector& OutHitNormal
    ) const
    {
        if (!GetWorld())
            return false;
            
        FVector Start = FootLocation + FVector(0, 0, TraceDistance);
        FVector End = FootLocation - FVector(0, 0, TraceDistance);
        
        FHitResult HitResult;
        FCollisionQueryParams QueryParams;
        QueryParams.AddIgnoredActor(GetOwner());
        
        bool bHit = GetWorld()->LineTraceSingleByChannel(
            HitResult,
            Start,
            End,
            TraceChannel,
            QueryParams
        );
        
        if (bHit)
        {
            OutHitLocation = HitResult.Location + FVector(0, 0, FootOffset);
            OutHitNormal = HitResult.Normal;
        }
        
        return bHit;
    }
    
    // 计算脚部旋转
    UFUNCTION(BlueprintPure, Category="Ground Trace")
    FRotator CalculateFootRotation(const FVector& HitNormal, const FRotator& CurrentRotation) const
    {
        FVector FootForward = CurrentRotation.Vector();
        FVector FootRight = FVector::CrossProduct(HitNormal, FootForward).GetSafeNormal();
        FVector AdjustedForward = FVector::CrossProduct(FootRight, HitNormal).GetSafeNormal();
        
        return FRotationMatrix::MakeFromXZ(AdjustedForward, HitNormal).Rotator();
    }
};
```

### 7.3 整合使用

```cpp
// Character 中整合 Control Rig 和地面检测

void AMyCharacter::UpdateFootIK(float DeltaTime)
{
    if (!ControlRigComponent || !GroundTraceComponent)
        return;
        
    UControlRig* CR = ControlRigComponent->GetControlRig();
    if (!CR)
        return;
        
    // 获取骨骼网格组件位置
    USkeletalMeshComponent* Mesh = GetMesh();
    
    // 获取原始脚部位置
    FVector LeftFootLoc = Mesh->GetSocketLocation(TEXT("foot_l"));
    FVector RightFootLoc = Mesh->GetSocketLocation(TEXT("foot_r"));
    
    // 执行地面追踪
    FVector LeftHitLoc, LeftHitNormal;
    FVector RightHitLoc, RightHitNormal;
    
    bool bLeftHit = GroundTraceComponent->TraceGround(LeftFootLoc, LeftHitLoc, LeftHitNormal);
    bool bRightHit = GroundTraceComponent->TraceGround(RightFootLoc, RightHitLoc, RightHitNormal);
    
    // 计算脚部旋转
    FRotator LeftFootRot = FRotator::ZeroRotator;
    FRotator RightFootRot = FRotator::ZeroRotator;
    
    if (bLeftHit)
    {
        LeftFootRot = GroundTraceComponent->CalculateFootRotation(
            LeftHitNormal, 
            Mesh->GetSocketRotation(TEXT("foot_l"))
        );
    }
    
    if (bRightHit)
    {
        RightFootRot = GroundTraceComponent->CalculateFootRotation(
            RightHitNormal,
            Mesh->GetSocketRotation(TEXT("foot_r"))
        );
    }
    
    // 平滑插值
    CurrentLeftFootLoc = FMath::VInterpTo(CurrentLeftFootLoc, 
        bLeftHit ? LeftHitLoc : LeftFootLoc, DeltaTime, InterpSpeed);
    CurrentRightFootLoc = FMath::VInterpTo(CurrentRightFootLoc,
        bRightHit ? RightHitLoc : RightFootLoc, DeltaTime, InterpSpeed);
    
    CurrentLeftFootRot = FMath::RInterpTo(CurrentLeftFootRot, LeftFootRot, DeltaTime, InterpSpeed);
    CurrentRightFootRot = FMath::RInterpTo(CurrentRightFootRot, RightFootRot, DeltaTime, InterpSpeed);
    
    // 设置 Control Rig 控制器值
    CR->SetControlValue<FVector>(TEXT("IK_Foot_L"), CurrentLeftFootLoc);
    CR->SetControlValue<FVector>(TEXT("IK_Foot_R"), CurrentRightFootLoc);
    CR->SetControlValue<FRotator>(TEXT("IK_FootRot_L"), CurrentLeftFootRot);
    CR->SetControlValue<FRotator>(TEXT("IK_FootRot_R"), CurrentRightFootRot);
    
    // 计算骨盆偏移（使角色整体下沉以适应地面）
    float LeftOffset = LeftFootLoc.Z - CurrentLeftFootLoc.Z;
    float RightOffset = RightFootLoc.Z - CurrentRightFootLoc.Z;
    float PelvisOffset = FMath::Min(LeftOffset, RightOffset);
    
    CR->SetControlValue<float>(TEXT("Pelvis_Offset"), PelvisOffset);
}
```

## 八、模块化 Rig（Modular Rig）

UE5.4 引入了模块化 Rig 系统，允许创建可复用的 Rig 模块。

### 8.1 模块定义

```cpp
// 模块设置
USTRUCT()
struct FRigModuleSettings
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere)
    FString Identifier;           // 模块标识符
    
    UPROPERTY(EditAnywhere)
    FString DisplayName;          // 显示名称
    
    UPROPERTY(EditAnywhere)
    FString Category;             // 分类
    
    UPROPERTY(EditAnywhere)
    FString Keywords;             // 关键词
    
    UPROPERTY(EditAnywhere)
    FString Description;          // 描述
    
    bool IsValidModule() const
    {
        return !Identifier.IsEmpty();
    }
};

// 模块实例
struct FRigModuleInstance
{
    FString Name;                 // 实例名称
    FString ParentPath;           // 父模块路径
    TWeakObjectPtr<UControlRig> ModuleControlRig;  // 模块 Control Rig
    
    // 连接器映射
    TMap<FRigElementKey, FRigElementKey> ConnectorMapping;
};
```

### 8.2 连接器系统

```cpp
// 连接器元素
struct FRigConnectorElement : public FRigBaseElement
{
    UPROPERTY()
    FRigConnectorSettings Settings;
    
    // 连接类型
    ERigConnectorType GetConnectorType() const { return Settings.Type; }
};

// 连接器设置
USTRUCT()
struct FRigConnectorSettings
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere)
    ERigConnectorType Type = ERigConnectorType::Primary;
    
    UPROPERTY(EditAnywhere)
    bool bOptional = false;
    
    UPROPERTY(EditAnywhere)
    TArray<FRigConnectorMatchRule> Rules;  // 匹配规则
};

// 连接器类型
UENUM()
enum class ERigConnectorType : uint8
{
    Primary,      // 主连接器（必须连接）
    Secondary,    // 次要连接器（可选）
};
```

## 九、性能优化建议

### 9.1 编译时优化

```cpp
// Control Rig 编译设置
USTRUCT()
struct FRigVMCompileSettings
{
    UPROPERTY(EditAnywhere)
    bool bEnableCompression = true;      // 启用字节码压缩
    
    UPROPERTY(EditAnywhere)
    bool bEnableBatchExecution = true;   // 启用批量执行
    
    UPROPERTY(EditAnywhere) 
    int32 MaxCallStackDepth = 128;       // 最大调用栈深度
    
    UPROPERTY(EditAnywhere)
    bool bInlineNodes = true;            // 内联简单节点
};
```

### 9.2 运行时优化

```cpp
// 最佳实践
class UOptimizedControlRig : public UControlRig
{
public:
    // 1. 缓存元素键
    virtual void Initialize(bool bRequestInit) override
    {
        Super::Initialize(bRequestInit);
        
        // 缓存常用元素键
        CachedPelvisKey = FRigElementKey(TEXT("Pelvis"), ERigElementType::Bone);
        CachedSpineKeys.Reserve(5);
        for (int32 i = 0; i < 5; ++i)
        {
            CachedSpineKeys.Add(FRigElementKey(
                *FString::Printf(TEXT("Spine_%d"), i),
                ERigElementType::Bone
            ));
        }
    }
    
    // 2. 批量获取变换
    void GetMultipleTransforms(TArray<FTransform>& OutTransforms)
    {
        URigHierarchy* H = GetHierarchy();
        if (!H) return;
        
        OutTransforms.Reset(CachedSpineKeys.Num());
        for (const FRigElementKey& Key : CachedSpineKeys)
        {
            OutTransforms.Add(H->GetGlobalTransform(Key));
        }
    }
    
    // 3. 使用脏标记避免不必要的更新
    void ConditionalUpdate()
    {
        if (!bNeedsUpdate)
            return;
            
        // 执行更新
        Execute(ForwardSolveEventName);
        
        bNeedsUpdate = false;
    }
    
private:
    FRigElementKey CachedPelvisKey;
    TArray<FRigElementKey> CachedSpineKeys;
    bool bNeedsUpdate = false;
};
```

### 9.3 LOD 配置

```cpp
// Control Rig LOD 设置
USTRUCT()
struct FControlRigLODSettings
{
    UPROPERTY(EditAnywhere)
    int32 LODLevel = 0;
    
    UPROPERTY(EditAnywhere)
    bool bDisableIK = false;
    
    UPROPERTY(EditAnywhere)
    bool bDisableFacialAnimation = false;
    
    UPROPERTY(EditAnywhere)
    float UpdateInterval = 0.0f;  // 0 = 每帧更新
};

// 在 Tick 中使用
void AMyCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // 根据距离计算 LOD
    float Distance = GetDistanceToCamera();
    int32 LOD = CalculateLOD(Distance);
    
    // 根据 LOD 调整 Control Rig 行为
    if (LOD >= 2)
    {
        // 降低更新频率
        ControlRigUpdateTimer += DeltaTime;
        if (ControlRigUpdateTimer < LODSettings[LOD].UpdateInterval)
            return;
        ControlRigUpdateTimer = 0.0f;
    }
    
    // 执行 Control Rig
    if (ControlRigComponent)
    {
        ControlRigComponent->GetControlRig()->Execute(ForwardSolveEventName);
    }
}
```

## 十、总结

### 核心要点回顾

1. **URigHierarchy** - 管理骨骼、控制器、空元素等层级结构
2. **URigVM** - 基于字节码的虚拟机执行引擎
3. **FRigUnit** - Rig Graph 中的基本执行单元
4. **UControlRig** - 整合所有组件的核心控制类
5. **模块化 Rig** - UE5.4 引入的可复用 Rig 组件系统

### Control Rig vs 传统动画

| 特性 | 传统动画 | Control Rig |
|------|----------|-------------|
| 数据来源 | 预制动画资源 | 程序化生成 |
| 灵活性 | 固定 | 高度可定制 |
| 运行时调整 | 有限 | 完全支持 |
| 性能开销 | 低 | 较高 |
| 适用场景 | 通用动画 | IK、物理模拟、程序化 |

### 下一步学习方向

- **Animation之旅_008**：动画通知系统（AnimNotify）深度解析
- **Animation之旅_009**：物理动画与 Ragdoll 系统
- **番外篇**：Motion Matching 与 Pose Search

## 参考资料

- Unreal Engine 5.6 源码：`Engine/Plugins/Animation/ControlRig/`
- Unreal Engine 5.6 源码：`Engine/Plugins/Runtime/RigVM/`
- [UE5 Control Rig 官方文档](https://docs.unrealengine.com/5.0/en-US/control-rig-in-unreal-engine/)
- [RigVM 技术规范](https://docs.unrealengine.com/5.0/en-US/rigvm-in-unreal-engine/)
