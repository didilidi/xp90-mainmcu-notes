# 锁屏历史记录分类方案（最终版）

## 需求

| 场景 | 期望显示 |
|------|----------|
| 按钮/空闲锁屏 → 手动滑块解锁 | "锁屏-解锁" |
| 高级报警强制解锁 | "锁屏-解锁" |
| 非运行/KVO 状态自动解锁 | "锁屏-解锁" |
| **门开锁屏 → 手动滑块解锁** | **"锁屏-手动解锁"** |
| 门开锁屏 → 门关自动解锁 | **不记录**（门开/门关是物理动作）|

**核心**：**只有"门锁→手动滑块"** 这一个场景区分"手动解锁"，其他场景都用"解锁"。

## 最终方案（yangyuxiang 设计）

### 关键设计

| 设计点 | 内容 |
|--------|------|
| "手动/自动"判断依据 | scence 端 `is_door_lock_screen` 全局状态（不是 REQ 字段）|
| REQ 触发方 | surface 状态机分支①②③（**不是**滑块事件）|
| 滑块事件 | 只清 surface 端状态，**不发 REQ** |
| RSQ 协议 | unlock 路径发 RSQ（result=1 来自 is_auto_lock=true），不靠 return 避免双义性 |
| 门锁分支 | 仍 return 不发 RSQ（保留我之前的门锁兜底）|

### 各场景走的路径

| 场景 | 触发 | REQ 字段 | scence 走哪 | 写历史 |
|------|------|----------|-------------|--------|
| 按钮锁屏 → 滑块 | 状态机分支③ | (false, auto=true, door=false) | 主分支 unlock | UNLOCK |
| 空闲锁屏 → 滑块 | 状态机分支③ | (false, auto=true, door=false) | 主分支 unlock | UNLOCK |
| 门开锁屏 → 滑块 | 状态机分支③ | (false, auto=true, door=false) | **门锁分支**（`is_door_lock_screen` 守卫）| **MANUAL_UNLOCK** |
| 门开锁屏 → 门关自动 | scence domain.cpp（注释）| — | — | 不写 |
| 高级报警强制解锁 | 状态机分支① | (false, auto=true, door=false) | 主分支 unlock | UNLOCK |
| 非运行/KVO 解锁 | 状态机分支② | (false, auto=true, door=false) | 主分支 unlock | UNLOCK |

## 4 处改动

### scence 端（commit 6138af4，`src/ui_msg_proc_menu.cpp`）

**line 38-53 门锁分支**：保留 `is_door_lock_screen` 守卫 + `SCREEN_MANUAL_UNLOCK` + return 不发 RSQ。

**line 85-94 主分支 unlock**：
```cpp
else
{
    rena_get_domain_param()->set_is_screen_lock(false);
    rena_history_record_param param;
    memset(&param, 0, sizeof(rena_history_record_param));
    param.format = 2;
    param.type = RENA_HISTORY_PARAM_TYPE_SCREEN_LOCK;
    param.object1 = RENA_HISTORY_PARAM_OBJECT_SCREEN_UNLOCK;  // 改回 UNLOCK
    rena_history_event_save(param);
    // 删 return → 继续走发 RSQ
}
if (lock_screen_req->is_auto_lock) { ... }  // 决定 RSQ result
lock_screen_rsp.is_door_lock = lock_screen_req->is_door_lock;
rena_send_msg_to_ui(RENA_UI_LOCK_SCREEN_RSQ, ...);
```

**line 103 修字段错配**：`is_door_lock = is_door_lock`（不是 is_auto_lock）。

### surface 端（commit f2001aae）

**`src/ui/lock.c` 分支②**（line 91-107）：补 `current_lock_state == true` 守卫 + 发 REQ。

**`src/ui/lock_view.c` 滑块事件**（line 280-294）：屏蔽之前加的发 REQ 代码，只清 surface 端状态。

## 为什么 yangyuxiang 的方案更合适

### 1. 准确理解需求

| 需求 | 我的 c2c2925 | yangyuxiang |
|------|--------------|-------------|
| "手动解锁"显示 | 所有滑块解锁场景 | **只门锁场景**（精准）|

