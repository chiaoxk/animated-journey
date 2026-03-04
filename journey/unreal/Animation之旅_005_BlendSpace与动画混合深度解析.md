# Animation之旅_005：BlendSpace与动画混合深度解析

> 本文深入剖析UE5动画系统中BlendSpace（混合空间）的实现原理，从数学基础到源码实现，带你理解这个强大的多维动画混合工具。

## 一、BlendSpace概述

### 1.1 什么是BlendSpace？

BlendSpace（混合空间）是UE提供的一种**基于参数的多动画混合系统**。它允许开发者通过一个或多个输入参数（如速度、方向），在多个预设的动画样本之间进行平滑插值混合。

**核心价值**：
- 用少量动画样本覆盖大范围的动作状态
- 自动处理动画之间的过渡混合
- 支持运行时参数驱动，实现流畅的姿态变化

### 1.2 BlendSpace类型

UE5提供了以下几种BlendSpace类型：

| 类型 | 维度 | 典型用途 |
|------|------|----------|
| `UBlendSpace1D` | 1D | 移动速度混合（Idle→Walk→Run） |
| `UBlendSpace` | 2D | 移动方向+速度混合 |
| `UAimOffsetBlendSpace` | 2D | 瞄准偏移（上下左右看） |
| `UAimOffsetBlendSpace1D` | 1D | 单轴瞄准偏移 |

### 1.3 类继承关系

```
UAnimationAsset
    └── UBlendSpace (2D混合空间)
            ├── UBlendSpace1D (1D混合空间)
            ├── UAimOffsetBlendSpace (2D瞄准偏移)
            └── UAimOffsetBlendSpace1D (1D瞄准偏移)
```

---

## 二、核心数据结构

### 2.1 FBlendParameter - 轴参数定义

```cpp
USTRUCT()
struct FBlendParameter
{
    GENERATED_BODY()

    // 轴名称（如"Speed"、"Direction"）
    UPROPERTY(EditAnywhere, DisplayName = "Name", Category=BlendParameter)
    FString DisplayName;

    // 最小值
    UPROPERTY(EditAnywhere, DisplayName = "Minimum Axis Value", Category=BlendParameter)
    float Min;

    // 最大值
    UPROPERTY(EditAnywhere, DisplayName = "Maximum Axis Value", Category=BlendParameter)
    float Max;

    // 网格划分数量
    UPROPERTY(EditAnywhere, DisplayName = "Grid Divisions", Category=BlendParameter)
    int32 GridNum;

    // 是否吸附到网格
    UPROPERTY(EditAnywhere, DisplayName = "Snap to Grid", Category = BlendParameter)
    bool bSnapToGrid;

    // 是否循环输入（用于方向轴）
    UPROPERTY(EditAnywhere, DisplayName = "Wrap Input", Category = BlendParameter)
    bool bWrapInput;
    
    // 获取网格大小
    float GetGridSize() const { return GetRange() / (float)GridNum; }
};
```

**关键属性解读**：
- `GridNum`：决定内部网格的精度，影响采样预计算
- `bWrapInput`：当为true时，输入可以从Max回绕到Min（如-180°到180°的方向）

### 2.2 FBlendSample - 动画样本点

```cpp
USTRUCT()
struct FBlendSample
{
    GENERATED_BODY()

    // 关联的动画序列
    UPROPERTY(EditAnywhere, Category=BlendSample)
    TObjectPtr<class UAnimSequence> Animation;

    // 在混合空间中的位置 (X, Y, Z对应3个轴)
    UPROPERTY(EditAnywhere, Category=BlendSample)
    FVector SampleValue;
    
    // 播放速率缩放
    UPROPERTY(EditAnywhere, Category = BlendSample)
    float RateScale = 1.0f;
    
    // 是否使用单帧混合
    UPROPERTY(EditAnywhere, Category = BlendSample)
    bool bUseSingleFrameForBlending = false;
    
    // 单帧混合时使用的帧索引
    UPROPERTY(EditAnywhere, Category = BlendSample)
    uint32 FrameIndexToSample = 0;
    
    // 获取样本播放长度
    ENGINE_API float GetSamplePlayLength() const;
};
```

### 2.3 FBlendSampleData - 运行时采样数据

```cpp
struct FBlendSampleData
{
    // 样本索引（指向SampleData数组）
    int32 SampleDataIndex;
    
    // 该样本的总权重
    float TotalWeight;
    
    // 当前播放时间
    float Time;
    
    // 上一帧播放时间
    float PreviousTime;
    
    // 播放速率
    float SamplePlayRate;
    
    // 关联的动画
    TWeakObjectPtr<UAnimSequence> Animation;
    
    // 逐骨骼混合数据
    TArray<float> PerBoneBlendData;
    
    // 标记同步记录
    FMarkerTickRecord MarkerTickRecord;
    
    // Delta时间记录
    FDeltaTimeRecord DeltaTimeRecord;
};
```

