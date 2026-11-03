# shell 脚本（下）：循环、逐行读取、算术、case、函数、巡检脚本

- 日期：2026-11-03（W5-D2）
- 考点：RHCSA EX200 —— 用循环（for 等）处理文件和命令行输入；处理脚本输入；在脚本中处理命令输出
- 环境：Rocky Linux 10.2（VirtualBox），node1，`~/w5d2`；装了 ShellCheck（EPEL）
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

掌握 for/while 循环、`while read` 逐行处理、管道子进程陷阱、`$(( ))`/`(( ))`、case、函数，写出一个带退出码的服务器巡检脚本 `inspect.sh`。

## 关键概念

| for 的列表 | 例子 | 文件名有空格 |
|---|---|---|
| 写出来 / 花括号 | `sshd httpd`、`{1..5}` | — |
| **通配符** | `*.log` | ✅ 安全（bash 直接匹配文件名） |
| 参数 | `"$@"` | ✅ |
| 命令输出 | `$(ls)`、`$(cut ...)` | ❌ 被按空格拆开 |

- `while IFS=: read -r a b c; do ...; done < 文件`：read 每次读一行；IFS（Internal Field Separator，内部字段分隔符）只对这次 read 生效；`-r` 不把 `\` 当转义；变量不够时剩下的全进最后一个；`_` 占位。
- **管道陷阱**：`cmd | while read` 的 while 在子进程里，循环里改的变量带不回来 → `done < 文件` 或 `done < <(命令)`（进程替换）。

| 写法 | 循环在哪 | 变量保留 |
|---|---|---|
| `cat f \| while read` | 子进程 | ❌ |
| `while read ...; done < f` | 当前 shell | ✅ |
| `while read ...; done < <(cmd)` | 当前 shell | ✅ |

- 算术：`$(( ))` 返回值；`(( ))` 返回退出码。两层：表达式值 0 = 假、非 0 = 真；`(( ))` 把"真"翻译成退出码 0（成功）→ 和"0 是成功"一致。`(( ))` 是 bash 语法，里面 `>` 是大于、变量不用 `$`；只有整数。坑：`n=0; (( n++ ))` 退出码是 1（后置自增取旧值 0）。
- case：`case "$1" in a) ...;; b|c) ...;; *) ...;; esac`，支持通配符。
- 函数：函数里的 `$1`/`"$@"` 是函数的参数；`local` 防止改到外面同名变量；`return N` 是退出码（用 `$?` 或 `if 函数`），`echo` 是输出（用 `x=$(函数)`）。
- 脚本没写 exit 时，退出码 = 最后一条命令的退出码（如 `(( bad == 0 ))`）。
- `shift` 位置参数左移；`set -- a b` 重设位置参数。
- `${f#模式}`/`${f##模式}` 从开头删最短/最长；`${f%模式}`/`${f%%模式}` 从末尾删（键盘上 `#` 在 `$` 左边、`%` 在右边）。`${pcent%\%}` 删掉末尾的 `%` 字符。
- `/proc` 是内核把内部数据显示成文件的虚拟文件系统；`/proc/loadavg` 前三个数是 1/5/15 分钟平均负载。
- `free -m` 以 MiB 显示，第 7 列 available；脚本里不用 `-h`（带单位无法比较）。
- `df --output=pcent,target -x tmpfs -x devtmpfs | tail -n +2`：只要使用率和挂载点、排除内存文件系统、跳过表头。
- `systemctl ... --no-legend`：不输出表头和末尾说明，只留数据行（给 awk / wc 用）。
- `unexpected end of file` = 某个成对结构没结尾（if/fi、do/done、case/esac、{}、引号），报在文件末尾，错在前面。

## 关键步骤

```bash
for f in *.log; do echo "[$f]"; done
while IFS=: read -r user _ uid _ _ home shell; do
    (( uid >= 1000 && uid < 60000 )) && printf '%-10s UID=%-6s %s\n' "$user" "$uid" "$shell"
done < /etc/passwd
while read -r line; do (( count++ )); done < <(grep -v nologin /etc/passwd)
shellcheck svc.sh; bash -n svc.sh
vim +18 svc.sh                    # 跳到第 18 行（vim svc.sh 18 会当成第二个文件）
```

```bash
#!/bin/bash
# inspect.sh [磁盘阈值，默认 80] —— 有问题时退出码为 1
threshold="${1:-80}"
problems=0
echo "===== $(hostname) $(date '+%F %T') ====="
read -r load1 _ < /proc/loadavg
echo "load(1min): $load1   cpus: $(nproc)"
avail_mb=$(free -m | awk '/^Mem:/{print $7}')
echo "mem available: ${avail_mb} MB"
echo "--- disk ---"
while read -r pcent target; do
    use=${pcent%\%}
    if (( use >= threshold )); then
        echo "WARN  $target ${use}%"; (( problems++ ))
    else
        echo "OK    $target ${use}%"
    fi
done < <(df --output=pcent,target -x tmpfs -x devtmpfs | tail -n +2)
echo "--- services ---"
for svc in sshd httpd firewalld chronyd; do
    if systemctl is-active --quiet "$svc"; then
        echo "OK    $svc"
    else
        echo "FAIL  $svc"; (( problems++ ))
    fi
done
failed=$(systemctl --failed --no-legend | wc -l)
if (( failed > 0 )); then
    echo "failed units: $failed"; (( problems++ ))
fi
echo "===== problems: $problems ====="
(( problems == 0 ))
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| `$(ls)` vs 通配符 | 实验 1 | 4 项 / 3 项 | ✅ |
| 普通用户列表 | `./users.sh` | UID 1000～60000 | ✅ 14 个 |
| 管道陷阱 | 实验 3 | 0 / 50 / 17，与 `wc -l`、`grep -vc` 一致 | ✅ |
| svc.sh 退出码 | check / check 含不存在服务 / 无参数 | 0 / 1 / 2 | ✅ |
| inspect 默认阈值 | `./inspect.sh` | 全 OK，退出码 0 | ✅ |
| inspect 阈值 15 | `./inspect.sh 15` | 4 块盘 WARN，problems 4，退出码 1 | ✅ |
| 输出重定向到文件 | `> /tmp/inspect-日期.txt` | 内容完整 | ✅ |

## 踩坑

- 实验 3 误敲 `{{ count++ }}` → 50 行 `{{: command not found` → 恰好证明循环跑了 50 次（在子进程里）。
- svc.sh：`for s in "$#"`（应为 `"$@"`）→ 输出 `FAIL 3`、`FAIL 2`，3/2 就是参数个数 → 从意外的值反推错误；ShellCheck 报 SC2066。`--no-legent` 拼错；`;;` 写成 `;`。
- inspect.sh：`if ...; do`（应为 then）→ `syntax error near unexpected token 'do'`；少了 `fi` → `line 42: unexpected end of file`（文件只有 41 行）。
- inspect.sh 运行前还有 bash -n 查不出的错：`threshold="$1"` 丢了默认值（全报 WARN）、`$loda1`、`--quite`（全报 FAIL）、`echo 'ok "$svc"'` 单引号不展开、`$"problems"`。→ 先 shellcheck 再运行。
- 坚持自己敲脚本练手感；改进练法：看懂 → 遮住原文凭记忆敲 → 对照 → `bash -n` + `shellcheck`。

## 卡点 / 没想明白的

- `(( ))` "结果非 0 是成功"和"0 是成功"看似矛盾 → 两层：表达式真假 → 翻译成退出码。
- `/proc/loadavg`、`free -m`、`${pcent%\%}`、`df --output ... | tail -n +2`（已补讲，见关键概念）。
- 热身：grep 退出码 1 和 2 记反（1 没找到，2 出错）。
- 达标题 4：以为 `((` 也是内置命令 → 它是 bash 语法（和 `[[` 一样在展开前处理），`[` 才是命令。
- 达标题 5：把 `$(函数)` 当成拿退出码 → `$(函数)` 拿输出，退出码用 `$?` / `if 函数`；`变量=函数名 参数` 语法不对，应为 `变量=$(函数名 参数)`。

## 回忆题

1. （概念）`cat f | while read l; do (( n++ )); done; echo $n` 为什么是 0？写出两种正确写法。
<details><summary>答案</summary>

管道两边各在独立的子进程，while 在子进程里加，结束后丢失。`while read -r l; do (( n++ )); done < f`；`while read -r l; do (( n++ )); done < <(cat f)`。
</details>

2. （操作）逐行读 /etc/passwd，打印 shell 是 `/sbin/nologin` 的用户名，并统计个数。
<details><summary>答案</summary>

`n=0; while IFS=: read -r u _ _ _ _ _ sh; do [[ $sh == /sbin/nologin ]] && { echo "$u"; (( n += 1 )); }; done < /etc/passwd; echo "total $n"`（用 `+= 1` 避免 `n++` 第一次退出码为 1）
</details>

3. （排障）脚本报 `line 42: syntax error: unexpected end of file`，文件只有 41 行。怎么查？
<details><summary>答案</summary>

文件读完了还有结构没结束：检查 if/fi、do/done、case/esac、{}、引号是否配对，错误在前面某处；shellcheck 能帮忙定位。
</details>

4. （概念）函数里 `return 1` 和 `echo 1` 怎么区分使用？调用方怎么拿到？
<details><summary>答案</summary>

return 设置退出码（成功/失败），用 `$?` 或 `if 函数; then`；echo 是输出数据，用 `x=$(函数)` 接收。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
