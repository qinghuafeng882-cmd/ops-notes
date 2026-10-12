# 速查表 W1：命令行、文件、vim、文本处理

> 只放最常用的写法和易错点。详细原理和踩坑见 `labs/` 对应笔记。

## 查帮助

| 用途 | 命令 |
|---|---|
| 配置文件格式 | `man 5 fstab` |
| 按关键词找命令（单个词） | `man -k password`（先 `sudo mandb`） |
| 手册页属于哪个包 | `rpm -qf $(man -w 5 fstab)` |
| 目录结构手册 | `man 7 hier`（需要 `man-pages` 包） |

⚠️ `man 5 xxx` 查不到 → `rpm -q man-pages`。`man -k` 不要拼短语。

## 文件与目录

| 用途 | 命令 |
|---|---|
| 看目录本身 | `ls -ld dir` |
| 看 inode | `ls -li` / `stat file` |
| 递归建目录 | `mkdir -p a/b/c` |
| 备份保留属性 | `cp -a file file.bak.$(date +%F)` |
| 花括号快速备份 | `cp nginx.conf{,.bak}` |
| 删除前看展开结果 | `echo rm /tmp/*.log` |
| 实时看日志 | `tail -f /var/log/messages` |
| 路径每一层权限 | `namei -l /path/to/file` |

通配符：`*` 任意、`?` 一个字符、`[0-9]`、`[!0-9]`。`*` 不匹配隐藏文件。

## 链接

| | 硬链接 `ln a b` | 软链接 `ln -s 目标 链接` |
|---|---|---|
| inode | 相同 | 独立 |
| 跨文件系统 / 链接目录 | ❌ | ✅ |
| 删源文件后 | 可用 | 失效 |

⚠️ 软链接里的相对路径相对于**链接所在目录**解析，不确定就用绝对路径。目录链接数 = 2 + 子目录数。

## find

```bash
find /var -type f -size +50M -mtime -30 -exec ls -lh {} +
find / -user harry -type f -exec cp -a {} /root/dst/ \; 2>/dev/null   # 先 mkdir -p 目标
find ~ -xtype l                          # 坏的符号链接
find / -perm -4000 -type f 2>/dev/null   # SUID 文件
find ~ -samefile file                    # 同一 inode 的所有名字
```

| 选项 | 含义 |
|---|---|
| `-mtime -N / +N / N` | N 天**以内** / **以前** / 恰好那天 |
| `-mmin -60` | 60 **分钟**以内 |
| `-size +100M` | 大于 100M（`-1M` 只能找到 0 字节，用 `-1024k`） |
| `-user` / `-group` | 属主 / 属组（不是 `-owner`） |
| `-name '*.log'` | **通配符**，必须加引号；不是正则 |
| `-exec cmd {} \;` | 每个文件执行一次 |
| `-exec cmd {} +` | 攒一批执行，**`{}` 必须在命令末尾** |

## 删了文件空间不释放

```bash
sudo lsof +L1                      # 找 (deleted) 但仍被打开的文件
truncate -s 0 /var/log/big.log     # 正在写的日志用截断，不要 rm
```

## vim

| 操作 | 按键 |
|---|---|
| 迷路 / 放弃修改退出 | `Esc` / `:q!` |
| 行尾追加 / 下方新行 | `A` / `o` |
| 删除 N 行 / 复制到末尾 | `10dd` / `yG` |
| 改整个单词 | `ciw` |
| 重复上次修改 | `.` |
| 跳到第 N 行 | `:50` 或 `vim +50 file` |
| 全文替换（逐个确认） | `:%s/旧/新/gc` |
| 只替换独立单词 | `:%s/\<no\>/NO/g` |
| 路径替换换分隔符 | `:%s#/var/www#/data/www#g` |
| 最后 5 行复制到开头 | `:$-4,$t0` |
| 批量注释 | `Ctrl-v` 选列 → `I#` → `Esc`，或 `:10,15s/^/#/` |
| 显示 Tab | `:set list`（Tab 显示 `^I`） |

