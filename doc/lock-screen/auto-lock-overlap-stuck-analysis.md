# 自动锁屏与高级报警叠加时锁屏界面卡死问题分析修复报告

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-06-09 | 初稿，覆盖根因分析、修复方案、场景验证 |

---

## 一、问题描述

### 1.1 现象

当输液泵的"输液自动锁屏"时间设置较短（例如 15 秒），且 VTBI 预置量计算的剩余输液时间与锁屏时间接近时（典型场景：50ml/h 速度 + 0.22ml 预置量 → 剩余 15.84s，锁屏 15s），会出现以下偶发问题：

1. 启动输液
2. **T+15s**：触发自动锁屏，锁屏界面覆盖在界面上
3. **T+16s**（剩余时间到）：触发 VTBI_FINISH 高级报警，报警 TOP 页面弹出
4. **关键异常**：锁屏界面残留在报警 TOP 页面之上，用户**滑动锁屏无法解锁**、**点击报警的确认/静音按钮也无反应**（按钮被锁屏遮挡）
5. 反复操作无效，最终以系统"重启"或跳回主页面结束，且 KVO 输液状态与 VTBI 完成报警同时轮询显示

### 1.2 触发条件

- 自动锁屏时间 ≈ VTBI 剩余时间（差 1-2 秒）
- 自动锁屏时间设置 > 0
- KVO 开关开启

### 1.3 偶发性

问题**偶发**而非 100% 复现，根因是 **IPC 时序竞争**：T+15s 自动锁屏触发的 REQ（自动锁屏响应）与 T+16s 报警触发的 alarm_nty 几乎同时到达 surface 端的消息队列，msg_thread 串行处理顺序受时序影响。

---

## 二、系统相关代码位置

| 组件 | 文件 | 关键函数/行号 |
|------|------|------------|
| surface 锁屏状态机 | `surface/src/ui/lock.c` | 300ms timer 分支①③④、109 行 `rena_unlock_screen()` |
| surface 锁屏界面 | `surface/src/ui/lock_view.c` | `lock_view_load`、`rena_slider_unlocker_event_cb` |
| surface 主页面层 | `surface/src/ui/manager.c` | 1124-1128 行 top_page 创建后的锁屏处理 |
| surface 锁屏 RSQ 处理器 | `surface/src/msgproc/proc_menu.c` | 18-49 行 |
| surface 报警 NTY 处理器 | `surface/src/msgproc/proc_notify.c` | 834-841 行 `switch_top_page` |
| scence 高级报警时解锁 RSQ | `scence/src/alarm_check.cpp` | 2709-2725 行 |
| scence 报警 NTY 发送 | `scence/src/alarm_manage.cpp` | 319-344 行 `rena_alarm_ui_send` |
| scence REQ→RSQ 响应 | `scence/src/ui_msg_proc_menu.cpp` | 32-105 行 `rena_proc_ui_msg_lock_screen_req` |

---

## 三、根因分析

### 3.1 数据流追踪（场景 A：alarm_nty 先到）

| 时刻 | 事件 | surface 端状态 |
|------|------|---------------|
| T+15s | 300ms timer 分支④触发 `rena_lock_screen()`，创建 lock_view A | `screen_is_locked=true`, `door_lock_screen=false`, lock_view A 存在 |
| T+15s+ε | lock.c 发 REQ(false, is_auto_lock=true) | — |
| T+15s+δ1 | scence 收 REQ，`is_door_lock_screen==false`，走正常解锁流程 | scence 端 `is_screen_lock=false`，发 RSQ(result=**1**, is_door_lock=**true**) |
| T+16s | VTBI 报警触发 | scence 端发 alarm_nty(top_page_id=PAGE_TOP_VTBI_FINISH)；`alarm_check.cpp:2709` 条件 `is_screen_lock==true` 不满足（scence 不知道 surface 自动锁屏），**不发 RSQ** |
| T+16s+δ1 | surface 收 alarm_nty | `proc_notify.c:841` → `switch_top_page(PAGE_TOP_VTBI_FINISH)` → `manager.c:1115-1132` 创建 top_page、`move_foreground`、**`alarm_type==HIGH && door_lock_screen==false` → `rena_set_screen_lock(false)`** |
| T+16s+δ2 | surface 收 RSQ(1, door=true) | `proc_menu.c:45` else 分支（错误情况处理）→ **no-op** |
| T+16s+ε | 300ms timer 跑 | `current_lock_state=false`（已被 manager 清），`pre_lock_state=true` → 分支③ 检测状态变化 → `rena_unlock_screen()` + 发 REQ(false) |

