# 速查表 W5：shell 脚本、计划任务、时间同步

> 只放最常用的写法和易错点。详细原理和踩坑见 `labs/` 对应笔记。

## 执行脚本

| 方式 | x 权限 | 在哪执行 | 变量保留 |
|---|---|---|---|
| `bash x.sh` | 不要 | 子进程 | ❌ |
| `./x.sh` | 要 | 子进程 | ❌ |
| `source ./x.sh` | 不要 | 当前 shell | ✅ |

⚠️ 自己的脚本带 `./`：不带 `/` 时直接执行和 source 都先查 PATH（`type -a 名字` 查同名）。
⚠️ source 的脚本里有 `exit` 会退出当前终端。

```bash
bash -n x.sh            # 只检查语法
bash -x x.sh            # 打印每行展开后的样子
type -a [               # 内置命令 / 外部命令 / 关键字
```

## 变量与引号

```bash
name=fqh                # = 两边不能有空格
echo "${name}_log"
today=$(date +%F)       # 命令替换
port="${2:-80}"         # $2 为空时用 80
export JAVA_HOME=/opt/java   # 子进程能继承
```

| 引号 | `$` | 空格拆分 | 通配符 |
|---|---|---|---|
| `'...'` | ❌ | ❌ | ❌ |
| `"..."` | ✅ | ❌ | ❌ |
| 无 | ✅ | ✅ | ✅ |

**引用变量一律加双引号。**

| 变量 | 含义 |
|---|---|
| `$0` `$1` `${10}` | 脚本名、参数 |
| `$#` | 参数个数 |
| `"$@"` | 全部参数，各自独立（传递参数用它） |
| `$?` | 上一条命令的退出码 |
| `$$` | 当前 shell 的 PID |

## 退出码

0 成功；非 0 失败。grep：1 没找到、2 出错。shell：127 找不到命令、126 不能执行。
`a && b` 成功才执行 b；`a || b` 失败才执行 b；`a; b` 都执行。脚本里 `exit N`。

## if / test

```bash
if [[ $# -ne 1 ]]; then
    echo "usage: $0 ARG" >&2      # 错误信息写到标准错误
    exit 2
elif id "$1" &>/dev/null; then    # if 判断的是命令的退出码
    echo "exists"
else
    echo "no" >&2; exit 1
fi
```

| 类别 | 写法 |
|---|---|
| 字符串 | `=` `!=` `-z`（空）`-n`（非空） |
| 整数 | `-eq -ne -lt -le -gt -ge`（不要用 `>`，在 `[ ]` 里是重定向） |
| 文件 | `-e` 存在 `-f` 文件 `-d` 目录 `-r -w -x` 权限 `-s` 非空 |

⚠️ `[` 是命令：`[`、`]` 两边要空格；变量不加引号时，空值会报 `unary operator expected`，或者更糟：`[ -n $a ]` 不报错但结果错。写 bash 脚本用 `[[ ]]`。

## 循环

```bash
for f in /var/log/*.log; do echo "$f"; done      # 遍历文件用通配符，不用 $(ls)
for s in "$@"; do ...; done                      # 遍历参数
for i in {1..5}; do ...; done
for ((i=1; i<=5; i++)); do ...; done
while IFS=: read -r user _ uid _; do ...; done < /etc/passwd
while read -r pcent target; do ...; done < <(df --output=pcent,target | tail -n +2)
```

⚠️ `cmd | while read` 在子进程，变量带不回 → 用 `< 文件` 或 `< <(命令)`。

## 算术

```bash
total=$(( a + b ))           # 取值
(( use > 80 )) && echo big   # 退出码：真 → 0
(( n += 1 ))                 # 比 n++ 安全（n=0 时 n++ 退出码为 1）
```

`(( ))` 里 `>` 是大于、变量不写 `$`、只有整数。

## case / 函数

