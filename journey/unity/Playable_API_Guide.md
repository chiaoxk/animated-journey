# Unity Playable API 深度解析

## 一、Playable API 概述

Unity Playable API 是 Unity 新一代动画系统的底层架构，设计目标是提供一个**灵活、可扩展、高性能**的图结构来处理动画、音频和脚本逻辑。

### 1.1 为什么需要 Playable API？

**传统 Animator Controller 的局限性：**

| 问题 | 描述 |
|------|------|
| 静态结构 | 状态机必须在编辑器中预先定义，运行时无法动态修改 |
| 有限的混合能力 | 只能使用预定义的 Blend Tree，无法自定义混合算法 |
| 难以与其他系统集成 | 与 Timeline、物理、IK 等系统的集成较为困难 |
| 性能开销 | 即使状态不活跃，整个状态机仍然会被评估 |
| 代码控制有限 | 主要依赖参数驱动，难以实现复杂的程序化动画 |

**Playable API 的解决方案：**

| 优势 | 描述 |
|------|------|
| 动态图结构 | 运行时可以任意创建、连接、断开节点 |
| 自定义混合 | 可以编写自定义 Playable 实现任意混合逻辑 |
| 统一接口 | 动画、音频、脚本使用相同的 Playable 架构 |
| 按需评估 | 只有连接到输出的活跃节点才会被评估 |
| 完全代码控制 | 所有操作都可以通过代码完成，支持程序化动画 |

---

## 二、核心概念架构

### 2.1 整体架构图

```
                              PlayableGraph
                    (整个动画系统的容器/管理器)
                                    │
                                    ▼
    ┌───────────────────────────────────────────────────────┐
    │                   PlayableOutput                       │
    │          (将 Playable 输出到具体目标)                  │
    │   ┌─────────────────────────────────────────────────┐ │
    │   │  AnimationPlayableOutput → Animator             │ │
    │   │  AudioPlayableOutput     → AudioSource          │ │
    │   │  ScriptPlayableOutput    → 脚本/通知            │ │
    │   └─────────────────────────────────────────────────┘ │
    └───────────────────────────────────────────────────────┘
                                    │
                                    │ GetSourcePlayable()
                                    ▼
    ┌───────────────────────────────────────────────────────┐
    │                     Playable (节点)                    │
    │   ┌─────────────────────────────────────────────────┐ │
    │   │  AnimationClipPlayable    - 播放单个动画片段    │ │
    │   │  AnimationMixerPlayable   - 混合多个动画        │ │
    │   │  AnimationLayerMixerPlayable - 层级混合         │ │
    │   │  AnimationScriptPlayable  - 自定义 Job          │ │
    │   │  AnimatorControllerPlayable - 包装 Controller   │ │
    │   └─────────────────────────────────────────────────┘ │
    └───────────────────────────────────────────────────────┘
```

### 2.2 核心类型说明

| 类型 | 职责 | 特点 |
|------|------|------|
| **PlayableGraph** | 整个图的容器 | 管理所有 Playable 和 Output 的生命周期 |
| **Playable** | 图中的节点 | 轻量级句柄（Handle），实际数据在原生层 |
| **PlayableOutput** | 输出端点 | 将图的计算结果输出到 Animator/AudioSource 等 |
| **PlayableBehaviour** | 自定义行为 | 可继承实现自定义逻辑（如脚本 Playable） |

---

## 三、Playable 的数据结构

### 3.1 Playable 是句柄（Handle）

```
  C# 层 (托管代码)                     Native 层 (原生代码)
  ┌────────────────────┐              ┌────────────────────┐
  │     Playable       │              │  PlayableInternal  │
  │  (struct, 16字节)  │──────────?  │  (实际数据存储)    │
  │                    │              │                    │
  │  - m_Handle (int)  │   句柄索引   │  - 输入连接数组    │
  │  - m_Version (int) │   ────────? │  - 输出连接数组    │
  │                    │              │  - 时间/速度/权重  │
  └────────────────────┘              │  - 播放状态       │
                                      │  - 类型信息       │
                                      └────────────────────┘
```

**优点：**
- **轻量级**：C# 层只有 16 字节，可以快速传递和复制
- **安全性**：版本号检测防止访问已销毁的 Playable
- **性能**：实际数据在原生层，避免 GC 压力
- **一致性**：所有 Playable 类型使用相同的句柄结构

### 3.2 Playable 的层次结构