---

## 三、三角化算法详解

BlendSpace的核心在于**如何根据输入参数确定周围样本点及其权重**。对于2D BlendSpace，UE使用**Delaunay三角剖分**算法。

### 3.1 Delaunay三角剖分原理

**Delaunay三角剖分**的核心性质是：任意三角形的外接圆内部不包含其他点。这确保了三角剖分的"最优性"——避免产生过于狭长的三角形。

```
           B
          /|\
         / | \
        /  |  \
       /   |   \
      A----+----C    外接圆内没有其他点D
           |
           D (在圆外)
```

### 3.2 FDelaunayTriangleGenerator - 三角化生成器

```cpp
class FDelaunayTriangleGenerator
{
public:
    enum ECircumCircleState
    {
        ECCS_Outside = -1,  // 点在外接圆外
        ECCS_On = 0,        // 点在外接圆上
        ECCS_Inside = 1,    // 点在外接圆内
    };
    
    // 执行三角化
    void Triangulate(EPreferredTriangulationDirection PreferredTriangulationDirection);
    
    // 添加样本点
    void AddSamplePoint(const FVector2D& NewPoint, const int32 SampleIndex);
    
private:
    // 样本点列表
    TArray<FVertex> SamplePointList;
    
    // 生成的三角形列表
    TArray<FTriangle*> TriangleList;
    
    // 外接圆测试 - Delaunay的核心判断
    ECircumCircleState GetCircumcircleState(const FTriangle* T, const FVertex& TestPoint);
    
    // 是否共线（退化情况）
    bool IsCollinear(const FVertex* A, const FVertex* B, const FVertex* C);
    
    // 翻转两个相邻三角形
    bool FlipTriangles(const int32 TriangleIndexOne, const int32 TriangleIndexTwo);
};
```

### 3.3 外接圆测试 - GetCircumcircleState

这是Delaunay三角剖分的**核心判断函数**，使用行列式计算：

```cpp
ECircumCircleState GetCircumcircleState(const FTriangle* T, const FVertex& TestPoint)
{
    // 首先对所有点进行归一化
    FVector2D NormalizedPositions[3];
    NormalizedPositions[0] = (T->Vertices[0]->Position - GridMin) * RecipGridSize;
    NormalizedPositions[1] = (T->Vertices[1]->Position - GridMin) * RecipGridSize;
    NormalizedPositions[2] = (T->Vertices[2]->Position - GridMin) * RecipGridSize;
    
    const FVector2D NormalizedTestPoint = (TestPoint.Position - GridMin) * RecipGridSize;

    // 计算3x3行列式
    // | Ax-Dx  Ay-Dy  (Ax²+Ay²)-(Dx²+Dy²) |
    // | Bx-Dx  By-Dy  (Bx²+By²)-(Dx²+Dy²) |
    // | Cx-Dx  Cy-Dy  (Cx²+Cy²)-(Dx²+Dy²) |
    
    const double M00 = NormalizedPositions[0].X - NormalizedTestPoint.X;
    const double M01 = NormalizedPositions[0].Y - NormalizedTestPoint.Y;
    const double M02 = NormalizedPositions[0].X * NormalizedPositions[0].X 
                     - NormalizedTestPoint.X * NormalizedTestPoint.X
                     + NormalizedPositions[0].Y * NormalizedPositions[0].Y 
                     - NormalizedTestPoint.Y * NormalizedTestPoint.Y;
    // ... M10, M11, M12, M20, M21, M22 类似计算
    
    // 行列式计算
    const double Det = M00*M11*M22 + M01*M12*M20 + M02*M10*M21 
                     - (M02*M11*M20 + M01*M10*M22 + M00*M12*M21);
    
    if (Det < 0.0)
        return ECCS_Outside;  // 在外接圆外
    else if (FMath::IsNearlyZero(Det))
        return ECCS_On;       // 在外接圆上
    else
        return ECCS_Inside;   // 在外接圆内
}
```

**数学原理**：
当顶点按逆时针排列时：
- Det > 0：测试点在外接圆**内部**
- Det < 0：测试点在外接圆**外部**
- Det = 0：测试点在外接圆**上**

### 3.4 三角化主流程

