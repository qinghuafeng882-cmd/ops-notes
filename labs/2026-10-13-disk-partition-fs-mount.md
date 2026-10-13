# 磁盘、分区、文件系统与挂载

- 日期：2026-10-13（W3-D1）
- 考点：RHCSA EX200 —— 创建/删除 MBR 和 GPT 分区；创建文件系统；挂载/卸载；用 UUID 或标签开机挂载
- 环境：Rocky Linux 10（VirtualBox），新加磁盘 sdg、sdh
- 状态：部分完成（实验 4 parted 命令拼错未重做，sdh 收尾清理未做）
- 来源：根据课程对话和终端记录整理

## 目标

新盘从"空白"到"开机自动挂载"走完全程：分区 → 格式化 → 挂载 → 写 fstab → 重启验证。

## 关键概念

- 新盘三步：**分区（fdisk/parted）→ 创建文件系统（mkfs）→ 挂载（mount + fstab）**。
- MBR：第 0 扇区 512 **字节** = 引导代码 446 + 分区表 64（4 × 16 字节）+ 签名 2 → 最多 4 个主分区；起始扇区号 32 位 × 512 字节 → 最大 **2TiB**。
- GPT：保护性 MBR + GPT 头 + 128 条分区条目，**首尾各一份**，带 CRC 校验；64 位扇区号。
- 本机 sda 是 GPT + BIOS 引导，所以有 1M 的 BIOS boot 分区。
- fdisk 改动存在内存里，`w` 才写入，`q` 放弃；parted 每条命令立即生效。
- 设备名（sdX）按检测顺序分配，会变；**UUID 写在文件系统超级块里，跟着文件系统走**；重新格式化 UUID 会变。
- XFS：RHEL 默认，可在线扩容（`xfs_growfs 挂载点`），**不能缩小**，小于 300MB 不能创建；ext4：可扩容（`resize2fs 设备`），可缩小但必须先卸载，inode 数格式化时固定，默认给 root 预留 5%。
- 挂载会**盖住**挂载点原有内容（不删除），卸载后重现。
- systemd 把 fstab 每一行转换成 `.mount` 单元（`/mnt/data1` → `mnt-data1.mount`），改 fstab 后要 `daemon-reload`。
- fstab 第 6 列：根分区 1、其他 ext4 2、XFS 一律 0。

## 关键步骤

```bash
# 识别空盘（不靠设备名：看大小、无子节点、无签名）
lsblk -f /dev/sdg /dev/sdh
sudo wipefs /dev/sdg                 # 不带参数只列出签名，无输出 = 空盘

# fdisk 交互：g（GPT）→ n → +1G → n → 回车用完剩余 → p 确认 → w
sudo fdisk /dev/sdg

# 格式化
sudo mkfs.xfs -L data1 /dev/sdg1
sudo mkfs.ext4 -L data2 /dev/sdg2

# fstab（先备份，追加）
sudo cp -a /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)
echo "UUID=0df9951b-2cfa-4db1-8cb1-cfcc5e073053 /mnt/data1 xfs defaults 0 0" | sudo tee -a /etc/fstab
echo "UUID=d03017a3-3628-4855-b663-1dbe403ea3f0 /mnt/data2 ext4 defaults 0 2" | sudo tee -a /etc/fstab

# 重启前三步检查
sudo findmnt --verify
sudo systemctl daemon-reload
sudo mount -a
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| sdg 是 GPT 两个分区 | `sudo fdisk -l /dev/sdg` | gpt，1G + 1022M | ✅ |
| fdisk `q` 不写入 | 删分区 2 后 `q`，再 `lsblk` | 分区 2 还在 | ✅ |
| XFS 拒绝覆盖 | 再次 `mkfs.xfs /dev/sdg1` | 拒绝，提示 `-f` | ✅ `appears to contain an existing filesystem` |
| 挂载盖住原文件 | 挂载前后 `ls /mnt/data1` | 挂载后看不到 hidden.txt，卸载后重现 | ✅ |
| target is busy | 在挂载点内 `umount` | 拒绝 | ✅ `fuser -vm` 显示 fqh 的 bash |
| fstab 检查 | `findmnt --verify` | 0 errors | ✅（只有 daemon-reload 的提醒） |
| mount 单元 | `systemctl list-units --type=mount \| grep data` | mnt-data1/2.mount active | ✅ |
| 重启后自动挂载 | `df -hT /mnt/data1 /mnt/data2` | 都已挂载 | ✅ 文件仍在 |
| 坏条目被检查工具拦住 | 写入假 UUID 后 `findmnt --verify` / `mount -a` | 报错 | ✅ 2 errors；`can't find UUID=...` |
| parted 建 MBR | `parted /dev/sdh print` | msdos | 未验证（命令拼错） |