```csharp
// Playable 是一个通用句柄，具体类型通过隐式转换获得
Playable playable;                              // 通用类型
AnimationClipPlayable clipPlayable = ...;      // 具体类型
playable = clipPlayable;                        // 隐式转换 (装箱)
clipPlayable = (AnimationClipPlayable)playable; // 显式转换 (需要类型匹配)
```

---

## 四、PlayableGraph 的工作流程

### 4.1 创建和连接流程

```csharp
// 1. 创建 Graph
var graph = PlayableGraph.Create("MyGraph");

// 2. 创建 Playable 节点
var clipPlayable1 = AnimationClipPlayable.Create(graph, clip1);
var clipPlayable2 = AnimationClipPlayable.Create(graph, clip2);
var mixerPlayable = AnimationMixerPlayable.Create(graph, 2);

// 3. 连接节点
graph.Connect(clipPlayable1, 0, mixerPlayable, 0);  // 输入端口 0
graph.Connect(clipPlayable2, 0, mixerPlayable, 1);  // 输入端口 1

// 4. 设置权重
mixerPlayable.SetInputWeight(0, 0.5f);
mixerPlayable.SetInputWeight(1, 0.5f);

// 5. 创建输出并连接
var output = AnimationPlayableOutput.Create(graph, "Animation", animator);
output.SetSourcePlayable(mixerPlayable);

// 6. 播放
graph.Play();
```

### 4.2 图结构示例

```
                         AnimationPlayableOutput
                                   │
                                   │ SetSourcePlayable
                                   ▼
                        AnimationMixerPlayable
                         (混合多个动画)
                        ╱                    ╲
                Input[0]                    Input[1]
               Weight=0.7                  Weight=0.3
                  ╱                            ╲
     AnimationClipPlayable              AnimationClipPlayable
          (idle.anim)                       (walk.anim)
          
结果: 输出 = idle * 0.7 + walk * 0.3
```

---

## 五、评估（Evaluate）机制

### 5.1 评估流程

```
  每帧更新 (根据 UpdateMode)
       │
       ▼
  ┌─────────────────────────────────────────┐
  │ 1. PrepareFrame (前向遍历)               │
  │    从 Output 开始，递归到所有子节点       │
  │    计算每个节点的时间和权重               │
  └─────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────┐
  │ 2. ProcessFrame (后向遍历)               │
  │    从叶子节点开始，计算动画姿势           │
  │    混合器节点混合子节点结果               │
  └─────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────┐
  │ 3. 输出到目标 (Animator)                 │
  │    最终姿势应用到骨骼                     │
  └─────────────────────────────────────────┘
```

### 5.2 按需评估的优化

```
                    AnimationLayerMixerPlayable (根)
                           ╱              ╲
                      Layer[0]          Layer[1]
                      Weight=1          Weight=0  ← 权重为0，不评估！
                         │                  │
                   MixerPlayable      MixerPlayable
                    ╱        ╲          ╱        ╲
               Clip1    Clip2      Clip3    Clip4
                              
只有 Layer[0] 及其子节点会被评估
Layer[1] 的子树完全跳过，节省 CPU 开销
```

---

## 六、Animancer 对 Playable API 的封装

### 6.1 类对应关系

| Animancer 类 | Playable API 类型 | 说明 |
|-------------|------------------|------|
| AnimancerGraph | PlayableGraph | 持有 PlayableGraph 实例 |
| AnimancerLayerList | AnimationLayerMixerPlayable | 层混合器 |
| AnimancerLayer | AnimationMixerPlayable | 单层内的状态混合 |
| ClipState | AnimationClipPlayable | 播放 AnimationClip |
| ManualMixerState | AnimationMixerPlayable | 自定义混合 |
| ControllerState | AnimatorControllerPlayable | 包装 AnimatorController |
| PlayableAssetState | PlayableAsset.CreatePlayable() | 播放 Timeline 等 |

### 6.2 Animancer 的图结构

```
  AnimationPlayableOutput
            │
            ▼
  UpdatableListPlayable (_PreUpdatePlayable)   ← Animancer 自定义节点
            │                                     用于 Pre/Post Update
            ├────────────────────┐
            │                    │
            ▼                    ▼
  AnimationLayerMixerPlayable   UpdatableListPlayable (PostUpdate)
   (AnimancerLayerList)
            │
    ┌───────┼───────┐
    │       │       │
    ▼       ▼       ▼
  Layer0  Layer1  Layer2  ...
    │
    ▼
  AnimationMixerPlayable (AnimancerLayer)
    │
    ├──────────┬──────────┐
    │          │          │
    ▼          ▼          ▼
  State0    State1    State2  ...
```

---

## 七、连接与断开的性能优化

