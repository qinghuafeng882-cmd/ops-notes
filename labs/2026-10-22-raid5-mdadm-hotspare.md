# 实验：用 mdadm 搭建 RAID 5 + 热备盘，模拟坏盘与自动重建

> 日期：2026-10-22　｜　环境：Rocky Linux 10（VirtualBox 虚拟机 node1）　｜　工具：mdadm 4.4

## 一、实验目标

1. 用 Linux 软 RAID（mdadm）搭建 3 块盘的 RAID 5，外加 1 块热备盘。
2. 模拟一块成员盘故障，观察阵列**降级 → 热备自动顶上 → 重建完成**的全过程。
3. 模拟"换新盘"，把热备补回来。
4. 全程用 SHA-256 校验和证明**数据前后完全一致**。

## 二、环境

| 项目 | 内容 |
| --- | --- |
| 宿主机 | Windows 11 笔记本，VirtualBox 7.2 |
| 虚拟机 | Rocky Linux 10，主机名 node1 |
| 实验盘 | 新增 4 块 1 GiB 虚拟盘：`sdj`、`sdk`、`sdl`、`sdm`（动态分配，挂在 SATA 控制器上） |
| 注意 | 系统里已有一个 RAID 1 阵列 `md0`（RHCSA 练习留下的），所以新阵列命名为 **md1** |
| 保护措施 | 实验前给虚拟机拍了快照，操作失误可一键还原 |

动手前先用 `lsblk` 确认哪些盘可以动。`sda`～`sdi` 都已被系统盘、LVM、已有阵列等占用，**本实验只操作 `sdj`～`sdm`**。

## 三、够用的原理

| 概念 | 要点 |
| --- | --- |
| RAID 5 | 条带 + 一份校验；最少 3 块；可用容量 = (n − 1) × 单盘；最多坏 1 块 |
| 校验 | P = D1 ⊕ D2，丢了任何一块，把剩下的异或起来就能算回来 |
| 校验轮换 | 校验块轮流放在各块盘上，避免固定校验盘成为写入瓶颈 |
| 热备盘 | 平时不参与阵列，成员盘故障时自动顶上并开始重建 |

## 四、步骤与关键输出

### 1. 创建 RAID 5 + 热备

```bash
sudo mdadm --create /dev/md1 --level=5 --raid-devices=3 /dev/sdj /dev/sdk /dev/sdl --spare-devices=1 /dev/sdm
```

创建时 mdadm 询问是否开启**写意图位图**（write-intent bitmap），选择开启。它记录正在写入的区域，断电或盘短暂掉线后只需同步被标记的区域，大幅缩短恢复时间。

`mdadm --detail /dev/md1` 关键输出：

```
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
             State : clean
    Active Devices : 3
     Spare Devices : 1
            Layout : left-symmetric
        Chunk Size : 512K

    Number   Major   Minor   RaidDevice State
       0       8      144        0      active sync   /dev/sdj
       1       8      160        1      active sync   /dev/sdk
       4       8      176        2      active sync   /dev/sdl

       3       8      192        -      spare   /dev/sdm
```

解读：

- **容量验证**：2044 MiB = 2 × 1022 MiB，符合 (n − 1) × 单盘。每块盘只用了 1022 MiB 而不是 1024，是因为盘的开头要存超级块、位图等元数据。
- **Chunk Size 512K**：条带深度，每块盘一次连续写 512K。
- **Layout left-symmetric**：校验块的具体轮换规则。
- **一个细节**：`sdl` 在 2 号槽位，成员编号却是 4，比热备盘 `sdm` 的 3 还大，说明它是后加入的。原因是 mdadm 创建 RAID 5 时，会**故意先建一个降级阵列**，把最后一块盘当成新盘做一次 recovery（读 2 块、算出第 3 块，全是顺序读写），比让 3 块盘一起 resync 校对更快。`dmesg` 里创建时刻的 `recover of RAID array md1` 也印证了这一点。

### 2. 建文件系统，写入测试数据

```bash
sudo mkfs.xfs /dev/md1
sudo mkdir -p /mnt/raid5
sudo mount /dev/md1 /mnt/raid5
sudo dd if=/dev/urandom of=/mnt/raid5/test.bin bs=1M count=500 status=progress
sha256sum /mnt/raid5/test.bin | tee ~/test.sha256
sudo mdadm --detail --scan /dev/md1 | sudo tee -a /etc/mdadm.conf
```

- `mkfs.xfs` 输出中 `sunit=128 swidth=256 blks`（块大小 4K）：sunit = 512K，等于 Chunk Size；swidth = 1024K，等于 2 块数据盘 × 512K。XFS **自动识别了 RAID 结构**，会尽量按整条带对齐写入，整条带写可以直接算校验，避免 RAID 5 的写惩罚。
- **校验和文件存在家目录，而不是阵列里**：它是检验阵列数据的"标准答案"，必须独立于被检验的对象。
- 写入 `/etc/mdadm.conf`：让阵列重启后仍以 md1 的名字被识别。

### 3. 模拟坏盘

```bash
sudo mdadm /dev/md1 --fail /dev/sdk; cat /proc/mdstat
```

故障瞬间：

