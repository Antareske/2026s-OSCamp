# 2026s-OSCamp

2026 春季开源操作系统训练营（Stage 7 专业阶段）个人工作产出仓库。

训练营主线：围绕 [StarryOS](https://github.com/Starry-OS/StarryOS) 的 Linux 网络应用兼容性，从系统调用测试出发，经完整应用适配与测试框架建设，深入到网络栈性能评估与优化准备，平行参与 OS 功能赛（proj57, ACT 推理）。

## 仓库结构

```
2026s-OSCamp/
├── README.md
├── reports/                     # 训练营报告
│   ├── 结营博客.md              #   Hexo 博客源文件（已发布至 rcore-os/blog）
│   └── 规划.md
├── 产出/                        # 各阶段文档与答辩材料
│   ├── 阶段1-系统调用测试/       #   epoll / file-sync / sigsetsize
│   │   ├── my-job.md
│   │   ├── epoll/
│   │   ├── file-sync/
│   │   └── sigsetsize/
│   ├── 阶段2-应用适配/           #   Nginx / Apache / VmX
│   │   ├── nginx/               #   测试方案 / PR 记录 / stress / CI 重构
│   │   ├── apache/              #   测试方案 / 审阅记录
│   │   └── VmX/
│   ├── 阶段3-性能优化与ACT/      #   网络栈分析 / 性能测试 / proj57
│   │   ├── net-bench/           #   性能分析 / 测试方法论 / QEMU 方案
│   │   ├── net-analysis/        #   网络命名空间分析
│   │   ├── ebpf/
│   │   └── proj57/              #   QEMU + RK3588 交付报告 & 复现文档
│   └── prep.pptx                #   答辩 PPT
└── images/                      #   文档配图
    ├── net-bench-vhost.png
    ├── net-bench-tap.png
    ├── net-bench-slirp.png
    └── act-single.png
```

## 工作概览

### 方案一：系统调用测试基础

以 epoll 事件复用和文件同步两组系统调用为目标，编写 Linux 兼容性测例，发现并修复 `syncfs`/`sync_file_range` 等实现偏差，所有测例在 x86_64 / riscv64 / aarch64 / loongarch64 四架构通过 CI。

**已合并 PR**：[#658](https://github.com/rcore-os/tgoskits/pull/658)、[#900](https://github.com/rcore-os/tgoskits/pull/900)、[#903](https://github.com/rcore-os/tgoskits/pull/903)

### 方案二：Linux 应用适配与测试框架

- **Nginx**：从零散脚本发展为 14 phase 完整测试框架，统一 runner 架构 + 三层 smoke/phase/debug 体系，4 架构 CI 支持。修复多 worker 卡死、EPOLLEXCLUSIVE 兼容等问题。
- **Apache**：参照 Nginx 架构从零实现 7 phase 测试，含 15 个 debug 探针。关键问题 TCP_DEFER_ACCEPT 通过 C 探针定位。
- **net-bench**：构建网络性能测试基础设施，支持 vhost/tap/slirp 三种拓扑、5 类性能指标、自动化结果汇总与 Linux 基线对比。
- **eBPF net_stats**：基于 kprobe/kretprobe 的协议栈内部收发观测工具。

**已合并 PR**：[#1014](https://github.com/rcore-os/tgoskits/pull/1014)、[#1018](https://github.com/rcore-os/tgoskits/pull/1018)、[#1038](https://github.com/rcore-os/tgoskits/pull/1038)、[#1267](https://github.com/rcore-os/tgoskits/pull/1267)、[#1297](https://github.com/rcore-os/tgoskits/pull/1297)、[#1316](https://github.com/rcore-os/tgoskits/pull/1316)

**开放 PR**：[#1311](https://github.com/rcore-os/tgoskits/pull/1311) (Apache)

### 方案三：网络性能优化与 OS 功能赛

- **StarryOS 网络栈评估**：系统分析 `ax-net` + smoltcp 网络实现的性能瓶颈，识别 4+ 次内存拷贝、全局大锁、批处理缺失、硬件 offload 缺失等问题，规划三阶段优化路线。
- **proj57 ACT 推理**：完成 QEMU（riscv64 CPU, tract ONNX）和 RK3588（NPU, RKNPU2 SDK）两阶段交付，NPU 执行阶段 StarryOS 已优于 Linux。镜像构建工作流固化。

## 相关链接

- 训练营主仓库：[rcore-os/tgoskits](https://github.com/rcore-os/tgoskits)
- 个人 fork：[Antareske/tgoskits](https://github.com/Antareske/tgoskits)
- 博客发布：[rcore-os/blog](https://rcore-os.cn/blog/)
- StarryOS：[Starry-OS/StarryOS](https://github.com/Starry-OS/StarryOS)
