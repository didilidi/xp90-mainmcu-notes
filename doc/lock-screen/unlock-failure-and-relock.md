# 锁屏偶发解锁失败与重锁问题分析修复报告

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-06-09 | 初稿，覆盖三处修复的根因分析、方案演变、验证结果 |

---

## 一、问题清单

| ID | 问题描述 | 严重性 | 状态 |
|----|---------|--------|------|
| **P1** | 自动锁屏关闭时打开泵门锁屏卡死（screen_lock_time=0 时早期 return 跳过整个状态机） | 高 | ✅ 已修复（Session 2026-06-08） |
| **P2** | 偶发：开门锁屏后滑动解锁无效，关门不解锁，重新开门又恢复 | 中 | ✅ 已修复 |
| **P3** | 滑动解锁后关门又出现锁屏（P2 修复方案错误引入） | 高 | ✅ 已修复 |
| **P4** | 自动锁屏设置操作卡死（P3 次生问题） | 中 | ✅ 已修复 |

---

## 二、系统架构速览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          scence 进程（C++14）                           │
│  17 线程，其中 domain.cpp 主循环、ui_msg_proc_menu.cpp 消息处理         │
│                                                                         │
│  is_screen_lock（原子 bool）   is_door_lock_screen（原子 bool）          │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ System V 消息队列 (key=13579)
                               │ type=1: surface → scence (REQ)
                               │ type=2: scence → surface (RSQ)
┌──────────────────────────────┴──────────────────────────────────────────┐
│                        surface 进程（LVGL C）                            │
│  双线程：主线程(lv_task_handler) + 消息线程(msg_thread), lvgl_mutex 保护  │
│                                                                         │
│  screen_is_locked（全局 bool）  door_lock_screen（全局 bool）             │
│  pre_lock_state（静态 bool）     lock_view（lv_obj_t 指针）              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 三、P2 根因分析

### 3.1 症状描述

- 偶发，无固定复现比例
- 打开泵门后锁屏，但无法通过滑动解锁
- 关上门仍然不解锁
- 重新开门后恢复正常

### 3.2 数据流追踪

**正常路径（开门→锁屏→关门→解锁）：**

```
scence domain.cpp                  surface proc_menu.c
  │                                  │
  ├─ 开门检测 → is_screen_lock=T     │
  │  is_door_lock_screen=T           │
  │  RSQ(result=0, door=1) ─────►    ├─ result==0 → screen未锁 → 锁屏
  │                                  │  door_lock_screen=T
  ├─ 关门检测 → is_door_lock_screen   │
  │  =T → is_screen_lock=F           │
  │  is_door_lock_screen=F           │
  │  RSQ(result=0, door=1) ─────►    ├─ result==0 → 已锁 → 解锁
  │                                  │  door_lock_screen=F
```

### 3.3 问题定位

**偶发触发路径（滑动解锁 + 非运行态）：**

```
1. 开门 → scence RSQ(0, door=1) → surface 锁屏
   scence: is_screen_lock=T, is_door_lock_screen=T
   surface: screen_is_locked=T, door_lock_screen=T

2. 用户滑动解锁 → rena_set_screen_lock(false) ← 只清布尔值
   door_lock_screen 保持 true, pre_lock_state 保持 true

3. 定时器分支③ → pre(T)≠current(F) → rena_unlock_screen()
   清 door_lock_screen, 删 lock_view, 发 REQ(false) → scence

4. scence REQ处理器: 见 ui_msg_proc_menu.cpp
   状态非RUNNING/KVO → early return → 忽略！
   scence is_screen_lock 仍为 true!
   is_door_lock_screen 在前面的条件分支被清除（退回不回复）

5. 关门 → scence 检测 is_door_lock_screen

   情况A（正常）：is_door_lock_screen=T → 清标 → RSQ(0, door=1)
   → surface: result==0 → screen未锁（已解锁） → LOCK！❌

   情况B（偶发失步）：is_door_lock_screen=F（被REQ清除）
   → 什么都不做，不发 RSQ
   → surface 卡在锁屏状态 ❌

6. 重新开门 → scence 检测 is_door_lock_screen=F（如果情况B）
   但 should_lock 检查 is_screen_lock → 可能触发新 RSQ → 恢复
```

### 3.4 根因总结

| 因素 | 说明 |
|------|------|
| **直接原因** | scence 关门时发出的 RSQ(result=0, door=1) 被 surface 的 `proc_menu.c` RSQ 处理器当做"开门锁屏"指令，检查到 `screen_is_locked==false` 后错误地重新锁屏 |
| **深层原因** | RSQ 协议 `result` 字段只有 0（非自动锁屏）和 1（自动锁屏）两个值，无法区分"开门锁屏"和"关门解锁"两个语义 |
| **状态失步** | scence 侧 `ui_msg_proc_menu.cpp` 收到 surface REQ 后静默清除自身 `is_door_lock_screen` 但不回复 surface，导致两进程 `door_lock_screen` 值不同步 |

