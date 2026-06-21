# ZMQ 实时仿真 UE 侧配合文档（srsRAN_4G UE）

## 1. 目标

与 gNB（srsRAN_Project）配合，验证基于 ZMQ 的分布式实时 5G 仿真。要求：

- gNB 和 UE 分别在两台机器上运行。
- **gNB 主动维持严格的 1 ms slot 发送节奏。**
- **UE 接收并处理样本，其 slot 节奏自然跟随 gNB 发送节奏。**
- gNB 到 UE 的传输延迟由 UE 侧缓冲吸收。
- 双方机器时间通过 PTP 同步。

**重要说明：** UE 使用 **srsRAN_4G** 项目，与 gNB 的 srsRAN_Project 是**两个不同的代码库**。UE 的 ZMQ radio 是 C 代码，架构和 gNB 不同，因此 UE 侧只需要改 `rf_zmq_get_time()`，不需要像 gNB 那样改 busy-wait。

本阶段为**最小可行验证**，暂不做 UE 漂移补偿和 PUB/SUB 改造。

---

## 2. 项目位置

| 角色 | 项目路径 | 可执行文件 |
|---|---|---|
| gNB | `/home/qijia/1Code/srsRAN_Project-main` | `build/apps/gnb/gnb` |
| UE | `/home/qijia/1Code/srsRAN_4G` | `build/srsue/src/srsue` |

---

## 3. UE 需要做的代码修改

### 3.1 修改 ZMQ radio 的 `rf_zmq_get_time()`

**文件：** `lib/src/phy/rf/rf_zmq_imp.c`

**第一步：添加 `#include <time.h>`**

在文件顶部已有 include 区域添加：
```c
#include <time.h>
```

**第二步：在 `rf_zmq_handler_t` 结构体中添加时间参考成员**

找到结构体定义（约第 35 行），在 `next_rx_ts` 后面添加：
```c
typedef struct {
  ...
  // Rx timestamp
  uint64_t next_rx_ts;

  // Time reference for get_time
  time_t start_sec;
  long   start_nsec;

  pthread_mutex_t tx_config_mutex;
  ...
} rf_zmq_handler_t;
```

**第三步：在 `rf_zmq_open_multi()` 中初始化启动时间**

找到 `rf_zmq_open_multi()` 函数（约第 202 行），在 `strcpy(handler->id, "zmq\0");` 之后添加：
```c
    handler->nof_channels     = nof_channels;
    strcpy(handler->id, "zmq\0");

    // Initialize time reference for rf_zmq_get_time
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    handler->start_sec  = ts.tv_sec;
    handler->start_nsec = ts.tv_nsec;

    rf_zmq_opts_t rx_opts = {};
```

**第四步：修改 `rf_zmq_get_time()` 返回真实时间**

找到 `rf_zmq_get_time()` 函数（约第 578 行），改为：
```c
void rf_zmq_get_time(void* h, time_t* secs, double* frac_secs)
{
  if (h) {
    rf_zmq_handler_t* handler = (rf_zmq_handler_t*)h;

    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);

    // Calculate elapsed seconds since radio open
    double elapsed = (double)(ts.tv_sec - handler->start_sec) +
                     (double)(ts.tv_nsec - handler->start_nsec) / 1e9;

    if (secs) {
      *secs = (time_t)elapsed;
    }

    if (frac_secs) {
      *frac_secs = elapsed - (time_t)elapsed;
    }
  }
}
```

**说明：** 原实现直接返回 0，导致 UE 协议栈没有时间参考。修改后返回基于 `CLOCK_MONOTONIC` 的相对时间。

### 3.2 不需要修改 busy-wait

UE 的 slot 节奏由 gNB 发送节奏驱动（通过 ZMQ 样本流）。UE 的 `sync_sa::run_thread()` 每次接收一个 slot 的样本，处理后再收下一个。因此 UE 不需要像 gNB 那样主动 busy-wait 守 1 ms。

### 3.3 不需要修改 ZMQ 缓冲区大小

srsRAN_4G UE 的 ZMQ 缓冲已经很大：
```c
#define ZMQ_MAX_BUFFER_SIZE (NSAMPLES2NBYTES(3072000)) // 10 subframes at 20 MHz
```
按 23.04 MHz 计算约为 **133 ms**，足以吸收网络抖动。不需要再调整。

---

## 4. 编译 UE

```bash
cd /home/qijia/1Code/srsRAN_4G/build
make -j$(nproc)
```

编译成功后，可执行文件为：
```
/home/qijia/1Code/srsRAN_4G/build/srsue/src/srsue
```

---

## 5. UE 运行环境配置

### 5.1 配置 PTP 时间同步

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

### 5.2 系统调优

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

### 5.3 启动 UE

```bash
sudo chrt -f 99 taskset -c 2,3 ./srsue -c ue_zmq.conf
```

