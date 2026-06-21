# ZMQ 实时仿真 UE 侧配合文档

## 1. 目标

与 gNB 配合，验证基于 ZMQ 的分布式实时 5G 仿真。要求：

- gNB 和 UE 分别在两台机器上运行。
- 各自独立维持严格的 **1 ms slot** 节奏（15 kHz SCS）。
- gNB 到 UE 的传输延迟由 UE 侧缓冲吸收。
- 双方时间通过 PTP 同步。

本阶段为**最小可行验证**，暂不做 UE 漂移补偿和 PUB/SUB 改造。如果本阶段 slot 时长能稳定在 1 ms，再进入下一阶段优化。

---

## 2. UE 需要做的代码修改

UE 必须与 gNB 使用**同一份修改后的 srsRAN 代码**。需要修改以下三个地方。

### 2.1 修改 ZMQ radio 的 `read_current_time()`

**文件：** `lib/radio/zmq/radio_session_zmq_impl.cpp`

**当前代码（约第 153 行）：**
```cpp
baseband_gateway_timestamp radio_session_zmq_impl::read_current_time()
{
  return 0;
}
```

**修改后：**
```cpp
baseband_gateway_timestamp radio_session_zmq_impl::read_current_time()
{
  auto now = std::chrono::steady_clock::now();
  auto us  = std::chrono::duration_cast<std::chrono::microseconds>(now - start_time).count();
  return static_cast<baseband_gateway_timestamp>(us * srate_hz / 1e6);
}
```

**文件：** `lib/radio/zmq/radio_session_zmq_impl.h`

**在 `radio_session_zmq_impl` 类中添加成员：**
```cpp
private:
  std::chrono::steady_clock::time_point start_time;
  double srate_hz = 0.0;
```

**在构造函数中初始化（`lib/radio/zmq/radio_session_zmq_impl.cpp`）：**
```cpp
radio_session_zmq_impl::radio_session_zmq_impl(...)
{
  ...
  srate_hz = config.sampling_rate_Hz;
  start_time = std::chrono::steady_clock::now();
  ...
}
```

**说明：** 原实现直接返回 0，导致 lower PHY 没有任何时间参考。修改后返回基于本地 steady_clock 的样本计数，使 pacing 有一个可计算的时间基准。

---

### 2.2 修改 lower PHY 为 busy-wait

**文件：** `lib/phy/lower/lower_phy_baseband_processor.cpp`

**找到以下代码（约第 132 行）：**
```cpp
if (elapsed < minimum_elapsed) {
  std::this_thread::sleep_until(*last_tx_time + minimum_elapsed);
}
```

**改为：**
```cpp
if (elapsed < minimum_elapsed) {
  auto target = *last_tx_time + minimum_elapsed;
  while (std::chrono::high_resolution_clock::now() < target) {
    __builtin_ia32_pause();
  }
}
```

**说明：** `sleep_until` 会被 OS 调度延迟，无法保证严格的 1 ms 间隔。busy-wait 让线程一直占用 CPU，直到目标时间到达，精度可达微秒级。

---

### 2.3 调大 ZMQ 缓冲区

**文件：** `lib/radio/zmq/radio_session_zmq_impl.h`

**当前值：**
```cpp
static constexpr unsigned DEFAULT_STREAM_BUFFER_SIZE = 614400;
```

**改为：**
```cpp
static constexpr unsigned DEFAULT_STREAM_BUFFER_SIZE = 115200;  // 5 slot @ 23.04 MHz
```

**说明：** 默认 614400 样本约 26 ms，太大；改为 5 slot（约 5 ms）可以吸收网络抖动，同时不会引入过多延迟。

---

## 3. UE 运行环境配置

### 3.1 配置 PTP 时间同步

**安装 linuxptp：**
```bash
sudo apt update
sudo apt install linuxptp
```

**启动 PTP（硬件时间戳）：**
```bash
sudo ptp4l -i <网卡名> -m -H
```

如果网卡不支持硬件时间戳，用软件时间戳：
```bash
sudo ptp4l -i <网卡名> -m -S
```

**同步系统时钟：**
```bash
sudo phc2sys -s <网卡名> -c CLOCK_REALTIME -m -n 24
```

**验证同步质量：**
```bash
sudo pmc -u -b 0 'GET TIME_STATUS_NP'
```

**要求：** `offsetFromMaster` 稳定在 **100 us 以内**。如果超过 500 us，需要检查网卡、交换机或改用直连网线。

---

### 3.2 系统调优

**调大 socket 缓冲：**
```bash
sudo sysctl -w net.core.rmem_max=33554432
sudo sysctl -w net.core.wmem_max=33554432
```

**CPU 性能模式：**
```bash
sudo apt install linux-tools-common
sudo cpupower frequency-set -g performance
```

**说明：** 避免 CPU 变频带来的时序抖动。

---

### 3.3 启动 UE

```bash
sudo chrt -f 99 taskset -c 2,3 ./srsue -c ue_zmq.conf
```

参数说明：
- `chrt -f 99`：使用 SCHED_FIFO 实时调度，优先级 99（最高）。
- `taskset -c 2,3`：把 UE 进程绑定到第 2、3 号核心。
- 请确保第 2、3 号核心上没有其他重要任务。

---

## 4. UE 配置文件示例

`ue_zmq.conf`：

```yaml
rf:
  freq_offset: 0
  tx_gain: 80
  rx_gain: 40

  srate: 23.04
  device_name: zmq
  device_args: tx_port=tcp://<gNB_IP>:2001,rx_port=tcp://<gNB_IP>:2000,base_srate=23.04e6

log:
  all_level: warning
  rf_level: warning
  phy_level: warning
  mac_level: warning
  rlc_level: warning
  pdcp_level: warning
```