### 7.1 KeepChildrenConnected 机制

```csharp
// Animancer 的优化策略：不活跃的状态断开连接
public override bool KeepChildrenConnected => _KeepChildrenConnected;

// 当状态 Weight = 0 且 IsPlaying = false 时：
// - KeepChildrenConnected = false (默认): 断开该状态的 Playable
// - KeepChildrenConnected = true: 保持连接，但权重为 0
```

### 7.2 比较

| 模式 | 活跃时 | 不活跃时 | 优点 | 缺点 |
|------|--------|----------|------|------|
| false (默认) | 连接 | 断开 | 减少不必要的评估 | 频繁连接/断开有开销 |
| true | 连接 | 保持连接(权重0) | 切换更快 | 可能有额外评估开销 |

---

## 八、自定义 Playable

### 8.1 使用 PlayableBehaviour

```csharp
// 自定义 Playable 行为
public class MyPlayableBehaviour : PlayableBehaviour
{
    // 图创建时调用
    public override void OnGraphStart(Playable playable) { }
    
    // 图销毁时调用
    public override void OnGraphStop(Playable playable) { }
    
    // 节点开始播放时调用
    public override void OnBehaviourPlay(Playable playable, FrameData info) { }
    
    // 节点停止播放时调用
    public override void OnBehaviourPause(Playable playable, FrameData info) { }
    
    // 每帧准备阶段调用（前向遍历）
    public override void PrepareFrame(Playable playable, FrameData info) { }
    
    // 每帧处理阶段调用（后向遍历）
    public override void ProcessFrame(Playable playable, FrameData info, object playerData) { }
}
```

### 8.2 使用 IAnimationJob（高性能）

```csharp
// Animation Job：在工作线程上执行，高性能
public struct MyAnimationJob : IAnimationJob
{
    // 处理根运动
    public void ProcessRootMotion(AnimationStream stream)
    {
        // 修改根骨骼的位置/旋转
    }
    
    // 处理动画
    public void ProcessAnimation(AnimationStream stream)
    {
        // 直接操作骨骼变换
        // 可用于程序化动画、IK等
    }
}

// 创建使用 Job 的 Playable
var jobPlayable = AnimationScriptPlayable.Create(graph, new MyAnimationJob());
```

---

## 九、最佳实践

### 9.1 性能优化要点

1. **减少图结构变化**
   - 避免频繁 Connect/Disconnect
   - 预创建所有需要的 Playable
   - 使用权重控制而非连接/断开

2. **合理使用 KeepChildrenConnected**
   - 频繁切换的动画：保持连接 (true)
   - 偶尔播放的动画：断开连接 (false，默认)

3. **使用 Animation Job**
   - 程序化动画用 IAnimationJob
   - 利用 Job System 多线程

4. **减少 Playable API 调用**
   - 缓存时间值
   - 批量设置权重

5. **及时销毁**
   - 不用的 Playable 及时销毁
   - 场景切换时销毁整个 Graph

### 9.2 常见陷阱

```csharp
// ? 错误：每帧创建新的 Playable
void Update()
{
    var clip = AnimationClipPlayable.Create(graph, animClip);  // 每帧创建！
}

// ? 正确：预创建并复用
AnimationClipPlayable _clipPlayable;

void Start()
{
    _clipPlayable = AnimationClipPlayable.Create(graph, animClip);
}

// ? 错误：忘记销毁 Graph
void OnDestroy()
{
    // 忘记调用 graph.Destroy() 会导致内存泄漏！
}

// ? 正确：始终销毁
void OnDestroy()
{
    if (graph.IsValid())
        graph.Destroy();
}
```

---

## 十、总结：Playable API 设计理念

| 理念 | 说明 |
|------|------|
| **图结构 (Graph-Based)** | 节点连接形成有向无环图，数据从叶子流向根 |
| **组合优于继承** | 通过组合不同 Playable 实现复杂功能 |
| **数据驱动** | 所有状态存储在原生层，C# 只持有句柄 |
| **延迟评估** | 只有活跃且连接的节点才被评估 |
| **统一接口** | 动画、音频、脚本使用相同的 Playable 架构 |
| **可扩展性** | 通过 PlayableBehaviour 和 IAnimationJob 自定义行为 |

**Animancer 在此基础上提供：**
- 更友好的 API 封装
- 自动状态管理
- 事件系统
- 过渡系统
- 调试工具

---

## 相关文档

- [Animancer 最佳实践文档](./Animancer_BestPractices.md) - Animancer 的详细使用指南和最佳实践
