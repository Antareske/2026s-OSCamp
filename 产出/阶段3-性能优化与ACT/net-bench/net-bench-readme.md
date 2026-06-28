# StarryOS 网络性能测试套件（net-bench）

面向 StarryOS 网络栈优化的性能测试套件，提供"入口 + 参数明确"的严肃测试流程，
覆盖吞吐 / PPS / 延迟 / CPU 效率 / 多核扩展等维度，并支持与 Linux 基线同拓扑对照。

## 设计理念

- **唯一严肃入口 `run.sh`**：架构 / 场景 / 加速器都显式指定，保证可复现、可对照。
- **智能入口 `bin/bench`（实验性）**：自动检测本机环境（WSL/裸 Linux/KVM/vhost）并推断参数，
  内部最终委托给 `run.sh` 执行。
- **公共流程封装**：常量（网段/端口/拓扑）、配置矩阵解析、iperf3 服务端生命周期、前置检查、
  环境指纹、结果汇总集中封装，消除散落硬编码。

## 快速开始

### 严肃测试（推荐，参数明确）

```bash
# 主力性能拓扑：TAP + vhost-net（需 sudo 预先配置网络）
sudo bash apps/starry/net-bench/bin/setup
bash apps/starry/net-bench/run.sh --scenario vhost --arch x86_64

# 多核扩展
bash apps/starry/net-bench/run.sh --scenario vhost-smp4 --arch x86_64 --repeat 5

# 功能冒烟（SLIRP，无需 sudo / 无需网络配置）
bash apps/starry/net-bench/run.sh --scenario slirp --arch x86_64

# 清理
sudo bash apps/starry/net-bench/bin/teardown
```

> TAP/vhost 场景需自行用 `bin/setup` 配好网络并启动 iperf3 服务端；`run.sh` 会自动管理 iperf3 服务端生命周期。

### 智能入口（实验性，开发期便捷）

```bash
bash apps/starry/net-bench/bin/bench vhost
bash apps/starry/net-bench/bin/bench-wsl    # WSL2 快捷壳
```

## 测试场景与配置矩阵

配置文件命名规范：`qemu/<scenario>-<arch>-<accel>.toml`

| 场景 | 拓扑 | 用途 |
|------|------|------|
| slirp | QEMU usermode | 功能冒烟（性能数据无意义）|
| tap | TAP（无 vhost）| 功能 / 趋势兜底 |
| vhost | TAP + vhost-net | 主力性能测试 |
| vhost-smp4 | TAP + vhost-net, smp=4 | 多核扩展 |
| tap-smp4 | TAP, smp=4 | vhost 不可用时的多核兜底 |

- arch：`aarch64` / `x86_64`
- accel：`kvm`（同架构 + KVM 可用）/ `tcg`（跨架构或无 KVM，仅功能验证）

## 测试覆盖（guest 侧自动运行）

| test-id | 说明 |
|---------|------|
| tcp1 | TCP 单流上行（guest → host）|
| tcp4 | TCP 4 并发流上行 |
| tcp1r | TCP 单流下行（host → guest）|
| udp1g | UDP 大包，目标 1 Gbit/s |
| udp64 | UDP 64B 小包 PPS |

默认每个 test-id：1 次 warmup + 5 次测量。延迟 / 短连接维度由 netperf（TCP_RR/UDP_RR/TCP_CRR）补充。

## 环境要求

TAP/vhost 场景的宿主配置由 `bin/setup` 一键接管：

```bash
sudo bash apps/starry/net-bench/bin/setup
```

`setup` 会自动：补齐依赖（`iperf3`/`iproute2`/`bridge-utils`/`jq`/`dnsmasq`）、
加载 `vhost_net` 模块、放开 `/dev/kvm` 与 `/dev/vhost-net` 权限、创建 `br0`/`tap0`
并配地址、启动 `dnsmasq` DHCP。iperf3 服务端由 `run.sh` 自管，不在此启动。清理用
`sudo bash apps/starry/net-bench/bin/teardown`。

WSL2 需启用嵌套虚拟化（Windows 侧配置），编辑 `%USERPROFILE%\.wslconfig`：

```ini
[wsl2]
nestedVirtualization=true
```

然后 `wsl --shutdown` 重启。

### guest 如何获取 IP

当前 StarryOS 在 `cargo xtask starry app qemu` 路径下的 guest 内核只支持 DHCP 获取地址。
SLIRP 场景 QEMU usermode 内建 DHCP 自动应答；TAP/vhost 场景 host 侧需要 dnsmasq DHCP
（`bin/setup` 会启动）。`run.sh` 的前置检查会校验 DHCP 服务端（:67 端口）是否就绪。

## 结果分析

```bash
# 汇总 mean/stddev（NOISY 标记 >10% stddev）
python3 apps/starry/net-bench/core/summarize.py apps/starry/net-bench/results/starry-*.txt

# 与 Linux 基线对比
bash apps/starry/net-bench/run-linux-baseline.sh aarch64 vhost --repeat 5
python3 apps/starry/net-bench/core/compare-baseline.py \
    results/summary-aarch64-vhost-*.txt \
    results/summary-linux-baseline-aarch64-vhost-*.txt

# CPU 效率（cycles/byte, IPC）
bash apps/starry/net-bench/run-with-perf.sh --arch aarch64 --scenario vhost
```

结果保存在 `results/`：`starry-*`（原始日志）、`summary-*`（汇总）、
`fingerprint-*`（环境指纹）、`perf-stat-*`（perf 数据）。

## 详细文档

- [快速参考](QUICK_START.md)
- [架构设计](STRUCTURE.md)
- [多队列问题](MULTIQUEUE_ISSUE.md)
