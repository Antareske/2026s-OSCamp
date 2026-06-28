# net-bench 架构设计

## 设计目标

net-bench 是"入口 + 参数明确"的严肃网络性能测试架构。核心原则：

1. **唯一严肃入口**：`run.sh` 显式接收架构 / 场景 / 加速器，保证可复现、可对照。
2. **智能入口实验化**：`bin/bench` 的环境自动检测属于实验性便捷能力，默认不参与
   严肃测试；它最终委托 `run.sh`，避免出现两套各自演化的执行路径。
3. **公共流程封装**：把常量、配置矩阵、iperf3 生命周期、前置检查、指纹、汇总集中
   到一处，消除散落硬编码、统一各入口行为。

## 关键模块

### 入口
- `run.sh`：唯一严肃入口（显式参数 `--scenario/--arch/--accel/--repeat`）
- `run-with-perf.sh`：perf stat 包裹变体
- `run-linux-baseline.sh`：Linux 同拓扑基线
- `prebuild.sh`：rootfs 预构建（被 axbuild discovery 调用）
- `bin/bench`：智能入口（实验性，委托 run.sh）
- `bin/bench-wsl`：WSL2 快捷壳
- `bin/setup`：配置网络（委托 env/setup-common.sh）
- `bin/teardown`：清理（委托 env/teardown.sh）

### 核心逻辑
- `core/lib.sh`：主机侧公共流程封装（常量、配置矩阵解析、iperf3 生命周期、前置检查、
  DHCP 校验、环境指纹、结果汇总）
- `core/net-bench-common.sh`：guest 侧基准核心（iperf3 + 标记协议）
- `core/net-bench.sh`：SLIRP guest 入口
- `core/net-bench-tap.sh`：TAP/vhost guest 入口
- `core/net-bench-netperf.sh`：netperf 延迟测试
- `core/summarize.py`：结果汇总（mean/stddev + NOISY 标记）
- `core/compare-baseline.py`：Starry vs Linux 基线对比

### 环境管理
- `env/detect-env.sh`：自动检测（JSON / human 输出）
- `env/setup-common.sh`：通用网络配置（br0/tap0/dnsmasq DHCP，状态化）
- `env/teardown.sh`：状态化回滚清理（读取 `.bench-state.json`）

### QEMU 配置矩阵
- `qemu/<scenario>-<arch>-<accel>.toml`：覆盖 20 个组合（5 场景 × 2 架构 × 2 加速模式）

### 构建配置
- `build-aarch64-unknown-none-softfloat.toml`：aarch64 构建配置（启用 virtio-net/virtio-blk）
- `build-x86_64-unknown-none.toml`：x86_64 构建配置

## 执行路径

### 严肃入口

```
run.sh --scenario S --arch A [--accel K] [--repeat N]
    │  source core/lib.sh
    ├─ 校验 arch/scenario/accel，推导默认 accel
    ├─ 解析配置: qemu/<S>-<A>-<K>.toml
    ├─ nb_check_scenario_prereq（kvm/vhost/tap 前置检查）
    ├─ nb_write_fingerprint（环境指纹）
    └─ for rep in 1..N:
         ├─ nb_start_iperf3（host 服务端）
         ├─ cargo xtask starry app qemu --test-case net-bench
         │      └─ prebuild.sh 安装 iperf3 + core/net-bench*.sh 到 rootfs
         │      └─ guest 跑 net-bench-common.sh（warmup + ITERS 迭代）
         ├─ nb_stop_iperf3
         └─ 收集日志
       nb_summarize（summarize.py 汇总）
```

### 智能入口（实验性）

```
bin/bench [scenario] [opts]
    ├─ env/detect-env.sh（检测 WSL/arch/kvm/vhost）
    ├─ 推断 ARCH/ACCEL，必要时 vhost→tap 降级
    ├─ env/setup-common.sh（配置网络，记录状态）
    ├─ 委托 run.sh --scenario ... --arch ... --accel ...
    └─ trap EXIT → env/teardown.sh（状态化回滚）
```

## 配置文件命名规范

`qemu/<scenario>-<arch>-<accel>.toml`

- scenario：`slirp` / `tap` / `vhost` / `vhost-smp4` / `tap-smp4`
- arch：`x86_64` / `aarch64`
- accel：`kvm` / `tcg`

`run.sh` 与 `bin/bench` 都通过 `nb_qemu_config` 解析该命名，新增场景/架构只需补齐
对应 toml 文件即可被两个入口识别。

## 自动回退（状态化）

`env/setup-common.sh` 把创建的资源与进程记录到 `.bench-state.json`：

```json
{
  "timestamp": "...",
  "created_resources": [
    {"type": "bridge", "name": "br0"},
    {"type": "tap", "name": "tap0"}
  ],
  "processes": [
    {"pid": 12345, "cmd": "iperf3 -s ..."}
  ]
}
```

`env/teardown.sh` 逆序读取并清理（SIGTERM→SIGKILL 终止进程、删除 tap/bridge）。

## 应用发现与 CI 关系

net-bench 列在 `apps/.ignore` 中，因此不参与默认 app 发现 / CI 全量跑——这与
"严肃性能测试是显式触发、非默认测试入口"的设计一致。通过
`cargo xtask starry app qemu --test-case net-bench ...` 显式指定时，发现逻辑会忽略
`apps/.ignore` 从而正常构建运行。

`prebuild.sh` 被 axbuild 的 discovery 自动识别为预构建脚本；它负责向 rootfs 注入
iperf3 及 guest 基准脚本。
