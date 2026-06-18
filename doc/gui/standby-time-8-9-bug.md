# 表面层 GUI 待机时间设置 8 与 9 失效问题分析修复报告

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-06-18 | 根因定位、滚轮数据流、修复方案、影响范围、遗留问题 |
| v1.1 | 2026-06-18 | 补充跨进程时间/数据流总图、scence↔surface 待机启动全流程分析 |

---

## 一、问题清单

| ID | 现象 | 严重性 | 状态 |
|----|------|--------|------|
| P1 | 待机滚轮 h/m/s 任一字段为 `8` 或 `9` 时整数值丢失，最终保存偏小或为 0 | 中 | ✅ 已修复 |

修复前后行为对照：

| 设置 | 修复前 value | 修复后 value |
|------|-------------|-------------|
| `8min` / `9min` | 0（丢弃） | 480 / 540 |
| `8s` / `9s` | 0（丢弃） | 8 / 9 |
| `1h8min0s` | 3600（8min 丢失） | 4080 |
| `0h8min8s` | 60（8s 丢失） | 488 |
| `18min` / `19min` / `28min` / `29min` | 正常 | 正常 |
| `0min` - `7min` | 正常 | 正常 |

---

## 二、根因

**两个独立机制的巧合：**

1. **数据源** — `surface/src/component/dynamic_roller.c:79` 用 `"%02d"` 格式写选项字符串，个位数字强制前导 0 → `8 → "08"`、`9 → "09"`
2. **解析端** — `surface/src/ui/standby_time_kb_view.c:401,404,407,540,543,546` 6 处 `strtoul(buf, NULL, 0)`，`base=0` 自动判进制

`strtoul` 自动判进制规则：

| 字符串 | 识别为 | 结果 |
|--------|--------|------|
| `"123"` | 十进制 | 123 |
| `"017"` | 八进制 | 15 |
| **`"08"`** | 八进制（`8` 非八进制位，停在 `0`） | **0** |
| **`"09"`** | 同上 | **0** |
| `"18"` / `"19"` / `"28"` / `"29"` | 十进制（无前导 0） | 正确 |
| `"00"` - `"07"` | 八进制 | 正确（恰好等于原值） |

18/19/28/29 正常是因为无前导 0；0-7 正常是因为八进制位恰好等于十进制位。**只有单个 8/9 既带前导 0 又不是合法八进制位**，正好踩中陷阱。

同模块对照（验证这是孤立 bug）：

| 文件 | strtoul base | 状态 |
|------|--------------|------|
| `dynamic_roller.c` | 10 | ✅ |
| `time_kb_view.c` / `time_setting_kb_view.c` / `date_or_time_set.c` / `history_event.c` | 10 | ✅ |
| **`standby_time_kb_view.c`** | **0** | **❌ 本次修复** |
| `sint_kb_view.c` / `uint_kb_view.c` / `sfloat_kb_view.c` | 0 | ⚠️ 暂不可触发（键入有去前导 0 保护、预设用 `%u`/`%d` 无前导 0），保留 |

---

## 三、standby 滚轮数据流与 4 个调用关系

> 用户诉求：搞懂 `rena_dynamic_roller_set_selected` × 2、`lv_roller_get_selected_str`、`strtoul` 这 4 者的关系和时间机制。

### 3.1 LVGL roller 内部模型

```
┌──────────────────────────────────────────────────────┐
│ lv_roller 对象                                         │
│   options    = "55\n56\n57\n58\n59\n00\n01\n02\n03\n04\n05"
│   selected   = 5                                       │
│   mode       = LV_ROLLER_MODE_NORMAL                   │
└──────────────────────────────────────────────────────┘
```

- `options`：滚轮所有候选项，`\n` 分隔。拖动时**不修改**这个池。
- `selected`：当前居中显示的索引。
- `lv_roller_get_selected_str(buf, len)`：取 `selected` 索引对应的字符串。
- `lv_roller_set_options(s)`：整体替换选项池。
- `lv_roller_set_selected(idx)`：只改索引。

### 3.2 4 个调用的时序

