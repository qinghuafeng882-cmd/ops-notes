# 归档压缩，以及 fstab 写错后的紧急模式救援

- 日期：2026-10-19（W3-D5）
- 考点：RHCSA EX200 —— 用 tar、gzip、bzip2 归档/压缩/解包；（部分）中断启动、紧急模式维护
- 环境：Rocky Linux 10（VirtualBox），救援在虚拟机控制台中操作
- 状态：完成
- 来源：根据课程对话、终端记录和控制台截图整理

## 目标

掌握 tar 打包、解包、只解单个文件和三种压缩；故意写坏 fstab，在紧急模式下修复；对比 nofail 的效果。

## 关键概念

- 归档（tar：多文件 → 一个文件，保留权限/属主/时间）≠ 压缩（gzip/bzip2/xz：一个文件变小）。
- tar 格式：每个文件 = 512 字节头 + 内容，顺序拼接，无索引。
- `c` 创建、`x` 解包、`t` 列出、`f 文件`（紧跟文件名）、`z/j/J` = gzip/bzip2/xz、`-C 目录` 解包到指定目录（目录要先存在）。
- 打包时去掉开头的 `/`：解包相对当前目录或 `-C`，避免覆盖系统文件 → 恢复备份先解到临时目录再比对。
- 压缩率 xz > bzip2 > gzip，速度相反。`gzip` 默认**删除原文件**（`-k` 保留）；`zcat`/`zgrep` 不解压直接看和搜。
- fstab 每行 → `.mount` 单元，属于 `local-fs.target`；挂载失败 → local-fs.target 失败 → `emergency.target`，网络和 sshd 不启动，只能用控制台。
- 紧急模式需要 root 密码；root 被锁定会提示 `the root account is locked`，只能走 GRUB 救援（W4）。
- `nofail`：非必需的数据盘、NFS 加；`/`、`/boot`、`/var` 等系统必需挂载点不加。

## 关键步骤

```bash
sudo tar czf etc.tar.gz /etc; sudo tar cjf etc.tar.bz2 /etc; sudo tar cJf etc.tar.xz /etc
tar tzf etc.tar.gz | head
sudo tar xzf etc.tar.gz -C restore
sudo tar xzf etc.tar.gz -C one etc/hosts       # 只解一个文件
gzip access.log; zcat access.log.gz | head; zgrep -c '" 404 ' access.log.gz; gunzip access.log.gz

# 救援前
sudo passwd -S root                            # P = 有密码，L/LK = 锁定

# 紧急模式（控制台，输入 root 密码后）
tail -3 /etc/fstab
sed -i '$d' /etc/fstab                         # 只读时先 mount -o remount,rw /
systemctl daemon-reload
mount -a
journalctl -b -p err | tail
reboot
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 压缩率 | `ls -lh etc.tar.*`、`du -sh /etc` | xz 最小 | ✅ /etc 23M → gz 4.7M、bz2 3.9M、xz 2.9M |
| 去掉开头 / | `tar tzf` | `etc/...` | ✅ |
| 解包一致 | `diff restore/etc/fstab /etc/fstab` | 一致 | ✅ |
| 只解一个文件 | `find one -type f` | 只有 etc/hosts | ✅ |
| 权限保留 | `ls -ld perm/common/admin` | drwxrws--- root sysmgrs | ✅ 文件属主、属组都保留 |
| gzip 删除原文件 | `ls -lh access.log*` | 只剩 .gz | ✅ 196K → 25K |
| zgrep | `zgrep -c '" 404 '` | 约 2000×0.2 | ✅ 400 |
| 坏 fstab 进紧急模式 | 控制台 | emergency.target | ✅ `Failed to mount mnt-broken.mount`、`Dependency failed for local-fs.target`、`Reached target emergency.target` |
| 紧急模式修复 | `sed -i '$d'`、`mount -a` | 无报错 | ✅ |
| nofail | 重启 | 正常开机 | ✅ `systemctl --failed` 0 个单元 |

## 踩坑

- 写入的坏行漏了 `UUID=`（`00000000-... /mnt/broken ...`）→ 被当成一个不存在的设备路径，挂载立即失败，所以没出现"A start job is running ... 1min30s"的等待（推测：写成 `UUID=` 时 systemd 会等待设备单元出现约 90 秒）。
- 紧急模式第一次输错 root 密码 → `Login incorrect`，重输即可。
- `time sudo tar czf ...` 第一条用时 7.5 秒包含了输入 sudo 密码的时间，三种压缩的用时对比不可信。
- 紧急模式日志里的 `kdump.service` 失败、`vmwgfx ... ERROR` 与本实验无关（虚拟机环境的问题）。
- nofail 那次 `journalctl -b | grep -i broken` 只匹配到 vmwgfx 的一行（推测：开机后不到 90 秒就查了，设备等待还没超时）。
- `systemctl daemon-reload` 忘了 sudo → 触发 polkit 认证提示。
- 热身：autofs 主配置目录写成 `/etc/auto.d/`，应为 `/etc/auto.master.d/`。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）为什么恢复备份时不要直接在 `/` 下执行 `tar xf`？
<details><summary>答案</summary>

会用备份里的旧文件覆盖当前所有同名文件；应先 `-C` 解到临时目录，对比确认后再拷贝。
</details>

2. （操作）把 `/usr/local` 打包成 `/root/local.tar.bz2`，再解到 `/tmp/test`。
<details><summary>答案</summary>

`tar cjf /root/local.tar.bz2 /usr/local`；`mkdir -p /tmp/test; tar xjf /root/local.tar.bz2 -C /tmp/test`
</details>

3. （排障）服务器重启后 SSH 连不上，控制台显示 emergency mode，最可能的原因和处理？
<details><summary>答案</summary>

fstab 某条挂载失败。控制台输入 root 密码 → `journalctl -xb` 找失败的 .mount → 必要时 `mount -o remount,rw /` → 修正或注释该行（非必需盘可加 nofail）→ `systemctl daemon-reload` → `mount -a` → reboot。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
