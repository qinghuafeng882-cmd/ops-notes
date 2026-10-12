# 文件操作、通配符、inode 与链接、find

- 日期：2026-09-22（W1-D2）
- 考点：RHCSA EX200 —— 创建/删除/复制/移动文件；创建硬链接和软链接；查找文件
- 环境：Rocky Linux 10（VirtualBox）
- 状态：完成
- 来源：根据课程对话整理，无完整终端记录

## 目标

理解 inode 和目录项，搞清硬链接/软链接的区别，熟练用 `find` 按条件找文件并批量处理。

## 关键概念

- **inode 里没有文件名**，文件名只存在目录里；目录 = "文件名 → inode 号"对照表。
- `rm` 只删目录项、链接数 -1；数据释放要两个条件：**链接数为 0 且没有进程打开**。
- 硬链接绑定 inode（不能跨文件系统、不能链接目录）；软链接存的是**路径字符串**，每次访问时现场查找。
- 软链接里的相对路径相对于**链接文件所在目录**解析 → 不确定就写绝对路径。
- 目录链接数 = 2 + 子目录个数（普通文件不算）。
- 通配符由 **Shell 先展开**再交给命令；`{}` 花括号展开不管文件存不存在。
- `find -mtime`：`-N` 以内，`+N` 以前，`N` 恰好那一天；`-size -1M` 会向上取整（只能找到 0 字节）。

## 关键步骤

```bash
# 硬链接 / 软链接
ln orig.txt hard.txt; ls -li orig.txt hard.txt   # inode 相同，链接数 2
ln -s /etc/fstab good-link                         # 软链接用绝对路径
find ~ -samefile hard.txt                          # 找同一 inode 的所有名字

# 跨文件系统硬链接（失败）
sudo ln ~/d2lab/hard.txt /mnt/disk-lab/cross-hard.txt

# 删了文件空间不释放
tail -f big.file > /dev/null &; rm big.file
sudo lsof +L1                                      # 找 (deleted) 但仍被打开的文件

# find
find /var -type f -size +50M -mtime -30 -exec ls -lh {} +
find ~ -xtype l                                    # 坏的符号链接
sudo find / -user harry -type f -exec cp -a {} /root/findfiles/ \; 2>/dev/null
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 跨文件系统硬链接 | `ln` 到 `/mnt/disk-lab` | 报错 | ✅ `Invalid cross-device link` |
| mtime 符号含义 | `touch -d "3/10/40 days ago"` 后分别 `find -mtime -7/+7/10` | 对应文件 | ✅（自述已跑通） |

## 踩坑

- 达标题写成 `-mtime 30` → 那是"恰好 30～31 天前"，"30 天内"要写 `-mtime -30`。
- `find / -name *.log` 不加引号 → 当前目录有匹配时 Shell 先展开，结果时好时坏甚至报错 → `-name` 后的模式一律加引号。
- `-xtype l` 只答了"好像是找坏链接" → 机制：`-xtype` 检查链接**指向目标**的类型，目标不存在才退回检查链接本身（`l`）。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）为什么硬链接不能跨文件系统？（从数据结构讲）
<details><summary>答案</summary>

目录项只存"文件名 → inode 号"，不存设备信息；inode 号只在单个文件系统内唯一，内核返回 EXDEV。
</details>

2. （操作）在 `/var` 下找大于 50M、30 天内修改过的普通文件，按 `ls -lh` 显示。
<details><summary>答案</summary>

`find /var -type f -size +50M -mtime -30 -exec ls -lh {} +`
</details>

3. （排障）删了 10G 日志，`df` 不降、`du` 降了。原因、排查、处理、预防？
<details><summary>答案</summary>

进程还开着文件；`lsof +L1`；让进程重新打开日志（如 `nginx -s reopen`）或重启进程；正在写的日志用 `truncate -s 0` 清空；预防用 logrotate + 磁盘告警。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
