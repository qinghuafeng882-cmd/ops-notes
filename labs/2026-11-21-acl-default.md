# 默认 ACL：只套到新建对象

- 日期：2026-11-21（北京时间 09:00 开课，10:04 课后 5 题做完）
- 考点：RHCSA EX200 —— 默认 ACL。访问 ACL 和 mask 沿用 [11-20](2026-11-20-acl-basics.md)
- 环境：Rocky Linux 10.2（VirtualBox），node1，用户 fqh，目录 `/srv/ops-exam`
- 状态：短课做完。模拟②仍不是完整闭卷。简历、JD、投递未做
- 来源：课堂终端记录。归档时没有重跑

## 目标

给 `/srv/ops-exam` 加上 exam02 的默认 `rx`，看新建文件和新建目录各自拿到什么，已有对象不动。

## 关键概念

- 开做前先说：默认 ACL 只影响之后新建的对象，不改已有文件和目录。用户答「不会」。
- 默认条目写在 `default:` 下。没有 `default:` 的是当前对象自己的访问 ACL。
- 新建目录会同时带上访问 ACL 和默认 ACL。新建的普通文件只有访问 ACL，没有 `default:` 条目。
- SGID 管新建对象的属组（这里仍是 ops_exam）。默认 ACL 管新建对象的 ACL 条目。两件事分开。
- `touch` 新建普通文件时请求的 mode 通常是 0666，不含执行位。这次文件上的 mask 是 `rw-`。不要把默认 ACL 写成会自动给普通文件执行权。
- `mkdir` 会请求执行位。这次新目录的 mask 是 `rwx`。
- 有效权限仍是条目和 mask 的交集。exam02 的 `r-x` 碰上文件 mask `rw-`，有效是 `r--`。
- 课堂补讲：默认 ACL 是创建时套用的权限模板，不是之后持续把旧文件改成一样。这不是把 ACL 简单说成「总是和 umask 取交集」。
- 为什么旧文件不会变，用户当时答不上来，并觉得课堂没讲。老师承认先前只给了结论，机制没讲透。课后第 2 题的 1 分不是因为用户独立把机制讲完整。

## 关键步骤

```bash
# 09:15 漏了 -m，出现 Usage
sudo setfacl d:u:exam02:rx /srv/ops-exam
# 补上 -m 后成功
sudo setfacl -m d:u:exam02:rx /srv/ops-exam
```

`getfacl`：属主 root，属组 ops_exam，flags `-s-`。访问 ACL 是 `user::rwx`、`user:exam02:r-x`、`group::rwx`、`mask::rwx`、`other::---`。新增的默认条目是 `default:user::rwx`、`default:user:exam02:r-x`、`default:group::rwx`、`default:mask::rwx`、`default:other::---`。

```bash
# 09:21
sudo -u exam01 touch /srv/ops-exam/default-file01.txt
```

fqh 直接 `getfacl` 是 `Permission denied`。`sudo getfacl` 成功：属主 exam01，属组 ops_exam。`user::rw-`，`user:exam02:r-x` 且 `#effective:r--`，`group::rwx` 且 `#effective:rw-`，`mask::rw-`，`other::---`。输出里没有 `default:` 条目。

```bash
# 09:42 空文件，cat 没有正文
sudo -u exam02 cat /srv/ops-exam/default-file01.txt
echo $?    # 0
```

```bash
# 09:46
sudo -u exam01 mkdir /srv/ops-exam/default-dir01
```

`sudo getfacl`：属主 exam01，属组 ops_exam，flags `-s-`。访问 ACL 是 `user::rwx`、`user:exam02:r-x`、`group::rwx`、`mask::rwx`、`other::---`，并且上面那组 `default:` 条目都在。

```bash
# 09:53
sudo -u exam01 touch /srv/ops-exam/default-dir01/nested-file01.txt
```

