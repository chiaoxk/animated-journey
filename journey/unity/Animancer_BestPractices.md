# Animancer 最佳实践文档

## 一、概述

Animancer 是一个强大的 Unity 动画系统，用于替代传统的 Animator Controller。它提供了更直观的代码驱动动画控制方式，支持动画混合、层级控制、状态机等高级功能。

### 核心优势

| 特性 | 传统 Animator | Animancer |
|------|--------------|-----------|
| 动画控制 | 状态机可视化编辑 | 代码直接控制 |
| 动态切换 | 需要参数触发 | 直接 Play() |
| 混合树 | Blend Tree | MixerState |
| 调试 | 状态机窗口 | Inspector 实时预览 |
| 性能 | 固定开销 | 按需创建状态 |

---

## 二、核心组件

### 2.1 AnimancerComponent

**作用**: Animancer 的主入口组件，挂载到需要播放动画的 GameObject 上。

```csharp
// 获取组件
AnimancerComponent animancer = GetComponent<AnimancerComponent>();

// 基本配置
animancer.Animator = GetComponent<Animator>();  // 关联 Animator
animancer.ActionOnDisable = AnimancerComponent.DisableAction.Pause;  // 禁用时行为
animancer.UpdateMode = AnimatorUpdateMode.Normal;  // 更新模式
```

**最佳实践**:
```csharp
[RequireComponent(typeof(Animator))]
[RequireComponent(typeof(AnimancerComponent))]
public class CharacterAnimation : MonoBehaviour
{
    [SerializeField] private Animator animator;
    [SerializeField] private AnimancerComponent animancer;

    private void Awake()
    {
        // 确保 Animator 没有 Controller（Animancer 接管动画）
        if (animator.runtimeAnimatorController != null)
        {
            Debug.LogWarning("移除 AnimatorController，使用 Animancer 控制动画");
            animator.runtimeAnimatorController = null;
        }
    }
}
```

---

### 2.2 AnimancerState

**作用**: 代表一个正在播放或可以播放的动画状态。

```csharp
// 播放动画并获取状态
AnimancerState state = animancer.Play(clip);

// 状态属性
state.Time;           // 当前时间（秒）
state.NormalizedTime; // 归一化时间（0-1）
state.Length;         // 动画长度
state.Duration;       // 考虑速度后的实际时长
state.Speed;          // 播放速度
state.Weight;         // 混合权重
state.IsPlaying;      // 是否正在播放
state.IsLooping;      // 是否循环
```

---

### 2.3 AnimancerGraph

**作用**: 管理整个动画图的核心类。

```csharp
AnimancerGraph graph = animancer.Graph;

// 图控制
graph.Speed = 1.5f;           // 全局速度
graph.IsGraphPlaying = true;  // 播放/暂停整个图
graph.PauseGraph();           // 暂停
graph.UnpauseGraph();         // 恢复
graph.Evaluate();             // 手动更新一帧
graph.Destroy();              // 销毁
```

---

## 三、播放动画

### 3.1 基础播放

```csharp
// 方式1：直接播放 AnimationClip
AnimancerState state = animancer.Play(clip);

// 方式2：带淡入时长播放
AnimancerState state = animancer.Play(clip, fadeDuration: 0.25f);

// 方式3：指定淡入模式
AnimancerState state = animancer.Play(clip, 0.25f, FadeMode.FromStart);
```

### 3.2 FadeMode 淡入模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `FixedSpeed` | 固定淡入速度 | 默认模式，自然过渡 |
| `FixedDuration` | 固定淡入时长 | 精确控制过渡时间 |
| `FromStart` | 从头开始播放 | 需要重置动画时间 |
| `NormalizedFromStart` | 归一化从头开始 | 循环动画同步 |

```csharp
// 推荐：大多数情况使用 FromStart，确保动画从头播放
animancer.Play(clip, 0.1f, FadeMode.FromStart);
```

### 3.3 使用 Transition（推荐）

**Transition** 封装了动画播放的所有参数，便于配置和复用。