## 踩坑

- `parted /dev/sdh mklable msdos` → 只打印了用法 → `mklabel` 拼错。
- 热身：`tee` 写 drop-in 报 No such file or directory → `sshd.service.d` 目录不存在 → 先 `mkdir -p`。
- 热身：`systemctl restart daemon-reload` → `daemon-reload` 是 systemctl 的子命令，不是服务。
- 热身：sshd 原单元里本来就有 `Restart=on-failure`，drop-in 是重复配置（`systemctl cat` 里能看到）。
- `echo "...0 0 | sudo tee -a /etc/fstab` 引号没闭合 → 出现 `>` 续行提示符 → Ctrl-C 重写。
- `findmnt --verity` 拼错；`umonut` 拼错。
- bind 挂载练习：执行了 `mount --bind / /mnt/rootview/data`，把根又挂到了 rootview 里面的 data 目录上 → `umount /mnt/rootview` 一直 target is busy → 原因是它下面还有一个子挂载，不是有进程在用 → 先卸里层 `umount /mnt/rootview/data`，再卸外层（或 `umount -R`）。本实验要看的路径应是 `/mnt/rootview/mnt/data1`。
- 对整个根做 `fuser -vm` 会列出几乎所有进程（`-m` 表示"使用这个文件系统的进程"），不能用来判断子挂载问题；普通用户执行还会刷出大量 Permission denied。
- 达标题：MBR 单位写错（bit/字节、MiB/字节、GiB/TiB），原理是对的。
- **加盘顶掉了一块旧盘（已解决）：** 现象：只看到 sdg、sdh 两块新盘；lab-vg 消失，sde 无任何签名，sdf 仍是 LVM2_member。排查：VirtualBox 介质管理里属于 node1 的盘有 9 块，SATA 控制器只挂了 8 块，端口数量设置为 8 → 加第三块新盘时顶掉了 lab-vg 的一块 PV。注意：介质管理的"关联到"也统计快照引用，被顶掉的盘仍显示已关联，要看虚拟机"存储"里的端口清单才准确。处理：端口数改为 10，重新挂回旧盘并调整顺序。结果：9 块盘全部识别，`pvs/vgs` 显示 lab-vg 两块 PV 完整，data-lv 已激活（`-wi-a-----`，未挂载）；fstab 按 UUID 写的 /mnt/data1、/mnt/data2 照常挂载。

## 卡点 / 没想明白的

- "/data 没挂上数据写进根分区"怎么验证（见回忆题 3）。

## 回忆题

1. （概念）为什么 fstab 要用 UUID 而不是 `/dev/sdb1`？UUID 什么时候会变？
<details><summary>答案</summary>

设备名按检测顺序分配，加盘换口就可能变，会挂错盘；UUID 写在文件系统超级块里，跟着文件系统走。重新 mkfs 后 UUID 改变。
</details>

2. （操作）给新盘 sdx 建一个 GPT 分区、格式化为 XFS，按 UUID 挂载到 /data 并开机生效。
<details><summary>答案</summary>

`fdisk /dev/sdx`（g、n、回车、w）→ `mkfs.xfs /dev/sdx1` → `mkdir -p /data` → `echo "UUID=$(blkid -s UUID -o value /dev/sdx1) /data xfs defaults 0 0" | sudo tee -a /etc/fstab` → `findmnt --verify` → `systemctl daemon-reload` → `mount -a`
</details>

3. （排障）`/data` 那块盘挂上了，根分区还是满的，`du` 找不到大文件，怎么验证？
<details><summary>答案</summary>

数据很可能在盘没挂上时写进了根分区的 `/data` 目录，挂载后被盖住。验证：`mkdir /mnt/rootview; mount --bind / /mnt/rootview; du -sh /mnt/rootview/data`（bind 挂载看到的是根分区本身，不受上层挂载影响）；或在停机窗口 `umount /data` 后直接看。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