```c
// t2: 用 preset 初始化（来自 kb_opt->preset）
rena_dynamic_roller_set_selected(roller_hour, hour);

// t3: 用 standby_time_remain 覆盖（当前实际保存值）
rena_dynamic_roller_set_selected(ui.roller_hour, standby_time_hour);

// t4-t6: 用户拖动 / 250ms 定时器 / 确认按钮
lv_roller_get_selected_str(ui.roller_hour, selected_str, 8);
hour_val = strtoul((const char *)selected_str, NULL, 0);  // bug 在此
```

**`set_selected(v)` 内部做两件事**：
1. 用 `v` 调用 `build_options()`，围绕 `v` 生成 [v-depth, v+depth] 的循环选项池（保证滚到边缘还有 depth 个可选项）
2. 调 `lv_roller_set_selected(depth)` 把指针居中

**`get_selected_str + strtoul` 是回算动作**：把当前选项池里被选中的字符串反算成数值（用于高亮快捷按钮 / 提交最终值），跟 `set_selected` 的"重建"是**两个独立动作**。

### 3.3 时序图

```
preset (uint32)              standby_time_remain (uint32)
    │ /3600, /60, %60             │ /3600, /60, %60
    ▼                             ▼
  h1,m1,s1                      h2,m2,s2
    │                             │
    │ set_selected (t2)           │ set_selected (t3, 覆盖 t2)
    ▼                             ▼
  ┌──────────────────────────────────┐
  │ roller_hour                      │  ← 同一个对象
  │   options = "..\n08\n.."         │
  │   selected = 15 (depth)          │
  └──────────────────────────────────┘
        ↑                                   ↑
        │ t4: 用户拖动                       │ t5: 250ms 定时器
        │ LVGL 改 selected 索引              │ get_selected_str → strtoul
        │ 触发 VALUE_CHANGED                  │ 对比快捷按钮
        │  → event_cb 用 base=10 重建          │  ← bug: 原 base=0
        │ t6: 确认按钮
        │  get_selected_str → strtoul
        │  → 提交最终值
        │  ← bug: 原 base=0
        ▼
    kb_param.confirm(&result)
        │ IPC
        ▼
    scence 写 DB + 启动 alarm
```

### 3.4 关键设计点

| # | 点 | 说明 |
|---|----|------|
| 1 | `set_selected` 的"居中重建" | 固定 depth 窗口，options 池围绕当前值动态生成 [v-depth, v+depth] 循环 wrap，保证边缘可滚 |
| 2 | **第一次 set_selected 是冗余的** | t2 用 preset，t3 用 standby_time_remain，**第二次完全覆盖第一次**。两次同步执行、效果相同，疑似早期遗留。**本次未删**（仅修 6 行 strtoul），记遗留问题 |
| 3 | `dynamic_roller.c` 用 `base=10`，standby 用 `base=0` | 事件回调（拖动时重建 options）用 base=10 正确；定时器和确认回调（回算当前值）用 base=0 翻车。**两个独立环节各用一套解析** |
| 4 | 拖动不影响 bug 触发 | 拖动时 options 重建用 base=10 的数值，滚轮"看起来"正常；bug 只在 t5/t6 两处回算时把已显示的 `"08"` 解析成 0 |
| 5 | buffer 8 字节够 | 滚轮最大字符串是 `"99\0"` = 3 字节，8 字节够用；bug 不是 buffer 溢出，是解析逻辑错 |

### 3.5 时间/数据流总图（surface ↔ scence 全链路）