```csharp
[Serializable]
public class ClipTransition : ITransition
{
    [SerializeField] private AnimationClip _Clip;
    [SerializeField] private float _Speed = 1;
    [SerializeField] private float _FadeDuration = 0.25f;
    // ...
}

// 在 Inspector 中配置
[SerializeField] private ClipTransition idleTransition;
[SerializeField] private ClipTransition walkTransition;

// 播放
animancer.Play(idleTransition);
```

**最佳实践**：使用 ScriptableObject 管理 Transition
```csharp
[CreateAssetMenu(menuName = "Animation/Transition Asset")]
public class TransitionAsset : ScriptableObject
{
    public ClipTransition transition;
}
```

---

## 四、动画事件

### 4.1 使用 Events 系统

```csharp
// 方式1：在归一化时间点添加事件
AnimancerState state = animancer.Play(clip);
state.Events(this).Add(0.5f, () => {
    Debug.Log("动画播放到一半");
});

// 方式2：动画结束事件
state.Events(this).OnEnd = () => {
    Debug.Log("动画播放完成");
    // 切换到下一个动画
    animancer.Play(nextClip);
};

// 方式3：清除事件（重要！避免事件累积）
state.Events(this).Clear();
```

### 4.2 事件最佳实践

```csharp
private void PlayActionAnimation(AnimationClip clip, Action onComplete)
{
    var state = animancer.Play(clip, 0.1f, FadeMode.FromStart);
    
    // 清除之前的事件
    state.Events(this).Clear();
    
    // 非循环动画才添加结束事件
    if (!clip.isLooping)
    {
        // 使用 exitTime 而不是 OnEnd，可以提前触发
        float exitTime = 1f - 0.08f; // 提前 0.08 秒
        state.Events(this).Add(exitTime, () => {
            onComplete?.Invoke();
        });
    }
}
```

---

## 五、动画混合器 (Mixer)

### 5.1 LinearMixerState（1D 混合）

用于基于单一参数（如速度）混合多个动画。

```csharp
// 创建线性混合器
var mixer = new LinearMixerState();

// 添加子动画和阈值
mixer.Add(idleClip, 0f);      // 速度 0 时播放 idle
mixer.Add(walkClip, 1f);      // 速度 1 时播放 walk
mixer.Add(runClip, 2f);       // 速度 2 时播放 run

// 播放混合器
animancer.Play(mixer);

// 通过参数控制混合
mixer.Parameter = currentSpeed;  // 根据当前速度自动混合
```

**使用 Transition 配置**:
```csharp
[SerializeField] private LinearMixerTransition locomotionMixer;

void Start()
{
    animancer.Play(locomotionMixer);
}

void Update()
{
    locomotionMixer.State.Parameter = moveSpeed;
}
```

### 5.2 CartesianMixerState（2D 笛卡尔混合）

用于基于二维参数（如移动方向）混合动画。

```csharp
// 创建 2D 混合器
var mixer = new CartesianMixerState();

// 添加方向动画
mixer.Add(idleClip, new Vector2(0, 0));       // 静止
mixer.Add(frontClip, new Vector2(0, 1));      // 前进
mixer.Add(backClip, new Vector2(0, -1));      // 后退
mixer.Add(leftClip, new Vector2(-1, 1));      // 左前
mixer.Add(rightClip, new Vector2(1, 1));      // 右前

// 播放
animancer.Play(mixer);

// 通过方向参数控制
mixer.Parameter = new Vector2(inputX, inputY);
```

**项目实际使用示例**（来自 CharacterAnim.cs）：
```csharp
private void InitMixerStates()
{
    var mixerState = new CartesianMixerState();
    
    // 初始化各方向动画
    InitClipState(mixerState, moveClips, ECharacterMoveAnim.Entry, new Vector2(0, 0));
    InitClipState(mixerState, moveClips, ECharacterMoveAnim.MoveLeft, new Vector2(-1, 1));
    InitClipState(mixerState, moveClips, ECharacterMoveAnim.MoveFront, new Vector2(0, 1));
    InitClipState(mixerState, moveClips, ECharacterMoveAnim.MoveRight, new Vector2(1, 1));
    InitClipState(mixerState, moveClips, ECharacterMoveAnim.MoveBack, new Vector2(0, -1));
    
    mixerStates[key] = mixerState;
}

private void Update()
{
    // 平滑更新方向参数
    currentDirection.x = Mathf.SmoothDamp(currentDirection.x, targetDirection.x, 
        ref velocity.x, smoothTime);
    currentDirection.y = Mathf.SmoothDamp(currentDirection.y, targetDirection.y, 
        ref velocity.y, smoothTime);
    
    if (currentAnimancerState is Vector2MixerState mixerState)
    {
        mixerState.Parameter = currentDirection;
    }
}
```

