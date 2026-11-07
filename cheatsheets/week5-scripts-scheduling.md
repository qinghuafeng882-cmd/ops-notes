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

## 排障：CPU、内存、负载

```bash
uptime; nproc                  # 负载 ÷ 核数，三个数看趋势
top                            # P 按 CPU，M 按内存，1 每个核，c 完整命令
vmstat 1 5                     # 第一行是开机以来平均，看第二行起
ps -eo pid,ppid,stat,%cpu,%mem,rss,cmd --sort=-%cpu | head -6
ps -eo pid,stat,cmd | awk '$2 ~ /D/'      # D 状态进程
free -h                        # 看 available
journalctl -k | grep -i -E 'oom|killed'   # OOM（正则里不要加空格）
cat /proc/PID/oom_score
```

| top CPU | 高了说明 |
|---|---|
| us | 应用计算多 |
| sy | 内核忙：I/O、网络、进程切换 |
| wa | CPU 空闲且有进程等 I/O（**CPU 忙时显示不出来**） |
| st | 虚拟机的 CPU 被宿主机拿走 |

| vmstat | 看什么 |
|---|---|
| r | 持续 > 核数：CPU 饱和 |
| b | 持续 > 0：I/O 饱和 |
| si / so | 持续非 0：内存真不够 |
| bi / bo | 磁盘读 / 写量 |

负载 = R + D 进程。负载高 + us 高 → CPU；负载高 + wa/b 高 → 磁盘；负载高 + CPU 空闲 + 磁盘不忙 → D 卡在无响应存储（NFS）。

内存：free 小正常；available 才是可用；swap 用了但 si/so ≈ 0 没关系。OOM 痕迹只在内核日志；`CONSTRAINT_MEMCG` = cgroup 上限（`MemoryMax=`），`CONSTRAINT_NONE` = 整机。

压测：`stress-ng --cpu 2 -t 60s`、`--hdd 2 --hdd-opts dsync`、`--vm 1 --vm-bytes 400M`；隔离：`sudo systemd-run --scope -p MemoryMax=200M -p MemorySwapMax=0 命令`。

## 排障：磁盘

```bash
df -hT; df -i                                   # 哪个文件系统满了；块满还是 inode 满
sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail     # 一层层往下钻
sudo find / -xdev -type f -size +100M -exec ls -lh {} + 2>/dev/null
du --inodes -x --max-depth=1 /var | sort -n     # 哪里文件最多
sudo lsof +L1                                   # 已删除但仍被打开（NLINK 0）
ls -l /proc/PID/fd                              # 看 (deleted) 的 fd 号
: > /proc/PID/fd/N                              # 不重启进程释放空间
: > app.log   或   truncate -s 0 app.log        # 清空正在写的日志，不要 rm
find /dir -type f -mmin +60 -delete             # 海量小文件，不用 rm *
journalctl --disk-usage; sudo journalctl --vacuum-size=200M
ls -li f1 f2                                    # inode 号相同 = 硬链接
```

| 现象 | 原因 | 查 |
|---|---|---|
| ENOSPC，df 100% | 块满 | du -x 逐层 |
| ENOSPC，df -h 有空间 | inode 满 | df -i、du --inodes |
| df 满、du 小 | 删了但进程开着 | lsof +L1 |
| 普通用户写满、root 还能写 | ext4 保留 5% | tune2fs -l |

处理顺序：df -h + df -i → 还在涨吗 → 找大头 → **止血**（截断、清缓存 / 旧日志） → 根因（logrotate、调试日志、备份） → 80% 告警。

```bash
iostat -dxsy 1 3            # -d 只看磁盘 -x 扩展 -s 短格式 -y 跳过开机平均
iostat -xzN 1 5             # -z 跳过空闲设备 -N 显示 LVM 名
pidstat -d 1 5              # 哪个进程在读写
```