```
┌─────────────────────────────────────────────────────────────────────────┐
│  surface 进程（LVGL C，主线程 + 消息线程）                                  │
│                                                                           │
│   用户输入                                                                 │
│   ┌─────────────────┐                                                     │
│   │ ① 长按电源键     │  ── key_power.c:212                                │
│   │ ② 下拉菜单待机   │  ── dropdown_menu.c:1546                            │
│   └────────┬────────┘                                                     │
│            │ RENA_UI_STANDBY_REQ (IPC, msg queue type=1)                 │
│            ▼                                                              │
│   ┌────────────────────────┐                                              │
│   │ 待机界面 standby.c     │  ← 250ms 定时器读 standby_time_remain       │
│   │  标签 + 倒计时显示     │     渲染 "Hh:Mm:Ss"                          │
│   └────────┬───────────────┘                                              │
│            │ 点"设置"按钮 → RENA_UI_KEYBOARD_REQ                         │
│            ▼                                                              │
│   ┌──────────────────────────────────┐                                    │
│   │ 待机时间键盘 standby_time_kb_view │  ← 本次 bug 现场                  │
│   │   roller_hour / minute / second │                                    │
│   │   ① rena_dynamic_roller_set_   │                                    │
│   │      selected(roller_hour,hour) │  t2 来自 preset                    │
│   │   ② rena_dynamic_roller_set_   │                                    │
│   │      selected(ui.roller_hour,  │  t3 来自 standby_time_remain        │
│   │      standby_time_hour)        │  ← 第二次覆盖第一次                 │
│   │   ③ 用户拖动 → LVGL 改 selected │                                    │
│   │      触发 event_cb (base=10)   │                                    │
│   │   ④ lv_roller_get_selected_str │                                    │
│   │      + strtoul(text,NULL,0)    │  ← t5 定时器 / t6 确认              │
│   │      ← strtoul 改成 base=10    │  ★ 本次修复点                       │
│   └────────┬─────────────────────────┘                                   │
│            │ 确认 → RENA_UI_CONFIG_SET_REQ                               │
│            │  index=STANDBY_TIME_SETTING, value=value                    │
└────────────┼──────────────────────────────────────────────────────────────┘
             │
             │ System V 消息队列 (key=13579)
             │ type=1: surface → scence (REQ)
             │ type=2: scence → surface (RSQ/NTY)
             │
┌────────────┼──────────────────────────────────────────────────────────────┐
│  scence 进程（C++14，17 线程）        ▼                                    │
│                                                                           │
│   ┌────────────────────────────────────┐                                  │
│   │ 消息分发 (ui_msg_process.cpp)      │                                  │
│   │   ① REQ → 调对应 handler            │                                  │
│   │   ② RSQ/NTY → 转发到 surface        │                                  │
│   └────────┬───────────────────────────┘                                  │
│            │                                                              │
│   ┌────────┴───────────────────────────────────────────────────────────┐  │
│   │ 业务处理                                                            │  │
│   │                                                                    │  │
│   │  CONFIG_SET_REQ(STANDBY_TIME)                                      │  │
│   │  ─→ ui_msg_proc_config_set.cpp:5719                                │  │
│   │     rena_proc_ui_config_set_standby_time(req, rsp)                │  │
│   │       ① set_standby_time_prev(v)  → user_setting（"上次值"）       │  │
│   │       ② set_standby_time_prev(v)  → domain（运行时）              │  │
│   │       ③ set_standby_time(v)       → domain（剩余时间初值）         │  │
│   │       ④ set + store common_setting  → 持久化到 DB                  │  │
│   │       ⑤ rena_infusion_state_standby_start()                       │  │
│   │       ⑥ rena_history_event_save(format=19, standby)  → 历史记录  │  │
│   │                                                                    │  │
│   │  STANDBY_REQ                                                       │  │
│   │  ─→ ui_msg_proc_standby.cpp:13                                    │  │
│   │     rena_proc_ui_msg_standby_req                                   │  │
│   │       ① 若正在运行 → return（不允许从运行态进入待机）              │  │
│   │       ② set_need_force_shutdown(0)                                 │  │
│   │       ③ rena_infusion_state_standby_start()                        │  │
│   │       ④ rena_history_event_save(format=1, standby)                │  │
│   │       ⑤ set_major_page_id(PAGE_MAJOR_STANDBY)                     │  │
│   │       ⑥ 发 RENA_UI_STANDBY_RSQ（携带 ui_flag）                     │  │
│   │                                                                    │  │
│   │  STANDBY_EXIT_REQ                                                  │  │
│   │  ─→ ui_msg_proc_standby.cpp:40                                    │  │
│   │     rena_infusion_state_standby_stop()                             │  │
│   │       ① history_event_save(format=2)                              │  │
│   │       ② 清零 standby_time_remain / standby_time                    │  │
│   │       ③ rena_standby_ui_exit() → 发 RENA_UI_STANDBY_EXIT_RSQ      │  │
│   │       ④ rena_state_change(&state_ready)  → 切回 ready 态           │  │
│   └────────┬───────────────────────────────────────────────────────────┘  │
│            │                                                              │
│   ┌────────┴───────────────────────────────────────────────────────────┐  │
│   │ 状态机 infusion_state.cpp                                            │  │
│   │                                                                    │  │
│   │  state_standby = {                                                  │  │
│   │    .state  = RENA_INFUSION_STATE_STANDBY,                           │  │
│   │    .entry  = rena_infusion_state_standby_entry,                     │  │
│   │    .process= rena_infusion_state_standby_process,                   │  │
│   │    .exit   = rena_infusion_state_standby_exit,                      │  │
│   │  }                                                                  │  │
│   │                                                                    │  │
│   │  entry():                                                           │  │
│   │    set_standby_time_remain(standby_time)   ← 初始 = 设定值          │  │
│   │    set_standby_time_remain(user_setting)   ← 同步到 user_setting    │  │
│   │    set_standby_start_tick(sys_uptime)                              │  │
│   │    (VP90) cancel TUBE_INSTALLATION / TUBE_PRESSURE 告警           │  │
│   │                                                                    │  │
│   │  process(): 每 100ms 调度器 tick 调一次                             │  │
│   │    time_used = (sys_uptime - start_tick) / 1000                     │  │
│   │    if (standby_time > time_used)                                    │  │
│   │        standby_time_remain = standby_time - time_used               │  │
│   │    else                                                             │  │
│   │        standby_time_remain = 0                                      │  │
│   │    同步到 user_setting                                               │  │
│   │                                                                    │  │
│   │  alarm_check.cpp:1028                                              │  │
│   │  rena_check_standby_finish()                                        │  │
│   │    if (remain > 0) || (standby_time == 0)  return                   │  │
│   │    rena_infusion_state_standby_stop()    ← 时间到，停待机           │  │
│   │                                                                    │  │
│   │  standby 结束 → 触发 RENA_ALARM_STANDBY_FINISH 告警                 │  │
│   │    → 切回 ready 态                                                 │  │
│   │    → surface 显示 alarm_standby_finish 弹窗                        │  │
│   └────────┬───────────────────────────────────────────────────────────┘  │
│            │                                                              │
│   ┌────────┴───────────────────────────────────────────────────────────┐  │
│   │ 周期通知 timer_task_thread.cpp:45                                    │  │
│   │   每 250ms（与 surface 渲染周期对齐）                                │  │
│   │   rena_get_user_setting()->send_nty()                               │  │
│   │     → 组装 rena_ui_user_setting_nty（包含 standby_time_remain）     │  │
│   │     → RENA_UI_USER_SETTING_NTY (IPC)                               │  │
│   └────────┬───────────────────────────────────────────────────────────┘  │
│            │                                                              │
└────────────┼──────────────────────────────────────────────────────────────┘
             │
             │  RENA_UI_USER_SETTING_NTY
             ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  surface 端接收                                                             │
│   msgproc/proc_notify.c:1118                                               │
│   rena_proc_ui_msg_user_setting_nty(data)                                  │
│     standby_time_remain = user_setting_nty->standby_time_remain           │
│   → 写到 global.c 的 rena_user_setting_params                             │
│                                                                           │
│   250ms 后 standby.c:171 standby_update_timer_cb 触发                      │
│     standby_time_set(standby_time_remain)  ← 读全局                        │
│     计算 h/m/s                                                            │
│     lv_label_set_text_fmt(label_standby_time, "%02d:%02d:%02d", ...)      │
│   → 用户看到倒计时刷新                                                     │
└───────────────────────────────────────────────────────────────────────────┘
```