### 5.3 ManualMixerState（手动混合）

完全手动控制各子动画权重。

```csharp
var mixer = new ManualMixerState();
mixer.Add(clip1);
mixer.Add(clip2);

// 手动设置权重
mixer.GetChild(0).Weight = 0.7f;
mixer.GetChild(1).Weight = 0.3f;
```

---

## 六、混合器深度解析

### 6.1 混合器类继承体系

```
                        AnimancerState
                              │
                              ▼
                      ManualMixerState
                              │
                              ▼
                    MixerState<TParameter>
                     ╱                  ╲
                    ╱                    ╲
                   ▼                      ▼
        MixerState<float>          MixerState<Vector2>
               │                          │
               ▼                          ▼
       LinearMixerState            Vector2MixerState
                                    ╱           ╲
                                   ╱             ╲
                                  ▼               ▼
                       CartesianMixerState   DirectionalMixerState
```

**各类职责**：

| 类名 | 职责 |
|------|------|
| `ManualMixerState` | 基础混合器，手动控制子状态权重 |
| `MixerState<T>` | 泛型参数混合器，根据参数和阈值自动计算权重 |
| `LinearMixerState` | 1D 线性混合（类似 Mecanim 的 1D Blend Tree） |
| `Vector2MixerState` | 2D 混合基类，支持 X/Y 双参数 |
| `CartesianMixerState` | 2D 笛卡尔混合（类似 Mecanim 的 2D Freeform Cartesian） |
| `DirectionalMixerState` | 2D 方向混合（类似 Mecanim 的 2D Freeform Directional） |

---

### 6.2 混合器核心数据结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MixerState<T>                               │
├─────────────────────────────────────────────────────────────────────┤
│  核心字段:                                                          │
│  ├── _Parameter: TParameter        // 当前混合参数值                │
│  ├── _Thresholds: TParameter[]     // 各子状态的阈值数组            │
│  └── WeightsAreDirty: bool         // 权重是否需要重算              │
│                                                                     │
│  继承自 ManualMixerState:                                           │
│  ├── ChildStates: AnimancerState[] // 子动画状态数组                │
│  ├── _ChildCount: int              // 子状态数量                    │
│  └── _SynchronizedChildren: List   // 需要同步时间的子状态          │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 6.3 混合器工作原理

#### 6.3.1 权重计算流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                        权重计算流程                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 设置 Parameter 值                                                │
│     │                                                                │
│     ▼                                                                │
│  2. SetWeightsDirty()                                                │
│     ├── WeightsAreDirty = true                                       │
│     └── Graph.RequirePreUpdate(this)  // 注册到预更新列表            │
│     │                                                                │
│     ▼                                                                │
│  3. 下一帧 Update() 被调用                                           │
│     │                                                                │
│     ▼                                                                │
│  4. RecalculateWeights()                                             │
│     ├── if (!WeightsAreDirty) return                                 │
│     ├── ForceRecalculateWeights()  // 子类实现具体算法               │
│     └── WeightsAreDirty = false                                      │
│     │                                                                │
│     ▼                                                                │
│  5. 权重应用到 Playable                                              │
│     └── Playable.SetChildWeight(state, weight)                       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

#### 6.3.2 LinearMixerState 权重计算算法

```csharp
// LinearMixerState.ForceRecalculateWeights() 核心逻辑
protected override void ForceRecalculateWeights()
{
    var parameter = Parameter;
    
    // 遍历所有子状态，找到参数所在的阈值区间
    for (int i = 0; i < childCount; i++)
    {
        var threshold = GetThreshold(i);
        var nextThreshold = GetThreshold(i + 1);
        
        if (parameter > threshold && parameter <= nextThreshold)
        {
            // 线性插值计算权重
            var t = (parameter - threshold) / (nextThreshold - threshold);
            Playable.SetChildWeight(state[i], 1 - t);
            Playable.SetChildWeight(state[i + 1], t);
            // 其他状态权重设为 0
            break;
        }
    }
    
    // ExtrapolateSpeed: 参数超过最大阈值时加速播放
    if (ExtrapolateSpeed && parameter > maxThreshold)
    {
        _Playable.SetSpeed(Speed * (parameter / maxThreshold));
    }
}
```