我过度设计了：把"按钮锁屏→滑块解锁"和"门开锁屏→滑块解锁"都标 MANUAL_UNLOCK。yangyuxiang 准确理解为**只有门锁场景才需要区分**。

### 2. 改动量更小

| 文件 | 我 c2c2925 | yangyuxiang |
|------|-----------|-------------|
| scence `ui_msg_proc_menu.cpp` | +5/-2 | +1/-3 |
| scence `history_event_def.h` | +1 | 0 |
| scence `history_event_def.cpp` | +1 | 0 |
| scence `i18n/lv_i18n.c` | +8 | 0 |
| scence `translation/*.yml` | +8 | 0 |
| surface `lock_view.c` | +10 | -10 |
| surface `lock.c` | 0 | +7 |
| **总计** | +33/-1（13 文件）| **+8/-13（2 文件）**|

### 3. RSQ(0) 双义性解决更优雅

| 方案 | 解决方式 | 评价 |
|------|----------|------|
| 我 c2c2925 | unlock 路径 return 不发 RSQ | 协议不对称 |
| yangyuxiang | REQ 的 `is_auto_lock` 决定 RSQ `result`（line 95-98）| 协议对称 |

**关键洞察**：所有 surface 状态机发的 REQ 都是 `is_auto_lock=true` → scence 发 RSQ result=1（自动确认）。**RSQ(0) 双义性问题自然消失**。

### 4. 滑块事件不发 REQ 更安全

| 方案 | 滑块事件 |
|------|----------|
| 我 c2c2925 | 滑块事件发 REQ（LVGL 事件回调中）|
| yangyuxiang | 滑块事件只清状态，状态机分支③ 发 REQ（异步）|

**好处**：
- 滑块事件回调中不发消息（避免 LVGL 事件回调栈问题）
- 状态机分支③ 统一处理所有解锁路径
- scence 端 `is_door_lock_screen` 状态先被滑块事件清掉 → 状态机分支③ 发 REQ 时 scence 走主分支 unlock → 写 UNLOCK
- scence 端 `is_door_lock_screen` 仍是 true → 状态机分支③ 发 REQ 时 scence 走门锁分支 → 写 MANUAL_UNLOCK

### 5. 不需要 `is_screen_lock` 守卫

我之前加的 `is_screen_lock` 守卫是为了防"状态机分支③ 重发 REQ 写两条历史"。**yangyuxiang 方案不需要**：

- 滑块事件 set `lock=false` → `pre_lock_state=true`（还没更新）
- 状态机分支③ 触发：`pre_lock_state(true) != current_lock_state(false)` → 发 REQ → **`pre_lock_state` 更新为 false**
- 下次 tick：`pre_lock_state(false) == current_lock_state(false)` → **不进分支③**

**只会触发一次**，scence 端只收到 1 条 REQ。

## 我之前方案被拒绝的部分

| 改动 | 原因 |
|------|------|
| 主分支 unlock 写 MANUAL_UNLOCK | 需求不要求"按钮/空闲锁屏的滑块解锁"区分 |
| unlock 路径 return 不发 RSQ | 不必要，用 `is_auto_lock` 决定 RSQ 更优雅 |
| `is_screen_lock` 守卫 | 不必要，状态机分支③ 只触发一次 |

## 相关 Commit

| 子模块 | commit | 说明 |
|--------|--------|------|
| scence | `c2c2925` | 初版（i18n + MANUAL_UNLOCK + return + 修字段）|
| scence | `6138af4` | 简化（删 return + 改回 UNLOCK）— **最终** |
| surface | `022083e6` | 初版（滑块事件发 REQ）|
| surface | `f2001aae` | 简化（屏蔽滑块发 REQ + lock.c 状态机分支发 REQ）— **最终** |

## 总结

最终方案的核心：
- **scence 端用 `is_door_lock_screen` 状态判断门锁分支**（不是 REQ 字段）
- **surface 端从状态机分支③ 统一发 REQ**（不是滑块事件）
- **REQ 的 `is_auto_lock=true` → RSQ result=1**，避开双义性
- **门锁场景写 MANUAL_UNLOCK，其他场景写 UNLOCK**

这是准确理解"只门锁场景区分手动解锁"需求的最简实现。
