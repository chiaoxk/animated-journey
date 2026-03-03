# UE 动画最佳实践（番外篇）

聚焦 UE5 动画系统在移动、格斗/武器技能等实际场景的落地方案，结合前文《Animation学习之旅》系列与源码调用链，给出结构化建议、调用序列与常见坑。

## 1. 总览与分层
- 主图结构：状态机（基础移动/空中/受击） → Slot/LayeredBlendPerBone → Additive overlay（表情/呼吸/瞄准） → Post-Process（IK/ControlRig）。
- 分层要点：下半身驱动移动，Slot 覆盖上半身技能；Additive 仅作用局部骨骼集，减少全身污染。

## 2. 状态机与转场
- 过渡条件：速度/加速度/地面检测/气力；格斗类增加“连段窗口”“可取消”“命中确认”。
- 防抖：Run⇄Sprint 使用加速度与延迟回落；空中进入技能时可锁定移动态，避免图震荡。
- Pose Cache：重复节点（长链 BlendSpace、循环蓄力段）使用 Pose Cache 减少评估开销。

## 3. 典型场景调用链
### 3.1 Idle → Run → Sprint → Jump
1) `UAnimInstance::NativeUpdateAnimation` 读速度/输入。
2) 状态机 `FAnimNode_StateMachine::UpdateState` 切换 Idle/Run/Sprint/JumpStart。
3) 移动用 BlendSpace：`FAnimNode_BlendSpacePlayer::Update/Evaluate` 采样序列；JumpStart/Loop 用 Sequence 或 Montage Section。
4) 根运动：按需在起跳段提取 `ExtractRootMotion`；空中多用移动组件物理。
5) `USkeletalMeshComponent::TickAnimation` → `RefreshBoneTransforms` 输出最终姿态；Post-Process IK 贴地/手部对位。

### 3.2 格斗轻/重击（无位移）
1) 触发轻击：AnimBP 置位标志 → 状态机切入 LightAttack（或 Slot 播放 Montage Section `Light_1`）。
2) Section 内事件：
   - 前摇：锁输入/锁朝向（可平滑旋转对齐目标）。
   - 挥砍开始通知：开武器碰撞盒 + Trail + CameraShake。
   - 命中窗口：命中则允许跳转 `Light_2`；未命中则收招。
   - 结束：关碰撞盒/Trail，释放输入。
3) Additive：上半身叠加瞄准/表情，遮罩肩颈+头部。
4) 网络：关键事件用 BranchingPoint，避免高延迟丢通知。

### 3.3 突进斩（位移技 + 伤害窗）
1) 进入技：Montage Section `Dash_Start` 开根运动突进；可提前对齐目标朝向。
2) 伤害窗：`Dash_Hit` 段开启碰撞盒；命中确认后可派生/取消；不中命则收招。
3) 结束段：`Dash_End` 释放根锁，恢复移动；必要时落地 IK。
4) 根运动策略：短突进用根运动，长位移或可打断技用移动组件以便网络修正。

### 3.4 空连 / 派生
1) 空中技能：状态机空中分支允许技能插队；蒙太奇 Section 挂 Sync Marker 确保拼接平滑。
2) 连段窗口：用 `AnimNotifyState`/`BranchingPoint` 开放输入窗口；命中确认后再跳转 Section，避免空挥连段。
3) HitStop：通过动画速率或全局时间膨胀短暂停顿；恢复时平滑过渡到下一段。

## 4. 根运动与同步标记
- 根锁：重击/投技启用 RootMotionRootLock；轻击可不锁减少滑步。
- Sync Marker：Run→Sprint→JumpStart、连段各段统一 Marker 名，减少切换破绽；镜像/循环时也要布局 Marker。
- 领导/跟随：组队演出或多角色同步，用 Leader/Follower Marker，确保时间对齐。

## 5. IK 与 ControlRig
- 脚 IK：冲刺/落地贴地；根运动技中适度抑制 IK，避免双算导致脚漂。
- 手 IK：双持/握柄对位；位移技中短暂关闭防止拉伸。
- AimOffset/LookAt：Additive 叠加，权重沿脊柱递减，避免肩部过扭。

## 6. 通知、命中盒与特效
- 碰撞盒：在攻击段开，命中或结束段关；多段斩逐段开关，防止多次命中。
- 轨迹与贴花：挥砍开始启动 Trail，结束关闭；投射体生成绑在可靠通知。
- 摄像机：冲刺/重击用曲线驱动 FOV/Lag/Shake，结束段回落。

## 7. 网络与复制
- 关键事件用 BranchingPoint（可靠）触发：伤害、投射体生成、根运动锁切换。
- 根运动同步：仅同步位置/旋转校正或段落时间戳，减少带宽；必要时服务器强制纠偏。
- 预测/回滚：能力系统层做本地预测，命中后校准 Section 时间；高延迟下检查 Marker 对齐与根运动平滑。

## 8. 性能与压缩
- 高频/快节奏技：关闭或调弱 Frame Stripping，降低压缩误差阈，避免末端抖动。
- 常用资产预热：`BeginCacheDerivedDataForCurrentPlatform` 或驻留请求，减少首播卡顿。
- 并行与缓存：开启多线程动画；长链节点使用 Pose Cache；曲线/骨骼变更后重压缩，防止过期数据。

## 9. 调试 Checklist
- `ShowDebug Animation`/`AnimGraph`：状态机、权重、曲线可视化。
- 根运动轨迹：检查滑步/漂移，确认 RootLock 是否正确。
- 命中盒显示：验证开关时机与伤害窗口一致。
- 网络模拟：高延迟/丢包下验证 BranchingPoint 触发、根运动纠偏、Marker 对齐。
- 性能：观察压缩体积估算接口、Pose Cache 命中率、锁竞争（读/写锁）。

## 10. 源码锚点（便于深挖）
- 序列采样与压缩：`UAnimSequence::GetAnimationPose`、`GetBoneTransform_Lockless`、`ExtractRootMotion`、`CreateDerivedDataKeyHash`、`IsCompressedDataOutOfDate`
- 状态机/图评估：`FAnimNode_StateMachine::UpdateState`、`FAnimInstanceProxy::UpdateAnimationNode`
- BlendSpace：`FAnimNode_BlendSpacePlayer::UpdateInternal/Evaluate`
- 蒙太奇与 Slot：`UAnimMontage::Advance`、Slot 混合
- 通知/同步标记：`HandleAssetPlayerTickedInternal`、`AdvanceMarkerPhaseAsLeader/Follower`
- 根运动网络：`RootMotionMode`、蒙太奇根运动提取

---

将本篇与已有《AnimInstance 与动画图》《BlendSpace 混合空间》《RootMotion 深度解析》《动画通知系统》《LinkedAnimGraph 与分层动画系统》《动画同步与网络复制》配合阅读，可快速搭建项目级动画方案。