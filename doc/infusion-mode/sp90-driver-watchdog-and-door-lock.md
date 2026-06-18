# XP90 驱动存活检测与门锁屏状态机方案说明

## 1. 背景与问题

输液泵（XP90）使用 ARM + DSP 双 MCU 架构：
- **scence 进程**（ARM Cortex-A7，Linux 5.4）：应用逻辑
- **驱动 MCU**（DSP）：通过 UART 实时上报门状态、电机、压力等传感器数据

**核心问题**：当驱动异常（被擦除、系统故障 1 等）时，scence 端 `is_door_close` 传感器值停留在默认值 `false`（门开），状态机误判为"门开"而触发虚假锁屏。

**解决方案**：利用 UART 通讯心跳检测驱动存活，驱动失效时让状态机停止工作。

---

## 2. 核心机制：驱动存活检测

### 2.1 既有基础设施（已存在于代码库）

**位置**：`scence/src/drv_uart_thread.cpp:20-23`

```cpp
uint64_t drv_uart_last_tick = 0;
bool drv_uart_1000ms_flag = false;  // 1 秒无数据
bool drv_uart_2000ms_flag = false;  // 2 秒无数据
bool drv_uart_3000ms_flag = false;  // 3 秒无数据
```

**位置**：`scence/src/drv_uart_thread.cpp:107-138`

```cpp
// select 200ms 轮询串口
retval = select(fd + 1, &fs_read, NULL, NULL, &tv);

if ((tick > 10000) && (drv_uart_last_tick > 0)) {
    if (tick - drv_uart_last_tick > 1000) drv_uart_1000ms_flag = true;
    if (tick - drv_uart_last_tick > 2000) drv_uart_2000ms_flag = true;
    if (tick - drv_uart_last_tick > 3000) drv_uart_3000ms_flag = true;
}

if (data_len > 0) {
    // 任何字节到达都更新 last_tick
    drv_uart_last_tick = rena_get_sys_uptime();
}
```

**关键**：每次串口收到**任何字节**就更新 `drv_uart_last_tick`，不要求是合法协议帧。

### 2.2 新增 API（本次实现）

**位置**：`scence/src/drv_uart_thread.cpp:240-247`

```cpp
bool rena_is_drv_uart_alive(void)
{
    if (drv_uart_last_tick == 0) {
        return false;  // 从未收到过数据
    }
    return (rena_get_sys_uptime() - drv_uart_last_tick) < 800;
}
```

**阈值选择理由**：
- 驱动实时数据周期约 200ms
- 800ms = 4 个数据周期，提供 3 个周期的容错余量
- 与现有 1000ms flag 系统保留 200ms 安全余量
- 比 500ms 更容忍瞬时抖动

---

## 3. 完整工作流程（一步步）

### 阶段 A：驱动 → 串口（数据上行）

```
[驱动 MCU]
  周期性发送实时数据帧
  - 包含: door_close, motor_data, pressure 等
  - 周期: ~100-300ms（推测，与 scence 轮询匹配）
       │
       │ UART 物理层
       ▼
[/dev/ttyS0]
```

### 阶段 B：scence drv_uart_thread 接收

```
[scence drv_uart_thread 线程]
  while(1) {
    select(fd, 200ms 超时)
       │
       ├─ 收到字节 ──→ drv_uart_buf.push(data)
       │              drv_uart_last_tick = now()  ← 关键：驱动存活信号
       │              解析协议帧（head/len/crc16）
       │              分发到 rena_proc_drv_msg_realtime()
       │
       └─ 超时 ──→ 检查 last_tick, 置位 1s/2s/3s flag
  }
```

### 阶段 C：协议帧处理

```
[rena_proc_drv_msg_realtime()]  // protocol.cpp:189
  1. 解析 rena_drv_realtime 结构
  2. 提取 key_group->key.door_close
  3. 调用 set_door_close()        // 更新 sensor_param
  4. 调用 rena_drv_protocol_set_last_tick()  // 更新协议 last_tick
  5. 更新其他传感器字段
```

### 阶段 D：sensor_param 更新