**示意图**：
```
Parameter: 1.5
Thresholds: [0, 1, 2]  (idle, walk, run)

      idle(0)     walk(1)      run(2)
        │           │            │
        ├───────────┼────────────┤
                    │     ↑      
                    │    1.5     
                    │            
Weight:  0%        50%          50%

结果: walk 和 run 各占 50% 权重
```

#### 6.3.3 CartesianMixerState 权重计算算法

CartesianMixerState 使用 **Gradient Band Interpolation** 算法：

```csharp
// CartesianMixerState.ForceRecalculateWeights() 核心逻辑
protected override void ForceRecalculateWeights()
{
    // 预计算混合因子 (只在阈值变化时计算一次)
    CalculateBlendFactors(childCount);
    
    for (int i = 0; i < childCount; i++)
    {
        var threshold = GetThreshold(i);
        var thresholdToParameter = Parameter - threshold;
        
        float weight = 1;
        
        // 对比所有其他状态，取最小权重
        for (int j = 0; j < childCount; j++)
        {
            if (j == i) continue;
            
            var newWeight = 1 - Vector2.Dot(thresholdToParameter, blendFactors[i][j]);
            if (weight > newWeight)
                weight = newWeight;
        }
        
        weights[i] = Mathf.Max(weight, 0);
    }
    
    // 归一化权重并应用
    NormalizeAndApplyWeights(totalWeight, weights);
}
```

**示意图**：
```
      (-1,1)────────(0,1)────────(1,1)
         │  左前     │  前进     │  右前
         │           │            │
         │     ●Parameter(0.3, 0.7)
         │           │            │
      (-1,0)────────(0,0)────────(1,0)
         │  左       │  静止     │  右
         │           │            │
      (-1,-1)───────(0,-1)──────(1,-1)
                     │  后退

当 Parameter = (0.3, 0.7) 时:
- 前进(0,1) 权重最高
- 右前(1,1) 次之
- 其他权重接近 0
```

---

## 七、AnimancerGraph 深度解析

### 7.1 核心字段

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            AnimancerGraph                                    │
│               (管理整个动画图的核心类，包含 Layers 和 States)                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  核心字段:                                                                  │
│  ├── Layers: AnimancerLayer[]      // 图中的所有层                         │
│  ├── States: AnimancerStateDictionary  // 状态字典                         │
│  ├── Playable: Playable            // 根 Playable（可以是动画轨道）        │
│  ├── DeltaTime: float              // 每帧时间（用于计算速度等）           │
│  └── MustBeUpdated: bool           // 标记是否需要更新                     │
│                                                                             │
│  设计目的:                                                                  │
│  管理动画图的所有层级、状态和转换，负责整个动画系统的更新和控制。          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 更新流程

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        AnimancerGraph 更新流程                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   调用 Graph.Update()                                                    │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────┐                                          │
│   │ 1. 遍历所有层             │                                          │
│   │    foreach (layer)        │                                          │
│   │    └─ layer.Update()      │                                          │
│   └──────────────────────────┘                                          │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────┐                                          │
│   │ 2. 更新根 Playable        │                                          │
│   │    Playable.Evaluate()    │                                          │
│   └──────────────────────────┘                                          │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────┐                                          │
│   │ 3. 重置 MustBeUpdated     │                                          │
│   │    MustBeUpdated = false  │                                          │
│   └──────────────────────────┘                                          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### 7.3 状态查找流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AnimancerState 的查找流程                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. States.Get(key)                                                        │
│        ├─? 如果存在，返回状态                                               │
│        └─? 否则，创建新状态并返回                                           │
│                                                                             │
│   2. 状态缓存                                                               │
│        ├─? 存储在 States 字典中                                             │
│        └─? 可以通过 Key 访问                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.4 状态与可播放体（Playable）

