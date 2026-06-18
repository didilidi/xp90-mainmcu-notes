# 锁屏系统工作流程说明

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-06-08 | 初稿，覆盖系统架构、消息协议、三大锁屏路径、状态机详解 |

---

## 一、系统架构总览

输液泵的锁屏系统运行在**两个独立进程**之间，通过 **System V 消息队列** 通信。

```
┌─────────────────────────────────────────────────────────────────┐
│                       scence 进程                                │
│                (应用逻辑层，C++，17 个线程)                       │
│                                                                  │
│   ┌──────────────────────┐  ┌──────────────────────────────┐    │
│   │ domain.cpp (主循环)   │  │ ui_msg_proc_menu.cpp        │    │
│   │ └─ 门传感器检测        │  │ └─ lock_screen_req 处理     │    │
│   │ └─ 泵门锁屏/解锁       │  │ └─ 状态同步                 │    │
│   └──────────────────────┘  └──────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│         System V 消息队列 (IPC key = 13579)                       │
│         type=1: surface → scence (REQ 请求)                      │
│         type=2: scence → surface (RSP 响应)                      │
│                           │                                      │
└───────────────────────────┼──────────────────────────────────────┘
                            │
┌───────────────────────────┼──────────────────────────────────────┐
│                           ▼                                      │
│                       surface 进程                                │
│                  (LVGL GUI 层，2 个线程)                          │
│                                                                  │
│   ┌────────────────┐    ┌────────────────┐    ┌──────────────┐  │
│   │ main.c (主线程)  │    │ msg_thread.c   │    │ lock.c       │  │
│   │ lv_task_handler │◄──►│ (消息线程)      │    │ (300ms 状态机)│  │
│   │ 1ms 循环        │    │ 收→处理→发     │    └──────┬───────┘  │
│   │ lvgl_mutex 保护  │    │ 100ms 循环      │           │         │
│   └────────────────┘    └────────────────┘           ▼           │
│                                              ┌──────────────┐   │
│                                              │ lock_view.c  │   │
│                                              │ (锁屏 UI)    │   │
│                                              └──────────────┘   │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│   │ proc_menu.c  │  │ manager.c    │  │ page_event_cb        │  │
│   │ (消息处理)    │  │ (页面管理)    │  │ (页面触摸事件回调)    │  │
│   └──────────────┘  └──────────────┘  └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.1 进程职责

| 进程 | 语言 | 线程数 | 职责 |
|------|------|--------|------|
| **scence** | C++14 | 17 pthread | 应用逻辑、输液状态、传感器、报警、数据库 |
| **surface** | C (LVGL 8.3) | 2 pthread | 图形界面渲染、触控交互、UI 页面管理 |

### 1.2 surface 进程的双线程模型

**文件：** `surface/src/main.c`，`surface/src/domain/msg_thread.c`

| 线程 | 函数 | 循环周期 | 职责 |
|------|------|----------|------|
| **主线程** | `main()` 中的 `while(1)` | 1ms | `lv_task_handler()` → LVGL 渲染 + 定时器回调 |
| **消息线程** | `msg_thread()` | 100ms | 收发 IPC 消息，处理 scence 响应 |

两个线程通过 `lvgl_mutex` 互斥，确保不同时操作 LVGL 对象：

```
主线程: lock(lvgl_mutex) → lv_task_handler() → unlock(lvgl_mutex) → sleep(1ms)
消息线程: lock(lvgl_mutex) → rena_proc_msg() → unlock(lvgl_mutex) → sleep(100ms)
```

---

## 二、通信协议 — System V 消息队列

### 2.1 队列属性

| 属性 | 值 |
|------|-----|
| 队列 key | `13579` |
| 消息类型 `type=1` | surface → scence（请求 REQ） |
| 消息类型 `type=2` | scence → surface（响应 RSP） |
| payload 大小 | 3072 字节 |

**消息结构体（`msg_thread.c`）：**
```c
struct msg_ui {
    long type;             // 1 = TO_CORE, 2 = TO_UI
    char payload[3072];    // 序列化的 rena_ui_msg_t
};
```

### 2.2 锁屏相关消息定义

**文件：** `surface/src/domain/msg_def.h`

```c
// ─── 锁屏请求 (surface → scence) ───
#define RENA_UI_LOCK_SCREEN_REQ  0x0150