```
md1 : active raid5 sdl[4] sdm[3] sdk[1](F) sdj[0]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]
      [>....................]  recovery =  0.0% (16/1046528) finish=545.0min speed=16K/sec
```

- `sdk[1](F)`：F 表示 Faulty（故障）。
- `[3/2]`：应有 3 块，现有 2 块在工作，阵列降级。
- `[U_U]`：方括号各位置**按槽位号**对应，`sdk` 在 1 号槽位，所以下划线在中间。
- `recovery`：热备盘 `sdm` 已经**自动**开始重建。
- `finish=545.0min`：重建刚开始速度还没提上来，估算严重偏大。

几秒后再看，已经是 `[3/3] [UUU]`，重建完成。

`dmesg` 记录的完整过程：

| 时间（开机后秒数） | 日志 | 含义 |
| --- | --- | --- |
| 3040.578 | `Disk failure on sdk, disabling device.` | 检测到故障，踢出 sdk |
| 3040.578 | `Operation continuing on 2 devices.` | 降级运行，业务不中断 |
| 3040.603 | `recover of RAID array md1` | 热备盘自动开始重建，无人工操作 |
| 3049.393 | `md1: recover done.` | 重建完成，用时约 9 秒 |

重建后的 `mdadm --detail`：

```
    Active Devices : 3
    Failed Devices : 1
     Spare Devices : 0

    Number   Major   Minor   RaidDevice State
       0       8      144        0      active sync   /dev/sdj
       3       8      192        1      active sync   /dev/sdm
       4       8      176        2      active sync   /dev/sdl

       1       8      160        -      faulty   /dev/sdk
```

`sdm` 接替了 1 号槽位。注意 **Spare Devices 变成了 0**：热备已经用掉，此时再坏一块就不会自动重建了。

数据校验：

```
$ sha256sum -c ~/test.sha256
/mnt/raid5/test.bin: OK
```

### 4. 换盘，补回热备

真实场景是拔掉坏盘、插入新盘，这里用 `sdk` 模拟"新盘"：

```bash
sudo mdadm /dev/md1 --remove /dev/sdk        # 先在软件里摘除故障盘
sudo mdadm --zero-superblock /dev/sdk        # 清除旧超级块，变成"全新的盘"
sudo mdadm /dev/md1 --add /dev/sdk           # 加入阵列
```

结果：

```
md1 : active raid5 sdk[5](S) sdl[4] sdm[3] sdj[0]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
```

阵列已经完整，新盘不会触发重建，而是成为热备（`(S)` = Spare），Spare Devices 恢复为 1。

## 五、结论

1. RAID 5 的容量公式、单盘容错能力得到验证。
2. 成员盘故障后，阵列降级但持续提供服务；热备盘**自动**顶上并完成重建，全程无需人工干预。
3. 热备只能顶一次，用掉后必须尽快换盘、补回热备。
4. 从故障到换盘的全过程，测试文件的 SHA-256 校验和始终一致，数据完好。
5. 1 GiB 重建约 9 秒；按约 200 MB/s 估算，一块 12 TB 的机械盘重建要约 17 小时，这就是生产环境中降级期的风险窗口。

## 六、踩坑与收获

- **预测 `[U_U]` 时我猜成了 `[UU_]`。**后来才明白方括号是按槽位号排列的，下划线在哪，就是哪个槽位缺盘。以后看到 `[_UU]`，马上就知道是 0 号位的盘出了问题。
- **看到 `finish=545.0min` 时以为要等好几个小时，实际 9 秒就完成了。**重建刚开始时的预计时间不可信，在真实环境里遇到不要慌，过几分钟再看。
- **校验和文件一开始没多想就存了，后来才理解为什么必须存在阵列外面。**检验依据和被检验的对象放在一起，阵列出问题时它也可能跟着坏，这和"备份必须独立于原数据"是同一个道理。
- **第一次知道超级块里有事件计数（Events）。**一块被踢出的旧盘，Events 会落后于阵列，mdadm 能据此认出它是"掉队的老成员"。临时掉线的好盘可以 `--re-add`，配合位图只补差异；真坏的盘或别处拿来的盘，要先 `--zero-superblock`。
- **实验环境里已经有很多盘在用。**`sdb` 并进了根分区的 LVM，`sdg`、`sdh` 是已有的 RAID 1，任何一个写错盘名都可能毁掉系统。而且 sdX 这种名字重启后可能变化，所以每次执行危险命令前都先 `lsblk` 确认一遍，这个习惯在生产环境里同样重要。

## 七、命令速查

```bash
# 创建
mdadm --create /dev/md1 --level=5 --raid-devices=3 盘1 盘2 盘3 --spare-devices=1 盘4
# 查看
cat /proc/mdstat
mdadm --detail /dev/md1
# 持久化
mdadm --detail --scan /dev/md1 >> /etc/mdadm.conf
# 故障与换盘
mdadm /dev/md1 --fail 盘
mdadm /dev/md1 --remove 盘
mdadm --zero-superblock 盘
mdadm /dev/md1 --add 盘
# 拆除实验环境
umount /mnt/raid5
mdadm --stop /dev/md1
mdadm --zero-superblock /dev/sdj /dev/sdk /dev/sdl /dev/sdm
# 再从 /etc/mdadm.conf 删除 md1 那一行
```
