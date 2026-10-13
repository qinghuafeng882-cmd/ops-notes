# 速查表 W3：存储

> 只放最常用的写法和易错点。详细原理和踩坑见 `labs/` 对应笔记。

## 动手前：确认操作对象

```bash
lsblk -f                 # 树形结构 + 文件系统类型、UUID、挂载点
sudo wipefs /dev/sdX     # 不带参数只列出签名，无输出 = 空盘
```

空盘三个条件：大小对、没有子节点、没有签名。**不要只看设备名**。本机 sda、sdb 是系统盘（根 LV 横跨两块），绝对不碰。

## 查看

| 命令 | 看什么 |
|---|---|
| `lsblk` / `lsblk -f` | 结构 / 加上类型、UUID |
| `sudo blkid` | UUID、LABEL、TYPE |
| `sudo blkid -s UUID -o value /dev/sdg1` | 只输出 UUID |
| `sudo fdisk -l /dev/sdg` | 分区表类型（gpt / dos）、起止扇区 |
| `df -hT` / `df -i` | 容量与类型 / inode |
| `findmnt` | 挂载树 |
| `du -sh 目录` | 目录实际占用 |

## 分区

**fdisk**（改动在内存里，`w` 写入，`q` 放弃）：

| 键 | 作用 |
|---|---|
| `g` / `o` | 新建 GPT / MBR 分区表 |
| `n` | 新建分区（大小写 `+1G`，回车 = 剩余全部） |
| `d` / `t` | 删除 / 改类型 |
| `p` | 打印 |
| `w` / `q` | 写入 / 不保存退出 |

**parted**（立即生效，可写脚本）：

```bash
sudo parted /dev/sdh mklabel msdos            # 或 gpt（是 mklabel 不是 mklable）
sudo parted /dev/sdh mkpart primary xfs 1MiB 1GiB
sudo parted /dev/sdh print
```

分区后看不到新设备：`sudo partprobe /dev/sdX`。
擦除分区表和签名：`sudo wipefs -a /dev/sdX`（执行前再看一眼设备名）。

| | MBR | GPT |
|---|---|---|
| 分区数 | 4 主（或 3 主 + 扩展 + 逻辑） | 128 |
| 最大磁盘 | 2TiB | 不受限 |
| 备份 | 无 | 首尾各一份 + CRC |

## 文件系统

```bash
sudo mkfs.xfs -L data1 /dev/sdg1       # 已有文件系统时拒绝，-f 强制
sudo mkfs.ext4 -L data2 /dev/sdg2
```

| | XFS | ext4 |
|---|---|---|
| 扩容 | `xfs_growfs 挂载点`（在线） | `resize2fs 设备`（在线） |
| 缩小 | ❌ 不能 | ✅ 先卸载 |
| inode | 动态 | 格式化时固定 |
| fstab 第 6 列 | 0 | 2（根分区 1） |

## 挂载

```bash
sudo mkdir -p /mnt/data1
sudo mount /dev/sdg1 /mnt/data1
sudo mount -o ro /dev/sdg1 /mnt/data1
sudo umount /mnt/data1                 # target is busy → 先 cd 出去
sudo fuser -vm /mnt/data1              # 谁在用
sudo lsof /mnt/data1
sudo mount --bind / /mnt/rootview      # 查看被挂载盖住的根分区目录
```

## /etc/fstab

```
UUID=xxxx   /mnt/data1   xfs    defaults   0 0
UUID=yyyy   /mnt/data2   ext4   defaults   0 2
```

```bash
sudo cp -a /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdg1)  /mnt/data1  xfs  defaults  0 0" | sudo tee -a /etc/fstab
sudo findmnt --verify            # ① 检查
sudo systemctl daemon-reload     # ② systemd 重新生成 .mount 单元
sudo mount -a                    # ③ 实际挂一遍
systemctl list-units --type=mount
```

| 选项 | 作用 |
|---|---|
| `nofail` | 挂载失败不影响开机（非必需的数据盘） |
| `_netdev` | 网络文件系统，等网络就绪 |
| `noatime` / `noexec` / `ro` | 不更新访问时间 / 禁止执行 / 只读 |

⚠️ 用 `UUID=` 不用 `/dev/sdX`；挂载点目录要先建好；重新格式化后 UUID 会变；fstab 有错误重启会进紧急模式 → 重启前必须三步检查。
