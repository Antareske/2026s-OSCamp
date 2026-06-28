---
title: 2026春季OS训练营工作规划与进度总结
date: 2026-06-28
categories:
    - OSTraining
tags:
    - author:Antareske
    - repo:https://github.com/Antareske/tgoskits
    - 2026S
    - StarryOS
---

# 工作规划概览

本文档总结训练营期间的工作规划、进度跟踪和技术路线选择。

---

## 一、总体规划

训练营工作分为三个方向：

1. **方向一**：系统调用测试与修复（epoll、文件同步等）
2. **方向二**：以 Nginx 为引导的应用适配与测试框架构建
3. **方向三**：以移动机器人为引导的全栈 StarryOS 改进（proj57 赛题 + 网络栈优化评估）

---

## 二、已完成工作

### 2.1 方向一：系统调用基础

**目标**：验证核心系统调用的 Linux 兼容性

**完成内容**：
- 事件复用系统调用测试与修复（epoll 系列）
- 文件同步系统调用语义对齐（fsync、fdatasync、sync_file_range 等）
- 信号掩码参数检查修复（sigsetsize 与 Linux ABI 对齐）

**对应 PR**（已合并）：
- #658: test(starry): Add syscall epoll tests
- #900: fix(epoll,sigmask-related): align sigsetsize checks with linux abi
- #903: fix(starry-kernel): align file sync syscalls with Linux semantics

---

### 2.2 方向二：Nginx 与 Apache 测试框架

**目标**：建立完整的网络应用测试体系

**完成内容**：

#### Nginx 测试框架（已全部合入 dev）
- 统一测试入口（smoke/phase/debug/stress 四模式）
- 三层 QEMU 配置结构（all/phase/debug）
- 多 worker 信号中断修复（EPOLLEXCLUSIVE 支持）
- CI 自动化测试集成

**对应 PR**（已合并）：
- #1014: test(starry,nginx): implement alpine app nginx CI
- #1018: fix(starry,nginx): multi-worker signal interruption and EPOLLEXCLUSIVE handling
- #1038: test(starry,nginx): implement alpine nginx normal tests

#### Apache 测试框架（未合并）
- 参考 Nginx 框架实现 Apache 功能性测试
- 支持 phase10-40 功能验证
- 发现并定位 TCP_DEFER_ACCEPT 潜在问题

**对应 PR**（Open）：
- #1311: test(starry,apache): add apache normal tests

#### Nginx 压力测试工具（未合并）
- 实现基于 wrk 的压力测试框架
- 支持多并发配置、多 worker 场景

---

### 2.3 方向三：全栈 StarryOS 改进

#### 3.3.1 proj57 赛题：ACT 推理

**目标**：在 StarryOS 上实现端到端自动驾驶推理

**完成内容**：

**QEMU 阶段**（5月28日-31日）：
- 成功在 QEMU StarryOS 中运行 ACT 推理程序
- 实现 ONNX Runtime 用户态推理
- 完善交付报告

**RK3588 阶段**（6月2日-15日）：
- 6月2日：开发板到货
- 6月4日-10日：研究香橙派镜像结构，手动构建启动链
- 6月11日：**关键突破** — 更换官方 DTB 支持 NPU 驱动，ACT 推理输出正确
- 6月13日：完善 RKNN 文档，准备申请 sg2002
- 6月15日：**固化镜像构建工作流**，实现文件系统 + 推理程序 overlay 的镜像构建自动化

**技术难点**：
- TF 卡镜像构建和构建流程固化：通过分析各类镜像资产布局得出构建方案并反复测试
- NPU 驱动支持：需要正确的设备树才能激活 RKNPU2 驱动

**对应 PR**：
- 相关工作在比赛仓库中完成

**后续计划**：
- 申请 sg2002 开发板继续验证

#### 3.3.2 StarryOS 网络栈优化评估工具

**目标**：为网络栈持续优化提供性能测试基础设施

**完成内容**：
- 实现 net-bench 工具（6月16日-2x日迭代开发）
- 支持多种 QEMU 网络模式对比（SLIRP/TAP/vhost-net）
- 集成 eBPF 观测工具（基于 CN-TangLin 的工作）
- 编写完整测试文档和原理图