```cpp
void FDelaunayTriangleGenerator::Triangulate(EPreferredTriangulationDirection PreferredTriangulationDirection)
{
    if (SamplePointList.Num() == 0) return;
    
    // 退化情况：只有1个点
    if (SamplePointList.Num() == 1)
    {
        FTriangle Triangle(&SamplePointList[0]);
        AddTriangle(Triangle);
        return;
    }
    
    // 退化情况：只有2个点
    if (SamplePointList.Num() == 2)
    {
        FTriangle Triangle(&SamplePointList[0], &SamplePointList[1]);
        AddTriangle(Triangle);
        return;
    }
    
    // 正常情况：3个及以上点
    SortSamples();  // 先排序
    
    // 增量式构建三角剖分
    for (int32 I = 2; I < SamplePointList.Num(); ++I)
    {
        GenerateTriangles(SamplePointList, I + 1);
    }
    
    // 处理共线退化情况
    if (TriangleList.Num() == 0)
    {
        if (AllCoincident(SamplePointList))
        {
            // 所有点重合
            FTriangle Triangle(&SamplePointList[0]);
            AddTriangle(Triangle);
        }
        else
        {
            // 所有点共线，创建退化三角形
            for (int32 PointIndex = 0; PointIndex < SamplePointList.Num() - 1; ++PointIndex)
            {
                FTriangle Triangle(&SamplePointList[PointIndex], &SamplePointList[PointIndex + 1]);
                AddTriangle(Triangle);
            }
        }
    }
    
    // 调整边方向以优化混合结果
    AdjustEdgeDirections(PreferredTriangulationDirection);
}
```

### 3.5 三角形翻转优化

当新加入的点导致某个三角形不再满足Delaunay条件时，需要进行**边翻转**：

```cpp
bool FDelaunayTriangleGenerator::FlipTriangles(const int32 TriangleIndexOne, const int32 TriangleIndexTwo)
{
    const FTriangle* A = TriangleList[TriangleIndexOne];
    const FTriangle* B = TriangleList[TriangleIndexTwo];

    // 找到B中不与A共享的点
    FVertex* TestPt = A->FindNonSharingPoint(B);

    // 如果该点在A的外接圆外，则已经是最优，不需要翻转
    if (GetCircumcircleState(A, *TestPt) != ECCS_Inside)
    {
        return false;
    }

    // 尝试构建新的两个三角形
    FTriangle NewTriangles[2];
    int32 TrianglesMade = 0;

    for (int32 VertexIndexOne = 0; VertexIndexOne < 2; ++VertexIndexOne)
    {
        for (int32 VertexIndexTwo = VertexIndexOne + 1; VertexIndexTwo < 3; ++VertexIndexTwo)
        {
            if (IsEligibleForTriangulation(A->Vertices[VertexIndexOne], A->Vertices[VertexIndexTwo], TestPt))
            {
                const FTriangle NewTriangle(A->Vertices[VertexIndexOne], A->Vertices[VertexIndexTwo], TestPt);
                const int32 VertexIndexThree = 3 - (VertexIndexTwo + VertexIndexOne);
                
                // 验证新三角形是否满足Delaunay条件
                if (GetCircumcircleState(&NewTriangle, *A->Vertices[VertexIndexThree]) == ECCS_Outside)
                {
                    NewTriangles[TrianglesMade++] = NewTriangle;
                }
            }
        }
    }
    
    // 成功创建2个新三角形则翻转成功
    if (TrianglesMade == 2)
    {
        AddTriangle(NewTriangles[0], false);
        AddTriangle(NewTriangles[1], false);
        return true;
    }
    
    return false;
}
```

**翻转示意图**：
```
  翻转前:          翻转后:
    A              A
   /|\            / \
  / | \          /   \
 B--+--C   =>   B-----C
  \ | /          \   /
   \|/            \ /
    D              D
```

---

## 四、运行时采样流程

### 4.1 核心函数：GetSamplesFromBlendInput

```cpp
bool UBlendSpace::GetSamplesFromBlendInput(
    const FVector& BlendInput, 
    TArray<FBlendSampleData>& OutSampleDataList, 
    int32& InOutCachedTriangulationIndex, 
    bool bCombineAnimations) const
{
    if (!bInterpolateUsingGrid)
    {
        // 直接使用三角化数据
        TArray<FWeightedBlendSample> WeightedBlendSamples;
        FVector NormalizedBlendInput = GetNormalizedBlendInput(BlendInput);
        
        // 从BlendSpaceData中获取样本权重
        BlendSpaceData.GetSamples(WeightedBlendSamples, DimensionIndices, 
                                   NormalizedBlendInput, InOutCachedTriangulationIndex);

        for (const FWeightedBlendSample& WeightedBlendSample : WeightedBlendSamples)
        {
            if (WeightedBlendSample.SampleWeight > ZERO_ANIMWEIGHT_THRESH)
            {
                FBlendSampleData BlendSampleData;
                BlendSampleData.SampleDataIndex = WeightedBlendSample.SampleIndex;
                BlendSampleData.TotalWeight = WeightedBlendSample.SampleWeight;
                BlendSampleData.Animation = SampleData[WeightedBlendSample.SampleIndex].Animation;
                BlendSampleData.SamplePlayRate = SampleData[WeightedBlendSample.SampleIndex].RateScale;
                OutSampleDataList.Push(BlendSampleData);
            }
        }
    }
    else
    {
        // 使用预计算的网格数据
        TArray<FGridBlendSample, TInlineAllocator<4>> RawGridSamples;
        GetRawSamplesFromBlendInput(BlendInput, RawGridSamples);
        
        // 合并网格样本
        for (const FGridBlendSample& GridSample : RawGridSamples)
        {
            float GridWeight = GridSample.BlendWeight;
            const FEditorElement& GridElement = GridSample.GridElement;

            for (int32 Ind = 0; Ind < GridElement.MAX_VERTICES; ++Ind)
            {
                const int32 SampleDataIndex = GridElement.Indices[Ind];
                if (SampleData.IsValidIndex(SampleDataIndex))
                {
                    int32 Index = OutSampleDataList.AddUnique(SampleDataIndex);
                    FBlendSampleData& NewSampleData = OutSampleDataList[Index];
                    NewSampleData.AddWeight(GridElement.Weights[Ind] * GridWeight);
                    // ...
                }
            }
        }
    }
    
    // 合并相同动画的样本
    if (bCombineAnimations)
    {
        // ... 合并逻辑
    }
    
    // 按权重排序并归一化
    OutSampleDataList.Sort([](const FBlendSampleData& A, const FBlendSampleData& B) 
    { 
        return B.TotalWeight < A.TotalWeight; 
    });
    
    // 移除权重过小的样本
    // ... 归一化权重
    
    return (OutSampleDataList.Num() != 0);
}
```

