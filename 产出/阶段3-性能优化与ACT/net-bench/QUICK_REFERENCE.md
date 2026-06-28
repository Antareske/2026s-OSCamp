# net-bench 测试快速参考卡

## 环境配置

```bash
# 一键配置（自动装依赖、加载模块、创建桥接和 TAP、启动 DHCP）
sudo bash apps/starry/net-bench/bin/setup

# 检测环境
bash apps/starry/net-bench/env/detect-env.sh

# 清理
sudo bash apps/starry/net-bench/bin/teardown
```

## Starry 性能测试

```bash
# 严肃入口（推荐）：显式参数
bash apps/starry/net-bench/run.sh --scenario vhost --arch x86_64
bash apps/starry/net-bench/run.sh --scenario vhost --arch x86_64 --repeat 5

# 多核扩展
bash apps/starry/net-bench/run.sh --scenario vhost-smp4 --arch x86_64 --repeat 5

# 功能冒烟（SLIRP，无需 sudo）
bash apps/starry/net-bench/run.sh --scenario slirp --arch x86_64

# 智能入口（实验性，开发期便捷）
bash apps/starry/net-bench/bin/bench vhost
bash apps/starry/net-bench/bin/bench-wsl
```

## Linux 基线测试

```bash
bash apps/starry/net-bench/run-linux-baseline.sh aarch64 vhost --repeat 5
bash apps/starry/net-bench/run-linux-baseline.sh aarch64 vhost-smp4 --repeat 5
```

## 性能对比

```bash
python3 apps/starry/net-bench/core/compare-baseline.py \
    results/summary-aarch64-vhost-*.txt \
    results/summary-linux-baseline-aarch64-vhost-*.txt
```

## CPU 效率测试

```bash
bash apps/starry/net-bench/run-with-perf.sh --arch aarch64 --scenario vhost
cat results/perf-stat-aarch64-vhost-*.txt
```

---

## 测试前降噪（可选）

```bash
# CPU 频率固定
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 关闭后台服务
sudo systemctl stop snapd docker

# 释放缓存
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches

# 检查负载
uptime
```

---

## 故障排查

### KVM 不可用
```bash
ls -l /dev/kvm
egrep -c '(vmx|svm)' /proc/cpuinfo
sudo modprobe kvm_intel  # 或 kvm_amd
```

### vhost-net 不可用
```bash
ls -l /dev/vhost-net
sudo modprobe vhost_net
```

### 网络连通性
```bash
ip addr show br0
brctl show br0
ss -tlnp | grep 5201  # iperf3 server
```

---

## 测试覆盖

### iperf3
- TCP 单流上行（tcp1）、TCP 4流上行（tcp4）、TCP 单流下行（tcp1r）
- UDP 大包吞吐（udp1g）、UDP 64B 小包 PPS（udp64）

### netperf（已集成）
- TCP_RR（TCP 请求-响应延迟）、UDP_RR、TCP_CRR（TCP 短连接速率）

### perf stat（已集成）
- cycles / instructions / IPC、cache-references / cache-misses、LLC-load-misses

---

## 关键指标

### 测量纪律
- ≥5 次迭代，warmup 过滤
- 标准差 > 10% 标注为 NOISY
- 环境指纹记录

### 核心 KPI
- **吞吐**: Mbit/s（TCP/UDP）
- **PPS**: pkt/s（UDP 64B）
- **延迟**: P50/P99 μs（netperf RR）
- **连接速率**: conn/s（netperf CRR）
- **CPU 效率**: cycles/byte, IPC
- **多核扩展**: 吞吐(smp4) / 吞吐(smp1)