```
[sensor_param]  // 全局单例
  door_close = true/false  // 门实际状态
  last_tick = now()        // 协议帧时间戳
```

### 阶段 E：domain_process 调用门状态机

```
[rena_domain_process()]  // domain.cpp 中
  cur_door_close = rena_get_domain_param()->get_is_door_close()
  drv_alive = rena_is_drv_uart_alive()
       │
       ├─ drv_alive == false ──→ return  // 状态机停止
       │
       └─ drv_alive == true ──→ 状态机工作
            ├─ prev=true, cur=false ──→ 锁屏 RSQ
            └─ prev=false, cur=true ──→ 解锁 RSQ
```

### 阶段 F：RSQ 发送

```
[scence → surface]
  RENA_UI_LOCK_SCREEN_RSQ {
    result: 0 (门开锁) / 2 (门关解锁)
    is_door_lock: true
  }
       │
       ▼
[surface proc_menu.c]
  创建/销毁 lock_view LVGL 对象
```

---

## 4. 门锁屏状态机

**位置**：`scence/src/domain.cpp:462-535`

```cpp
{
    static bool prev_door_close_state = true;  // 上一次门状态
    bool cur_door_close_state = rena_get_domain_param()->get_is_door_close();

    // 守卫：驱动失效时状态机停止
    if (!rena_is_drv_uart_alive()) {
        return;
    }

    // 门开 → 锁屏
    if (prev_door_close_state && !cur_door_close_state) {
        if (should_lock) {
            set_is_screen_lock(true);
            set_is_door_lock_screen(true);
            记录历史;
            发送锁屏 RSQ;
        }
    }

    // 门关 → 解锁
    if (!prev_door_close_state && cur_door_close_state) {
        if (is_door_lock_screen) {
            set_is_screen_lock(false);
            set_is_door_lock_screen(false);
            记录历史;
            发送解锁 RSQ;
        }
    }

    prev_door_close_state = cur_door_close_state;
}
```

**关键设计**：
- `prev` 是 static，跨多次 `rena_domain_process()` 调用保持
- 启动时 `prev=true`（假设门初始为关）
- 状态机只看**变化**（prev != cur），不主动根据当前值触发
- 驱动失效时整个状态机停止，不修改锁屏状态

---

## 5. 今日解决的问题清单

### 5.1 Session 起始状态
原始状态机（无驱动存活检测）：
```cpp
if (prev_door_close_state && !cur_door_close_state) {
    // 触发锁屏
}
```

**问题**：
- 驱动擦除时 sensor 默认 false（门开）
- 启动时 `prev=true, cur=false` 触发虚假锁屏
- 出现"驱动擦除后开机立即显示锁屏"问题

### 5.2 解决过程

**尝试 1（用户手动实现）**：加 `have_door_close_state` 守卫
```cpp
static bool have_door_close_state = false;
if (cur) have_door_close_state = true;
if (prev && !cur && have_door_close_state) { /* 锁 */ }
```
- ✅ 解决了驱动擦除+门关的虚假锁屏
- ❌ 引入新问题：门开+不关机+重启后不显示锁屏（门曾开着，但 have=false 拦截）

**尝试 2（Plan B）**：加 `drv_alive_confirmed` 守卫（基于驱动存活）
- ✅ 解决了门开+重启问题
- ❌ 引入新问题：驱动从未活过时（驱动擦除+门开）状态机不工作

**尝试 3（持久化）**：加 NVRAM 持久化门状态
- ✅ 解决了所有问题
- ❌ 引入新问题：正常开机+手动合门导致"锁屏闪一下"
- ❌ 用户认为持久化逻辑不合理（驱动异常也要锁屏？）

**最终方案（本次）**：回归原始 + 仅保留驱动存活检测
```cpp
if (!rena_is_drv_uart_alive()) {
    return;  // 驱动失效，状态机完全停止
}
```
- ✅ 驱动擦除 → 不锁屏（核心需求）
- ✅ 驱动异常 → 不锁屏
- ✅ 系统故障 1 → 不锁屏（因为协议帧停发，800ms 后 is_drv_alive=false）
- ✅ 驱动正常 + 门开 → 锁屏
- ✅ 驱动正常 + 门关 → 解锁
- ✅ 接受"驱动正常+手动合门时锁屏闪一下"