**关键矛盾点**：表面看起来分支③ 会删 lock_view A，但实际：

- **分支①** 守卫 `alarm_type==HIGH && door_lock_screen==false`（manager.c:1124 已设 alarm_type==HIGH）
- 分支① 内 `if (current_lock_state==true)` → **false**（已被 manager 清）→ **不进清理逻辑** → return
- **分支③** 不会跑（被分支① 的 return 跳过）

**结果**：lock_view A **始终残留**在 top_page 之上，**遮挡**报警按钮。

### 3.2 场景 B：REQ 响应 RSQ 先到

| 时刻 | 事件 | surface 端状态 |
|------|------|---------------|
| T+15s+δ1 | surface 收 RSQ(1, door=true) | `proc_menu.c:45` else → no-op |
| T+16s+δ1 | surface 收 alarm_nty | manager.c:1127 `rena_set_screen_lock(false)`，lock_view A 残留 |
| T+16s+ε | 300ms timer | 同 3.1，lock_view A 残留 |

**两种场景结果相同**：lock_view A 残留。

### 3.3 用户感知

- **场景 A 与 B 都发生**：lock_view A 覆盖在 top_page（1424×280）之上 → 用户触摸在 lock_view A 区域被 `rena_lock_view_event_cb` 拦截 → 不发 REQ → 按钮无反应
- 反复操作无果，最终触发 scence 端 watchdog 复位或异常路径

---

## 四、修复方案

### 4.1 修复目标

1. **避免 lock_view 在高级报警后残留**
2. **保留门锁屏功能**（开门锁屏是独立功能，不能破坏）
3. **不影响正常自动锁屏逻辑**（无报警时仍正常显示锁屏）
4. **不破坏已有的 KVO 状态、KVO 字段推送逻辑**

### 4.2 修复 R2：manager.c 高级报警时直接删 lock_view

**问题**：`manager.c:1124-1128` 在高级报警且非门锁屏时仅调用 `rena_set_screen_lock(false)`（清布尔），不删除 lock_view 对象。

**修复**：改用 `rena_unlock_screen()`（删 lock_view + 清 door_lock_screen + 删 timer）。

**文件**：`surface/src/ui/manager.c:1124-1128`

```c
// 修复前
if (rena_get_alarm_type() == RENA_ALARM_TYPE_HIGH && rena_get_door_lock_screen() == false)
{
    rena_refresh_lock_prev_time();
    rena_set_screen_lock(false);  // ← 仅清布尔
}
else
{
    rena_set_lock_view_foreground();
}

// 修复后
if (rena_get_alarm_type() == RENA_ALARM_TYPE_HIGH && rena_get_door_lock_screen() == false)
{
    rena_refresh_lock_prev_time();
    rena_unlock_screen();  // ← 删 lock_view
}
else
{
    rena_set_lock_view_foreground();
}
```

**安全保证**：守卫条件 `door_lock_screen==false` 确保门锁屏不进入此分支，门锁屏功能完全保留。

### 4.3 修复 R1：proc_menu.c RSQ 处理器区分门/非门锁场景

**问题**：`proc_menu.c:24-40` 依赖单一 `screen_is_locked` 标志判断 LOCK/UNLOCK 路径。当 `screen_is_locked` 已被 manager.c:1124 清掉但 `lock_view` 残留时，会误判为 LOCK 路径并创建新 lock_view（防御性修复）。

**修复**：在 `is_door_lock==true`（门锁屏）分支保留原始逻辑（依赖当前状态）；在 `is_door_lock==false`（非门锁屏）分支增加 `lock_view` 残留检查。

**前提**：需要新增 `rena_is_lock_view_exist()` API（因 `lock_view` 是 lock.c 的 static 变量）。

#### 4.3.1 新增 API

**文件**：`surface/src/ui/lock.h` 末尾追加：

```c
bool rena_is_lock_view_exist(void);
```

**文件**：`surface/src/ui/lock.c` 末尾追加：

```c
bool rena_is_lock_view_exist(void)
{
    return lock_view != NULL;
}
```

#### 4.3.2 RSQ 处理器重写

**文件**：`surface/src/msgproc/proc_menu.c:18-49`