**关键时序点**：

| 时间点 | 事件 | 涉及进程 |
|--------|------|---------|
| t0 | 用户点电源键/菜单待机 | surface |
| t1 | `RENA_UI_STANDBY_REQ` → scence | surface→scence |
| t2 | scence 状态机切到 STANDBY，记录 history，设 ui_flag | scence |
| t3 | `RENA_UI_STANDBY_RSQ` 回 surface，surface 切到 STANDBY 页面 | scence→surface |
| t4 | 用户在 standby 页面点"设置" | surface |
| t5 | `RENA_UI_KEYBOARD_REQ` → scence，scence 回 RSP 含 preset | 双向 |
| t6 | surface 加载 standby_time_kb_view（**bug 现场**） | surface |
| t7 | 用户拖滚轮 → 250ms 定时器回算 → 点确认 | surface |
| t8 | `RENA_UI_CONFIG_SET_REQ` → scence，携带 value（**bug 触发点**） | surface→scence |
| t9 | scence 写 DB + 启动 standby 状态机 | scence |
| t10 | scence 周期 250ms `USER_SETTING_NTY` 推 standby_time_remain | scence→surface |
| t11 | surface 定时器渲染倒计时 | surface |
| t12 | 倒计时归零 → scence 触发 STANDBY_FINISH 告警 → 切回 ready | scence |