编辑系统文件用 `sudoedit`。`E325` swap 提示：确认没人在编辑 → `R` 恢复 → 删 `.xxx.swp`。

## 重定向与管道

| 写法 | 含义 |
|---|---|
| `> f` / `>> f` | 覆盖 / **追加** |
| `2> f` | 只存报错 |
| `> f 2>&1` 或 `&> f` | 全部存入（`2>&1` 写最后） |
| `2>/dev/null` | 丢弃报错（结果不对劲时先去掉它看报错） |
| `echo x \| sudo tee -a /etc/f` | sudo 写文件（`sudo echo > file` 不行） |
| `cat > f << 'EOF'` | 写多行文件；内容有 `$` 时 EOF 加引号 |
| `find ... -print0 \| xargs -0 cmd` | 文件名带空格 |

⚠️ 管道只传 stdout。sudo 管不到当前 Shell 做的事：**重定向、通配符展开**。

## grep

```bash
sudo grep -Ev '^(#|$)' /etc/ssh/sshd_config      # 只看有效配置
grep -n PermitRootLogin file                     # 改配置前先看原始行
grep 'A' f | grep -c 'B'                         # 两个条件"与"（正则没有 &）
grep -rl 关键词 /etc 2>/dev/null                 # 哪些文件包含
```

`-i` 忽略大小写、`-v` 取反、`-c` 计数、`-w` 整词、`-o` 只输出匹配部分、`-E` 扩展正则、`-A/-B/-C 3` 上下文。

正则：`^` 行首、`$` 行尾、`.` 一个字符、`*` 前一个字符任意次、`.*` 任意内容；`-E` 才有 `+ ? | () {n,m}`。

## 统计模板

```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -5   # Top N
```

`uniq` 只去**相邻**重复 → 先 `sort`。`cut` 只能单字符分隔，多空格用 awk。

## sed

```bash
sed -n '100,105p' file                                       # 打印指定行
sed '/^#/d; /^$/d' file                                      # 删注释和空行
sed 's/old/new/g' file | diff file -                         # 先预览
sed -i".bak.$(date +%F-%H%M%S)" -E 's/^#?Port .*/Port 2222/' file   # 改配置（带时间戳备份）
sed -i -E 's/^PASS_MAX_DAYS[[:space:]]+.*/PASS_MAX_DAYS\t90/' file  # 空格或 Tab 都能匹配
```

⚠️ 不加 `-i` 只打印不改文件；用了 `? + |` 必须 `-E`；`-i后缀` 和 `-E` 之间要空格；`.bak` 不带时间戳会被下一次覆盖；`#` 后有空格时写 `^#? *`；只替换要改的部分，不要 `.*` 吞掉整行路径。

## awk

```bash
awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd   # 先 head -3 看分隔符
awk '$9 == 404 && $7 == "/admin"' access.log                 # 多条件
awk '{sum += $NF} END {printf "%.3f\n", sum/NR}' access.log  # 平均值
awk '$9 >= 500 {n++} END {printf "%.2f%%\n", n/NR*100}' f    # 百分比
df -h | awk 'NR>1 && $5+0 > 80 {print $6, $5}'
awk -v max="$limit" '$10 > max' f                            # 传 Shell 变量
```

`$NF` 最后一列、`$(NF-3)` 倒数第 4 列、`NR` 行号、`-F:` 分隔符。程序用单引号。

## 工具语法对照

| | 匹配 | 替换 |
|---|---|---|
| Shell / `find -name` | 通配符 | — |
| grep | 正则（`-E` 扩展） | — |
| vim | 正则 | `:%s/旧/新/g` |
| sed | 正则（`-E` 扩展） | `s/旧/新/g`（不写 `%`） |
| awk | `/.../`，内容里 `/` 要转义 | — |