**对应 PR**（未合并）：
- net-enhance 分支：StarryOS 网络栈优化项目

---

### 2.4 其他基础设施修复

**完成内容**：
- 增强 VmPeak & VmHWM 内存监控功能
- 修复 x86 QEMU 动态启动路径
- 修复 somehal x86 IPI 向量问题

**对应 PR**（已合并）：
- #1316: feat(starry): enhance VmPeak & VmHWM
- #1267: fix(starry): route app qemu through dynamic boot
- #1297: fix(somehal): send x86 helper IPI on IPI vector

---

## 三、时间线与关键节点

### 5月26日-6月1日：Nginx 框架初建
- 5-26：迁移并实现 Nginx CI，并根据测试计划整理进度
- 5-27：定位多 worker 卡死问题并修复信号语义 #1018
- 5-28：重构 nginx CI 并调整测试计划 #1014；**QEMU StarryOS 首次跑通 ACT 推理**
- 5-29：跑通大部分功能性测试；定位个别问题 #1038
- 6-01：补充 #1018 测例；完善 nginx CI

### 6月2日-6月15日：RK3588 攻坚
- 6-02：rk3588 到货；开始 nginx 压测；研究最小支持 Nginx 的杰哥内核
- 6-03：处理 EPOLLEXCLUSIVE PR #1018；实现 onnx runtime ACT 推理；参考 Nginx 的 Apache 功能性测试
- 6-04：烧录 TF 卡 + rk3588 板开机，研究板测方案
- 6-05：响应 nginx CI 测试 PR #1038，记录到性能敏感的偶发 http 错误；继续推进网络命名空间
- 6-06：尝试使用开发容器进行板测
- 6-07：研究香橙派镜像结构，手动构建新版启动链的 starry 镜像
- 6-08：尝试从 tgoskits 网络连接香橙派-失败
- 6-09：调整 starry 镜像；实现 rknn 用户态推理程序
- 6-10：更新 starry 镜像以支持多核；实现镜像局部更新脚本
- 6-11：**关键突破** — 更换镜像为官方香橙派 dtb 以支持 NPU 驱动，在板上跑通 ACT 推理，输出正确
- 6-12/13：响应 nginx 测试的修改意见，变基为最新进度；对齐 apache 测试程序；完善 rknn 文档，准备申请 sg2002

### 6月15日-6月2x日：多线推进
- 6-15：
  - 升级 rk3588 推理程序的各阶段计时功能
  - 在香橙派上做了 ACT 推理的 starry vs linux 测试，撰写对比文档
  - 定位了 starry 的 VmX 内存监控功能不全的问题 #1316
  - **固化了针对 rk3588 的文件系统+推理程序 overlay 的镜像构建工作流，并撰写文档**
  - 完善比赛的 qemu 阶段交付报告
  - 重新整理比赛项目内容，撰写文档和交付报告
  - 响应 nginx 测试修改建议 #1038
  - 提交 x86 qemu 测试构建路径的修复 #1267
  - 尝试开发一种 3588/2002 的高效流水线 ACT 推理程序
  - 拟出 starryos 网络栈优化的项目方案，主要解决性能问题

- 6-16：
  - 拟出 starryos 网络栈优化项目的 qemu 的性能测试方案
  - 进一步完善流水线推理程序的构想

- 6-18：
  - 升级 nginx & apache 的应用测试 #1038 #1311
  - 增强 VmPeak & VmHWM #1316
  - starry 的网络性能测试工具
  - 修复 somehal x86 构建路径问题 #1297

- 6-2x：
  - 实现 net-bench，迭代过程见该分支 commit

---

## 四、PR 统计

### 已合并（9个）
1. #658: test(starry): Add syscall epoll tests
2. #900: fix(epoll,sigmask-related): align sigsetsize checks with linux abi
3. #903: fix(starry-kernel): align file sync syscalls with Linux semantics
4. #1014: test(starry,nginx): implement alpine app nginx CI
5. #1018: fix(starry,nginx): multi-worker signal interruption and EPOLLEXCLUSIVE handling
6. #1267: fix(starry): route app qemu through dynamic boot
7. #1297: fix(somehal): send x86 helper IPI on IPI vector
8. #1038: test(starry,nginx): implement alpine nginx normal tests
9. #1316: feat(starry): enhance VmPeak & VmHWM