**注意：**
- `tx_port` 和 `rx_port` 必须与 gNB 配置中的端口对应。
- 日志级别先设成 warning，减少日志 IO 开销。
- `<gNB_IP>` 替换为 gNB 机器的实际 IP 地址。

---

## 5. 需要收集的数据

### 5.1 日志文件

启动时重定向日志到文件：
```bash
./srsue -c ue_zmq.conf > /tmp/ue.log 2>&1
```

### 5.2 slot 时长统计

运行后用以下 Python 脚本分析 PHY 日志：

```python
import re
import statistics

pattern = re.compile(r'(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d+) \[PHY\s*\] \[D\] \[\s*(\d+)\.(\d+)\]')

slots = {}
with open('/tmp/ue.log', 'r') as f:
    for line in f:
        m = pattern.search(line)
        if m:
            ts_str, frame, slot = m.groups()
            h, mi, s = ts_str.split('T')[1].split(':')
            sec = float(h) * 3600 + float(mi) * 60 + float(s)
            key = (int(frame), int(slot))
            if key not in slots or sec < slots[key]:
                slots[key] = sec

# 计算相邻 slot 时间差
times = sorted(slots.values())
diffs = [(times[i] - times[i-1]) * 1000 for i in range(1, len(times))]

print(f"样本数: {len(diffs)}")
print(f"平均: {statistics.mean(diffs):.3f} ms")
print(f"中位数: {statistics.median(diffs):.3f} ms")
print(f"最小: {min(diffs):.3f} ms")
print(f"最大: {max(diffs):.3f} ms")
print(f"标准差: {statistics.stdev(diffs):.3f} ms")
```

**目标：**
- 平均 slot 时长 ≈ **1.000 ms**
- 标准差 < **0.1 ms**
- 最大跳变 < **0.5 ms**

### 5.3 PTP 同步状态

```bash
sudo pmc -u -b 0 'GET TIME_STATUS_NP'
```

记录 `offsetFromMaster` 和 `meanPathDelay`。

### 5.4 CPU 占用

```bash
top -p $(pgrep -d',' srsue)
```

或：
```bash
htop
```

预期：UE 进程占用 1–2 个核心接近 100%（busy-wait 导致）。

---

## 6. 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|---|---|---|
| UE slot 时长不稳定，忽大忽小 | PTP 未同步或 offset 太大 | 检查 `pmc` 输出，确保 offset < 100 us |
| UE CPU 占用 100% | busy-wait 正常行为 | 确认绑定到独立核心，不影响其他关键任务 |
| UE 无法接入 gNB | 端口配置不匹配 / 防火墙 | 检查 `device_args`，确认能 ping 通 gNB |
| 大量 ZMQ timeout | 网络延迟大或丢包 | 检查 ping 和交换机，必要时直连 |
| slot 标准差 > 0.5 ms | 有其他进程抢 CPU | 确认 `taskset` 生效，关闭其他占用 CPU 的程序 |
| PTP offset 跳动大 | 网卡不支持硬件时间戳 / 交换机不稳 | 换直连网线，或改用 `-S` 软件时间戳再测 |

---

## 7. 与 gNB 的协调清单

在启动测试前，双方确认以下事项：

- [ ] 双方使用同一份 srsRAN 代码（已修改 `read_current_time()`、busy-wait、buffer size）。
- [ ] 双方都已编译通过。
- [ ] 双方都已配置 PTP 并验证同步（offset < 100 us）。
- [ ] 双方都已执行系统调优（socket buffer、CPU performance）。
- [ ] gNB 配置文件中的 ZMQ 端口与 UE 配置文件对应。
- [ ] 双方都使用 `chrt -f 99 taskset -c ...` 启动。
- [ ] 双方都记录日志到文件。
- [ ] **启动顺序：先启动 gNB，再启动 UE。**

---

## 8. 测试流程

1. **gNB 机器：** 配置 PTP → 系统调优 → 启动 gNB。
2. **UE 机器：** 配置 PTP → 系统调优 → 启动 UE。
3. 双方运行至少 **5 分钟**。
4. 收集日志，用脚本分析 slot 时长。
5. 同时检查 PTP offset 是否稳定。
6. 把结果反馈给 gNB 侧，共同判断是否需要进入下一阶段（漂移补偿 / PUB/SUB）。

---

## 9. 需要反馈的内容

测试完成后，请向 gNB 侧提供：

1. `/tmp/ue.log` 文件。
2. slot 时长统计脚本的输出结果。
3. `sudo pmc -u -b 0 'GET TIME_STATUS_NP'` 的输出。
4. `top` 或 `htop` 中 UE 进程的 CPU 占用截图。
5. 是否成功接入 gNB，是否有异常告警。

---

## 10. 注意事项

- 本阶段**不修改 ZMQ 模式**（保持 REQ/REP），也不做 UE 漂移补偿。
- 本阶段的核心目标是：**验证 PTP + busy-wait + SCHED_FIFO 是否能让 UE 独立维持 1 ms slot。**
- 如果本阶段成功，下一步才会考虑：
  - 把 REQ/REP 改成 PUB/SUB，降低握手延迟。
  - 在 UE 侧加入漂移补偿，长时间运行更稳定。

---

## 11. 联系方式

如遇到问题，请提供以下信息：

- `/tmp/ue.log` 的前 100 行和最后 200 行。
- `pmc -u -b 0 'GET TIME_STATUS_NP'` 的输出。
- `top` 中 UE 进程的 CPU 占用截图。
- UE 机器配置（CPU 型号、核心数、网卡型号）。