### 5.3 新增的代码

| 文件 | 改动 |
|------|------|
| `scence/src/drv_uart_thread.cpp` | 新增 `rena_is_drv_uart_alive()` 函数（8 行）|
| `scence/include/drv_uart_thread.h` | 新增函数声明 |
| `scence/src/domain.cpp` | 门状态机增加 4 行守卫（early return）|

**总计**：约 12 行新代码，复用既有 `drv_uart_last_tick` 基础设施。

---

## 6. 关键文件位置

| 文件 | 作用 |
|------|------|
| `scence/src/drv_uart_thread.cpp:20-23` | `drv_uart_last_tick` 全局变量定义 |
| `scence/src/drv_uart_thread.cpp:138` | UART 字节到达时更新 last_tick |
| `scence/src/drv_uart_thread.cpp:240-247` | `rena_is_drv_uart_alive()` 实现（800ms 阈值）|
| `scence/src/protocol.cpp:189-249` | `rena_proc_drv_msg_realtime()` 协议帧处理 |
| `scence/src/protocol.cpp:242` | `set_door_close(key_group->key.door_close)` |
| `scence/src/alarm_check.cpp:2326-2351` | 系统故障 1 检测（6 秒阈值，复用类似机制）|
| `scence/src/domain.cpp:462-535` | 门锁屏状态机 |
| `scence/src/domain.cpp:467-470` | 驱动存活守卫（early return）|

---

## 7. 行为对照表

| 场景 | 行为 | 备注 |
|------|------|------|
| 驱动擦除 + 门关 + 开机 | 不锁 | 驱动从未活 → early return |
| 驱动擦除 + 门开 + 重启 | 不锁 | 同上 |
| 正常开机（门关）| 不锁 | prev=true, cur=true，无变化 |
| 正常开门 | 锁 | prev=true, cur=false |
| 正常关门 | 解锁 | prev=false, cur=true |
| 正常开门后拔排线 | 保持锁屏 | 800ms 后 early return |
| 锁屏后驱动恢复+关门 | 解锁 | 驱动恢复，状态机工作 |
| 正常开机时门已开 | **首次检测到"开"会锁** | prev=true（静态初值）, cur=false |
| 系统故障 1（6 秒无协议包）| 状态机停止 | 6 秒内无协议帧，但任何字节会更新 last_tick，所以实际更早停止 |

---

## 8. 风险与边界

### 8.1 已知边界

1. **正常开机时门已开**会立即锁屏
   - 这是状态机的固有行为（`prev=true` 初值假设门关）
   - 用户接受

2. **驱动恢复后**状态机重新工作
   - `is_drv_alive` 实时检查（每次 domain_process 调用）
   - 驱动恢复后下次调用即可正常

3. **800ms 阈值**与系统故障 1 的 6 秒阈值协同
   - 800ms 是"驱动刚停止活动"的快速检测
   - 6 秒是"协议帧停止"的报警阈值
   - 两者独立工作，800ms 先触发锁屏状态机停止

### 8.2 与其他机制的关系

- **系统故障 1 报警**：800ms < 6 秒，状态机比报警更早停止
- **手动解锁**：slider unlock 通过 REQ/RSP 路径，与状态机独立
- **自动锁屏**（lock.c 300ms 定时器）：通过 `screen_is_locked` 标志，与门锁独立
- **高级报警解锁**（alarm_check.cpp:2731）：HIGH 报警+非门锁时自动解锁

---

## 9. 总结

**核心思路**：驱动都失效了还检测什么锁屏 - 状态机不工作。

**实现方式**：
1. 复用既有 `drv_uart_last_tick` UART 心跳基础设施
2. 新增 8 行 `rena_is_drv_uart_alive()` 封装
3. 门状态机增加 4 行 early return 守卫

**关键设计原则**：
- 只信任能说话的传感器的数据
- 状态机只看"变化"，不主动判定当前值
- 驱动失效时保持当前状态不变