### Open（1个）
1. #1311: test(starry,apache): add apache normal tests

### 未提交 PR 的工作
- Nginx 压力测试框架（nginx-stress 分支）
- 网络栈优化评估工具（net-enhance 分支）
- proj57 赛题相关工作（比赛仓库）

---

## 五、技术经验总结

### 5.1 测试设计经验

建立了 **smoke + phase + debug** 三层测试体系：
- **smoke**：CI 快速验证，覆盖基础功能（APK 安装、服务启动）
- **phase**：阶段性递进覆盖的功能测试（phase10-70），每个 phase 验证一组相关特性
- **debug**：自由度较高的用于定位问题的测试，支持单步调试、日志增强

这一模式在 Nginx 和 Apache 测试中均得到验证，可推广到其他应用测试。

### 5.2 镜像构建经验

**RK3588 TF 卡镜像构建和构建流程固化**：
- 通过分析各类镜像资产布局（香橙派官方镜像、社区镜像）得出构建方案
- 反复测试启动链：U-Boot → DTB → Kernel → Rootfs
- 实现 overlay 机制：在不重新打包整个镜像的情况下更新用户态程序
- 固化为可复现的构建工作流（6月15日完成）

### 5.3 性能测试设计

**net-bench 设计经验**：
- 难以搭建成套的测试体系，且兼顾多维度且稳定的测试要求，实际上是复杂的设计需求
- 需要考虑：网络模式（SLIRP/TAP/vhost-net）、并发数、测试时长、统计方式
- eBPF 观测工具的集成提供了内核侧的可观测性

---

## 六、主要困难与弯路

### 6.1 Nginx 测试稳定性
在保证稳定性和文档一致性时花了很多精力，主要体现在：
- 多 worker 场景下的信号中断问题调试
- EPOLLEXCLUSIVE 支持的正确性验证
- CI 环境下的偶发性错误定位

### 6.2 网络性能测试复杂性
StarryOS 网络性能测试 net-bench 难以搭建成套的测试体系，且兼顾多维度且稳定的测试要求，实际上是复杂的设计需求。net-bench 和相关 www 文档可见这一点。

### 6.3 RK3588 镜像构建
TF 卡镜像的构建和构建流程固化：通过分析各类镜像资产布局得出构建方案并反复测试。6月11日是关键突破点，解决了 NPU 驱动激活问题。

### 6.4 开发节奏问题
**弯路**：没有践行"小步快跑"的方针，且一开始的 AI 驾驭能力不足，导致很多想法（尤其是方案三）暂未落地。

---

## 七、协作致谢

### 直接协作

**学员**：
- **aptacc2421、WellDown64**：组员之间的帮助，讨论交流了包括但不限于 tgoskits、OS 功能比赛等话题
- **LetsWalkInLine（龙同学）**：从其手中交接了初步的 nginx starry app

**助教与导师**：
- **Mr Graveyard**：一阶段指导
- **Ajax**：rk3588 的帮助
- **周睿老师**：rk3588 和 PR 等方面的帮助

### 间接协作

**eBPF 工作**：用到了 eBPF，需要感谢 blog 例子 eg2（CN-TangLin）中提及的 eBPF 的实现工作

---

## 八、后续计划

### 8.1 近期（训练营结束后）
- 响应 Apache 测试 PR #1311 的修改意见
- 申请 sg2002 开发板继续 proj57 赛题验证
- 完善 net-bench 工具并提交 PR

### 8.2 中期
- 基于 net-bench 的测试结果，推进 StarryOS 网络栈优化
- 实现流水线 ACT 推理程序（RK3588/sg2002）
- 探索网络命名空间支持

### 8.3 长期
- 持续参与 StarryOS 社区，推动 Linux 应用兼容性提升
- 探索更多嵌入式场景的 StarryOS 适配

---

## 附录：工作仓库

- **个人 fork**：https://github.com/Antareske/tgoskits
- **上游仓库**：https://github.com/rcore-os/tgoskits
- **比赛仓库**：proj57 相关工作
