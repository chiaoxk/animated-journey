# 🎬 Animation 之旅

> 通过源码级别的学习，深入理解 **Unreal Engine 5** 动画系统的设计理念与实现细节。

---

## 📖 项目简介

本系列是对 UE5 动画系统的系统性深度解析，涵盖从基础动画资产到高级网络同步的完整知识体系。每篇文章均从源码出发，配合架构图与代码示例，帮助开发者真正理解动画系统的底层原理。

---

## 📚 文章目录

| 篇目 | 主题 | 难度 |
|------|------|------|
| [001 - AnimSequence 深度解析](journey/Animation之旅_001_AnimSequence深度解析.md) | 动画序列，UE 动画系统的核心数据载体 | ⭐⭐ |
| [002 - AnimInstance 与动画图](journey/Animation之旅_002_AnimInstance与动画图.md) | 动画蓝图运行时与 AnimNode 执行架构 | ⭐⭐⭐ |
| [003 - BlendSpace 混合空间](journey/Animation之旅_003_BlendSpace混合空间.md) | 混合空间，多动画混合的核心资产 | ⭐⭐⭐ |
| [004 - 动画蒙太奇与 Slot 系统](journey/Animation之旅_004_动画蒙太奇与Slot系统.md) | AnimMontage，UE 中最强大的动画播放控制工具 | ⭐⭐⭐ |
| [005 - BlendSpace 与动画混合深度解析](journey/Animation之旅_005_BlendSpace与动画混合深度解析.md) | 从数学基础到��码实现的多维动画混合 | ⭐⭐⭐⭐ |
| [006 - 骨骼重定向与 IK 系统](journey/Animation之旅_006_骨骼重定向与IK系统.md) | 骨骼重定向原理与 IK 系统深度解析 | ⭐⭐⭐⭐ |
| [007 - Control Rig 与程序化动画](journey/Animation之旅_007_ControlRig与程序化动画.md) | 基于 RigVM 的程序化动画解决方案 | ⭐⭐⭐⭐ |
| [008 - Root Motion 深度解析](journey/Animation之旅_008_RootMotion深度解析.md) | 根骨骼运动的实现原理与网络同步 | ⭐⭐⭐⭐ |
| [009 - 动画通知系统 AnimNotify 深度解析](journey/Animation之旅_009_动画通知系统AnimNotify深度解析.md) | AnimNotify 与 AnimNotifyState 的架构与实现 | ⭐⭐⭐ |
| [010 - Linked Anim Graph 与分层动画系统](journey/Animation之旅_010_LinkedAnimGraph与分层动画系统深度解析.md) | 多 AnimInstance 协作与动画分层核心机制 | ⭐⭐⭐⭐ |
| [011 - 动画同步与网络复制](journey/Animation之旅_011_动画同步与网络复制.md) | 多人游戏中的动画同步策略 | ⭐⭐⭐⭐⭐ |
| [番外篇 - 动画压缩算法详解](journey/Animation之旅_番外篇_动画压缩算法详解.md) | UE5 动画压缩的原理、格式与实现细节 | ⭐⭐⭐⭐ |

---

## 🗺️ 知识体系

```
UE5 动画系统
├── 数据层
│   ├── UAnimSequence       （动画序列）
│   ├── UBlendSpace         （混合空间）
│   └── UAnimMontage        （动画蒙太奇）
├── 运行时层
│   ├── UAnimInstance       （动画蓝图实例）
│   ├── Linked Anim Graph   （链接动画图）
│   └── Control Rig / RigVM （程序化动画）
├── 高级特性
│   ├── Root Motion         （根骨骼运动）
│   ├── 动画压缩算法
│   └── 网络同步与复制
```

---

## 🛠️ 适用人群

- 有一定 UE 开发经验，希望深入理解动画系统原理的开发者
- 对游戏引擎底层架构感兴趣的技术同学
- 希望通过源码学习提升工程能力的学习者

---

## 📌 说明

- 引擎版本：**Unreal Engine 5.x**
- 文章持续更新中，欢迎 Star ⭐ 关注