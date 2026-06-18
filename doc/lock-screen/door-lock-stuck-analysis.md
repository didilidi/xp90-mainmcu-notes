# 泵门锁屏卡死问题分析与修复说明

## 问题现象

当输液自动锁屏功能关闭时，打开泵门会触发自动锁屏，但此后屏幕出现卡死（界面无响应、触控不生效、锁屏无法滑动解锁）。而当输液自动锁屏功能开启时（输液运行中自动锁屏触发后），泵门锁屏正常，关门可自动取消锁屏。

## 涉及模块

| 模块 | 文件 | 作用 |
|------|------|------|
| surface UI | `surface/src/ui/lock.c` | 锁屏管理器，含 300ms 定时器状态机 |
| surface UI | `surface/src/msgproc/proc_menu.c` | 锁屏响应消息处理 |
| scence 逻辑 | `scence/src/domain.cpp` | 泵门传感器检测，开门锁屏/关门解锁 |

## 根因分析

### 执行流对比

**自动锁屏开启时（正常）：**

1. 滴注运行中，`screen_lock_time > 0`，`rena_page_lock_timer_cb` 每 300ms 正常运行
2. 泵门打开 → scence 发送 `RENA_UI_LOCK_SCREEN_RSQ` 至 surface
3. surface 创建 `lock_view`，调用 `switch_menu_state(COLLAPSE)` 收起菜单
4. 定时器回调持续运行：第 138-143 行刷新空闲计时器，第 107-122 行检测状态转换
5. 用户滑动解锁 → surface 侧 `is_screen_lock = false` → 定时器检测到状态转换 → 调用 `rena_unlock_screen()` 清理对象 → 一切正常

**自动锁屏关闭时（卡死）：**

1. 滴注未运行，`screen_lock_time == 0`，`rena_page_lock_timer_cb` 第 63-66 行**直接 return**，状态机完全跳过
2. 泵门打开 → scence 发送锁屏响应 → surface 创建 `lock_view`，`pre_lock_state = true`
3. `switch_menu_state(COLLAPSE)` 创建新的 dropdown_menu，覆盖在 lock_view 上方
4. **状态机不运行** → 以下问题同时发生：
   - lock_view 被 dropdown_menu 遮挡，`rena_set_lock_view_foreground()` 不被调用
   - `pre_lock_state` 始终保持 `true`，即使用户已通过滑块解锁
   - `lock_view` 静态指针未被置 NULL，`door_lock_screen` 未清除
5. 后续任何锁屏/解锁事件触发时，状态机无法正确解析当前状态，操作 stale 对象，界面完全不响应

### 根本原因

`lock.c:63-66` 的过早 return 导致在自动锁屏关闭时，锁屏状态机被完全跳过，无法执行核心的状态转换检测（第 107-122 行）和锁屏状态下的空闲计时器刷新（第 138-143 行），从而引发状态不同步、对象泄漏和 UI 遮挡三个问题叠加，最终导致屏幕卡死。

## 修复方案

### 修改 1：`surface/src/ui/lock.c` — 去掉过早 return，仅包围空闲超时

**位置：** 第 63-66 行（注释掉）+ 第 124-139 行（添加保护条件）

**改动：**

- 注释掉 `if (rena_get_screen_lock_time() == 0) { return; }`，让状态机始终运行
- 在第 125 行增加 `if (rena_get_screen_lock_time() > 0)` 守卫，仅当自动锁屏开启时才执行空闲超时自动锁屏逻辑

**效果：**

- 状态机持续运行：高级报警解锁（第 72-87 行）、非运行状态解锁（第 96-104 行）、状态转换检测（第 107-122 行）始终生效
- 空闲超时锁屏仅在 `screen_lock_time > 0` 时触发，与原有行为一致

```c
// 改动前 (63-66行)
if (rena_get_screen_lock_time() == 0)
{
    return;
}

// 改动后：注释掉以上代码

// 改动前 (124行)
if (curr_time - touch_tick >= rena_get_screen_lock_time() * 1000)

// 改动后 (125行)
if (rena_get_screen_lock_time() > 0)    // 新增保护条件
{
    curr_time = page_lock_timer->last_run;
    if (curr_time - touch_tick >= rena_get_screen_lock_time() * 1000)
    { ... }
}
```

### 修改 2：`surface/src/msgproc/proc_menu.c` — lock_view 置前

**位置：** 第 33 行之后，增加第 34 行

**改动：**

在 `switch_menu_state(PAGE_MENU_STATE_COLLAPSE)` 之后调用 `rena_set_lock_view_foreground()`，确保锁屏 overlay 始终在 dropdown_menu 之上，不被遮挡。

```c
// 改动后
rena_non_auto_lock_screen_proc(true);
switch_menu_state(PAGE_MENU_STATE_COLLAPSE);
rena_set_lock_view_foreground();    // 新增
```

## 影响范围

- **仅影响 VP90 / SP90 产品**（泵门锁屏功能仅在这两个产品开启）
- 不影响自动锁屏功能原有逻辑（空闲超时判断保护条件已隔离）
- 不影响高级报警自动解锁、KVO 结束解锁等原有行为
- 无新增内存分配，无接口变更

## 测试建议

| 场景 | 步骤 | 预期结果 |
|------|------|----------|
| 自动锁屏关闭 + 开门锁屏 | 设置自动锁屏=关闭 → 打开泵门 | 锁屏出现，可滑动解锁，不卡死 |
| 自动锁屏关闭 + 关门 | 开门锁屏后，关闭泵门 | 锁屏自动消失，恢复正常界面 |
| 自动锁屏开启 + 输液锁屏 | 开始输液 → 等待自动锁屏时间 | 正常自动锁屏，可解锁 |
| 自动锁屏开启 + 开门 | 输液运行中已锁屏 → 打开泵门 | 锁屏正常，关门消失 |
| 高级报警 + 锁屏 | 运行中锁屏 → 触发高级报警 | 自动解锁，弹出报警界面 |
| 非运行状态 + 开门锁屏 | 停止输液 → 开门 | 锁屏正常，关门消失 |

## 相关文件清单

| 文件 | 修改说明 |
|------|----------|
| `surface/src/ui/lock.c:63-66` | 注释掉 `screen_lock_time == 0` 的过早 return |
| `surface/src/ui/lock.c:125` | 增加 `screen_lock_time > 0` 保护条件 |
| `surface/src/msgproc/proc_menu.c:34` | 新增 `rena_set_lock_view_foreground()` |
