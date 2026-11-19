# Nginx access log 字段拆分与汇总脚本

- 日期：2026-11-18
- 考点：RHCSA EX200 —— 简单 shell 脚本（`if` / `[`、退出码、重定向）；Nginx access log 为运维扩展
- 环境：Rocky Linux 10（VirtualBox），node1 192.168.56.11（日志与脚本）/ node2 192.168.56.12（发请求）
- 状态：完成（边界情况未验证，见下表）
- 来源：课堂终端记录。归档时没有在另一台机器上重跑

## 目标

看清 node1 上 Nginx `main` 格式的 access log 怎么被 awk 拆开，并写成只统计状态码的脚本 `~/nginx-log-summary.sh`。

## 关键概念

- 本课生效配置：`access_log /var/log/nginx/access.log main;`
- `main` 格式字符串（`sudo nginx -T | grep -A 3 'log_format'` 显示出的三截）：

```text
'$remote_addr - $remote_user [$time_local] "$request" '
'$status $body_bytes_sent "$http_referer" '
'"$http_user_agent" "$http_x_forwarded_for"'
```

- `nginx -T` 的检查提示在 stderr，有两行，不进管道。少写 `2>/dev/null` 时终端上能看见。原文没有抄下来。
- `grep -A` 是 After：`-A 3` 是匹配行之后的 3 行，空行也算。
- awk 默认按空白拆字段，双引号不会把 `"GET / HTTP/1.1"` 收成一列。本课这条格式里 `$1` 是 IP、`$7` 是路径、`$9` 是状态码。不要套到别的 `log_format`。
- `sort | uniq -c`：`uniq` 只合并相邻的相同行，所以要先排序。
- `wc -l 文件` 输出行数和文件名；`wc -l < 文件` 只有行数。
- `bash -n` 读入命令、不执行，只查语法。无输出不能证明文件存在。「no execution」是记忆说法，不是官方全称。
- `>&2` 把错误送到 stderr，正常结果留在 stdout。
- `sudo ls -l /var/log/nginx/`：当前 `access.log` 是 0 字节，旧文件带有日期。没有抄完整列表，也没有确认日期是写在文件名里还是在 `ls` 的时间列。为什么轮转没有核实（推测）。

## 关键步骤

```bash
# node1：日志路径。检查提示在 stderr，这里丢掉
sudo nginx -T 2>/dev/null | grep -E 'access_log|log_format'
# access_log /var/log/nginx/access.log main;

sudo ls -l /var/log/nginx/

# 不加 2>/dev/null：stderr 两行检查提示不进管道，会显示出来
sudo nginx -T | grep -A 3 'log_format'

# node2 访问了 HTTPS 的 / 和 /not-exist（User-Agent 为 curl/8.12.1）。具体 curl 参数没有留在记录里
# node1：刚写入的两行
sudo tail -n 2 /var/log/nginx/access.log
sudo awk '{print $1,$7,$9}' /var/log/nginx/access.log
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c
```

`tail -n 2` 实测：

```text
192.168.56.12 - - [18/Nov/2026:07:10:05 +0800] "GET / HTTP/1.1" 200 23 "-" "curl/8.12.1" "-"
192.168.56.12 - - [18/Nov/2026:07:10:29 +0800] "GET /not-exist HTTP/1.1" 404 153 "-" "curl/8.12.1" "-"
```

`awk '{print $1,$7,$9}'`：`192.168.56.12 / 200` 和 `192.168.56.12 /not-exist 404`。

`awk '{print $9}' | sort | uniq -c`：`1 200` 与 `1 404`。`uniq -c` 的前导空格没有按原样抄。

讲解后的课堂脚本（`~/nginx-log-summary.sh`，只这一版）：

```bash
#!/bin/bash
log_file="$1"
if [ ! -r "$log_file" ]; then
 echo "文件不可读，退出！" >&2
 exit 1
fi
echo "请求总数：$(wc -l < "$log_file")"
echo "状态码统计如下："
awk '{print $9}' "$log_file" | sort | uniq -c
```

