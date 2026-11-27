# ACL 基础：setfacl、mask 与有效权限

- 日期：2026-11-20（课堂 07:37–10:51）
- 考点：RHCSA EX200 —— ACL（`setfacl` / `getfacl`）。默认 ACL 继承没讲，这课不是 ACL 全部
- 环境：Rocky Linux 10（VirtualBox）。目录沿用当天的 `/srv/ops-exam`（`drwxrws--- root ops_exam`），已有 `/srv/ops-exam/test.txt`
- 状态：基础补课做完。模拟②第 2 题已撤出闭卷，不能写成闭卷通过
- 来源：课堂终端记录

## 目标

不把 exam02 加进 ops_exam，单独给这个用户目录上的 `rx`，看清 mask 和「能进目录」与「能新建文件」不是一回事。

## 关键概念

- 模拟题原要求给 exam02 `rx`。语法还没教，当场撤出闭卷评分，改成下面的补课。
- `setfacl -m` 是修改条目，`-x` 是删一条。`getfacl` 看结果。默认 ACL（`-d`）没讲。
- 有效权限是该条目和 mask 的交集。mask 不限制 owner，也不限制 other。`ls -ld` 中间原来的组权限位，有 ACL 时对应该 mask。
- 目录上的写权限管的是在目录里创建、删除条目。它不等于已经存在的文件内容能不能写。
- `touch` 已有文件通常只改时间。属主即使没有 `w`，往往仍能改自己文件的时间；不是属主时，有 `w` 也可以改。不能把「是属主」当成「默认就有 `w`」。
- `getfacl` 打出 `Removing leading /` 是提示，不是失败。

## 关键步骤

```bash
sudo useradd exam02
```

exam02 的 uid 是 3102，主组是 exam02，不在 ops_exam 里。

正确授权：

```bash
sudo setfacl -m u:exam02:rx /srv/ops-exam
```

这次 `getfacl` 的结果：属主 root，属组 ops_exam，flags `-s-`，`user::rwx`，`user:exam02:r-x`，`group::rwx`，`mask::rwx`，`other::---`。

```bash
sudo -u exam02 ls /srv/ops-exam
sudo -u exam02 sh -c 'cd /srv/ops-exam && pwd'
sudo -u exam02 touch /srv/ops-exam/exam02-test.txt
```

`ls` 能看到 `test.txt`。`cd` 成功并打印了路径。当时还没有 `exam02-test.txt`，`touch` 是 `Permission denied`。

```bash
sudo setfacl -m m::r-- /srv/ops-exam
```

exam02 那条和 group 那条都变成 `effective:r--`，`cd` 失败。随后自行改回：

```bash
sudo setfacl -m m::rwx /srv/ops-exam
```

`cd` 又成功。

独立验收时临时改成 `rwx`，`touch` 创建出 `exam02-test.txt`。再改回 `rx` 时又漏了 `-m`，改正后才是：

```bash
sudo setfacl -m u:exam02:rwx /srv/ops-exam
sudo -u exam02 touch /srv/ops-exam/exam02-test.txt
sudo setfacl -m u:exam02:rx /srv/ops-exam
```

改回 `rx` 之后，再 `touch` 同一个 `exam02-test.txt` 仍然成功，因为文件已经在，只是改时间。换成还不存在的名字就失败：

```bash
sudo -u exam02 touch /srv/ops-exam/exam02-new.txt
```

`Permission denied`。不能创建。

```bash
sudo -u exam02 sh -c 'echo "ACL test" >> /srv/ops-exam/exam02-test.txt'
```

追加成功。当前用户 fqh 直接 `cat /srv/ops-exam/exam02-test.txt` 是 `Permission denied`。`sudo cat` 和 `sudo -u exam02 cat` 都读到 `ACL test`。

删除条目的命令只在课后口答，没有执行。结束时 ACL 里仍有 `user:exam02:r-x`。口答的命令是：

