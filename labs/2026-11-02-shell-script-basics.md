# shell 脚本（上）：执行方式、变量与引号、位置参数、退出码、if/test

- 日期：2026-11-02（W5-D1）
- 考点：RHCSA EX200 —— 编写简单 shell 脚本：条件执行（if、test、[]）、处理位置参数、在脚本中处理命令输出
- 环境：Rocky Linux 10.2（VirtualBox），node1，`~/w5d1`
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

搞清楚脚本怎么被执行（shebang、PATH、子进程 vs source），引号对变量展开的影响，位置参数和退出码，`if` 判断的本质，写出带参数检查和退出码的实用脚本。

## 关键概念

- shebang `#!/bin/bash`：`./x.sh` 时内核读开头 `#!`，用后面的程序解释这个文件。

| 执行方式 | 要 x 权限 | 看 shebang | 在哪执行 | 变量结束后还在 |
|---|---|---|---|---|
| `bash x.sh` | ❌ | ❌ | 新的子进程 | ❌ |
| `./x.sh` | ✅ | ✅ | 新的子进程 | ❌ |
| `source ./x.sh` / `. ./x.sh` | ❌ | ❌ | **当前 shell 自己** | ✅（普通变量，不是环境变量） |

- source 是 bash 内置命令，由当前 bash 自己读文件逐行执行；文件里的 `cd`、变量会影响当前终端，`exit` 会**退出当前终端**。
- 不带 `/` 的文件名：直接执行和 `source` 都会**先在 PATH 里找** → 自己的脚本一律带 `./`。
- 当前目录不在 PATH 中是安全设计（防止同名恶意程序）。

| | 外部命令 | 内置命令 | 关键字 |
|---|---|---|---|
| 例子 | `ls`、`/usr/bin/[` | `cd`、`source`、`export`、`exit`、`[`、`test` | `if`、`fi`、`for`、`[[` |
| 处理时机 | 展开、拆分后执行 | 展开、拆分后执行 | **展开之前**就被 bash 识别 |

- 改变当前 shell 状态的命令（cd、source、export、exit）只能是内置命令。
- 变量赋值等号两边不能有空格；`$(cmd)` 命令替换；`${2:-80}` 为空时用默认值；`export` 让子进程继承。

| 引号 | `$` 展开 | 按空格拆分 | 通配符 |
|---|---|---|---|
| `'...'` | ❌ | ❌ | ❌ |
| `"..."` | ✅ | ❌ | ❌ |
| 不加 | ✅ | ✅ | ✅ |

- 特殊变量：`$0` 脚本名、`$1..` 参数（第 10 个 `${10}`）、`$#` 个数、`"$@"` 每个参数独立、`$*` 合成一个、`$?` 上条命令退出码、`$$` 当前 shell 的 PID（Process ID）。
- 退出码：0 成功，非 0 失败（不同数字区分原因）。grep：1 没找到、2 出错；shell：127 命令找不到、126 找到但不能执行。`&&` 成功才执行下一条，`||` 失败才执行，`;` 不管结果都执行。
- `if` 后面是**命令**，判断它的退出码。`[` 是一条命令，所以 `[` 后、`]` 前必须有空格。
- test：字符串 `=` `!=` `-z`（空）`-n`（非空）；整数 `-eq -ne -lt -le -gt -ge`（`>` 在 `[ ]` 里是重定向）；文件 `-e -f -d -r -w -x -s`。
- `[[ ]]` 是关键字：空变量不加引号也不出错，支持 `&&` `||` 模式匹配；写 bash 脚本推荐用。
- 错误信息写到 `>&2`（标准错误）：和正常输出分开，正常输出被重定向或进管道时，错误仍能显示 / 单独收集。
- 调试：`bash -n` 只查语法；`bash -x` 打印展开后的每一行（`+` 开头）。

## 关键步骤

```bash
# 执行方式
bash hello.sh; chmod +x hello.sh; ./hello.sh
type -a hello.sh                         # PATH 里的同名文件
echo $$; source ./hello.sh; echo "after: $greeting"
type -a [; type [[

# 引号
f="my file"; ls $f; ls "$f"
bash -x -c 'f="my file"; ls $f'          # + ls my file

# 参数
./args.sh a "b c" d                      # "$@" 3 个，$* 4 个

# 退出码
./check_user.sh ops; echo $?             # 0 / 1 / 2
./check_user.sh nobody2 > /dev/null      # 错误信息仍显示

# 不加引号的陷阱
a=""; [ -n $a ] && echo "认为不为空？！"  # 错误结果，不报错
[ $a = "" ] && echo yes                  # unary operator expected
```