```bash
bash -n ~/nginx-log-summary.sh
sudo bash ~/nginx-log-summary.sh /var/log/nginx/access.log
sudo bash ~/nginx-log-summary.sh /no-such-file
echo $?
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 日志路径 | `sudo nginx -T 2>/dev/null \| grep -E 'access_log\|log_format'` | `access_log /var/log/nginx/access.log main;` | ✅ |
| main 格式 | `sudo nginx -T \| grep -A 3 'log_format'` | 上面三截格式字符串；stderr 两行检查提示另显 | ✅ |
| 两条请求落盘 | `sudo tail -n 2 /var/log/nginx/access.log` | `/` 为 200、`/not-exist` 为 404，来源 192.168.56.12 | ✅ |
| 字段位置 | `sudo awk '{print $1,$7,$9}'` | IP、路径、200/404 | ✅ |
| 状态码次数 | `awk '{print $9}' \| sort \| uniq -c` | `1 200`、`1 404` | ✅ |
| 语法 | `bash -n ~/nginx-log-summary.sh` | 无输出 | ✅ |
| 脚本汇总 | `sudo bash ~/nginx-log-summary.sh /var/log/nginx/access.log` | `请求总数：2`，然后 `1 200` 与 `1 404` | ✅ |
| 不可读 | `sudo bash ~/nginx-log-summary.sh /no-such-file` 后 `echo $?` | `文件不可读，退出！`，`$?` 为 1 | ✅ |
| 空文件 | | | 未验证 |
| 不带参数 | | | 未验证 |
| 路径含空格 | | | 未验证 |
| 普通用户读不了日志 | | | 未验证 |
| 并发写入时计数是否一致 | | | 未验证 |
| 重启后脚本和日志配置仍在 | | | 未验证 |

## 踩坑

书写时发现的问题也记在这里，不要求先跑出报错。下面三条都是初稿未运行，经对话检查发现并修改；没有实际报错输出。

- `if -r "$log_file"; then` → 漏了测试用的方括号；当时不会写否定 → 改为 `[ ! -r "$log_file" ]`。初稿未运行，经对话检查发现并修改；没有实际报错输出。
- `exit(1)`，并且这段 `if` 没有 `fi` → `exit` 这里不用括号，应写成 `exit 1`；`if` 要用 `fi` 收尾 → 改为 `exit 1`，补上 `fi`。初稿未运行，经对话检查发现并修改；没有实际报错输出。
- `wc -l $log_file` → 变量没有加引号，而且 `wc -l` 后面直接跟文件名时会把文件名一起打出来 → 改为 `wc -l < "$log_file"`。初稿未运行，经对话检查发现并修改；没有实际报错输出。

## 卡点 / 没想明白的

- 口答 5/5。独立写仍不熟的是 `[ ! -r ]`、`exit 1`、`fi`。口答对不等于能自己写出来。
- 当前 `access.log` 为 0 字节、旧文件带日期。轮转原因未核实（推测）。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题，没有为凑题型另编。

1. （概念）`sort | uniq -c` 里为什么 `sort` 在前？
<details><summary>批改要点</summary>

`uniq` 只统计连续相同的行，不先排序就把隔开的相同状态码当成多次。
</details>

2. （排障）`bash -n ~/nginx-log-summary.sh` 没有输出，能不能说明这个文件存在？
<details><summary>批改要点</summary>

不能。`bash -n` 只查语法、不执行命令。无输出只说明这次检查没有打出语法错误。
</details>

3. （概念）`>&2` 起什么作用？
<details><summary>批改要点</summary>

错误信息走 stderr，正常结果留在 stdout。
</details>

4. （取舍）`wc -l 文件名` 和 `wc -l < 文件` 差在哪？
<details><summary>批改要点</summary>

前者行数和文件名都有；`<` 重定向只给出行数。
</details>

5. （概念）变量为什么加双引号？
<details><summary>批改要点</summary>

防止按空白拆开，通配符也不展开。
</details>

作答结果：5/5，五题都对。

- 最薄弱：独立写出 `if [ ! -r "$log_file" ]; then ... exit 1; fi`。口答 5/5 不等于写得熟。

## 复习记录

- [x] D+1（11-19，0.5/1（合并复习，三篇合计 4.5/5，不是本篇单独 5 题）；写成 `if [ ! -r $log_file ]; then`，变量没有加引号）
- [ ] D+3
- [ ] D+7
