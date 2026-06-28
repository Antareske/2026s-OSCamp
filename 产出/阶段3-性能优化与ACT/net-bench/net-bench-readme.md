# StarryOS 网络性能基线测试

自动化的 StarryOS 网络性能测试套件，支持 WSL2 和裸 Linux 环境。

## 快速开始

### 本地开发（智能模式）
适合开发者快速验证，自动检测环境并配置：

```bash
# WSL2 环境
bash apps/starry/net-bench/bin/bench-wsl

# 裸 Linux 环境
bash apps/starry/net-bench/bin/bench
```

### CI / 自动化（明确模式，推荐）
适合 CI 和生产环境，明确指定所有参数：

```bash
# 性能测试（需要 KVM + TAP 网络配置）
cargo xtask starry app qemu \
    --test-case net-bench \
    --arch x86_64 \
    --qemu-config apps/starry/net-bench/qemu/vhost-x86_64-kvm.toml

# 功能冒烟（无需 sudo，SLIRP usermode 网络）
cargo xtask starry app qemu \
    --test-case net-bench \
    --arch x86_64 \
    --qemu-config apps/starry/net-bench/qemu/slirp-x86_64-kvm.toml
```

## 核心特性

- **环境自动检测** - 自动识别平台、架构、KVM 可用性
- **智能配置** - 自动创建网络、启动服务、规避已有资源
- **自动回退** - 测试结束或失败时自动清理
- **多场景支持** - vhost/tap/slirp，单核/多核

## 测试场景

| 场景 | 说明 | 用途 |
|------|------|------|
| vhost | TAP+vhost-net | 主力性能测试 |
| vhost-smp4 | TAP+vhost-net+SMP4 | 多核扩展 |
| tap | TAP（无 vhost） | 降级方案 |
| slirp | SLIRP usermode | 功能冒烟 |

## 测试覆盖

每次测试包含（guest 侧自动运行）：

| test-id | 说明 |
|---------|------|
| tcp1 | TCP 单流上行（guest → host）|
| tcp4 | TCP 4 并发流上行 |
| tcp1r | TCP 单流下行（host → guest）|
| udp1g | UDP 大包，目标 1 Gbit/s |
| udp64 | UDP 64B 小包 PPS |

默认每个 test-id：1 次 warmup + 5 次测量

## 环境要求

### 基础依赖
```bash
sudo apt-get install -y \
    iperf3 \
    bridge-utils \
    jq \
    dnsmasq
```

### WSL2 环境
- Windows 11（或 Win10 21H2+ with KB5020030）
- 嵌套虚拟化（在 `%USERPROFILE%\.wslconfig` 添加）:
  ```ini
  [wsl2]
  nestedVirtualization=true
  ```
- 重启 WSL: `wsl --shutdown`

### 裸 Linux 环境
- CPU 支持硬件虚拟化（Intel VT-x / AMD-V）
- KVM 模块已加载：`lsmod | grep kvm`

## 使用示例

### 本地开发测试（智能模式）
```bash
# 一键测试（自动检测环境）
bash apps/starry/net-bench/bin/bench

# 重复 5 次
bash apps/starry/net-bench/bin/bench vhost --repeat 5

# 多核测试
bash apps/starry/net-bench/bin/bench vhost-smp4
```

### CI / 生产环境（明确模式，推荐）
CI 环境应明确指定所有关键参数以保证可重复性和可控性：

```bash
# 直接使用 cargo xtask + 完整 QEMU 配置路径
cargo xtask starry app qemu \
    --test-case net-bench \
    --arch x86_64 \
    --qemu-config apps/starry/net-bench/qemu/vhost-x86_64-kvm.toml

# 或使用 bin/bench 明确指定架构和场景
bash apps/starry/net-bench/bin/bench vhost --arch x86_64 --skip-setup

# slirp 模式（功能冒烟，无需 sudo 网络配置）
cargo xtask starry app qemu \
    --test-case net-bench \
    --arch x86_64 \
    --qemu-config apps/starry/net-bench/qemu/slirp-x86_64-kvm.toml
```