`sudo getfacl`：属主 exam01，属组 ops_exam。`user::rw-`，`user:exam02:r-x` 且 `#effective:r--`，`group::rwx` 且 `#effective:rw-`，`mask::rw-`，`other::---`。没有 `default:` 条目。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 加上默认条目 | `sudo setfacl -m d:u:exam02:rx /srv/ops-exam` 后 `getfacl` | 访问 ACL 还在，并多出 `default:` | ✅ 09:15 |
| 新建普通文件 | `sudo -u exam01 touch .../default-file01.txt` 后 `sudo getfacl` | 只有访问 ACL，mask `rw-`，exam02 有效 `r--` | ✅ 09:21 |
| exam02 能读空文件 | `sudo -u exam02 cat` 后 `echo $?` | 无正文，退出码 0 | ✅ 09:42 |
| 新建目录 | `sudo -u exam01 mkdir .../default-dir01` 后 `sudo getfacl` | 访问 ACL 和全部 `default:` 都在，flags `-s-` | ✅ 09:46 |
| 子目录里再建文件 | `touch .../nested-file01.txt` 后 `sudo getfacl` | 与 `default-file01.txt` 同类：mask `rw-`，无 `default:` | ✅ 09:53 |
| 已有文件被默认 ACL 改掉 | 对旧文件做前后 `getfacl` | 不应改变 | 未实测。只解释过 |
| 进入 `default-dir01` | `cd` | | 未做 |
| 写入被拒绝 | | | 未做 |
| 删除默认规则 | | | 未做 |
| `chmod` | | | 未做 |
| 重启后仍在 | | | 未验证 |

## 踩坑

- `sudo setfacl d:u:exam02:rx /srv/ops-exam` → 漏了 `-m`，出现 `Usage` → 改成 `sudo setfacl -m d:u:exam02:rx /srv/ops-exam`。课后第 1 题又漏了一次，首答 0.5，后面订正全对，不改这 0.5。
- fqh 对 `default-file01.txt` 跑 `getfacl` → `Permission denied` → fqh 没有这个目录的穿越权 → 改用 `sudo getfacl`。
- 已有文件会不会变，用户只答「不会」，为什么说不上来 → 先前只给了结论 → 补讲成创建时的模板，不是持续同步。第 2 题仍是 1 分，不写成用户自己把机制讲完整。
- 合并复习里把 `uniq` 说成对相邻相同行「排序」→ `uniq` 是合并统计，排序是 `sort` 的事 → 仍给 1 分。
- 合并复习里写成 `Last_IO_Errors` → 多了一个 `s`，字段是 `Last_IO_Error` → 首答 0.5，订正不改分。

## 卡点 / 没想明白的

- 文件上的 `r-x` 为什么有效只剩 `r--`：mask 是 `rw-`，交集丢掉 `x`。这次 mask 来自 touch 请求的 0666，不是默认 ACL 另发了执行权。
- 目录 mask 是 `rwx`，因为 `mkdir` 请求执行位。

## 面试题（5 道，对话中作答后批改）

下面是 10:04 做完的课后 5 题。总分 4.5/5。

1. （操作）写出给目录添加默认条目的完整命令。
<details><summary>批改要点</summary>

`sudo setfacl -m d:u:exam02:rx /srv/ops-exam`。又漏了 `-m`。0.5 分。随后订正全对，保留首答分数。
</details>

2. （概念）设了默认 ACL 之后，已经存在的文件会变吗？
<details><summary>批改要点</summary>

不会。用户答了「不会」，为什么没说上来，并觉得课堂没讲。老师承认只给过结论，机制没讲透。补讲：创建时的权限模板，不是持续同步。1 分。这 1 分不是独立把机制讲完整得来的。
</details>

3. （概念）条目是 `r-x`、mask 是 `rw-` 时，有效权限是什么。
<details><summary>批改要点</summary>

交集是 `r--`。1 分。
</details>

4. （概念）没有 `default:` 的条目和 `default:` 条目各管什么。
<details><summary>批改要点</summary>

没有 `default:` 的是当前对象的访问权限。`default:` 用于之后新建的对象。1 分。
</details>

5. （概念）为什么这次文件和目录的 mask 不一样。
<details><summary>批改要点</summary>

`touch` 不请求执行位，`mkdir` 请求执行位，所以这次 mask 不同。1 分。普通文件没有因此得到执行权。
</details>

作答结果：4.5/5。薄弱点：又漏 `-m`；「不会」背后的机制当时说不上来。

## 复习记录

起算日是 11-21。这三项都还没做。

- [ ] D+1
- [ ] D+3
- [ ] D+7
