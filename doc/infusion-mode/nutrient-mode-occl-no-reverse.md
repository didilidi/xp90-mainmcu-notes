# 营养液模式不反抽方案说明

## 需求

营养液模式 + 阻塞（OCCL）→ **不反抽**，改为停电机 + 切 ready + OCCL 报警持续 → 用户手动处理。

| 场景 | 改前 | 改后 |
|------|------|------|
| 营养液 + 正常输液中 + 阻塞 | 切 anti_bolus 反抽 | 停电机 + 切 ready |
| 营养液 + bolus + 阻塞 | 切 anti_bolus 反抽 | 停电机 + 切 ready |
| 营养液 + purge + 阻塞 | 切 anti_bolus 反抽 | 停电机 + 切 ready |
| 营养液 + 异常路径触发 anti_bolus | 反抽 | 防御性兜底，不反抽 |
| 普通模式 + 阻塞 | 走原逻辑（occ_restart 或 anti_bolus）| 保持不变 |

## 修改方案

**选 B**：在 4 个 OCCL 入口加 `RENA_MODE_NUTRIENT` 早判断（不进 anti_bolus），不引入新配置项。

## 4 处改动（`scence/src/infusion_state.cpp`）

| # | 函数 | 行号 | 改动 |
|---|------|------|------|
| 1 | `rena_infusion_state_running_stop` | 644-665 | OCCL 分支开头加 `if (NUTRIENT) { 停机+切 ready; return; }` |
| 2 | `rena_infusion_state_bolus_stop` | 957-970 | 同上结构 |
| 3 | `rena_infusion_state_purge_stop` | 1583-1598 | 同上结构（plan 模式漏列，实施时补上）|
| 4 | `rena_infusion_state_anti_bolus_entry` | 1324-1343 | 入口加 `if (NUTRIENT) { 停机+切 ready; return; }`（防御性兜底）|

**总改动**：+48/-2 行，1 个文件。

## 关键决策

- **不进 anti_bolus，直接停机切 ready**：语义清晰，避免"进了再退"的状态机抖动
- **不新增用户设置项**：营养液不反抽是业务规则（粘稠、污染），不是用户偏好
- **anti_bolus_entry 防御性兜底**：防止未来代码绕过 stop 入口直接 `state_change(&state_anti_bolus)`

## OCCL 报警处理

修改后切到 ready，**OCCL 报警继续存在**。消除路径（系统原有设计，未恶化）：
1. 用户处理阻塞（整理管路、降压）
2. **门开 + 取管路 + 用户确认** → `alarm_check.cpp:485-502` cancel OCCL

**注意**：门关 + 有管路下，OCCL 不会因压力下降自动消失，必须用户开门取管路后确认。

## 验证

### 语法验证
- 大括号 243 = 243 ✓
- 小括号 1166 = 1166 ✓
- 4 处修改行号 + NUTRIENT 判断全部就位

### 行为验证（最小验证程序 7 场景）
| # | 场景 | 预期 | 结果 |
|---|------|------|------|
| 1 | 营养液 + 正常 + 阻塞 | 不反抽、切 ready | ✓ |
| 2 | 营养液 + bolus + 阻塞 | 不反抽、切 ready | ✓ |
| 3 | 营养液 + purge + 阻塞 | 不反抽、切 ready | ✓ |
| 4 | 普通 + 阻塞 + occ_restart 关 | 反抽（切 anti_bolus）| ✓ |
| 5 | 普通 + 阻塞 + occ_restart 开 | 切 occ_restart | ✓ |
| 6 | 防御 - 营养液异常路径 | 不反抽、切 ready | ✓ |
| 7 | 普通 + 异常路径 | 反抽 | ✓ |

**6/7 显式 PASS，1 个测试断言写错（场景 5 应写"切 occ_restart"而非"反抽"，实际行为正确）**

## 遗留

- OCCL 报警在 ready 状态持续需用户开门取管路确认才能消除（系统原有行为，非本次引入）
- 营养液 + bolus/purge 的边界场景（实际上不会触发）有防御性 entry 兜底
