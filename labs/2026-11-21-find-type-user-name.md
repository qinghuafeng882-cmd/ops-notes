# find：-type、-user、-name 与退出码

- 日期：2026-11-21（北京时间 10:45–11:02）
- 考点：RHCSA EX200 —— `find` 按类型、属主、文件名查找。不是 RHCSA 模拟②的完整闭卷
- 环境：Rocky Linux 10.2（VirtualBox），node1，用户 fqh，目录 `/srv/ops-exam`
- 状态：短课做完，课后 5/5。简历、JD、投递未做
- 来源：课堂终端记录。归档时没有重跑

## 目标

在 `/srv/ops-exam` 下面找出 exam01 的普通文件，分清类型、属主、文件名和大小条件，并知道没匹配到时退出码仍可以是 0。

## 关键概念

- `-t`、`-u` 不是 `-type`、`-user` 的缩写。
- `-name '*.txt'` 要加引号。不加引号时，Bash 可能先按当前目录把 `*.txt` 展开。当前目录没有匹配时，默认可能把通配符原样留下。不把「每次都会展开」写死。`-name` 只比文件名（basename），不比整条路径。
- 多个条件默认是并且（AND），要同时满足。这一例里把条件前后互换，结果相同。课堂这样讲，没有另跑一遍互换后的命令。
- `-size +1M` 是大于 1 MiB，1 MiB = 1048576 字节。这节没有讲 `-size 1M` 和 `-size -1M` 的单位进位边界。
- 这种默认把路径打到屏幕上的用法，一个都没匹配到也可以退出码 0。0 不是「命令没跑成」。
- `find` 也会检查起始目录自己。`/srv/ops-exam` 的属主是 root，所以 `-user exam01` 不会把它列出来。

## 关键步骤

任务是递归找出 `/srv/ops-exam` 里 exam01 的普通文件。最初写的是 `sudo find /srv/ops-exam -t f -u exam01`。这条没有执行。

```bash
# 10:48
sudo find /srv/ops-exam -type f -user exam01
```

三行：

```text
/srv/ops-exam/test.txt
/srv/ops-exam/default-file01.txt
/srv/ops-exam/default-dir01/nested-file01.txt
```

10:50 口答在上面加上 `-name`，对。10:51 口答 `-name` 匹配的是文件名，对。10:53 口答把类型、名字、属主、大小写在一起，对。

```bash
# 10:55
sudo find /srv/ops-exam -type f -name '*.txt' -user exam01 -size +1M
echo $?
```

没有输出。`echo $?` 是 0。没有为了让它有结果去造大文件。

10:57 口答 `sudo find /srv/ops-exam -type d -user exam01`，对。

```bash
# 10:58
sudo find /srv/ops-exam -type d -user exam01
```

只输出 `/srv/ops-exam/default-dir01`。

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| exam01 的普通文件 | `sudo find /srv/ops-exam -type f -user exam01` | 三个已有 `.txt` | ✅ 10:48，就是上面三行 |
| 大于 1 MiB | 加上 `-name '*.txt' -size +1M` | 没有这样的文件 | ✅ 无输出，`$?` 为 0。不是执行失败 |
| exam01 的目录 | `sudo find /srv/ops-exam -type d -user exam01` | 不含属主为 root 的起点 | ✅ 只有 `default-dir01` |
| `-size` 有命中 | | | 未实测。没有创建大文件 |
| `-size 1M`、`-size -1M` 的进位 | | | 未讲，未测 |
| `-delete`、`-exec`、按 mtime、复制、移动 | | | 未做 |
| 目录名本身是 `.txt` 的查找 | `-type d -user exam01 -name '*.txt'` | 默认同时满足 | 只口答，没有实执行 |
| 重启后这些文件仍在 | | | 未验证 |

## 踩坑

- 写成 `sudo find /srv/ops-exam -t f -u exam01` → `-t`、`-u` 不是类型和属主的缩写，要用 `-type`、`-user` → 没有执行，所以没有报错输出。改正后的命令是 10:48 那条。

## 卡点 / 没想明白的

- 无输出却是 0：这条 find 是把匹配路径打出来，没匹配到也算正常结束。
- 起始目录没有出现在 `-type d -user exam01` 的结果里，因为它的属主是 root。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题。总分 5/5。

1. （操作）找出 exam01 的、名字以 `.txt` 结尾的普通文件。
<details><summary>批改要点</summary>

`sudo find /srv/ops-exam -type f -name '*.txt' -user exam01`。10:59，1 分。
</details>

2. （概念）`-name '*.txt'` 的引号做什么。
<details><summary>批改要点</summary>

防止 Bash 先展开通配符。11:00，1 分。
</details>

3. （排障）find 没有输出，退出码却是 0，是不是失败了。
<details><summary>批改要点</summary>

这种默认打印路径的用法，没有匹配也是正常结束，退出码可以是 0。11:01，1 分。
</details>

4. （操作）找出 exam01 的目录。
<details><summary>批改要点</summary>

`sudo find /srv/ops-exam -type d -user exam01`。11:02，1 分。
</details>

5. （概念）`-type d -user exam01 -name '*.txt'` 是什么关系。
<details><summary>批改要点</summary>

默认同时满足。1 分。没有另跑一条去找名字是 `.txt` 的目录，不编造那样的输出。
</details>

作答结果：5/5。最初的缩写 `-t`、`-u` 没有执行，仍然是书写上的坑。

## 复习记录

起算日是 11-21。

- [x] D+1（11-22，1/1（合并复习，五题合计 4.5/5，不是本篇单独 5 题）；无输出且退出码 0，在默认打印路径的用法里是正常的没有匹配）
- [x] D+3（11-26，0.5/1（合并复习，六题合计 5/6，不是本篇单独 6 题）；`-type f -user exam01` 拼写正确，但 `*.txt` 漏了单引号，且以普通用户查 `/srv/ops-exam` 时需加 `sudo`）
- [x] D+7（11-28，1/1（合并复习，今日到期 5 篇 6 题合计 5/6、含 11-20 弱项回看 7 题总计 6/7，不是本篇单独 7 题）；一次写出完整命令 `sudo find /srv/ops-exam -type f -user exam01 -name '*.txt'`，已修正 D+3 的 `sudo` 与引号弱项）