**CI 最佳实践：**
- 明确指定 `--arch` 和 `--qemu-config`，不依赖自动检测
- 性能测试使用 `vhost-*-kvm.toml`（需 KVM + 网络配置）
- 功能冒烟使用 `slirp-*-*.toml`（无需 sudo，快速验证）
- 使用 `--skip-setup` 避免每次测试重复网络配置
- 配置文件命名规范：`<scenario>-<arch>-<accel>.toml`

### 环境管理
```bash
# 检测环境（无需 sudo）
bash apps/starry/net-bench/env/detect-env.sh

# 配置 TAP 网络（需要 sudo，仅用于 vhost/tap 场景）
bash apps/starry/net-bench/bin/setup

# 查看状态
bash apps/starry/net-bench/bin/teardown status

# 清理（需要 sudo）
bash apps/starry/net-bench/bin/teardown
```

**注意：**
- `slirp` 场景使用 QEMU usermode 网络，无需网络配置和 sudo
- `vhost` 和 `tap` 场景需要 TAP 桥接网络，需要 sudo 配置

### 高级用法
```bash
# 强制指定架构
bash apps/starry/net-bench/bin/bench vhost --arch aarch64

# 跳过配置（环境已配好）
bash apps/starry/net-bench/bin/bench vhost --skip-setup

# 测试后不清理（调试用）
bash apps/starry/net-bench/bin/bench vhost --no-cleanup
```

## 目录结构

```
apps/starry/net-bench/
├── README.md           # 本文档
├── bin/                # 统一入口
│   ├── bench          # 主入口（自动检测）
│   ├── bench-wsl      # WSL2 快捷入口
│   ├── setup          # 配置环境
│   └── teardown       # 清理环境
├── env/                # 环境管理
│   ├── detect-env.sh  # 自动检测
│   ├── setup-common.sh# 通用配置
│   └── teardown.sh    # 自动回退
├── core/               # 核心逻辑
│   ├── net-bench-common.sh  # guest 测试核心
│   ├── net-bench-tap.sh     # TAP/vhost 入口
│   ├── summarize.py         # 结果汇总
│   └── compare-baseline.py  # 基线对比
├── qemu/               # QEMU 配置
│   ├── vhost-x86_64-kvm.toml
│   ├── vhost-aarch64-tcg.toml
│   └── ...
├── docs/               # 详细文档
│   ├── QUICK_START.md       # 快速参考
│   ├── STRUCTURE.md         # 架构设计
│   ├── MULTIQUEUE_ISSUE.md  # 多队列问题
│   └── TODO.md              # 待办事项
└── results/            # 测试结果
```

## 常见问题

### KVM 不可用
```bash
# WSL2: 启用嵌套虚拟化（见上方"环境要求"）
# 裸 Linux: 检查权限
sudo chmod 666 /dev/kvm
```

### vhost-net 不可用
```bash
sudo modprobe vhost_net
```

### 测试失败
```bash
# 查看环境状态
bash apps/starry/net-bench/bin/teardown status

# 清理后重试
sudo bash apps/starry/net-bench/bin/teardown
bash apps/starry/net-bench/bin/bench
```

## 结果分析

测试结果保存在 `results/` 目录：

```bash
# 查看汇总
cat results/summary-*.txt

# 手动汇总
python3 core/summarize.py results/starry-*.txt

# 基线对比
python3 core/compare-baseline.py \
    results/summary-starry-*.txt \
    results/summary-linux-*.txt
```

## 详细文档

- [快速参考](docs/QUICK_START.md) - 命令速查
- [架构设计](docs/STRUCTURE.md) - 目录结构和设计原则
- [多队列问题](docs/MULTIQUEUE_ISSUE.md) - 技术问题说明
- [待办事项](docs/TODO.md) - 功能规划

## 获取帮助

```bash
bash apps/starry/net-bench/bin/bench --help
```

## 许可证

遵循项目根目录的 LICENSE