typedef struct {
    bool result;          // true=锁屏, false=解锁
    bool is_auto_lock;    // 是否是自动锁屏触发（菜单按钮锁屏=false）
    bool is_door_lock;    // 当前未使用
} rena_ui_lock_screen_req;

// ─── 锁屏响应 (scence → surface) ───
#define RENA_UI_LOCK_SCREEN_RSQ  0x0151

typedef struct {
    uint8_t result;       // 0=非自动锁屏, 1=自动锁屏（来自 is_auto_lock）
    rena_ui_flag ui_flag; // 页面标志
    bool is_door_lock;    // 是否是泵门触发
} rena_ui_lock_screen_rsp;

// ─── 锁屏时间设置 ───
#define RENA_UI_LOCK_TIME_SET_REQ    0x0230
#define RENA_UI_LOCK_TIME_SET_RSP    0x0231
#define RENA_UI_LOCK_TIME_SET_VIEW_REQ  0x0232
#define RENA_UI_LOCK_TIME_SET_VIEW_RSP  0x0233
```

### 2.3 消息类型汇总

| 方向 | 消息 ID | 时机 | 负载 |
|------|---------|------|------|
| surface→scence | `LOCK_SCREEN_REQ` | 空闲超时/用户主动锁屏/用户解锁 | result, is_auto_lock |
| scence→surface | `LOCK_SCREEN_RSQ` | 泵门开/关、响应 REQ | result, is_door_lock |
| surface→scence | `LOCK_TIME_SET_REQ` | 用户修改自动锁屏时间 | screen_lock_time |
| scence→surface | `LOCK_TIME_SET_RSP` | 保存成功 | result, ui_flag |

### 2.4 surface 侧消息收发流程

```
                消息线程 (msg_thread)
                      │
                      │  msgrcv(type=2, IPC_NOWAIT)
                      ▼
            ┌─────────────────────┐
            │ 有消息？            │←── 没有消息 → break（进入发送阶段）
            └─────────┬───────────┘
                      │ 有
                      ▼
        lvgl_mutex.lock()
              │
              ▼
        rena_proc_msg(payload)
              │
              ▼  dispatch by msg_type ───→  proc_menu.c: rena_proc_ui_msg_lock_screen_rsp()
                                              │
                                              ├─ rena_non_auto_lock_screen_proc() → lock.c
                                              ├─ switch_menu_state()
                                              └─ rena_set_lock_view_foreground()
        lvgl_mutex.unlock()
              │
              ▼
          继续收下一批
                      │
                      ▼
            ┌─────────────────────┐
            │ 发送队列有消息？    │
            └─────────┬───────────┘
                      │ 有
                      ▼
        msg.type = 1 (TO_CORE)
        msgsnd(qid, IPC_NOWAIT)
```

**`rena_send_msg()` 函数（安全地在持锁期间调用）：**
```c
void rena_send_msg(msg_type, body, len) {
    msg = malloc(sizeof(rena_ui_msg_t));
    memcpy(msg->body, body, len);
    TsQueue.push(send_msg_queue, msg);  // 只入队，不发 IPC
}
```

消息只入队列，实际的 `msgsnd` 在消息线程释放 `lvgl_mutex` 后异步发送，避免在持锁期间阻塞。

---

## 三、锁屏核心数据结构

### 3.1 scence 侧 — 线程安全原子变量

**文件：** `scence/include/domain_param.h`

```cpp
std::atomic<bool> is_screen_lock;        // 是否锁屏
std::atomic<bool> is_door_lock_screen;   // 是否是泵门触发的锁屏
```

### 3.2 scence 侧 — 用户设置

**文件：** `scence/include/user_setting.h`

```cpp
uint16_t screen_lock_time;   // 自动锁屏时间(秒)，0=关闭，默认300(5分钟)
```

### 3.3 surface 侧 — 全局变量

```c
// user_setting.c
static bool screen_is_locked = false;    // 屏幕锁状态 (user_setting.c:9)

