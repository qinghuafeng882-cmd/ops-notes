---
name: lab-wrapup
description: >-
  Wraps up a Linux, RHCSA, or ops lab session by organizing real terminal logs into a structured
  lab note, generating 5 interview questions, updating cheatsheets and README progress, and
  committing/pushing after sanitization checks. Use when finishing a lab (even partially).
---

# 实验收尾（Lab Wrapup）

## 什么时候用

做完一次实验（哪怕只做了一半），需要将终端记录整理成标准笔记、完成课后题并归档推送时使用。

## 输入信息

- 实验主题，以及对应的 RHCSA EX200 考点或运维扩展模块；
- 终端记录（`history` 输出、复制的终端文字或用户截图）；
- （可选）本次卡点或没想明白的问题。

## 执行步骤

1. **生成标准笔记**：严格按 [labs/_template.md](../../../labs/_template.md) 编写 `labs/YYYY-MM-DD-主题.md`。
2. **只记录真实发生的事**：
   - 只记录终端里真正执行过的命令与真实输出，未贴出的输出不编造；
   - 补充说明或推理必须明确标注「（推测）」。
3. **提取「踩坑」**：从报错、拼写错误、重试、撤销操作中提取，每条写成「现象 → 原因 → 解决」。没有踩坑就写「无」，不要硬编。
4. **填写「验证」表**：写清证明配置生效的命令、预期结果、实测结果，以及**重启后是否仍生效**。没验证过的明确写「未验证」或「未做」。
5. **准备 5 道面试题（对话中作答并批改）**：
   - 题型分布：1 道概念题 + 2 道排障题 + 1 道操作题 + 1 道取舍题；
   - 严格一题一问、不超纲；每题附 `<details><summary>批改要点</summary>...</details>`；
   - 在对话中让用户作答并逐题批改，不要做成让用户课后自填的回忆单；作答后把实际得分与批改结果写入笔记。
6. **标注复习起算日**：在 `## 复习记录` 写明起算日与预计 `D+1` / `D+3` / `D+7` 日期，保持 `- [ ]` 未勾选状态。
7. **同步速查表与进度表**：将本课新命令与关键结论补进对应 `cheatsheets/weekN-*.md`，并在 `README.md` 进度表新增记录。
8. **敏感信息校验与 Git 推送**：
   - 检查是否混入真实密码（密码统一脱敏为 `'...'` 或 `...`）或单位真实主机名、IP、拓扑、账号、内部域名（实验虚机地址 `192.168.56.x` 可保留）；发现可疑信息立刻停下提醒用户；
   - 确认无误后执行 `git add`、`git commit`、`git push`；
   - 最后单独提醒用户：**这次实验最薄弱的点是什么，下次复习先看哪里**。

## 不要做

- 不要写成教程，不要贴大段 `man` 手册。
- 不要美化记录：失败的尝试、拼写错误和未完成项也是笔记的一部分。
- 不要省略用户自己提到的卡点，也不要给未做过的检查打勾。