### 4.2 2D三角化查找 - GetSamples2D

这是**增量式三角形查找**的核心实现：

```cpp
void FBlendSpaceData::GetSamples2D(
    TArray<FWeightedBlendSample>& OutWeightedSamples,
    const TArray<int32>& InDimensionIndices,
    const FVector& InNormalizedSamplePosition,
    int32& InOutTriangleIndex) const
{
    int32 Index0 = InDimensionIndices[0];
    int32 Index1 = InDimensionIndices[1];
    FVector2D P(InNormalizedSamplePosition[Index0], InNormalizedSamplePosition[Index1]);

    // 从缓存的三角形索引开始搜索
    if (InOutTriangleIndex < 0 || InOutTriangleIndex >= Triangles.Num())
    {
        InOutTriangleIndex = Triangles.Num() / 2;  // 从中间开始
    }

    // 增量式搜索
    for (int32 Attempt = 0; Attempt != Triangles.Num(); ++Attempt)
    {
        const FBlendSpaceTriangle* Triangle = &Triangles[InOutTriangleIndex];
        
        // 找到目标点最"外部"的边
        float LargestDistance = UE_KINDA_SMALL_NUMBER;
        int32 LargestEdgeIndex = INDEX_NONE;
        
        for (int32 VertexIndex = 0; VertexIndex != 3; ++VertexIndex)
        {
            FVector2D Corner = Triangle->Vertices[VertexIndex];
            FVector2D EdgeNormal = Triangle->EdgeInfo[VertexIndex].Normal;
            float Distance = (P - Corner) | EdgeNormal;  // 点到边的有向距离
            
            if (Distance > LargestDistance)
            {
                LargestDistance = Distance;
                LargestEdgeIndex = VertexIndex;
            }
        }
        
        if (LargestEdgeIndex < 0)
        {
            // 点在三角形内部！计算重心坐标
            FVector Weights = FMath::GetBaryCentric2D(P, 
                Triangle->Vertices[0], Triangle->Vertices[1], Triangle->Vertices[2]);
            
            OutWeightedSamples.Push(FWeightedBlendSample(Triangle->SampleIndices[0], Weights[0]));
            OutWeightedSamples.Push(FWeightedBlendSample(Triangle->SampleIndices[1], Weights[1]));
            OutWeightedSamples.Push(FWeightedBlendSample(Triangle->SampleIndices[2], Weights[2]));
            return;
        }
        
        // 移动到相邻三角形
        if (Triangle->EdgeInfo[LargestEdgeIndex].NeighbourTriangleIndex >= 0)
        {
            InOutTriangleIndex = Triangle->EdgeInfo[LargestEdgeIndex].NeighbourTriangleIndex;
        }
        else
        {
            // 在边界上，需要沿边界行走找到最近点
            // ... 边界处理逻辑
        }
    }
}
```

### 4.3 重心坐标插值

当确定了包含输入点的三角形后，使用**重心坐标（Barycentric Coordinates）**计算权重：

```cpp
static FVector GetBaryCentric2D(const FVector2D& Point, 
    const FVector2D& A, const FVector2D& B, const FVector2D& C)
{
    // 计算重心坐标
    double a = ((B.Y - C.Y) * (Point.X - C.X) + (C.X - B.X) * (Point.Y - C.Y)) 
             / ((B.Y - C.Y) * (A.X - C.X) + (C.X - B.X) * (A.Y - C.Y));
    
    double b = ((C.Y - A.Y) * (Point.X - C.X) + (A.X - C.X) * (Point.Y - C.Y)) 
             / ((B.Y - C.Y) * (A.X - C.X) + (C.X - B.X) * (A.Y - C.Y));

    return FVector(a, b, 1.0 - a - b);
}
```