状态封装了一个可播放体（Playable），并通过该可播放体与 Unity 的动画系统接口。
可播放体的创建、更新和销毁由状态管理，而状态的激活与去激活则由图（Graph）管理。

---

## 八、调试技巧

### 8.1 AnimancerDebug

`AnimancerDebug` 提供了一种方便的方式来查看和修改运行时的 Animancer 状态。

```csharp
// 在需要的地方添加调试代码
void OnDrawGizmos()
{
    AnimancerDebug.DrawAllGraphs();
}

// 注意：过多的 Gizmos 调试信息可能影响性能，发布时请移除
```

### 8.2 Inspector 调试

在运行时选中带有 AnimancerComponent 的物体，可以在 Inspector 中：
- 查看当前播放的状态和权重
- 查看图结构
- 手动修改参数进行测试

---

## 九、AnimancerState 深度解析

### 9.1 类继承体系总览

```
                              AnimancerNodeBase
                                     │
                                     │ (基础节点，管理 Playable)
                                     │
                                     ▼
                              AnimancerNode
                                     │
                                     │ (添加 Weight/Fade/IK 等功能)
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
       AnimancerLayer         AnimancerState          (其他扩展)
              │                      │
              │      ┌───────────────┼───────────────┐
              │      │               │               │
              │      ▼               ▼               ▼
              │  ClipState   ManualMixerState  ControllerState
              │                      │
              │      ┌───────────────┴───────────────┐
              │      │                               │
              │      ▼                               ▼
              │  MixerState<float>          MixerState<Vector2>
              │      │                               │
              │      ▼                               ▼
              │  LinearMixerState           Vector2MixerState
              │                                      │
              │                        ┌─────────────┴─────────────┐
              │                        │                           │
              │                        ▼                           ▼
              │              CartesianMixerState        DirectionalMixerState
              │
              └──────────────────────────────────────────────────────────────────┘
```

---

### 9.2 基类职责详解

#### 9.2.1 AnimancerNodeBase

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AnimancerNodeBase                                    │
│                       (所有节点的最基础类)                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  核心职责:                                                                  │
│  ├── 管理 Playable 实例                                                    │
│  ├── 维护 Graph 引用                                                       │
│  ├── 维护 Parent 父节点引用                                                │
│  └── 定义 Speed 播放速度                                                   │
│                                                                             │
│  核心属性:                                                                  │
│  ├── Graph: AnimancerGraph        // 所属的动画图                          │
│  ├── Parent: AnimancerNodeBase    // 父节点                                │
│  ├── Layer: AnimancerLayer        // 所属层（向上查找）                    │
│  ├── Playable: Playable           // 内部 Playable 对象                    │
│  ├── Speed: float                 // 播放速度                              │
│  └── EffectiveSpeed: float        // 实际速度（乘以所有父节点速度）        │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 9.2.2 AnimancerNode

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            AnimancerNode                                     │
│                    (AnimancerLayer 和 AnimancerState 的基类)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  继承: AnimancerNodeBase                                                    │
│                                                                             │
│  新增职责:                                                                  │
│  ├── 权重管理 (Weight)                                                     │
│  ├── 淡入淡出 (Fade)                                                       │
│  ├── 子节点管理 (Children)                                                 │
│  └── IK 设置继承                                                           │
│                                                                             │
│  核心属性:                                                                  │
│  ├── Index: int                   // 在父节点中的索引                      │
|  ├── Weight: float                // 混合权重 (0-1)                        │
|  ├── TargetWeight: float          // Fade 目标权重                         │
|  ├── FadeSpeed: float             // Fade 速度                             │
|  └── EffectiveWeight: float       // 实际权重（乘以所有父节点权重）        │
│                                                                             │
│  核心方法:                                                                  │
│  ├── SetWeight(float)             // 设置权重                              │
│  ├── StartFade(target, duration)  // 开始淡入淡出                          │
│  ├── CancelFade()                 // 取消淡入淡出                          │
│  └── Stop()                       // 停止                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 9.2.3 AnimancerState

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AnimancerState                                     │
│                         (所有动画状态的基类)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  继承: AnimancerNode                                                        │
│                                                                             │
│  核心职责:                                                                  │
│  ├── 管理动画播放状态                                                      │
│  ├── 时间控制 (Time, NormalizedTime)                                       │
│  ├── 事件系统 (Events)                                                     │
│  └── 活跃状态管理 (IsActive)                                               │
│                                                                             │
│  核心属性:                                                                  │
│  ├── Key: object                  // 状态字典中的注册键                    │
│  ├── IsPlaying: bool              // 时间是否自动推进                      │
│  ├── Time: float                  // 当前时间（秒）                        │
│  ├── NormalizedTime: float        // 归一化时间（0-1）                     │
│  ├── Length: float                // 动画长度（抽象，子类实现）            │
│  ├── IsLooping: bool              // 是否循环                              │
|  ├── Duration: float              // 考虑速度后的实际时长                  │
|  ├── RemainingDuration: float     // 剩余时长                              │
|  └── AverageVelocity: Vector3     // 根运动平均速度                        │
│                                                                             │
│  事件相关:                                                                  │
│  ├── SharedEvents: AnimancerEvent.Sequence  // 共享事件序列                │
│  ├── OwnedEvents: AnimancerEvent.Sequence   // 自有事件序列                │
│  └── NormalizedEndTime: float     // 结束事件触发时间                      │
│                                                                             │
│  核心方法:                                                                  │
│  ├── Play()                       // 立即播放（Weight=1, IsPlaying=true）  │
│  ├── Stop()                       // 停止并重置                            │
│  ├── Destroy()                    // 销毁状态                              │
│  ├── SetGraph(graph)              // 设置所属图                            │
│  ├── SetParent(node)              // 设置父节点                            │
│  ├── MoveTime(time, normalized)   // 移动时间（触发事件和根运动）          │
│  └── Clone(context)               // 克隆状态（抽象）                      │
│                                                                             │
│  设计目的:                                                                  │
│  作为所有具体动画状态的抽象基类，定义动画状态的通用接口和行为。            │
│  提供时间控制、事件触发、活跃状态管理等核心功能。                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.3 具体状态类详解

