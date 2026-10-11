# ops-notes

个人运维知识库：Linux 运维 / RHCSA 学习记录、排障复盘、可复用的 Skill。

## 目录

| 目录 | 放什么 | 命名 |
|---|---|---|
| `labs/` | 每次实验一页笔记 | `YYYY-MM-DD-主题.md`，如 `2026-10-12-lvm-extend.md` |
| `troubleshooting/` | 过往排障经历的脱敏复盘 | `主题.md`，如 `l2-loop-remediation.md` |
| `skills/` | 写给 AI 执行的固定流程 | `动作名.md`，如 `lab-wrapup.md` |

每个目录下的 `_template.md` 是模板，新建笔记时复制一份。

## 硬规矩

**单位的任何数据都不进这个仓库**：真实主机名、IP、拓扑、配置、截图、账号、内部系统名。
排障复盘只写脱敏后的现象、思路和方法。

## 日常用法

1. 做完实验 → 把终端记录交给 AI，按 `skills/lab-wrapup.md` 整理成笔记
2. 笔记里的回忆题按 D+1 / D+3 / D+7 自测，在笔记底部打勾
3. 同一件事做第二遍 → 考虑写成新的 Skill