// page.c
bool door_lock_screen = false;           // 泵门锁屏标志 (page.c:23)

// user_setting.h
uint16_t screen_lock_time;               // 自动锁屏超时(秒) (user_setting.h:185)
```

### 3.4 surface 侧 — lock.c 内部状态

```c
static lv_timer_t *page_lock_timer;      // 300ms LVGL 定时器
static lv_obj_t *lock_view;              // 锁屏 overlay LVGL 对象指针
static uint32_t touch_tick;              // 最近一次触摸时间戳
static uint32_t prev_time;               // 上一次定时器运行时间（时间跳变检测）
static bool pre_lock_state;              // 上一次检测到的锁状态
```

### 3.5 scence ↔ surface 状态对应关系

| scence | surface | 说明 |
|--------|---------|------|
| `is_screen_lock` | `screen_is_locked` | 锁屏布尔状态 |
| `is_door_lock_screen` | `door_lock_screen` | 是否门锁触发 |
| `screen_lock_time` | `screen_lock_time` | 自动锁屏超时 |

**注意：** 两个进程各自维护一份状态，通过消息队列同步。不是所有路径都会同步双方状态（例如泵门触发锁屏时 scence 直接发 RSP，surface 在 `rena_set_door_lock_screen(true)` 时也没有再发 REQ 回 scence）。

---

## 四、三大锁屏路径详解

### 路径 A：泵门自动锁屏 / 解锁

**触发条件：** 泵门传感器状态变化

**调用链：**

```
scence domain.cpp 主循环 (每帧)
    │
    ├─ rena_domian_check_door_close() → 读取传感器
    │
    ├─ 关门→开门: 判断 should_lock
    │   ├─ SP90: !get_is_screen_lock()
    │   ├─ VP90: !is_running && !get_is_screen_lock()
    │   └─ 满足条件:
    │       ├─ set_is_screen_lock(true)
    │       ├─ set_is_door_lock_screen(true)
    │       ├─ 记录历史事件
    │       └─ send_msg_to_ui(RSQ, result=0, is_door_lock=true)
    │
    └─ 开门→关门: 判断 get_is_door_lock_screen()
        └─ 满足条件:
            ├─ set_is_screen_lock(false)
            ├─ set_is_door_lock_screen(false)
            ├─ 记录历史事件
            └─ send_msg_to_ui(RSQ, result=0, is_door_lock=true)
               ▲
               │ System V 消息队列
               │
surface msg_thread
    │
    └─ rena_proc_ui_msg_lock_screen_rsp()
        │
        ├─ result == 0 (非自动锁屏)
        │   ├─ 当前未锁 → 锁屏
        │   │   ├─ set_door_lock_screen(true)
        │   │   ├─ rena_lock_screen()
        │   │   │   ├─ set_screen_lock(true)
        │   │   │   ├─ pre_lock_state = true
        │   │   │   └─ lv_obj_create + lv_obj_move_foreground + lock_view_load()
        │   │   ├─ switch_menu_state(COLLAPSE)
        │   │   └─ rena_set_lock_view_foreground()   ← 保证 overlay 在最前
        │   │
        │   └─ 当前已锁 → 解锁
        │       └─ rena_non_auto_lock_screen_proc(false) → rena_unlock_screen()
        │
        └─ result == 1 → 错误处理（忽略）
```

**关键特点：**
- scence **主动**发消息，不需要 surface 先请求
- `result=0` 表示"非自动锁屏"，通知 surface 调用 `rena_non_auto_lock_screen_proc()`
- `is_door_lock=true` 用于区分门锁和普通锁屏

---

### 路径 B：空闲超时自动锁屏

**触发条件：** `screen_lock_time > 0` 且触摸空闲时间超过设定值

**调用链：** 完全在 surface 侧的 `lock.c` 300ms 定时器内完成

```
300ms LVGL 定时器 (lock.c: rena_page_lock_timer_cb)
    │
    ├─ [前置] 时间跳变检测: last_run - prev_time > 1500ms → 重置 touch_tick
    │
    ├─ ① 高级报警 + 非门锁 → 解锁 + 发 REQ(false) 通知 scence
    │
    ├─ ② 非运行/KVO + 非门锁 → 解锁 (离开运行页面时自动解除锁屏)
    │
    ├─ ③ 已解锁 + pre_lock_state != current_lock_state → 解锁 + 发 REQ(false)
    │
    ├─ ④ 已解锁 + 空闲超时 + screen_lock_time > 0 → 锁屏 + 发 REQ(true)
    │
    └─ ⑤ 已锁定 → 刷新 touch_tick (防空闲计时器误触发)
