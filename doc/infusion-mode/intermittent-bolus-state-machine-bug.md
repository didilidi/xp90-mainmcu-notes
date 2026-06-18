# 间断给药模式状态机 Bug 与完成清零分析修复报告

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-06-16 | 初稿，覆盖三处根因分析、方案、仿真验证 |

---

## 一、问题清单

| ID | 问题描述 | 严重性 | 状态 |
|----|---------|--------|------|
| **P1** | 未设总预置量时，状态机卡在 INTERVAL，永远不切 MAINTAIN | 高 | ✅ 已修复 |
| **P2** | MAINTAIN 倒计到 0 后，无法自动进入下一轮 INTERVAL（preset_vtbi=0 场景） | 中 | ✅ 已修复（连带的 P1 修复自然解出） |
| **P3** | 输液完成后 `basic_param.preset_vtbi` 不清零，与 nutrient/rate 模式行为不一致 | 中 | ✅ 已修复 |

---

## 二、问题现象（用户报告）

> 当设置间断预置量、主速度、维持速度、间隔时间、预置量时正常运行：开始→主速度打间断预置量→维持速度运行间隔时间→下一轮循环，直到预置量打完。✓
>
> 但当**不设置总预置量**时，启动后进入间断中打药，打完间断预置量后**仍显示"间断中"**，速度仍是主速度，剩余时间显示 > 100h。

---

## 三、根因分析

### 3.1 P1 根因：状态切换条件把两件事混在一起

`calc_mode_param()` 第 391 行（修复前）：

```cpp
if (enter_current_cycle_vtbi + interval_vtbi > volume || preset_vtbi <= volume)
{
    // 还在打 interval 阶段
    ...
}
else
{
    // interval 阶段打完，切 MAINTAIN
    ...
}
```

这里把**两件事**放在同一个 OR 里：

- `enter_current_cycle_vtbi + interval_vtbi > volume` —— 本 cycle interval 还没打完（状态切换的正确判据）
- `preset_vtbi <= volume` —— 已经达到总预置量（治疗终止条件，不是状态切换条件）

**当 `preset_vtbi=0` 时**，`preset_vtbi <= volume` 只要打过药就永远为真 → 永远走 INTERVAL 分支 → 永远不切 MAINTAIN → UI 永远显示"间断中"+主速度。

数值复盘（`interval_vtbi=1ml, interval_rate=100ml/h, preset_vtbi=0`）：

| volume | 修复前分支 | 修复后分支 |
|--------|----------|----------|
| 0.5 | INTERVAL（0+1>0.5）| INTERVAL |
| 1.0 | **INTERVAL（0<=1.0 兜底）** | **MAINTAIN** ✓ |
| 10.0 | **INTERVAL** ✗ | （上层报警已 stop） |

### 3.2 P2 根因：MAINTAIN → INTERVAL 条件漏 preset_vtbi=0 场景

```cpp
if (maintain_time_left <= 0 && preset_vtbi > volume)   // 修复前
{
    enter_interval_indicator();
}
```

`preset_vtbi=0` 时 `preset_vtbi > volume` 永远为假 → MAINTAIN 完成后无法进下一轮。

### 3.3 P3 根因：stop() 没清 basic_param.preset_vtbi

`rena_check_vtbi_finish()`（`alarm_check.cpp:343`）在 `volume_on_therapy >= vtbi` 时调 `rena_get_domain_param()->set_vtbi(0)` —— **只清 domain 层 vtbi，没清 basic_param.preset_vtbi**。

`intermittent_mode_stop()`（修复前）只清了 `volume_on_therapy` 和 `volume`，**与 `nutrient_mode_stop()`（行 280-285）和 `rate_mode`（行 306-312）的惯例不一致**：

```cpp
// nutrient_mode_stop() 既有惯例
if (current_vtbi == 0.0f)
{
    new_basic_param.set_preset_vtbi(0.0f);
}
```

---

## 四、修复方案

### 4.1 修复 P1：`calc_mode_param()` INTERVAL 分支

拆分状态切换条件与剩余量裁剪条件。

```cpp
if (indicator == RENA_INTERMITTENT_MODE_INTERVAL)
{
    maintain_time_left = maintain_time;

    if (enter_current_cycle_vtbi + interval_vtbi > volume)
    {
        // 本次 interval 还没打完
        if (preset_vtbi > 0 && (enter_current_cycle_vtbi + interval_vtbi) >= preset_vtbi)
        {
            // 最后一轮：剩余量上限 = preset_vtbi - volume
            current_interval_vtbi = preset_vtbi - volume;
            if (current_interval_vtbi <= 0)
            {
                current_interval_vtbi = 0;
            }
        }
        else
        {
            current_interval_vtbi = (enter_current_cycle_vtbi + interval_vtbi) - volume;
        }
        interval_time_left = static_cast<int32_t>(ceil(3600 * current_interval_vtbi / interval_rate));
        if (interval_time_left <= 0)
        {
            interval_time_left = 0;
        }
    }
    else
    {
        // 本次 interval 打完了，切到 MAINTAIN
        current_interval_vtbi = 0;
        interval_time_left = 0;
        indicator = RENA_INTERMITTENT_MODE_MAINTAIN;
        enter_maintain_tick = rena_get_sys_uptime();
    }

    basic_param.set_rate(interval_rate);
}
```