| iostat | 看什么 |
|---|---|
| r_await / w_await | 每个请求耗时（ms，含排队）。SSD ≈ 1 以下，机械盘十几；和基线比 |
| aqu-sz | 持续偏高 = 排队、饱和 |
| %util | 机械盘接近 100 = 饱和；**SSD / NVMe / RAID 不可靠** |
| f/s | flush 请求（dsync / fsync 产生） |

⚠️ wa 高 ≠ 磁盘慢（限速、NFS 时设备可能空闲）；wa 低 ≠ 磁盘没事（CPU 忙时记不出来）。以 iostat 的设备指标为准。

`mkfs.xfs -N` 是"只显示不格式化"；ext4 指定 inode 数用 `mkfs.ext4 -N`。`df -h` 向上取整，精确用 `df -B1K`。

## 排障：服务与网络（A4）

先用报错和耗时分流，再逐层验证：

| curl 报错 | 耗时 | 回包来源 | 原因 |
|---|---|---|---|
| Connection refused（7） | ms | 对端 RST | 没有进程监听这个 IP:端口 |
| No route to host（7） | ms | 对端 ICMP admin prohibited | 防火墙 REJECT |
| No route to host（7） | ~3 s | 无；本机邻居表 FAILED | ARP 失败：同网段找不到 MAC |
| Connection timed out（28） | = `-m` 值 | 无 | 防火墙 DROP |

```bash
ss -lntp 'sport = :80'                  # 精确过滤端口（grep :80 会匹配 :8090）；看监听地址
systemctl status <PID>                  # 由 PID 反查属于哪个 unit
systemctl cat <服务名>                   # 看 unit 文件（ExecStart 里有没有 --bind 127.0.0.1）
curl -v -m 5 --noproxy "*" http://IP:PORT/   # -v 看握手；-m 总时限；--noproxy 不走代理
ip route get <IP>                       # 走哪个网卡、用哪个源地址
ip neigh show <IP>                      # FAILED / INCOMPLETE：ARP 失败；REACHABLE / STALE：ARP 正常
firewall-cmd --get-active-zones         # 网卡在哪个 zone
firewall-cmd --list-all                 # 默认 zone 的运行时规则
firewall-cmd --permanent --list-all     # 同一 zone 的永久规则
firewall-cmd --add-rich-rule='rule family="ipv4" port port="8888" protocol="tcp" drop'   # 富规则：丢弃
sudo timeout 20 tcpdump -nn -i enp0s8 host <IP> and port 8888   # 抓包看 SYN / RST / ICMP；-n 端口仍翻译，-nn 都不翻译
getenforce; ls -Z <文件>                 # Enforcing？文件标签是什么
sudo ausearch -m AVC -ts recent         # SELinux 拒绝记录（-m 类型，-ts 起始时间）
namei -l <文件路径>                      # 逐级看权限（DAC）
sudo restorecon -v <文件>                # 按策略恢复标签（-v 显示改动）
```

| 现象 | 查什么 |
|---|---|
| refused，但服务 active | `ss -lntp`：监听地址是 127.0.0.1 还是 *？端口对不对？ |
| No route，耗时几 ms | `firewall-cmd --list-all`；抓包看 ICMP |
| No route，耗时约 3 s | `ip neigh`：目标 IP 是否存在、是否开机 |
| timed out | 防火墙 DROP：抓包看 SYN 有没有回包 |
| 403 | `ausearch -m AVC`（SELinux）→ `namei -l`（权限）→ httpd 配置（如 `Require all denied`） |

⚠️ 监听 `127.0.0.1` 只能本机访问；`*` 才是所有地址。服务 active ≠ 在监听你要的端口。
⚠️ `reload` 会丢掉只存在于运行时的规则。`--runtime-to-permanent` 会覆盖永久配置，先对比两边再执行。
⚠️ error_log 的 `Permission denied` 不能说明是 SELinux，DAC 和 SELinux 都会打印它。
⚠️ 防火墙用 `reject-with tcp-reset` 时也会得到 refused，所以 refused 不等于"防火墙已放行"。