```

**空闲超时逻辑（分支④）：**
```c
if (rena_get_screen_lock_time() > 0) {         // 仅当自动锁屏开启
    curr_time = page_lock_timer->last_run;
    if (curr_time - touch_tick >= rena_get_screen_lock_time() * 1000) {
        rena_lock_screen();                    // 锁屏
        rena_send_msg(REQ, {result=true, is_auto_lock=true});
    }
}
```

锁屏后 surface 发 `REQ` 给 scence，scence 确认后回复 `RSQ`（但不影响 UI 侧的锁屏效果，锁屏已执行）。

**触摸刷新（`page_event_cb` in manager.c）：**
```c
void page_event_cb(lv_event_t *e) {
    if (触摸事件) {
        rena_refresh_lock_prev_time();   // touch_tick = 当前时间
        send_msg(RENA_UI_TOUCH_NTY);     // 通知 scence 有触摸
    }
}
```

---

### 路径 C：用户主动锁屏 / 解锁

**触发条件：** 用户点击下拉菜单的"锁屏"按钮，或按功耗键

**调用链：**

```
用户点击"锁屏"按钮 (dropdown_menu.c / proc_key_power.c)
    │
    └─ rena_send_msg(REQ, {result=true, is_auto_lock=false})
        │
        ▼  System V 消息队列
        │
scence ui_msg_proc_menu.cpp: rena_proc_ui_msg_lock_screen_req()
    │
    ├─ [VP90/SP90] 门锁状态下的解锁请求 → 直接清除, 不回复
    │
    ├─ bolus/purge 状态 → 拒绝（直接 return）
    │
    ├─ 非 running/kvo 状态
    │   ├─ 请求锁屏 → 弹警告 "非运行状态，无法锁屏"
    │   └─ 请求解锁 → 拒绝
    │
    ├─ 高级报警中 → 拒绝锁屏
    │
    ├─ 执行锁屏/解锁
    │   ├─ set_is_screen_lock(result)
    │   └─ 记录历史事件
    │
    └─ 回复 RSQ
        ├─ result = is_auto_lock ? 1 : 0
        └─ is_door_lock = req->is_auto_lock  ← 注意：此处 is_door_lock 被误赋为 is_auto_lock
