---
name: new-lesson
description: >-
  Teaches a new Linux, RHCSA, or operations lesson and guides hands-on VM labs step-by-step.
  Use when starting a new course module or lab session on node1/node2: first explain principles
  and terminology, guide one command block at a time based on real terminal output, and wrap up
  with lab-wrapup and review-quiz rules.
---

# 新课讲授与带做实验（New Lesson）

## 什么时候用

开始学习一节新的 Linux / RHCSA / 运维扩展课程，并在实验虚拟机（`node1` / `node2`）上动手做实验时使用。

## 前置检查（开课前必做）

1. 按仓库 [AGENTS.md](../../../AGENTS.md) 执行 `git pull --ff-only`，拉取远端最新提交。
2. 扫描 `labs/` 底部的「复习记录」与近期薄弱点，提醒用户当天到期的 `D+1` / `D+3` / `D+7` 复习任务（如需先复习，转用 `review-quiz`）。
3. 确认本课编号、主题、目标及当前实验环境状态（`node1` / `node2` 服务、端口、配置等）。

## 执行步骤

1. **先讲原理，再动手做实验。** 在给出第一条实操命令前，精炼讲清：
   - 为什么要学这个机制、解决什么工程问题；
   - 核心概念对比与取舍（尽量用对比表）；
   - 本次实验的步骤路线图。
2. **新术语先解释再用。** 第一次出现的专业术语或缩写（如 `InnoDB`、事务快照、`DDL`/`DML`、`SIGHUP`、`AVC`）先用通俗语言讲清含义，并按规则首次出现附英文原文/全称。
3. **分步推进，每步只给当前这一小段命令。**
   - 明确说明在 `node1` 还是 `node2`（或宿主机）执行，以及关键参数的作用；
   - 等用户贴出真实终端输出或截图后，先解读输出、回答疑问，再给下一步。
4. **实时抓住报错和踩坑。**
   - 用户遇到报错、拼写错误或意外现象时，先讲清底层原因（如 bash 重定向先于命令启动、SQL 定位、SELinux 域转换），留作笔记「踩坑」素材。
5. **课后收尾与归档（遵循 `lab-wrapup`）。**
   - 按 `labs/_template.md` 生成 `labs/YYYY-MM-DD-主题.md`，并将新命令/事实补进当周 `cheatsheets/weekN-*.md`、更新 `README.md` 进度表；
   - 在对话中出 **3～5 道课后面试题**（按课体量弹性决定；遵守 `review-quiz`：一题一问、不超纲），用户作答后批改并写入笔记；
   - 做脱敏检查后执行 `git add`、`git commit`、`git push`。

## 验证清单

- 笔记内所有命令均来自真实终端记录，未编造输出或跳过未做项。
- 密码统一写为 `'...'` 或 `...`，不含任何单位真实主机名、IP、账号、域名或系统名。
- `git status` 干净且已成功推送至远端。

## 不要做

- 不要还没讲原理就直接甩命令让用户执行。
- 不要使用未解释的新术语。
- 不要一次把整节课的所有命令全贴出来。
- 不要把实验密码或任何单位真实数据写进仓库。
- 不要在课后题里叠问、连问或超纲。