```bash
sudo setfacl -x u:exam02 /srv/ops-exam
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| exam02 不在 ops_exam | `id` | uid 3102，主组 exam02 | ✅ |
| 命名用户 rx | `sudo setfacl -m u:exam02:rx /srv/ops-exam` 后 `getfacl` | `user:exam02:r-x`，mask `rwx` | ✅ |
| 列出、进入 | `sudo -u exam02 ls`；`cd && pwd` | 能看到 `test.txt`，pwd 成功 | ✅ |
| 尚无写权限时新建 | `touch exam02-test.txt` | `Permission denied` | ✅ |
| mask 收紧 | `sudo setfacl -m m::r-- /srv/ops-exam` | exam02 与 group 的 effective 都是 `r--`，`cd` 失败 | ✅ |
| 恢复 mask | `sudo setfacl -m m::rwx /srv/ops-exam` | `cd` 成功 | ✅ 自己改回的 |
| 临时 rwx 后创建 | `u:exam02:rwx` 再 touch | 文件创建成功 | ✅ |
| 回到 rx 后再 touch 同名 | `touch exam02-test.txt` | 成功，只改已有文件的时间 | ✅ |
| 回到 rx 后新建另一个名字 | `touch exam02-new.txt` | `Permission denied` | ✅ |
| 已有文件追加 | `echo "ACL test" >> exam02-test.txt` | 成功 | ✅ |
| 谁能读 | fqh 直接 cat；`sudo cat`；`sudo -u exam02 cat` | 只有第一种 `Permission denied`，后两种是 `ACL test` | ✅ |
| 删除 exam02 条目 | `sudo setfacl -x u:exam02 /srv/ops-exam` | 条目消失 | 未执行。口答 1 分。结束时条目还在 |
| 默认 ACL 继承 | | | 未讲，未做 |

## 踩坑

- 第一次 `setfacl`、后来恢复授权、课后第 1 题，都漏了 `-m` → 出现 `Usage`，`-m` 才是修改 → 改成 `sudo setfacl -m u:exam02:rx /srv/ops-exam`。第 1 题因此 0.5 分。
- `getfacl` 出现 `Removing leading /` → 这是提示，不是错误 → 命令不用改。
- `m::r--` 之后 group 那一行也是 `effective:r--`，`cd` 失败 → 有效权限是条目和 mask 的交集，mask 管到命名用户和属组，不管 owner 和 other → 改回 `sudo setfacl -m m::rwx /srv/ops-exam`，`cd` 成功。
- 回到 `rx` 后 `touch` 同名文件仍成功，误当成还能新建 → 文件已在，touch 只改时间 → 改用还不存在的 `exam02-new.txt`，得到 `Permission denied`。
- fqh 直接 `cat` 失败，当时用普通 `cat` 当验证 → 当前用户不是 exam02，也不是 root → 改用 `sudo cat` 和 `sudo -u exam02 cat`，两边都是 `ACL test`。

## 卡点 / 没想明白的

- 目录允许创建、删除名字，不等于文件内容已经可写。`echo >>` 写的是已有文件。
- 属主不一定有 `w`。`touch` 改时间不能用来证明有写权限。
- `-x` 只说对了，没有真的删。复习时不要把「还留着 exam02 的 rx」记成已经删过。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题。总分 4.5/5。

1. （操作）写出给 exam02 加 `rx` 的命令。
<details><summary>批改要点</summary>

`sudo setfacl -m u:exam02:rx /srv/ops-exam`。漏了 `-m`。0.5 分。
</details>

2. （概念）用户条目是 `rwx`，mask 是 `r-x`，有效权限是什么。
<details><summary>批改要点</summary>

交集是 `r-x`。1 分。
</details>

3. （排障）已经有 `exam02-test.txt` 时，`touch` 同名文件为什么还会成功。
<details><summary>批改要点</summary>

改的是已有文件的时间戳，不是新建。1 分。补充：属主不一定有 `w`。
</details>

4. （操作）删掉 exam02 这条 ACL。
<details><summary>批改要点</summary>

`sudo setfacl -x u:exam02 /srv/ops-exam`。1 分。只口答，没有执行。结束时 `getfacl` 里仍有 exam02 的 `rx`。
</details>

5. （概念）`ls -ld` 中间那组权限位在有 ACL 时对应什么。
<details><summary>批改要点</summary>

对应 mask。1 分。
</details>

作答结果：4.5/5。薄弱点：反复漏 `-m`；把 touch 已有文件当成新建；用 fqh 的 `cat` 验证；mask 会限制属组。

## 复习记录

起算日是 11-20。

- [x] D+1（11-21，2/2（合并复习，六题合计 5.5/6，折合 4.58/5，不是 5/5，也不是本篇单独 5 题）；`sudo setfacl -m u:exam02:rx /srv/ops-exam` 1 分；`r-x` 与 mask `r--` 的有效权限是 `r--` 1 分）
- [x] D+3（11-23，1/1（合并复习，五题合计 5/5，不是本篇单独 5 题）；`sudo setfacl -m u:exam02:rx /srv/ops-exam`）
- [x] D+7（11-27，0.5/1（合并复习，今日到期 4 篇合计 3.5/4、含弱项回看 6 题总计 5.5/6，不是本篇单独 6 题）；答对 `group` 受 `mask` 限制，但误把 `other` 也算入受 `mask` 限制，已纠正 `mask` 不管 `owner` 和 `other`）