```c
void rena_proc_ui_msg_lock_screen_rsp(void *data)
{
    rena_ui_msg_t *msg_recv = (rena_ui_msg_t *)data;
    rena_ui_lock_screen_rsp *rev_rsp = (rena_ui_lock_screen_rsp *)msg_recv->body;

    if (rev_rsp->result == 0)
    {
        if (rev_rsp->is_door_lock)
        {
            // 门锁 RSQ：保留原始依赖当前状态的判断逻辑，保护门锁屏功能
            if (rena_get_screen_lock() == false)
            {
                rena_set_door_lock_screen(true);
                rena_non_auto_lock_screen_proc(true);
                switch_menu_state(PAGE_MENU_STATE_COLLAPSE);
            }
            else
            {
                rena_non_auto_lock_screen_proc(false);
            }
        }
        else
        {
            // 非门锁 RSQ：处理 lock_view 残留问题
            if (rena_get_screen_lock() == false && !rena_is_lock_view_exist())
            {
                // 真正需要锁（无 lock_view 残留）
                rena_non_auto_lock_screen_proc(true);
                switch_menu_state(PAGE_MENU_STATE_COLLAPSE);
            }
            else if (rena_is_lock_view_exist())
            {
                // 有 lock_view 残留（screen_is_locked 已被其他路径清掉）→ 清理
                rena_unlock_screen();
            }
        }
    }
    else if (rev_rsp->result == 2)
    {
        rena_non_auto_lock_screen_proc(false);
    }
    else
    {
        // 错误情况处理
    }
}
```

### 4.4 未实施的候选修复

#### 4.4.1 R3（lock.c:69-83 分支① 守卫条件）

**候选内容**：把 `if (current_lock_state==true)` 改为 `if (lock_view != NULL)`。

**未实施原因**：R2 修复后，manager.c:1124 在 alarm_nty 时已直接调 `rena_unlock_screen()` 删 lock_view。300ms timer 后续跑时 lock_view 已是 NULL，分支① 不会进清理逻辑。R3 是冗余修复。

#### 4.4.2 R4（rena_lock_screen 先删旧 lock_view）

**候选内容**：`rena_lock_screen()` 函数先 delete 旧的 lock_view 再创建新的。

**未实施原因**：R2 修复后，lock_view 不会在报警路径上残留到 LOCK 路径。R4 是防御性修复，目前不必要。

---

## 五、场景验证

| 场景 | 触发条件 | 预期结果 | 验证 |
|------|---------|---------|------|
| 1. 纯门锁屏 | 开门→锁屏→关门→解锁 | 门锁屏功能正常 | ✅ |
| 2. 自动锁屏（非报警） | T+15s 空闲 | 锁屏正常显示，用户可滑动解锁 | ✅ |
| 3. **原 bug：自动锁屏 + VTBI 报警** | T+15s 锁屏 + T+16s VTBI 报警 | 锁屏标识消失，按钮可点，报警正常处理 | ✅ |
| 4. 门锁屏 + 高级报警 | 开门锁屏后触发 OCCL 等 | 门锁屏保留（safety 设计） | ✅ |
| 5. 自动锁屏滑动解锁后关门 | 自动锁屏→滑动解锁→关门 | 表面无锁屏（scence 端不发 RSQ） | ✅ |
| 6. 门锁屏滑动解锁后关门 | 门锁屏→滑动解锁→关门 | 表面无锁屏 | ✅ |
| 7. 非门锁 RSQ 误判防御 | 任意 RSQ(0, door=false) 到达 | 不会误判 LOCK 路径 | ✅ |

---

## 六、改动清单

| # | 文件 | 行号 | 改动 |
|---|------|------|------|
| 1 | `surface/src/ui/lock.h` | 末尾 | 新增 `bool rena_is_lock_view_exist(void);` |
| 2 | `surface/src/ui/lock.c` | 末尾 | 新增 `rena_is_lock_view_exist()` 实现 |
| 3 | `surface/src/ui/manager.c` | 1127 | `rena_set_screen_lock(false);` → `rena_unlock_screen();` |
| 4 | `surface/src/msgproc/proc_menu.c` | 18-49 | RSQ 处理器重写：门锁保留原逻辑，非门锁处理 lock_view 残留 |

---

## 七、相关历史

- 本次修复建立在前两个 session 的修复基础之上：
  - Session 2026-06-08 修复了 `lock.c` 早期 return 导致泵门锁屏卡死
  - Session 2026-06-09 v2 修复了锁屏偶发解锁失败（`result=2` 协议扩展、`lock_view` 状态同步）
  - Session 2026-06-09 v3（本报告）修复了自动锁屏与高级报警叠加的锁屏残留

- 本次修复**未涉及** KVO 字段无刷新问题（属另一独立问题，需要单独排查）

---

## 八、相关文档

- `doc/泵门锁屏卡死问题分析与修复说明.md`（2026-06-08）
- `doc/锁屏系统工作流程说明.md`（2026-06-08）
- `doc/锁屏偶发解锁失败与重锁问题分析修复报告.md`（2026-06-09 v2）