**重心坐标的特性**：
- 三个权重之和恒为1
- 每个权重表示该顶点对目标点的贡献
- 当点在顶点上时，对应权重为1，其他为0
- 当点在边上时，只有两个权重非零

---

## 五、输入平滑与过滤

### 5.1 FBlendFilter - 输入滤波器

```cpp
struct FBlendFilter
{
    TArray<FInputFilter> FilterPerAxis;  // 每个轴一个滤波器
};
```

### 5.2 FInterpolationParameter - 插值参数

```cpp
USTRUCT()
struct FInterpolationParameter
{
    GENERATED_BODY()

    // 平滑时间
    UPROPERTY(EditAnywhere, DisplayName = "Smoothing Time")
    float InterpolationTime = 0.f;

    // 阻尼比（仅SpringDamper模式）
    UPROPERTY(EditAnywhere)
    float DampingRatio = 1.f;

    // 最大速度限制
    UPROPERTY(EditAnywhere)
    float MaxSpeed = 0.f;

    // 插值类型
    UPROPERTY(EditAnywhere, DisplayName = "Smoothing Type")
    TEnumAsByte<EFilterInterpolationType> InterpolationType = EFilterInterpolationType::BSIT_SpringDamper;
};
```

### 5.3 插值类型说明

| 类型 | 描述 | 特点 |
|------|------|------|
| `BSIT_Average` | 平均值滤波 | 简单但可能有延迟 |
| `BSIT_Linear` | 线性插值 | 恒定速度到达目标 |
| `BSIT_Cubic` | 三次插值 | 平滑的加减速 |
| `BSIT_Exponential` | 指数衰减 | 快速接近然后缓慢到达 |
| `BSIT_SpringDamper` | 弹簧阻尼器（默认） | 物理感强，可调节阻尼 |

### 5.4 FilterInput - 输入过滤实现

```cpp
FVector UBlendSpace::FilterInput(FBlendFilter* Filter, const FVector& BlendInput, float DeltaTime) const
{
    FVector FilteredBlendInput = BlendInput;
    
    if (Filter)
    {
        for (int32 AxisIndex = 0; AxisIndex < Filter->FilterPerAxis.Num(); ++AxisIndex)
        {
            // 处理循环输入
            if (BlendParameters[AxisIndex].bWrapInput)
            {
                Filter->FilterPerAxis[AxisIndex].WrapToValue(
                    BlendInput[AxisIndex], 
                    BlendParameters[AxisIndex].Max - BlendParameters[AxisIndex].Min);
            }
            
            // 应用滤波
            FilteredBlendInput[AxisIndex] = Filter->FilterPerAxis[AxisIndex]
                .UpdateAndGetFilteredData(BlendInput[AxisIndex], DeltaTime);
        }
    }
    
    return FilteredBlendInput;
}
```

---

## 六、权重平滑过渡

### 6.1 目标权重插值

BlendSpace支持**样本权重的平滑过渡**，避免在样本间突然切换：

```cpp
// UBlendSpace类中
UPROPERTY(EditAnywhere, Category = SampleSmoothing, meta = (DisplayName = "Weight Speed"))
float TargetWeightInterpolationSpeedPerSec = 0.0f;

UPROPERTY(EditAnywhere, Category = SampleSmoothing, meta = (DisplayName = "Smoothing"))
bool bTargetWeightInterpolationEaseInOut = true;
```

### 6.2 InterpolateWeightOfSampleData