```

surface 收到 `RSQ` 后，由 `proc_menu.c` 处理，与路径 A 相同的处理函数。

**注意：** scence 的 `ui_msg_proc_menu.cpp:103` 有一处疑似 bug：
```cpp
lock_screen_rsp.is_door_lock = lock_screen_req->is_auto_lock;
```
`is_door_lock` 被赋值为请求中的 `is_auto_lock` 字段，语义上混淆。但由于 surface 侧只有在 `result=0` 且 `is_door_lock=true` 时才做门锁标记，而自动锁屏路径 `result=1`，所以实际不影响泵门锁屏功能。

---

## 五、surface 侧锁屏状态机详解

### 5.1 位置与频率

- **文件：** `surface/src/ui/lock.c`
- **函数：** `rena_page_lock_timer_cb()`
- **触发：** 每 300ms（`lv_timer_create(rena_page_lock_timer_cb, 300, NULL)`）

### 5.2 状态变量作用

| 变量 | 初始化于 | 更新于 | 用途 |
|------|----------|--------|------|
| `pre_lock_state` | `rena_lock_view_init()` → false | `rena_lock_screen()` → true, `rena_unlock_screen()` → false | 检测状态转换（锁↔解锁） |
| `touch_tick` | `rena_lock_view_init()` → `last_run` | `rena_refresh_lock_prev_time()` | 空闲超时计时起点 |
| `prev_time` | `rena_lock_view_init()` → `last_run` | 每次定时器回调 | 时间跳变检测 |
| `lock_view` | `rena_lock_screen()` → new lv_obj | `rena_unlock_screen()` → NULL | 管理 LVGL 对象生命周期 |

### 5.3 分支逻辑流程图

```
rena_page_lock_timer_cb()
    │
    ├── 时间跳变检测 (last_run - prev_time > 1500ms)
    │   └─ 是 → touch_tick = last_run (防止系统时间修改导致锁屏)
    │
    ├── prev_time = t->last_run
    │
    ├── 分支①: 高级报警 + 非门锁
    │   ├─ 条件: alarm_type == HIGH && door_lock_screen == false
    │   └─ 动作: rena_unlock_screen() + 发 REQ(false) → return
    │
    ├── 分支②: 非运行/KVO + 非门锁
    │   ├─ 条件: state != RUNNING && state != KVO && door_lock_screen == false
    │   └─ 动作: rena_unlock_screen() → return
    │
    ├── 当前解锁 (!current_lock_state)?
    │   ├── 是:
    │   │   ├── 分支③: pre_lock_state != current_lock_state (状态变化)
    │   │   │   └── rena_unlock_screen() + 发 REQ(false) → return
    │   │   │
    │   │   └── 分支④: screen_lock_time > 0 (空闲超时锁屏)
    │   │       ├─ 条件: curr_time - touch_tick >= screen_lock_time * 1000
    │   │       └─ 动作: rena_lock_screen() + 发 REQ(true)
    │   │
    │   └── 否 (当前锁定):
    │       └── 分支⑤: rena_refresh_lock_prev_time()
```

---

## 六、锁屏 UI 层执行流程

### 6.1 文件位置

| 文件 | 内容 |
|------|------|
| `surface/src/ui/lock_view.c` | 锁屏 UI 控件创建/销毁 |
| `surface/src/ui/lock_view.h` | lock_view_load / lock_view_unload 声明 |

### 6.2 lock_view_load — 创建锁屏 UI

**调用者：** `lock.c:rena_lock_screen()`

```
rena_lock_screen()
    ├─ set_screen_lock(true)
    ├─ pre_lock_state = true
    ├─ lock_view = lv_obj_create(lv_scr_act())   // 创建 overlay 容器
    │   └── lv_obj_move_foreground(lock_view)    // 移到最前
    └─ lock_view_load(lock_view)                  // 填充 UI 控件
        │
        ├─ 重置状态标志: is_entering/entered/exiting/exited_unlock
        ├─ 清空 ui 结构体
        │
        ├─ 容器: position(0,0), size(1424,280)
        │   ├─ 解锁区域 (417,108, 491,122) - 青色圆角矩形
        │   │   ├─ 方向箭头 ×3 (流动动画提示向右滑动)
        │   │   ├─ 虚线圆 + 解锁图标 (右侧)
        │   │   ├─ 滑条 (30,21, 400,80)
        │   │   │   ├─ range: 0-100, 初始值: 8
        │   │   │   ├─ style_main → 圆角
        │   │   │   └─ style_knob → 圆形 + 锁图标
        │   │   └─ 无限循环方向箭头动画 (1500ms)
        │   │
        │   ├─ 锁图标区域 (1300,102, 116,116) - 圆形
        │   │   └─ locker 图片
        │   │
        │   └─ 5 秒退出定时器
        │
        └─ 注册事件回调
            ├─ rena_lock_view_event_cb → 触摸锁屏区域 → 入场动画 + 重置退出定时器
            └─ rena_slider_unlocker_event_cb → 滑条交互
```

### 6.3 滑动解锁交互

```
用户触摸滑条 → LV_EVENT_PRESSED 或 DRAGGING
    │
    ├─ is_entered_unlock == true (初始状态)
    │   └─ 跳过入场动画，UI 可见
    │
    ├─ 重置 5 秒退出定时器
    │
    ├─ 用户拖动 → LV_EVENT_VALUE_CHANGED
    │   └─ 值 < 8 时钳位到 8
    │
    └─ 用户松手 → LV_EVENT_RELEASED
        │
        ├─ 滑条值 < 100:
        │   └─ 弹回初始值 8
        │
        └─ 滑条值 ≥ 100:
            ├─ 删除退出定时器
            ├─ 发 RENA_UI_TOUCH_NTY (通知 scence 有触摸)
            ├─ set_screen_lock(false) (surface 侧设为未锁)
            └─ 创建新的 5 秒退出定时器
                └─ 5 秒后 → 播放淡出动画 (500ms)
                    └─ 设置 is_exited_unlock = true