```bash
#!/bin/bash
# disk_check.sh 挂载点 [阈值，默认 80]
mount_point="$1"
threshold="${2:-80}"
if [[ -z "$mount_point" ]]; then
    echo "usage: $0 MOUNT_POINT [THRESHOLD]" >&2; exit 2
fi
if [[ ! -d "$mount_point" ]]; then
    echo "$mount_point is not a directory" >&2; exit 2
fi
use=$(df --output=pcent "$mount_point" | tail -1 | tr -d ' %')
if [[ "$use" -ge "$threshold" ]]; then
    echo "WARNING: $mount_point is ${use}% full (threshold ${threshold}%)"; exit 1
else
    echo "OK: $mount_point is ${use}% full"; exit 0
fi
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 无 x 权限 | `./hello.sh` | Permission denied | ✅ |
| 子进程 vs 当前 shell | `bash`/`./` 与 `source ./` 打印的 PID | 前两者不同，source 与 `$$` 相同 | ✅ 7971 = 7971 |
| source 后变量保留 | `echo $greeting` | hello from script | ✅ |
| 不加引号被拆分 | `bash -x` | `+ ls my file` | ✅ |
| `"$@"` vs `$*` | `./args.sh a "b c" d` | 3 个 / 4 个 | ✅ |
| 退出码 | `check_user.sh` 三种情况 | 0 / 1 / 2 | ✅ |
| `>&2` | `> /dev/null` 后仍显示错误 | 显示 | ✅ |
| disk_check | `/`、`/ 10`、不存在目录、无参数 | 0 / 1 / 2 / 2 | ✅ 17% |
| 命令找不到 | `./dsk_check.sh` | 退出码 127 | ✅ |
| `[ -n $a ]` 陷阱 | 空变量 | 错误地判为非空 | ✅ |

## 踩坑

- 直接敲 `hello.sh` 和 `source hello.sh` 执行的都是 `/usr/local/bin/hello.sh`（W2 的 systemd 实验脚本，循环写 `/var/log/hello.log`）→ 刷出 Permission denied、`$greeting` 为空，**不是**因为子进程 → `type -a` 发现同名文件，`source ./hello.sh` 重做。
- `[ -z $mount_point ]` 空变量时变成 `[ -z ]`：一个参数时 test 判断"是否非空字符串"→ 碰巧为真；换成 `[ -n $a ]` 就悄悄得出错误结论 → 不加引号最危险的是**不报错但结果错**。
- 用法提示写成 `>$2`（应为 `>&2`）→ 重定向到以 `$2` 命名的文件。
- `[ -n $a ]; echo ...` 用了 `;`：不管结果都执行 → 要用 `&&`。
- 中文引号 `“为空”` 不是 shell 引号，被原样打印。
- 拼写：`pritnf`、`chomd`、`dsk_check.sh`、`args.dh`。

## 卡点 / 没想明白的

- source 和 `bash x.sh` 的区别、由谁执行（已补讲：当前 bash 自己）。
- 内置命令和关键字的区别（已补讲）。
- 以为 source 会把变量"导出成环境变量" → 只是当前 shell 的普通变量。
- `>&2` 的作用答成"更醒目" → 是为了和正常输出分开。
- 达标题 6 太笼统：监控脚本里 `grep -q ERROR app.log` 退出码 1 是正常，2 说明日志丢了要告警。

## 回忆题

1. （概念）`bash x.sh`、`./x.sh`、`source ./x.sh` 各在哪个进程执行？脚本里的 `cd /tmp` 对你的终端分别有没有影响？
<details><summary>答案</summary>

前两个在新的子进程，`cd` 不影响当前终端；source 由当前 bash 自己执行，终端会进入 /tmp（脚本里的 exit 也会退出当前终端）。`./` 需要 x 权限并看 shebang，另两种只需读权限。
</details>

2. （操作）写一个脚本：必须带 1 个参数（用户名），没带就向标准错误输出用法并以 2 退出；用户存在输出 UID、退出 0，不存在退出 1。
<details><summary>答案</summary>

`if [[ $# -ne 1 ]]; then echo "usage: $0 USER" >&2; exit 2; fi`；`if id "$1" &>/dev/null; then echo "UID $(id -u "$1")"; exit 0; else echo "no such user" >&2; exit 1; fi`
</details>

3. （排障）脚本里 `if [ -n $1 ]; then 处理; fi`，不带参数运行时它也去"处理"了，没有任何报错。为什么？怎么改？
<details><summary>答案</summary>

`$1` 为空且没加引号，展开后变成 `[ -n ]`，只剩一个参数，test 判断字符串 "-n" 非空 → 真。改成 `[ -n "$1" ]` 或 `[[ -n $1 ]]`，或直接用 `$# -eq 0` 判断参数个数。
</details>

4. （排障）`source myenv.sh` 执行的竟然不是当前目录的 myenv.sh，为什么？
<details><summary>答案</summary>

文件名不带 `/` 时，source 先在 PATH 里找，PATH 里有同名文件就执行那个。用 `source ./myenv.sh`，用 `type -a myenv.sh` 检查。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