```cpp
bool UBlendSpace::InterpolateWeightOfSampleData(
    float DeltaTime, 
    const TArray<FBlendSampleData>& OldSampleDataList, 
    const TArray<FBlendSampleData>& NewSampleDataList, 
    TArray<FBlendSampleData>& FinalSampleDataList) const
{
    for (const FBlendSampleData& OldSample : OldSampleDataList)
    {
        bool bTargetSampleExists = false;
        FBlendSampleData ModifiedSample = OldSample;
        
        // 在新样本中查找相同的样本
        for (const FBlendSampleData& NewSample : NewSampleDataList)
        {
            if (NewSample.SampleDataIndex == OldSample.SampleDataIndex)
            {
                // 找到了，插值到新权重
                FBlendSampleData InterpData = NewSample;
                SmoothWeight(InterpData.TotalWeight, InterpData.WeightRate, 
                            OldSample.TotalWeight, OldSample.WeightRate, 
                            NewSample.TotalWeight, DeltaTime, 
                            TargetWeightInterpolationSpeedPerSec, 
                            bTargetWeightInterpolationEaseInOut);
                
                // 逐骨骼权重插值
                for (int32 BoneIdx = 0; BoneIdx < InterpData.PerBoneBlendData.Num(); ++BoneIdx)
                {
                    SmoothWeight(InterpData.PerBoneBlendData[BoneIdx], 
                                InterpData.PerBoneWeightRate[BoneIdx],
                                OldSample.PerBoneBlendData[BoneIdx], 
                                OldSample.PerBoneWeightRate[BoneIdx], 
                                NewSample.TotalWeight,
                                DeltaTime, 
                                PerBoneBlendValues[BoneIdx].InterpolationSpeedPerSec, 
                                bTargetWeightInterpolationEaseInOut);
                }
                
                if (InterpData.TotalWeight > ZERO_ANIMWEIGHT_THRESH)
                {
                    FinalSampleDataList.Add(InterpData);
                    bTargetSampleExists = true;
                }
                break;
            }
        }
        
        // 如果新样本中没有这个样本，则平滑衰减到0
        if (!bTargetSampleExists)
        {
            FBlendSampleData InterpData = OldSample;
            SmoothWeight(InterpData.TotalWeight, InterpData.WeightRate, 
                        OldSample.TotalWeight, OldSample.WeightRate, 
                        0.0f, DeltaTime, 
                        TargetWeightInterpolationSpeedPerSec, 
                        bTargetWeightInterpolationEaseInOut);
            
            if (InterpData.TotalWeight > ZERO_ANIMWEIGHT_THRESH)
            {
                FinalSampleDataList.Add(InterpData);
            }
        }
    }
    
    // 处理新增的样本（从0平滑增加）
    // ... 
    
    return FinalSampleDataList.Num() > 0;
}
```

### 6.3 权重平滑计算

```cpp
static void SmoothWeight(float& Output, float& OutputRate, 
    float Input, float InputRate, float Target, 
    float DeltaTime, float Speed, bool bUseEaseInOut)
{
    if (Speed <= 0.0f)
    {
        Output = Target;
        return;
    }

    if (bUseEaseInOut)
    {
        // 临界阻尼平滑
        Output = Input;
        OutputRate = InputRate;
        FMath::CriticallyDampedSmoothing(Output, OutputRate, Target, 0.0f, 
            DeltaTime, SmoothingTimeFromSpeed(Speed));
    }
    else
    {
        // 恒定速度插值
        Output = FMath::FInterpConstantTo(Input, Target, DeltaTime, Speed);
    }
}

// Speed到SmoothingTime的转换
static float SmoothingTimeFromSpeed(float Speed)
{
    return Speed > FLT_EPSILON ? 1.0f / (UE_EULERS_NUMBER * Speed) : 0.0f;
}
```

---

## 七、逐骨骼混合

### 7.1 FPerBoneInterpolation - 逐骨骼插值设置

```cpp
USTRUCT()
struct FPerBoneInterpolation
{
    GENERATED_BODY()

    // 骨骼引用
    UPROPERTY(EditAnywhere)
    FBoneReference BoneReference;

    // 该骨骼的权重变化速度
    UPROPERTY(EditAnywhere, meta=(DisplayName="Weight Speed"))
    float InterpolationSpeedPerSec = 6.f;
};
```

### 7.2 GetPerBoneInterpolationIndex

```cpp
int32 UBlendSpace::GetPerBoneInterpolationIndex(
    const FCompactPoseBoneIndex& InCompactPoseBoneIndex, 
    const FBoneContainer& RequiredBones, 
    const IInterpolationIndexProvider::FPerBoneInterpolationData* InData) const
{
    const FSortedPerBoneInterpolationData* Data = 
        static_cast<const FSortedPerBoneInterpolationData*>(InData);
    
    for (int32 Iter = 0; Iter < Data->Data.Num(); ++Iter)
    {
        const FPerBoneInterpolation& PerBoneInterpolation = Data->Data[Iter].PerBoneBlend;
        const FBoneReference& SmoothedBone = PerBoneInterpolation.BoneReference;
        
        FSkeletonPoseBoneIndex SkelBoneIndex = SmoothedBone.GetSkeletonPoseIndex(RequiredBones);
        
        // 骨骼重映射（跨骨架支持）
        if (SkeletonRemapping.IsValid())
        {
            const int32 RemappedIndex = SkeletonRemapping.GetTargetSkeletonBoneIndex(SkelBoneIndex.GetInt());
            SkelBoneIndex = FSkeletonPoseBoneIndex(RemappedIndex);
        }

        const FCompactPoseBoneIndex SmoothedBoneCompactIndex = 
            RequiredBones.GetCompactPoseIndexFromSkeletonPoseIndex(SkelBoneIndex);
        
        // 检查是否匹配或是其子骨骼
        if (SmoothedBoneCompactIndex == InCompactPoseBoneIndex)
        {
            return Data->Data[Iter].OriginalIndex;
        }
        
        if (SmoothedBoneCompactIndex != INDEX_NONE && 
            RequiredBones.BoneIsChildOf(InCompactPoseBoneIndex, SmoothedBoneCompactIndex))
        {
            return Data->Data[Iter].OriginalIndex;
        }
    }
    
    return INDEX_NONE;
}
```

