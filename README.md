# XP90 MainMCU 文档索引

本仓库是**独立的** XP90 输液泵主 MCU 固件设计/分析/修复文档库，与 `linux-learning` 个人学习仓完全分离。

> 维护关系：本仓仅供本地 + GitHub 备份，**不再**受 `opencode-pro` 主仓 `AGENTS.md` 的"不 commit"约束。

## 目录结构

```
.
├── README.md                   # 本文件
├── .gitattributes              # 行尾规范（统一 LF）
└── doc/                        # 文档目录
    ├── alarm/                  # 报警模块
    │   └── sp90-tube-fall-off-fp-fix.md
    ├── infusion-mode/          # 输液模式
    │   ├── sp90-driver-watchdog-and-door-lock.md
    │   ├── nutrient-mode-occl-no-reverse.md
    │   └── intermittent-bolus-state-machine-bug.md
    ├── lock-screen/            # 锁屏模块
    │   ├── workflow.md
    │   ├── history-classification.md
    │   ├── unlock-failure-and-relock.md
    │   ├── door-lock-stuck-analysis.md
    │   └── auto-lock-overlap-stuck-analysis.md
    ├── gui/                    # GUI/界面模块
    │   └── standby-time-8-9-bug.md
    └── notes/                  # 笔记/会话总结
        ├── claude.md
        └── session-2026-06-09.md
```

## 命名规范

- 文件名使用英文 `kebab-case`
- 命名格式：`<模块/场景关键词>-<动作/类型>.md`
- 常用动作/类型后缀：
  - `-analysis.md` — 问题分析
  - `-fix.md` / `<-bug>.md` — 修复方案
  - `-workflow.md` — 工作流程说明
  - `-classification.md` — 分类方案
  - `-session-YYYY-MM-DD.md` — 会话总结

## 模块索引

### alarm（报警）

| 文档 | 简述 |
| --- | --- |
| [sp90-tube-fall-off-fp-fix.md](doc/alarm/sp90-tube-fall-off-fp-fix.md) | SP90 延长管脱落检测误报修复（P0 互斥抑制+确认计数） |

### infusion-mode（输液模式）

| 文档 | 简述 |
| --- | --- |
| [sp90-driver-watchdog-and-door-lock.md](doc/infusion-mode/sp90-driver-watchdog-and-door-lock.md) | XP90 驱动存活检测与门锁屏状态机方案 |
| [nutrient-mode-occl-no-reverse.md](doc/infusion-mode/nutrient-mode-occl-no-reverse.md) | 营养液模式阻塞不反抽方案 |
| [intermittent-bolus-state-machine-bug.md](doc/infusion-mode/intermittent-bolus-state-machine-bug.md) | 间断给药模式状态机 Bug 与完成清零修复 |

### lock-screen（锁屏）

| 文档 | 简述 |
| --- | --- |
| [workflow.md](doc/lock-screen/workflow.md) | 锁屏系统工作流程 |
| [history-classification.md](doc/lock-screen/history-classification.md) | 锁屏历史记录分类方案（最终版） |
| [unlock-failure-and-relock.md](doc/lock-screen/unlock-failure-and-relock.md) | 锁屏偶发解锁失败与重锁问题修复 |
| [door-lock-stuck-analysis.md](doc/lock-screen/door-lock-stuck-analysis.md) | 泵门锁屏卡死问题分析 |
| [auto-lock-overlap-stuck-analysis.md](doc/lock-screen/auto-lock-overlap-stuck-analysis.md) | 自动锁屏与高级报警叠加卡死问题修复 |

### gui（GUI/界面）

| 文档 | 简述 |
| --- | --- |
| [standby-time-8-9-bug.md](doc/gui/standby-time-8-9-bug.md) | 表面层 GUI 待机时间设置 8/9 失效修复 |

### notes（笔记）

| 文档 | 简述 |
| --- | --- |
| [claude.md](doc/notes/claude.md) | 用户个人笔记（勿动） |
| [session-2026-06-09.md](doc/notes/session-2026-06-09.md) | 2026-06-09 会话总结 |

## 维护说明

- 新增文档请放入对应模块子目录
- 跨模块问题可在 `doc/` 下新建子目录
- 弃用/已合入代码的文档请标注 `[DEPRECATED]` 前缀
- 修改路径或重命名后请同步更新本索引
- 提交时遵守 `.gitattributes`：所有文本文件统一 LF 行尾