```bash
case "$1" in
    start)     ... ;;
    stop|halt) ... ;;
    *)         echo "usage: $0 start|stop" >&2; exit 2 ;;
esac

check() {
    local svc="$1"                     # local！
    systemctl is-active --quiet "$svc" # 最后一条的退出码就是函数的退出码
}
check sshd && echo ok                  # 退出码：$? / if
out=$(myfunc arg)                      # 输出：$(...)
```

`shift` 参数左移；`set -- a b c` 重设参数；脚本不写 exit 时退出码 = 最后一条命令的。

## 参数展开

| 写法 | `f=report.tar.gz` |
|---|---|
| `${f#*.}` / `${f##*.}` | `tar.gz` / `gz`（从开头删） |
| `${f%.gz}` / `${f%%.*}` | `report.tar` / `report`（从末尾删） |
| `${p%\%}` | 去掉末尾的 `%` |
| `${1:-80}` | 为空时用 80 |

## 常用取数

```bash
read -r load1 _ < /proc/loadavg              # 1 分钟负载
free -m | awk '/^Mem:/{print $7}'            # available（MiB）
df --output=pcent,target -x tmpfs -x devtmpfs | tail -n +2
systemctl --failed --no-legend | wc -l       # --no-legend：不要表头和说明
```

## 排错

| 报错 | 原因 |
|---|---|
| `unexpected end of file` | 成对结构没结尾（fi / done / esac / } / 引号） |
| `unexpected token 'do'` | if 后面写成了 do（应为 then） |
| 输出里出现 `$svc` 字样 | 单引号里的变量不展开 |

`shellcheck x.sh` → `bash -n x.sh` → 运行。`vim +18 x.sh` 跳到第 18 行。

## 计划任务：cron

```
分 时 日 月 星期  命令
*/5 * * * *       每 5 分钟
30 2 * * *        每天 02:30
30 8 * * 1-5      工作日 08:30
0 0 1 * *         每月 1 号
0 3 * * 0         每周日 03:00（0 和 7 都是周日）
```

```bash
crontab -e / -l              # 不要用 -r（全删）
sudo crontab -u fqh -l       # 看别人的
sudo ls /var/spool/cron/     # 用户 crontab 存放处
cat /etc/crontab; ls /etc/cron.d/    # 系统级：时间 + 用户名 + 命令
sudo grep CMD /var/log/cron | tail   # 执行记录（或 journalctl -t CROND）
```

⚠️ cron 环境：PATH 极简 → **绝对路径**；不读 .bashrc；输出 → **`>> /tmp/x.log 2>&1`**；**`%` 写成 `\%`**。
⚠️ 日和星期同时指定是"或"。

## 计划任务：systemd timer

```ini
# /etc/systemd/system/inspect.service
[Unit]
Description=Daily inspection
[Service]
Type=oneshot
ExecStart=/usr/local/bin/inspect.sh

# /etc/systemd/system/inspect.timer
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=10min
[Install]
WantedBy=timers.target
```

```bash
sudo systemd-analyze verify /etc/systemd/system/inspect.timer   # 先查拼写
sudo systemctl daemon-reload && sudo systemctl enable --now inspect.timer   # enable 的是 timer
systemctl list-timers                       # NEXT / LAST
sudo systemctl start inspect.service        # 手动跑一次
journalctl -u inspect.service -n 30
systemd-analyze calendar "Mon..Fri 08:30" --iterations=3
sudo install -m 755 ~/x.sh /usr/local/bin/x.sh    # 脚本放系统目录（SELinux 标签 bin_t）
```

| OnCalendar | 含义 |
|---|---|
| `*-*-* 02:00:00` / `daily` | 每天 |
| `Mon..Fri 08:30` | 工作日 |
| `*:0/5` | 每 5 分钟 |

| | cron | timer |
|---|---|---|
| 日志 | 自己重定向 | journal |
| 失败 | 看不见 | failed |
| 关机错过 | 丢（cron.daily 由 anacron 补） | Persistent=true 补跑 |

anacron = anachronistic cron，补跑 /etc/cron.daily 等。
