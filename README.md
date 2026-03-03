# 🎬 Animated Journey

> Unreal Engine 动画系统系列深度解析文档

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/chiaoxk/animated-journey)](https://github.com/chiaoxk/animated-journey/stargazers)
[![GitHub issues](https://img.shields.io/github/issues/chiaoxk/animated-journey)](https://github.com/chiaoxk/animated-journey/issues)

---

## 📖 项目简介

本项目是一套关于 **Unreal Engine 动画系统** 的系列深度解析文档，涵盖从基础到进阶的各个核心主题，包括动画序列、动画蓝图、混合空间、蒙太奇、IK 系统、Control Rig、Root Motion、动画通知、分层动画、网络同步，以及番外篇动画压缩算法等内容。

每篇文章力求原理清晰、结合实践，帮助读者系统性地掌握 UE 动画开发技术。

---

## 📚 内容目录

| 编号 | 文章 | 简介 |
|------|------|------|
| 001 | [AnimSequence 深度解析](journey/Animation之旅_001_AnimSequence深度解析.md) | 深入剖析动画序列的结构、采样与播放机制 |
| 002 | [AnimInstance 与动画图](journey/Animation之旅_002_AnimInstance与动画图.md) | 动画实例的生命周期与动画蓝图图表详解 |
| 003 | [BlendSpace 混合空间](journey/Animation之旅_003_BlendSpace混合空间.md) | 一维/二维混合空间的原理与配置实践 |
| 004 | [动画蒙太奇与 Slot 系统](journey/Animation之旅_004_动画蒙太奇与Slot系统.md) | Montage 的结构、分段控制与 Slot 播放机制 |
| 005 | [BlendSpace 与动画混合深度解析](journey/Animation之旅_005_BlendSpace与动画混合深度解析.md) | 混合空间高级特性与多维动画融合策略 |
| 006 | [骨骼重定向与 IK 系统](journey/Animation之旅_006_骨骼重定向与IK系统.md) | 跨骨架重定向流程与各类 IK 求解器应用 |
| 007 | [ControlRig 与程序化动画](journey/Animation之旅_007_ControlRig与程序化动画.md) | 利用 Control Rig 实现运行时程序化动画 |
| 008 | [RootMotion 深度解析](journey/Animation之旅_008_RootMotion深度解析.md) | Root Motion 的提取、应用与网络同步要点 |
| 009 | [动画通知系统 AnimNotify 深度解析](journey/Animation之旅_009_动画通知系统AnimNotify深度解析.md) | Notify 与 NotifyState 的原理与自定义扩展 |
| 010 | [LinkedAnimGraph 与分层动画系统深度解析](journey/Animation之旅_010_LinkedAnimGraph与分层动画系统深度解析.md) | 模块化动画图链接与多层动画叠加方案 |
| 011 | [动画同步与网络复制](journey/Animation之旅_011_动画同步与网络复制.md) | 多人游戏下动画状态的同步策略与实现 |
| 番外 | [动画压缩算法详解](journey/Animation之旅_番外篇_动画压缩算法详解.md) | 关键帧压缩、曲线编码与运行时解压原理 |

---

## 👥 适合人群

- 正在学习或深入研究 Unreal Engine 动画系统的**游戏程序员**
- 希望掌握 UE 动画蓝图、混合空间等工具的**技术美术（TA）**
- 对动画底层机制感兴趣的 **UE 引擎开发者**
- 准备面试 UE 相关岗位、需要系统梳理动画知识的**求职者**

---

## 🚀 如何使用

**方式一：在线阅读**

直接在 GitHub 上点击上方目录中的链接，即可在浏览器中阅读每篇文章。

**方式二：克隆到本地**

```bash
git clone https://github.com/chiaoxk/animated-journey.git
cd animated-journey/journey
```

使用任意支持 Markdown 的编辑器（如 VS Code、Typora、Obsidian）打开 `journey/` 目录即可本地阅读。

---

## 🤝 贡献与反馈

欢迎提交 [Issue](https://github.com/chiaoxk/animated-journey/issues) 反馈问题或建议，也欢迎通过 [Pull Request](https://github.com/chiaoxk/animated-journey/pulls) 贡献内容或修正错误。

---

## 📄 License

本项目采用 [MIT License](LICENSE) 授权，欢迎自由使用与分享。