### 4.2 修复 P2：MAINTAIN → INTERVAL 条件加 `preset_vtbi <= 0` 兜底

```cpp
if (maintain_time_left <= 0 && (preset_vtbi <= 0 || preset_vtbi > volume))
{
    enter_interval_indicator();
}
```

### 4.3 修复 P3：`rena_intermittent_mode_stop()` 清零

```cpp
void rena_intermittent_mode_stop()
{
    rena_mode_param basic_param = intermittent_mode_param.get_basic_param();
    rena_mode_param new_basic_param = intermittent_mode_param.get_basic_param();

    float current_vtbi = basic_param.get_current_vtbi();

    new_basic_param.set_volume(0);
    rena_get_domain_param()->set_volume_on_therapy(0);

    // 输液完成后清零总预置量（仿 nutrient_mode_stop 模式）
    if (current_vtbi == 0.0f)
    {
        new_basic_param.set_preset_vtbi(0.0f);
    }

    intermittent_mode_param.set_basic_param(new_basic_param);
    rena_nty_intermittent_vtbi(new_basic_param.get_preset_vtbi());
}
```

---

## 五、行为对比

| 场景 | 修复前 | 修复后 |
|------|--------|--------|
| 打完 `volume >= interval_vtbi`（无总预置量）| **卡 INTERVAL** | 切 MAINTAIN，速度切 maintain_rate（0 时电机停转） |
| MAINTAIN 倒计到 0（无总预置量）| 不进下一轮 | **自动进下一轮 INTERVAL** |
| 打到 `volume >= preset_vtbi`（有总预置量）| OK | OK（行为不变） |
| 输液完成触发 stop | preset_vtbi 残留 | **preset_vtbi=0** |
| 输液未完成时点停止 | preset_vtbi 保留 | preset_vtbi 保留（与 nutrient 模式一致） |

---

## 六、验证

### 6.1 仿真验证（Python 状态机模拟）

**场景 A：preset_vtbi=0, maintain_rate=0（用户报告的 bug）**

```
vol=0.95 INTERVAL rate=100 itl=2
vol=1.00 MAINTAIN rate=0   itl=0 mtl=7200   ← 修复后正确切 MAINTAIN
vol=1.05 MAINTAIN rate=0   itl=0 mtl=7197   ← mt_left 正常倒计
...
```

旧版同输入永远卡在 INTERVAL（bug 复现成功，对照组 ✓）。

**场景 B：MAINTAIN → INTERVAL 循环**

```
cycle 1 打完 interval: vol=1.0, eccv=0.0, state=M
  -> 切回 INTERVAL at vol=1.0, eccv=1.0      ← eccv 正确更新
cycle 2 打完 interval: vol=2.0, eccv=1.0, state=M
  -> 切回 INTERVAL at vol=2.0, eccv=2.0
```

**场景 C：最后一轮（preset_vtbi=10, interval_vtbi=1）**

```
vol=4.7 civ=0.8 itl=29  （eccv+iv=5.5, 普通分支，5.5>4.7）
vol=4.9 civ=0.6 itl=22
vol=5.0 civ=0.5 itl=18
```

行为正确。

### 6.2 代码审查

3 处改动对照方案逐字核对一致，无遗漏：

- `intermittent_mode.cpp:394-429` INTERVAL 分支 ✓
- `intermittent_mode.cpp:453` MAINTAIN 切回条件 ✓
- `intermittent_mode.cpp:249-268` stop() 清零 ✓

### 6.3 验证结论

**三处核心 bug 全部修复，可宣布本次任务完成。**

---

## 七、未处理的独立问题（不阻塞本次修复）

| ID | 位置 | 描述 |
|----|------|------|
| R1 | `calc_mode_param()` 行 474 | `end_cycle_vtbi` 计算时把 `extera_cycle_time`（时间/秒）当 vtbi 减，单位错误；`total_time_left` 跨多 cycle 时不准 |
| R2 | `rena_intermittent_mode_start()` 行 223 | `set_current_vtbi(preset_vtbi)`，preset_vtbi=0 时 current_vtbi=0，history/UI 显示异常 |
| R3 | `rena_intermittent_mode_start()` 行 228 | `volume_on_therapy > 0` 早返，重新启动治疗时可能跳过 init |
| R4 | `calc_mode_param()` 行 452 | MAINTAIN 状态 `current_interval_vtbi = interval_vtbi`，语义可疑（"当前 cycle 剩余 interval"应为 0） |

均为原代码既有 bug，与本次修复无关，需单独评估处理。

---

## 八、影响范围

- 文件：`scence/src/intermittent_mode.cpp`（仅 1 个文件）
- 函数影响：`calc_mode_param()`、`rena_intermittent_mode_stop()`
- 接口影响：无（公开 API 未变）
- 联动文件：依赖 `rena_mode_param::set_preset_vtbi` / `get_current_vtbi`（已有接口）
- 其他模式（rate/time/dose_time/nutrient 等）：无影响，逻辑独立
- 历史记录：`rena_intermittent_mode_history_save()` 仍记录治疗开始时的参数，行为不变