---

## 八、姿态提取与混合

### 8.1 GetAnimationPose - 获取混合姿态

```cpp
void UBlendSpace::GetAnimationPose_Internal(
    TArray<FBlendSampleData>& BlendSampleDataCache, 
    TArrayView<FPoseLink> InPoseLinks,
    FAnimInstanceProxy* InProxy, 
    bool bInExpectsAdditivePose,
    const FAnimExtractContext& ExtractionContext,
    FAnimationPoseData& OutAnimationPoseData) const
{
    if (BlendSampleDataCache.Num() == 0)
    {
        ResetToRefPose(OutAnimationPoseData.GetPose());
        return;
    }

    const int32 NumPoses = BlendSampleDataCache.Num();
    
    // 为每个样本准备姿态容器
    TArray<FCompactPose, TInlineAllocator<8>> ChildrenPoses;
    TArray<FBlendedCurve, TInlineAllocator<8>> ChildrenCurves;
    TArray<UE::Anim::FStackAttributeContainer, TInlineAllocator<8>> ChildrenAttributes;
    TArray<float, TInlineAllocator<8>> ChildrenWeights;
    
    ChildrenPoses.AddZeroed(NumPoses);
    ChildrenCurves.AddZeroed(NumPoses);
    ChildrenAttributes.AddZeroed(NumPoses);
    ChildrenWeights.AddZeroed(NumPoses);
    
    // 初始化
    for (int32 Idx = 0; Idx < NumPoses; ++Idx)
    {
        ChildrenPoses[Idx].SetBoneContainer(&OutAnimationPoseData.GetPose().GetBoneContainer());
        ChildrenCurves[Idx].InitFrom(OutAnimationPoseData.GetCurve());
    }

    // 提取每个样本的姿态
    for (int32 I = 0; I < NumPoses; ++I)
    {
        FCompactPose& Pose = ChildrenPoses[I];

        if (SampleData.IsValidIndex(BlendSampleDataCache[I].SampleDataIndex))
        {
            const FBlendSample& Sample = SampleData[BlendSampleDataCache[I].SampleDataIndex];
            ChildrenWeights[I] = BlendSampleDataCache[I].GetClampedWeight();

            if (InPoseLinks.Num() > 0)
            {
                // 嵌套图模式：评估链接的动画图
                FPoseContext ChildPoseContext(InProxy, bInExpectsAdditivePose);
                InPoseLinks[BlendSampleDataCache[I].SampleDataIndex].Evaluate(ChildPoseContext);
                ChildrenPoses[I] = MoveTemp(ChildPoseContext.Pose);
                ChildrenCurves[I] = MoveTemp(ChildPoseContext.Curve);
                ChildrenAttributes[I] = MoveTemp(ChildPoseContext.CustomAttributes);
            }
            else
            {
                // 直接采样动画序列
                if (Sample.Animation && Sample.Animation->GetSkeleton() != nullptr)
                {
                    const float Time = FMath::Clamp(BlendSampleDataCache[I].Time, 
                                                    0.f, Sample.Animation->GetPlayLength());
                    FAnimationPoseData ChildPoseData = { Pose, ChildrenCurves[I], ChildrenAttributes[I] };
                    Sample.Animation->GetAnimationPose(ChildPoseData, ExtractionContext);
                }
                else
                {
                    ResetToRefPose(Pose);
                }
            }
        }
        else
        {
            ResetToRefPose(Pose);
        }
    }

    // 执行混合
    if (PerBoneBlendValues.Num() > 0)
    {
        if (bAllowMeshSpaceBlending && !bContainsRotationOffsetMeshSpaceSamples)
        {
            // 网格空间逐骨骼混合
            FAnimationRuntime::BlendPosesTogetherPerBoneInMeshSpace(
                ChildrenPoses, ChildrenCurves, ChildrenAttributes,
                this, BlendSampleDataCache, OutAnimationPoseData);
        }
        else
        {
            // 本地空间逐骨骼混合
            FAnimationRuntime::BlendPosesTogetherPerBone(
                ChildrenPoses, ChildrenCurves, ChildrenAttributes,
                this, BlendSampleDataCache, OutAnimationPoseData);
        }
    }
    else
    {
        // 标准混合
        FAnimationRuntime::BlendPosesTogether(
            ChildrenPoses, ChildrenCurves, ChildrenAttributes, 
            ChildrenWeights, OutAnimationPoseData);
    }

    // 归一化旋转
    OutAnimationPoseData.GetPose().NormalizeRotations();
}
```

---