---

## 四、待机启动与生效全流程（scence ↔ surface 跨进程分析）

> 本节梳理"按下待机按钮 → 倒计时 → 到时退出"完整链路上的全部关键节点，含两个进程的所有 IPC 消息、状态机切换、持久化、告警联动。

### 4.1 触发入口（surface）

```c
// 入口 1：长按电源键
// surface/src/ui/key_power.c:205-214
void rena_key_power_standby_event_cb(lv_event_t *e) {
    if (CLICKED) {
        rena_ui_standby_req standby_req;
        rena_send_msg(RENA_UI_STANDBY_REQ, &standby_req, ...);
    }
}

// 入口 2：下拉菜单点击"待机"图标
// surface/src/ui/dropdown_menu.c:1539-1548
static void rena_menu_standby_event_cb(lv_event_t *e) {
    if (CLICKED && rena_menu_page_check()) {
        rena_ui_standby_req standby_req;
        rena_send_msg(RENA_UI_STANDBY_REQ, &standby_req, ...);
    }
}
```

`rena_ui_standby_req` 结构体当前无字段（占位用），触发信号靠消息类型本身。

### 4.2 scence 接收并启动状态机

```cpp
// scence/src/ui_msg_proc_standby.cpp:13-38
void rena_proc_ui_msg_standby_req(void *data) {
    // 1. 守卫：运行态不允许进待机
    if (state >= RUNNING && state <= OCC_RESTART && state != STANDBY) return;

    // 2. 取消强制关机标志
    set_need_force_shutdown(0);

    // 3. 切状态机
    rena_infusion_state_standby_start();

    // 4. 写历史记录
    rena_history_event_save({format=1, type=STANDBY});

    // 5. 通知 surface 切到 STANDBY 页面
    rena_set_major_page_id(PAGE_MAJOR_STANDBY);
    rena_send_msg_to_ui(RENA_UI_STANDBY_RSQ, ...);
}
```

### 4.3 状态机 entry / process / exit

```cpp
// scence/src/infusion_state.cpp
void rena_infusion_state_standby_entry() {
    set_standby_time_remain(get_standby_time());        // 剩余 = 设定值
    user_setting->set_standby_time_remain(...);          // 同步缓存
    set_standby_start_tick(get_sys_uptime());            // 记起点
    #if VP90
        cancel_alarm(TUBE_INSTALLATION);
        cancel_alarm(TUBE_PRESSURE);
    #endif
}

void rena_infusion_state_standby_process() {
    if (get_standby_time() <= 0) return;
    time_used = (sys_uptime - start_tick) / 1000;        // 秒
    if (standby_time > time_used)
        set_standby_time_remain(standby_time - time_used);
    else
        set_standby_time_remain(0);
    user_setting->set_standby_time_remain(...);          // 同步
}

// 倒计时归零由 alarm_check 触发
// scence/src/alarm_check.cpp:1028
bool rena_check_standby_finish() {
    if (remain > 0 || standby_time == 0) return false;
    rena_infusion_state_standby_stop();
    return true;
}
```

`process()` 由 scence 调度器**每 100ms 调一次**（参见 `domain.cpp` 主循环），每 tick 重算一次剩余时间。

### 4.4 surface 接收 RSQ 并切页

```c
// surface/src/msgproc/proc_standby.c:6-13
void rena_proc_ui_msg_standby_rsp(void *data) {
    switch_menu_state(PAGE_MENU_STATE_COLLAPSE);
    switch_major_page(PAGE_MAJOR_NULL);
    switch_top_page(PAGE_TOP_NULL, NULL);
    switch_power_page(PAGE_POWER_NULL);
    switch_major_page(PAGE_MAJOR_STANDBY);   // ← 触发 standby.c:37 加载
}
```

`standby.c:standby_load()` 创建标签、定时器（250ms `standby_update_timer_cb`），定时器每次读 `rena_get_user_setting_params()->standby_time_remain` 并刷新 `label_standby_time` 显示。

### 4.5 进入待机时间设置（用户点 standby 页面"设置"按钮）