参数说明：
- `chrt -f 99`：使用 SCHED_FIFO 实时调度，优先级 99（最高）。
- `taskset -c 2,3`：把 UE 进程绑定到第 2、3 号核心。
- 请确保这两个核心上没有其他重要任务。

---

## 6. UE 配置文件示例

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
- 如果需要查看 TTI 执行时间统计，可以在配置中设置 `have_tti_time_stats = true`。

---

## 7. 需要收集的数据

### 7.1 日志文件

启动时重定向日志到文件：
```bash
./srsue -c ue_zmq.conf > /tmp/ue.log 2>&1
```

### 7.2 TTI 执行时间统计

如果启用了 `have_tti_time_stats = true`，日志中会出现 TTI 执行时间的统计信息。需要关注：
- 每个 TTI 的处理时间是否稳定
- 是否有 TTI 超时或跳变

### 7.3 PTP 同步状态

```bash
sudo pmc -u -b 0 'GET TIME_STATUS_NP'
```

记录 `offsetFromMaster` 和 `meanPathDelay`。

### 7.4 CPU 占用

```bash
top -p $(pgrep -d',' srsue)
```

或：
```bash
htop
```

预期：UE 进程占用 1–2 个核心（主要来自 PHY 处理，不是 busy-wait）。

---

## 8. 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|---|---|---|
| UE 无法接入 gNB | 端口配置不匹配 / 防火墙 | 检查 `device_args`，确认能 ping 通 gNB |
| 大量 ZMQ timeout | 网络延迟大或丢包 | 检查 ping 和交换机，必要时直连 |
| TTI 处理时间不稳定 | PTP 未同步或 offset 太大 | 检查 `pmc` 输出，确保 offset < 100 us |
| CPU 占用过高 | 有其他进程抢 CPU | 确认 `taskset` 生效，关闭其他占用 CPU 的程序 |
| PTP offset 跳动大 | 网卡不支持硬件时间戳 / 交换机不稳 | 换直连网线，或改用 `-S` 软件时间戳再测 |

---

## 9. 与 gNB 的协调清单

在启动测试前，双方确认以下事项：

- [ ] gNB 已修改 `read_current_time()`、busy-wait、buffer size 并编译通过。
- [ ] UE 已修改 `rf_zmq_get_time()` 并编译通过。
- [ ] 双方都已配置 PTP 并验证同步（offset < 100 us）。
- [ ] 双方都已执行系统调优（socket buffer、CPU performance）。
- [ ] gNB 配置文件中的 ZMQ 端口与 UE 配置文件对应。
- [ ] 双方都使用 `chrt -f 99 taskset -c ...` 启动。
- [ ] 双方都记录日志到文件。
- [ ] **启动顺序：先启动 gNB，再启动 UE。**

---

## 10. 测试流程

1. **gNB 机器：** 配置 PTP → 系统调优 → 启动 gNB。
2. **UE 机器：** 配置 PTP → 系统调优 → 启动 UE。
3. 双方运行至少 **5 分钟**。
4. gNB 侧分析 PHY 日志中的 slot 时长（目标：平均 1.000 ms，标准差 < 0.1 ms）。
5. UE 侧分析 TTI 执行时间和接入状态。
6. 同时检查 PTP offset 是否稳定。
7. 把结果反馈给 gNB 侧，共同判断是否需要进入下一阶段（UE 漂移补偿 / PUB/SUB）。

---

## 11. 需要反馈的内容

测试完成后，请向 gNB 侧提供：

1. `/tmp/ue.log` 文件。
2. TTI 执行时间统计（如果启用了 `have_tti_time_stats`）。
3. `sudo pmc -u -b 0 'GET TIME_STATUS_NP'` 的输出。
4. `top` 或 `htop` 中 UE 进程的 CPU 占用截图。
5. 是否成功接入 gNB，是否有异常告警。
6. UE 机器配置（CPU 型号、核心数、网卡型号）。

---

## 12. 注意事项

- 本阶段**不修改 ZMQ 模式**（保持 REQ/REP），也不做 UE 漂移补偿。
- 本阶段的核心目标是：**gNB 侧稳定维持 1 ms slot，UE 能成功接入并稳定处理。**
- UE 的 slot 节奏天然跟随 gNB，因此不需要 UE 侧 busy-wait。
- 如果本阶段成功，下一步才会考虑：
  - 把 REQ/REP 改成 PUB/SUB，降低握手延迟。
  - 在 UE 侧加入漂移补偿，长时间运行更稳定。

---

## 13. 联系方式

如遇到问题，请提供以下信息：

- `/tmp/ue.log` 的前 100 行和最后 200 行。
- `pmc -u -b 0 'GET TIME_STATUS_NP'` 的输出。
- `top` 中 UE 进程的 CPU 占用截图。
- UE 机器配置（CPU 型号、核心数、网卡型号）。