```

### 6.4 lock_view_unload — 销毁锁屏 UI

**调用者：** `lock.c:rena_unlock_screen()`

```c
void lock_view_unload(lv_obj_t *page) {
    lv_obj_remove_event_cb(ui.slider_unlocker, rena_slider_unlocker_event_cb);
    lv_obj_remove_event_cb(page, rena_lock_view_event_cb);
    lv_timer_del(exit_unlock_timer);  // 删除退出定时器
    lv_anim_del(&ui.lock_path_anim);  // 删除方向箭头动画
    lv_style_reset(&style_main);      // 清理样式
    lv_style_reset(&style_knob);
}
```

`rena_unlock_screen()` 在 unload 之后调用 `lv_obj_del(lock_view)` 彻底删除 LVGL 对象，并将 `lock_view = NULL`、`pre_lock_state = false`。

---

## 七、完整场景流程演示

### 7.1 场景：泵门开锁屏 → 滑动解锁 → 泵门关

```
时间          scence                           surface
───           ──────                           ───────
T0            domain.cpp 主循环
              ├─ 检测到门: close→open
              ├─ should_lock=true (SP90/VP90非运行)
              ├─ is_screen_lock=true
              ├─ is_door_lock_screen=true
              └─ 发 RSQ(result=0, door=1)
                │
                │  ─── System V 消息队列 ───►
                │
T1                                          msg_thread 收到 RSQ
                                            lvgl_mutex.lock()
                                            proc_menu.c:
                                            ├─ set_door_lock_screen(true)
                                            ├─ rena_lock_screen()
                                            │   ├─ set_screen_lock(true)
                                            │   ├─ pre_lock_state=true
                                            │   ├─ lv_obj_create(lv_scr_act())
                                            │   └─ lock_view_load()
                                            ├─ switch_menu_state(COLLAPSE)
                                            └─ rena_set_lock_view_foreground()
                                            lvgl_mutex.unlock()
                                            
                                            主线程: lv_task_handler()
                                            ├─ 渲染 lock_view UI ✓
                                            └─ 300ms 定时器运行
                                                ├─ screen_lock_time=0
                                                ├─ 分支①: 非高级报警 → 跳过
                                                ├─ 分支②: 非运行+KVO+门锁=true
                                                │   └─ door_lock==true → 跳过
                                                ├─ ③④: 当前锁定 → 进入分支⑤
                                                └─ 分支⑤: 刷新 touch_tick ✓
T2      用户触摸滑条 → 拖动到 100 → 松手
                                            lock_view.c:
                                            ├─ rena_set_screen_lock(false)
                                            ├─ 发 RENA_UI_TOUCH_NTY
                                            └─ 启动 5s 退出定时器
                                            
                                            300ms 定时器:
                                            当前解锁 + pre_lock=true != false
                                            → 分支③: rena_unlock_screen()
                                            │   ├─ set_screen_lock(false)
                                            │   ├─ set_door_lock_screen(false)
                                            │   ├─ lock_view = NULL
                                            │   └─ pre_lock_state = false
                                            └─ 发 REQ(result=false)
                │
                │  ◄─── System V 消息队列 ───
                │
T3   ui_msg_proc_menu.cpp:
    ├─ is_door_lock_screen=true
    ├─ !result=true → 清除
    ├─ is_screen_lock=false
    ├─ is_door_lock_screen=false
    └─ 记录历史事件 ✓

T4      用户关泵门

    domain.cpp 主循环
    ├─ 检测到门: open→close
    ├─ is_door_lock_screen=false (已清除)
    └─ 条件不满足 → 跳过 (不会重复发消息)