```c
// surface/src/ui/standby.c:148-158
void standby_setting_event_cb(lv_event_t *e) {
    if (CLICKED) {
        rena_ui_keyboard_req keyboard_req;
        keyboard_req.index = RENA_KB_INDEX_STANDBY_TIME_SETTING;
        rena_send_msg(RENA_UI_KEYBOARD_REQ, &keyboard_req, ...);
    }
}
```

scence 端处理 KEYBOARD_REQ，组装 `preset = standby_time_prev`（**注意：这里用 prev 不是 remain**），回 RSP。

surface 收到 RSP → `proc_keyboard.c:2784` → `switch_minor_page(PAGE_MINOR_KB_STANDBY_TIME_SETTING, &standby_time_kb_opt)` → 加载 `standby_time_kb_view.c:90`。

### 4.6 待机时间键盘（**bug 现场**）

详见第三节。这里只补充 surface→scence 的接口：

```c
// surface/src/ui/standby_time_kb_view.c:521-566 (修复后)
void standby_time_kb_confirm_event_cb(lv_event_t *e) {
    ...
    hour_val   = strtoul(..., NULL, 10);   // 修复后
    minute_val = strtoul(..., NULL, 10);   // 修复后
    second_val = strtoul(..., NULL, 10);   // 修复后
    value = hour_val * 3600 + minute_val * 60 + second_val;
    ...
    kb_param.confirm(&result);  // → rena_config_set_standby_time_confirm
}
```

```c
// surface/src/msgproc/proc_keyboard.c:3347-3364
void rena_config_set_standby_time_confirm(standby_time_kb_result_t *result) {
    rena_ui_config_set_req req;
    req.index = RENA_CONFIG_INDEX_STANDBY_TIME_SETTING;
    req.type  = RENA_CONFIG_TYPE_UINT32;
    req.is_valid = result->is_valid;
    req.config_value.uint32_value = result->value;   // 修复后 value 正确
    rena_send_msg(RENA_UI_CONFIG_SET_REQ, &req, ...);
}
```

### 4.7 scence 接收 CONFIG_SET_REQ 并持久化

```cpp
// scence/src/ui_msg_proc_config_set.cpp:5719-5744
void rena_proc_ui_config_set_standby_time(req, rsp) {
    uint32_t standby_time = req->config_value.uint32_value;

    user_setting->set_standby_time_prev(standby_time);  // ① user_setting
    domain_param->set_standby_time_prev(standby_time);  // ② domain
    domain_param->set_standby_time(standby_time);       // ③ domain（剩余初值）

    common_standby_time.set_uint32_value(standby_time);
    rena_set_common_setting(STANDBY_TIME, ...);          // ④ 写内存缓存
    rena_store_common_setting(STANDBY_TIME, ...);        // ⑤ 持久化到 DB

    rena_infusion_state_standby_start();                // ⑥ 切状态机
    rena_history_event_save({format=19, type=STANDBY});  // ⑦ 历史记录
}
```

> **修复前的隐藏 bug**：步骤 ① 取的 `result->value` 在用户设置 8/9 时为 0，scence 把 `standby_time=0` 写库 + 启动状态机。状态机 `process()` 看到 `standby_time <= 0` 直接 return，相当于**没进待机**，但页面可能已经被切到 STANDBY。**用户感知**："我点了 8min 但没倒计时"。

### 4.8 倒计时渲染（surface 定时器）

```c
// surface/src/ui/standby.c:171-209
void standby_update_timer_cb(lv_timer_t *timer) {
    standby_time_set(rena_get_user_setting_params()->standby_time_remain);
    // ↑ 这个值由 scence 通过 RENA_UI_USER_SETTING_NTY 周期推送
    //   scence/src/timer_task_thread.cpp:45 → user_setting.cpp:1008 send_nty()
    //   → user_setting.cpp:1235 发送 NTY
    //   surface/proc_notify.c:1118 接收 → 写 rena_user_setting_params

    if (ui.standby_time <= 0) {
        lv_label_set_text(label_standby, _("STR_STANDBY"));
        lv_label_set_text(label_standby_time, " ");
    } else {
        lv_label_set_text_fmt(label_standby_time, "%02d%s:%02d%s:%02d%s",
            ui.standby_time_hour, _("STR_HOUR"),
            ui.standby_time_min,   _("STR_MINUTE"),
            ui.standby_time_sec,   _("STR_SECOND"));
    }
}
```