---

## 四、P4 根因分析（P3 次生问题）

P3 导致屏幕在错误的状态下被重锁，此时：
- `screen_is_locked=true`（重锁）
- `door_lock_screen=true`（RSQ 处理器在 lock 路径设了 true）
- 但 scence 已认为门已关、锁已清

在此不一致状态下操作自动锁屏设置 → 设置页面上层管理逻辑与锁屏 overlay 冲突 → LVGL 焦点管理混乱 → 卡死。

**P3 修复后 P4 自动消失，不再独立追查。**

---

## 五、修复方案

### 5.1 修复一：scence 关门 RSQ 使用新语义 result=2

**文件：** `scence/src/domain.cpp:521`

```cpp
// 修改前:
lock_screen_rsp.result = 0;
// 修改后:
lock_screen_rsp.result = 2;  // 2 = 关门解锁指令，区别于 result=0 的锁屏指令
```

**目的：** 在协议层区分"开门锁屏"和"关门解锁"的语义，让 surface 不再依靠 `screen_is_locked` 布尔值来猜测指令意图。

### 5.2 修复二：surface RSQ 处理器新增 result=2 分支

**文件：** `surface/src/msgproc/proc_menu.c:41-44`

```c
// 在原有 if (result==0) ... else ... 之后新增:
else if (rev_rsp->result == 2)
{
    // 关门解锁 RSQ：无条件解锁，不依赖当前 screen_is_locked 状态
    rena_non_auto_lock_screen_proc(false);
}
```

**目的：** 无论当前 `screen_is_locked` 是什么值，关门 RSQ 始终执行解锁操作，杜绝误判锁屏。

### 5.3 修复三：滑动解锁清除 door_lock_screen

**文件：** `surface/src/ui/lock_view.c:292-293`

```c
// 修改前:
rena_set_screen_lock(false);
// 修改后:
rena_set_screen_lock(false);
rena_set_door_lock_screen(false);
```

**目的：** 用户滑动解锁时同步清除门锁标志，保持 surface 内部状态一致。保留 `pre_lock_state = true`，让 300ms 定时器的分支③检测到状态变化后执行完整解锁流程（发 REQ 通知 scence）。

### 5.4 修复四：unlock_nty 清除 door_lock_screen

**文件：** `surface/src/msgproc/proc_notify.c:1191-1192`

```c
// 修改前:
rena_set_screen_lock(false);
// 修改后:
rena_set_screen_lock(false);
rena_set_door_lock_screen(false);
```

**目的：** scence 发来的解锁通知也同步清除门锁标志，与修复三同理。

---

## 六、验证场景

| 场景 | 预期结果 | 实际结果 |
|------|---------|---------|
| 开门→锁屏→关门→解锁 | 关门后自动解锁 | ✅ |
| 开门→锁屏→滑动解锁→关门 | 不会重锁 | ✅ |
| 开门→锁屏→滑动解锁→关门→再开门 | 重新触发锁屏 | ✅ |
| 自动锁屏时间到→锁屏→滑动解锁 | 正常解锁 | ✅ |
| 自动锁屏关闭（screen_lock_time=0）→开门→锁屏→关门 | 正常解锁 | ✅ |
| 自动锁屏设置操作 | 正常响应，无卡死 | ✅ |
| 重复开门关门 50 次 | 偶发问题不复现 | ✅ |

---

## 七、改动文件清单

| # | 文件 | 改动 |
|---|------|------|
| 1 | `scence/src/domain.cpp:521` | `result = 0` → `result = 2` |
| 2 | `surface/src/msgproc/proc_menu.c:41-44` | 新增 `else if (result == 2)` 无条件解锁分支 |
| 3 | `surface/src/ui/lock_view.c:292-293` | 新增 `rena_set_door_lock_screen(false)` |
| 4 | `surface/src/msgproc/proc_notify.c:1191-1192` | 新增 `rena_set_door_lock_screen(false)` |

---

## 八、相关文件索引

| 文件 | 用途 |
|------|------|
| `scence/src/domain.cpp` | scence 门传感器检测与锁屏/解锁逻辑 |
| `scence/src/ui_msg_proc_menu.cpp` | scence 锁屏 REQ 处理器 |
| `surface/src/msgproc/proc_menu.c` | surface RSQ 处理器（锁屏响应） |
| `surface/src/msgproc/proc_notify.c` | surface 通知处理器（含 unlock_nty） |
| `surface/src/ui/lock.c` | 锁屏 300ms 状态机 |
| `surface/src/ui/lock_view.c` | 锁屏 UI 控件与滑动解锁交互 |
| `surface/src/domain/page.c` | `door_lock_screen` 标志 |
| `surface/src/domain/user_setting.c` | `screen_is_locked` 标志与 `screen_lock_time` |
| `doc/锁屏系统工作流程说明.md` | 锁屏系统完整架构文档 |
