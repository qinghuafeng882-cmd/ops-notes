# 给 AI 助手的说明

这是 fqh 的个人运维知识库，用于 Linux 运维 / RHCSA 学习和排障经验积累。

## 规则

1. **不写入任何单位数据。** 发现真实主机名、公网或内网 IP、拓扑、设备配置、账号密码、内部域名或系统名时，停下来提醒我，不要写入。实验环境里的虚拟机地址（如 192.168.56.x）可以保留。
2. **只记录真实发生的事。** 不编造没执行过的命令和没出现过的输出；推测的内容标注「（推测）」。
3. **新笔记使用对应目录下的 `_template.md`。**
4. **不扩写成教程。** 笔记是给我自己复习用的，简洁优先。
5. **同类任务第二次出现时，提议把它写成 `skills/` 下的新 Skill。**
6. **开课前先同步。** 每次新对话开始上课、实验或复习前，AI 主动执行 `git pull --ff-only` 拉取远端最新提交，并检查最近笔记的「复习记录」与上次薄弱点，核对当天到期的 D+1 / D+3 / D+7 复习任务。
7. **收尾由 AI 直接归档并推送。** 课后或复习结束后，AI 直接按 `skills/lab-wrapup.md` / `skills/review-quiz.md` 写入笔记、更新 `cheatsheets/` 与 `README.md` 进度表；确认无单位敏感数据及真实密码后，由 AI 直接执行 `git add`、`git commit`、`git push`（仅在检测到疑似敏感信息或 Git 冲突时停下提醒我）。

## 目录

- `labs/`：实验笔记
- `troubleshooting/`：排障复盘（已脱敏）
- `skills/`：可复用流程
- `cheatsheets/`：每周速查表，每学完一课把新内容补进当周的速查表

## 可用 Skill

- `skills/lab-wrapup.md`：实验收尾，把终端记录整理成笔记 + 面试题
- `skills/triage-site-unreachable.md`：网站或服务连不上时（refused、No route、timed out、403），按报错分流，再逐层验证
- `skills/review-quiz.md`：D+1 / D+3 / D+7 间隔复习快问快答（一题一问、不超纲）与复习记录更新
- `skills/new-lesson.md`：新课讲授与带做实验（先讲原理和术语，再分步实操，最后按 `lab-wrapup` 收尾）