### 4.9 退出路径

#### 主动退出（用户点 standby 页面"退出"按钮）

```c
// surface/src/ui/standby.c:160-169
void standby_exit_event_cb(lv_event_t *e) {
    if (CLICKED) {
        ui.standby_time = 0;
        rena_send_msg(RENA_UI_STANDBY_EXIT_REQ, ...);
    }
}
```

```cpp
// scence/src/ui_msg_proc_standby.cpp:40-43
void rena_proc_ui_msg_standby_exit_req(void *data) {
    rena_infusion_state_standby_stop();
}

// scence/src/infusion_state.cpp:1772-1789
int8_t rena_infusion_state_standby_stop() {
    history_event_save({format=2, type=STANDBY, object=STOP});
    set_standby_time_remain(0);
    user_setting->set_standby_time_remain(0);
    set_standby_time(0);
    rena_standby_ui_exit();                  // 发 RENA_UI_STANDBY_EXIT_RSQ
    rena_state_change(&state_ready);         // 切回 ready 态
}
```

```c
// surface/src/msgproc/proc_standby.c:15-18
void rena_proc_ui_msg_standby_exit_rsp(void *data) {
    switch_major_page(PAGE_MAJOR_NULL);
}
```

#### 被动退出（倒计时归零）

```cpp
// scence/src/alarm_check.cpp:1028-1036
bool rena_check_standby_finish() {
    if (remain > 0 || standby_time == 0) return false;
    rena_infusion_state_standby_stop();  // 走同一 stop 路径
    return true;
}
// 后续 alarm_manage 触发 RENA_ALARM_STANDBY_FINISH 高级告警
// surface 端收到 → 加载 alarm_standby_finish.c 弹窗
```

### 4.10 完整时序图（含进程边界）

```
surface                                                scence
───────                                                ──────
用户点待机按钮
   │
   │── RENA_UI_STANDBY_REQ ──────────────────────────►│
   │                                                  │ ① 守卫检查
   │                                                  │ ② rena_infusion_state_standby_start
   │                                                  │    ├─ entry: 初始 remain = 0（首次进入）
   │                                                  │ ③ history_event_save
   │                                                  │ ④ set_major_page_id
   │◄── RENA_UI_STANDBY_RSQ ──────────────────────────│
   │                                                  │
switch_major_page(STANDBY)                            │ 调度器 tick (100ms)
   │                                                  │ ├─ process: 算 remain
standby_update_timer_cb (250ms)                       │ │   同步到 user_setting
   │ 读 remain（首轮 = 0，显示"待机"无倒计时）        │ │
   │                                                  │ │
用户点"设置"按钮                                     │ │
   │                                                  │ │
   │── RENA_UI_KEYBOARD_REQ ────────────────────────►│ │
   │                                                  │ 组装 preset = standby_time_prev
   │◄── RENA_UI_KEYBOARD_RSP (preset) ───────────────│ │
   │                                                  │ │
switch_minor_page(KB_STANDBY_TIME)                    │ │
standby_time_kb_view_load                             │ │
   │ set_selected × 2 + roller 显示                  │ │
   │                                                  │ │
用户拖滚轮 → event_cb (base=10, 正常)                │ │
   │                                                  │ │
250ms 定时器 / 用户点确认                             │ │
   │ get_selected_str + strtoul  (★★★ 修复后 base=10)│ │
   │                                                  │ │
   │── RENA_UI_CONFIG_SET_REQ ──────────────────────►│ │
   │   index=STANDBY_TIME_SETTING                    │ ⑤ rena_proc_ui_config_set_standby_time
   │   value = h*3600+m*60+s (修复后正确)            │ │  ├─ set_standby_time_prev
   │                                                  │ │  ├─ set_standby_time
   │                                                  │ │  ├─ set + store common_setting → DB
   │                                                  │ │  ├─ rena_infusion_state_standby_start
   │                                                  │ │  │  ├─ entry: remain = 设定值
   │                                                  │ │  │  └─ process: 启动倒计时
   │                                                  │ │  └─ history_event_save
   │                                                  │ │
   │                                                  │ 调度器 tick (100ms)
   │                                                  │ ├─ process: 持续更新 remain
   │                                                  │ │
   │                                                  │ timer_task (250ms)
   │                                                  │ └─ send_nty → 打包 user_setting
   │◄── RENA_UI_USER_SETTING_NTY ────────────────────│
   │                                                  │
proc_notify → 写 standby_time_remain                  │
standby_update_timer_cb (250ms)                       │
   │ 读 remain，刷新 "Hh:Mm:Ss" 倒计时标签            │
   │                                                  │
   │    ... 倒计时中 ...                              │ ... 倒计时中 ...
   │                                                  │
   │                                                  │ process: remain 归零
   │                                                  │ alarm_check: 检测到 finish
   │                                                  │ ├─ rena_infusion_state_standby_stop
   │                                                  │ ├─ history_event_save(STOP)
   │                                                  │ ├─ 清零 remain
   │                                                  │ ├─ rena_state_change(ready)
   │                                                  │ └─ 触发 RENA_ALARM_STANDBY_FINISH
   │◄── RENA_UI_STANDBY_EXIT_RSQ ───────────────────│
   │                                                  │
   │◄── RENA_UI_ALARM_NTY (STANDBY_FINISH) ─────────│
   │                                                  │
switch_major_page(NULL)                               │
加载 alarm_standby_finish 弹窗                       │
```