## 九、Aim Offset特殊处理

### 9.1 Aim Offset的特点

Aim Offset是一种特殊的BlendSpace，主要用于角色瞄准时的姿态偏移：

- **必须是Additive动画**：基于基础姿态的偏移
- **通常使用Mesh Space旋转**：确保在世界空间中看起来正确
- **典型参数**：Yaw（偏航）和Pitch（俯仰）

### 9.2 UAimOffsetBlendSpace

```cpp
UCLASS()
class UAimOffsetBlendSpace : public UBlendSpace
{
    GENERATED_UCLASS_BODY()

    virtual bool IsValidAdditiveType(EAdditiveAnimationType AdditiveType) const override
    {
        // 只接受Mesh Space Rotation Additive
        return (AdditiveType == AAT_RotationOffsetMeshSpace);
    }
    
    virtual bool IsValidAdditive() const override 
    { 
        return ContainsMatchingSamples(AAT_RotationOffsetMeshSpace); 
    }
};
```

---

## 十、性能优化建议

### 10.1 样本数量控制

- 2D BlendSpace建议**不超过16个样本点**
- 每个三角形最多3个样本参与混合
- 样本过多会增加内存和序列化开销

### 10.2 使用Grid模式

```cpp
// 启用网格预计算模式
UPROPERTY(EditAnywhere, Category = InputInterpolation, meta = (DisplayName="Use Grid"))
bool bInterpolateUsingGrid = false;
```

- **Grid模式**：编辑器预计算，运行时O(1)查找
- **Triangle模式**：运行时三角形遍历，但利用缓存通常很快

### 10.3 权重平滑设置

```cpp
// 设为0禁用权重平滑，减少每帧计算
TargetWeightInterpolationSpeedPerSec = 0.0f;
```

### 10.4 输入平滑设置

```cpp
// 禁用输入平滑
InterpolationParam[0].InterpolationTime = 0.f;
InterpolationParam[1].InterpolationTime = 0.f;
```

---

## 十一、调试技巧

### 11.1 编辑器中查看三角化

在BlendSpace编辑器中，可以直观看到：
- 三角剖分结果
- 当前采样位置
- 各样本的权重分布

### 11.2 运行时调试命令

```cpp
// 启用BlendSpace调试日志
#define DEBUG_LOG_BLENDSPACE_TRIANGULATION
```

### 11.3 代码调试点

关键断点位置：
1. `GetSamplesFromBlendInput` - 采样入口
2. `GetSamples2D` / `GetSamples1D` - 三角形/线段查找
3. `GetAnimationPose_Internal` - 姿态混合
4. `InterpolateWeightOfSampleData` - 权重过渡

---

## 十二、总结

### 12.1 BlendSpace核心流程

```
1. 输入参数 (Speed, Direction, etc.)
      ↓
2. 输入过滤/平滑 (FilterInput)
      ↓
3. 查找所属三角形 (GetSamples2D)
      ↓
4. 计算重心坐标权重
      ↓
5. 权重平滑过渡 (InterpolateWeightOfSampleData)
      ↓
6. 提取各样本姿态 (GetAnimationPose)
      ↓
7. 姿态混合 (BlendPosesTogether)
      ↓
8. 输出最终姿态
```

### 12.2 关键技术点

| 技术点 | 实现方式 |
|--------|----------|
| 空间划分 | Delaunay三角剖分 |
| 权重计算 | 重心坐标插值 |
| 输入平滑 | 多种滤波器（弹簧阻尼器等） |
| 权重过渡 | 临界阻尼平滑 |
| 逐骨骼混合 | 骨骼层级继承 |
| 姿态混合 | 加权线性混合/网格空间混合 |

### 12.3 最佳实践

1. **合理布局样本点**：均匀分布，避免极端狭长三角形
2. **选择合适的平滑参数**：根据游戏风格调整响应速度
3. **注意Additive类型一致性**：同一BlendSpace中的动画类型要匹配
4. **利用Sync Markers**：确保循环动画正确同步
5. **性能优先时禁用平滑**：在不需要平滑过渡时关闭相关功能

---

## 参考资料

1. UE5源码: `Engine/Source/Runtime/Engine/Private/Animation/BlendSpace.cpp`
2. UE5源码: `Engine/Source/Runtime/Engine/Private/Animation/BlendSpaceHelpers.cpp`
3. UE5源码: `Engine/Source/Runtime/Engine/Classes/Animation/BlendSpace.h`
4. [Delaunay Triangulation - Wikipedia](https://en.wikipedia.org/wiki/Delaunay_triangulation)
5. [Barycentric Coordinates - Wikipedia](https://en.wikipedia.org/wiki/Barycentric_coordinate_system)

---

> 下一篇预告：**Animation之旅_006：骨骼重定向与IK系统** - 将深入探讨骨骼重定向的实现原理，以及各种IK解算器的工作机制。