```

### 7.2 场景：空闲超时自动锁屏

```
时间          scence                           surface
───           ──────                           ───────
T0      输液运行中, screen_lock_time=60s
                                            用户触摸 → page_event_cb
                                            ├─ rena_refresh_lock_prev_time()
                                            │  touch_tick = 当前时间
                                            └─ 发 RENA_UI_TOUCH_NTY

T0+60s  无触摸 → 300ms 定时器:
                                            ├─ 分支④: screen_lock_time>0
                                            ├─  idle_time = T0+60s - T0 = 60s = 锁屏时间
                                            ├─ rena_lock_screen()
                                            └─ 发 REQ(result=true, is_auto_lock=true)
                │
                │  ─── System V 消息队列 ───►
                │
T1   ui_msg_proc_menu.cpp:
    ├─ result=true → 请求锁屏
    ├─ 非 bolus/purge → 继续
    ├─ running 状态 → 继续
    ├─ 非高级报警 → 继续
    ├─ set_is_screen_lock(true)
    ├─ 记录历史事件
    └─ 回复 RSQ(result=1)
                │
                │  ◄─── System V 消息队列 ───
                │
                                            proc_menu.c:
                                            result==1 → 错误处理分支（忽略）
                                            (锁屏已在 surface 侧完成)
```

---

## 八、安全机制

### 8.1 高级报警自动解锁

**位置：** `lock.c:72-86` + `scence/src/alarm_check.cpp:2709-2726`

当 `alarm_type == HIGH` 且当前锁屏不是泵门触发时，自动解锁：

```
300ms 定时器:
├─ alarm_type == HIGH && door_lock_screen == false
│   └─ current_lock_state == true → rena_unlock_screen()
│       └─ 发 REQ(result=false, is_auto_lock=true)
│           └─ scence: set_is_screen_lock(false)
```

**设计理由：** 高级报警（如堵塞、气泡、门系统错误）需要用户立即操作，不能因为锁屏而阻碍。

### 8.2 非运行状态自动解锁

**位置：** `lock.c:96-104`

当输液不在 `RUNNING` 或 `KVO` 状态且不是泵门触发时，自动解锁。确保用户离开运行页面后不需要额外解锁。

### 8.3 门锁状态下禁止菜单解锁

**位置：** `scence/src/ui_msg_proc_menu.cpp:38-53`

```cpp
// VP90/SP90: 泵门锁屏状态下，只响应解锁请求
if (is_door_lock_screen && !result) {
    set_is_screen_lock(false);
    set_is_door_lock_screen(false);
    return;  // 不回 surface，避免 toggle
}
```

门锁屏只能通过关门解除，防止用户通过菜单/功耗键解锁门锁状态，确保安全策略。

### 8.4 Bolus/Purge 拒绝锁屏

**位置：** `scence/src/ui_msg_proc_menu.cpp:55-59`

Bolus（快速推注）和 Purge（冲洗）是短时关键操作，不允许锁屏干扰。

### 8.5 状态机时间跳变保护

**位置：** `lock.c:57-61`

```c
if (t->last_run - prev_time > 1500) {
    touch_tick = t->last_run;  // 防止系统时间修改导致立即锁屏
}
prev_time = t->last_run;
```

---

## 九、关键函数调用关系图

```
rena_proc_ui_msg_lock_screen_rsp()          proc_menu.c
  │
  ├─ rena_non_auto_lock_screen_proc(bool)   lock.c
  │   │
  │   ├─ true → rena_lock_screen()
  │   │   ├─ rena_set_screen_lock(true)
  │   │   ├─ lv_obj_create()
  │   │   ├─ lv_obj_move_foreground()
  │   │   └─ lock_view_load()
  │   │       ├─ lv_slider_create()
  │   │       ├─ lv_anim_start()            ← 方向箭头动画
  │   │       └─ lv_timer_create()          ← 5s 退出定时器
  │   │
  │   └─ false → rena_unlock_screen()
  │       ├─ rena_set_screen_lock(false)
  │       ├─ rena_set_door_lock_screen(false)
  │       ├─ lock_view_unload()
  │       ├─ lv_obj_del()
  │       └─ pre_lock_state = false
  │
  ├─ switch_menu_state()
  └─ rena_set_lock_view_foreground()