---

## 五、修复方案

文件：`surface/src/ui/standby_time_kb_view.c`

6 处 `strtoul(..., NULL, 0)` → `strtoul(..., NULL, 10)`：

| 行号 | 函数 | 上下文 |
|------|------|--------|
| 401 | `standby_time_kb_view_update_timer_cb` | 250ms 定时器，刷新快捷按钮高亮 |
| 404 | 同上 | |
| 407 | 同上 | |
| 540 | `standby_time_kb_confirm_event_cb` | 用户点确认，提交最终值 |
| 543 | 同上 | |
| 546 | 同上 | |

diff：

```diff
-    hour_val   = strtoul((const char *)selected_str, NULL, 0);
+    hour_val   = strtoul((const char *)selected_str, NULL, 10);

-    minute_val = strtoul((const char *)selected_str, NULL, 0);
+    minute_val = strtoul((const char *)selected_str, NULL, 10);

-    second_val = strtoul((const char *)selected_str, NULL, 0);
+    second_val = strtoul((const char *)selected_str, NULL, 10);
```

不改 `"%02d"` 格式的原因：`dynamic_roller` 是公共组件，6 个文件共用，改格式会改变所有时间/日期滚轮的视觉（`"08"` 变 `" 8"`）。

---

## 六、验证

### 6.1 离线 strtoul 行为

```c
strtoul("08", NULL, 0) → 0   // 八进制陷阱
strtoul("09", NULL, 0) → 0
strtoul("08", NULL, 10) → 8  // 修复后
strtoul("09", NULL, 10) → 9
strtoul("18", NULL, 0)  → 18 // 无前导 0，自动十进制，OK
```

### 6.2 模拟器 / 板级（待补）

- 模拟器 `cd surface/simulator && make && ./build/bin/demo` 跑全部 case
- ARM `cd scence && mkdir -p out && cd out && cmake ../pc && make` 烧板回归

---

## 七、影响范围

| 维度 | 影响 |
|------|------|
| 文件 | 1 个 — `surface/src/ui/standby_time_kb_view.c` |
| 行数 | 6 行（`NULL, 0` → `NULL, 10`） |
| 接口 / UI / 性能 / IPC / DB / i18n | 全部无变化 |

---

## 八、遗留问题

| # | 项 | 风险 | 处置 |
|---|----|------|------|
| 1 | `standby_time_kb_view_load` 第一次 `set_selected` 用 preset，第二次用 `standby_time_remain` 覆盖。两次同步执行，第二次完全替代第一次 → **第一次是死代码** | 低（不影响功能） | 建议删 212-218 行，仅保留 220-225 行 |
| 2 | `sint_kb_view.c` / `uint_kb_view.c` / `sfloat_kb_view.c` 各 4-5 处 `strtoul(..., NULL, 0)` | 中（潜伏） | 当前不可触发（键入栈有去前导 0、预设用 `%u`/`%d` 无前导 0），下次重构时统一改 base=10 |
| 3 | `dynamic_roller.c:90` `end < 0` 分支 end_fmt 缺尾部 `\n` | 无功能影响 | 风格统一，下次顺手补 |