#### 9.3.1 ClipState

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ClipState                                       │
│                      (播放单个 AnimationClip)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  继承: AnimancerState                                                       │
│  Playable 类型: AnimationClipPlayable                                       │
│                                                                             │
│  核心职责: 封装 AnimationClipPlayable，播放单个 AnimationClip               │
│                                                                             │
│  使用场景:                                                                  │
│  ├── 播放普通动画片段                                                      │
│  ├── 作为 Mixer 的子状态                                                   │
│  └── 最常用的状态类型                                                      │
│                                                                             │
│  示例:                                                                      │
│  var state = animancer.Play(clip);  // 自动创建 ClipState                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 9.3.2 ControllerState

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ControllerState                                     │
│                   (包装 AnimatorController)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  继承: AnimancerState                                                       │
│  Playable 类型: AnimatorControllerPlayable                                  │
│                                                                             │
│  使用场景:                                                                  │
│  ├── 迁移现有项目时保留原有 Animator Controller                            │
│  └── 混合使用 Animancer 和 Controller                                      │
│                                                                             │
│  注意事项:                                                                  │
│  ├── 不推荐新项目使用（违背 Animancer 设计理念）                           │
│  └── 性能可能不如纯 Animancer                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 9.3.3 PlayableAssetState

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PlayableAssetState                                    │
│                      (播放 PlayableAsset)                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  继承: AnimancerState                                                       │
│                                                                             │
│  使用场景:                                                                  │
│  ├── 播放 Timeline 资产                                                    │
│  └── 播放自定义 PlayableAsset                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.4 状态生命周期

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AnimancerState 生命周期                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 创建阶段                                                                │
│     ├── new ClipState(clip) 或 animancer.Play(clip)                        │
│     └── Graph.States.Register(state)     // 注册到字典                     │
│                                                                             │
│  2. 连接阶段                                                                │
│     ├── state.SetGraph(graph)            // 设置所属图                     │
│     └── state.SetParent(layer)           // 连接到层                       │
│                                                                             │
│  3. 播放阶段                                                                │
│     ├── layer.Play(state)                // 开始播放                       │
│     │   ├─ state.IsPlaying = true                                          │
│     │   └─ state.Weight = 1                                                │
│     └── 每帧更新时间、事件、淡入淡出                                        │
│                                                                             │
│  4. 停止阶段                                                                │
│     ├── state.Stop()                                                       │
│     │   ├── Weight = 0                                                     │
│     │   ├── IsPlaying = false                                              │
│     │   └── Time = 0                                                       │
│     └── Playable 可能断开连接（取决于 KeepChildrenConnected）              │
│                                                                             │
│  5. 销毁阶段                                                                │
│     └── state.Destroy() → 销毁 Playable，从字典注销                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9.5 状态类选择指南

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         状态类选择决策树                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                        需要播放什么？                                        │
│                              │                                              │
│         ┌────────────────────┼────────────────────┐                        │
│         │                    │                    │                        │
│         ▼                    ▼                    ▼                        │
│   单个动画片段          多个动画混合       AnimatorController              │
│         │                    │                    │                        │
│         ▼                    │                    ▼                        │
│     ClipState               │              ControllerState                 │
│                              │                                              │
│              ┌───────────────┴───────────────┐                             │
│              │                               │                             │
│              ▼                               ▼                             │
│        自动混合参数                    手动控制权重                        │
│              │                               │                             │
│    ┌─────────┴─────────┐                    ▼                             │
│    │                   │              ManualMixerState                     │
│    ▼                   ▼                                                   │
│  1D参数             2D参数                                                 │
│    │                   │                                                   │
│    ▼                   ▼                                                   │
│ LinearMixerState     混合类型？                                            │
│                          │                                                 │
│               ┌──────────┴──────────┐                                      │
│               │                     │                                      │
│               ▼                     ▼                                      │
│          XY坐标位置            方向+强度                                   │
│               │                     │                                      │
│               ▼                     ▼                                      │
│      CartesianMixerState   DirectionalMixerState                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 十、最佳实践总结