rena_page_lock_timer_cb()                   lock.c (300ms)
  │
  ├─ 时间跳变检测
  ├─ 分支①: 高级报警 → rena_unlock_screen() + send_msg(REQ)
  ├─ 分支②: 非运行/KVO → rena_unlock_screen()
  ├─ 分支③: 状态变化 → rena_unlock_screen() + send_msg(REQ)
  ├─ 分支④: 空闲超时 → rena_lock_screen() + send_msg(REQ)
  └─ 分支⑤: 已锁定 → rena_refresh_lock_prev_time()

rena_slider_unlocker_event_cb()             lock_view.c (滑条)
  │
  ├─ LV_EVENT_RELEASED:
  │   ├─ 值 ≥ 100 → rena_set_screen_lock(false)
  │   │           → send_msg(TOUCH_NTY)
  │   │           → 重启 5s 退出定时器
  │   └─ 值 < 100 → 弹回 8
  │
  └─ LV_EVENT_VALUE_CHANGED:
      └─ 值 < 8 → 钳位到 8

rena_exit_unlock_timer_cb()                 lock_view.c (5s 超时)
  └─ 播放 500ms 淡出动画 → is_exited_unlock = true

rena_proc_ui_msg_lock_screen_req()          scence/ui_msg_proc_menu.cpp
  │
  ├─ 门锁清除检测
  ├─ bolus/purge 拒绝
  ├─ 非运行/KVO 拒绝
  ├─ 高级报警拒绝
  ├─ set_is_screen_lock()
  └─ send_msg_to_ui(RSQ)

scence_domain_door_process()                scence/domain.cpp
  │
  ├─ 开门 → 判断 should_lock → RSQ(result=0, door=1)
  └─ 关门 → 判断 is_door_lock_screen → RSQ(result=0, door=1)
```

---

## 十、相关文件清单

### surface 侧

| 文件路径 | 作用 |
|----------|------|
| `surface/src/ui/lock.c` | 锁屏状态机（300ms 定时器）、锁屏/解锁操作入口 |
| `surface/src/ui/lock.h` | lock.c 公共函数声明 |
| `surface/src/ui/lock_view.c` | 锁屏 UI 控件（滑条、箭头、锁图标） |
| `surface/src/ui/lock_view.h` | lock_view.c 公共函数声明 |
| `surface/src/ui/lock_time_view.c` | 自动锁屏时间设置页面 |
| `surface/src/ui/manager.c` | 页面管理、page_event_cb（触摸刷新锁屏计时）、rena_set_lock_view_foreground |
| `surface/src/msgproc/proc_menu.c` | 锁屏响应处理函数 |
| `surface/src/msgproc/proc_machine_setting.c` | 锁屏时间设置响应处理 |
| `surface/src/msgproc/proc_key_power.c` | 功耗键锁屏处理 |
| `surface/src/domain/msg_thread.c` | 消息线程（IPC 收发） |
| `surface/src/domain/send_msg.c` | 消息发送队列（TsQueue） |
| `surface/src/domain/msg_def.h` | 消息结构体定义 |
| `surface/src/domain/user_setting.c` | screen_is_locked、screen_lock_time 存取 |
| `surface/src/domain/page.c` | door_lock_screen 标志 |
| `surface/src/domain/page.h` | 页面 ID 枚举、门锁屏 API |

### scence 侧

| 文件路径 | 作用 |
|----------|------|
| `scence/src/domain.cpp` | 泵门传感器检测、开门锁屏/关门解锁 |
| `scence/src/ui_msg_proc_menu.cpp` | 锁屏请求处理（状态验证、历史记录） |
| `scence/src/ui_msg_proc_lock_time.cpp` | 锁屏时间设置处理 |
| `scence/src/ui_msg_process.cpp` | 消息路由注册 |
| `scence/src/alarm_check.cpp` | 高级报警自动解锁 |
| `scence/include/domain_param.h` | is_screen_lock、is_door_lock_screen |
| `scence/include/user_setting.h` | screen_lock_time |
| `scence/include/ui_msg_def.h` | 消息类型/结构体定义 |