### 10.1 状态复用

```csharp
// ? 好：通过 Key 复用状态
var state = animancer.States.GetOrCreate(clip);  // 自动复用
animancer.Play(state);

// ? 避免：每次创建新状态
var state = new ClipState(clip);  // 手动创建时注意管理生命周期
```

### 10.2 Mixer 预创建

```csharp
// ? 好：预创建 Mixer，避免运行时分配
private LinearMixerState _locomotionMixer;

void Awake()
{
    _locomotionMixer = new LinearMixerState();
    _locomotionMixer.Add(idleClip, 0f);
    _locomotionMixer.Add(walkClip, 1f);
    _locomotionMixer.Add(runClip, 2f);
}

void OnEnable() => animancer.Play(_locomotionMixer);
void Update() => _locomotionMixer.Parameter = speed;
```

### 10.3 事件管理

```csharp
// ? 好：播放前清理事件
var state = animancer.Play(clip);
state.Events(this).Clear();
state.Events(this).Add(0.9f, OnAnimationEnd);

// ? 避免：重复添加事件导致多次触发
state.Events(this).Add(0.9f, OnAnimationEnd);  // 不清理会累积
```

### 10.4 资源销毁

```csharp
void OnDestroy()
{
    // ? 始终销毁 Graph
    if (animancer.Graph.IsValid())
        animancer.Graph.Destroy();
}
```

---

## 附录：常用 API 速查

| 操作 | API |
|------|-----|
| 播放动画 | `animancer.Play(clip)` |
| 带淡入播放 | `animancer.Play(clip, 0.25f)` |
| 从头播放 | `animancer.Play(clip, 0.25f, FadeMode.FromStart)` |
| 停止动画 | `state.Stop()` |
| 暂停图 | `animancer.Graph.PauseGraph()` |
| 恢复图 | `animancer.Graph.UnpauseGraph()` |
| 设置速度 | `state.Speed = 2f` |
| 设置时间 | `state.NormalizedTime = 0.5f` |
| 添加事件 | `state.Events(this).Add(0.5f, callback)` |
| 清理事件 | `state.Events(this).Clear()` |
| 销毁图 | `animancer.Graph.Destroy()` |

---

## 相关文档

- [Playable API 深度解析](./Playable_API_Guide.md) - Unity Playable API 的底层原理
