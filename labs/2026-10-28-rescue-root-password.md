# 救援：重置 root 密码、修复无法启动

- 日期：2026-10-28（W4-D6）
- 考点：RHCSA EX200 —— 中断启动过程获得系统访问权限（重置 root 密码）；启动到不同 target；修复无法启动的系统
- 环境：Rocky Linux 10.2（VirtualBox 控制台），node1；实验前快照 `before-rescue`
- 状态：完成
- 来源：根据课程对话、控制台截图和终端记录整理

## 目标

用 GRUB 的 `e` 临时改内核参数，分别进入 rescue.target、rd.break、init=/bin/bash、dracut 紧急 shell 和 rescue 内核；重置 root 密码并处理 SELinux 标签；现场修复"找不到根 LV"。

## 关键概念

- 能碰到控制台 ≈ 能拿到 root：GRUB 的 `e` 可以改内核参数，让系统停在检查密码之前。防范：机房门禁 / BMC（Baseboard Management Controller）密码、GRUB 密码（`grub2-setpassword`）、LUKS（Linux Unified Key Setup）磁盘加密。
- GRUB 编辑：`e` 编辑，`Ctrl+X` 启动，`Esc` 放弃。**只影响这一次开机**，BLS 条目不变（实测 `/proc/cmdline` 有 rootx，`grubby --info` 仍是 root）。
- `journalctl -b 0` 永远是当前这次开机，每重启一次编号整体前移；要稳定引用用 boot ID（`--list-boots` 里的长串）。

| 停在哪 | 参数 | 根分区 | 要 root 密码（RHEL 10 实测） | SELinux 策略 | 适用 |
|---|---|---|---|---|---|
| rescue.target | `systemd.unit=rescue.target` | 读写，fstab 本地文件系统都挂了 | ✅ | ✅ | 系统大体正常，在不启动普通服务的状态下维护（如某个服务卡住开机） |
| emergency.target | `systemd.unit=emergency.target` | 只挂根，**只读**，fstab 其他条目不挂 | ✅ | ✅ | fstab 写坏 |
| dracut 紧急 shell（rd.break 或出错自动进） | `rd.break` | 根已只读挂在 `/sysroot`，还没 switch_root | ✅ **（RHEL 8 以前不要）** | ❌ | 根分区挂载问题、switch_root 前修改真正的根 |
| init=/bin/bash | `init=/bin/bash` | 已 switch_root，`/` 只读 | ❌ | ❌ | **RHEL 10 上忘记 root 密码用这个** |
| rescue 内核 | 菜单选 `0-rescue` | 正常启动 | 正常登录 | ✅ | 正常 initramfs 坏了或缺驱动、新内核起不来 |

- rd.break 流程：`mount -o remount,rw /sysroot` → `chroot /sysroot`（否则改的是 initramfs 里的文件）→ `passwd` → `touch /.autorelabel` → `exit` 两次。
- init=/bin/bash 流程：`mount -o remount,rw /` → `passwd` → `touch /.autorelabel` → **`exec /sbin/init`**。bash 是 PID 1（`echo $$` = 1），exit 会 kernel panic；exec 在同一进程里换成 systemd。
- **为什么必须 relabel**：passwd 写新文件再替换（新 inode），此时 SELinux 策略没加载 → 新 shadow 是 `unlabeled_t` → 登录时 `unix_chkpwd` 读 shadow 被拦 → **所有用户**都提示密码错误。
- `/.autorelabel`（根目录下以点开头的隐藏文件）≠ `./autorelabel`（当前目录下的普通文件）。
- 开机时 systemd 发现 `/.autorelabel` → `selinux-autorelabel.service` 全盘重打标签 → 删掉文件 → 自动重启（node1 约 20 秒）。
- `cp -a` 会保留原文件的 SELinux 标签（-a 含保留上下文），和 mv 一样。
- root 设密码可以无视 `BAD PASSWORD` 字典检查警告。
- rescue 条目：内核文件与正常内核完全相同（安装时复制，`cmp` 验证）；initramfs 是通用型（79M、2629 个文件）对比 host-only（39M、2194 个文件）。
- initramfs 只激活 `rd.lvm.lv=` 点名的 LV；systemd 在 initramfs 里按"等设备出现"工作，手动激活后立刻 `Found device` 继续。

## 关键步骤

```bash
# 实验前
sudo passwd -S root; sudo grub2-editenv - list; ls -Z /etc/shadow

# rescue.target（GRUB e，linux 行末尾）
systemd.unit=rescue.target
systemctl list-units --type=service --state=running | head; findmnt /backup; systemctl default

# rd.break（要 root 密码）
mount -o remount,rw /sysroot; chroot /sysroot; passwd; ls -Z /etc/shadow; touch /.autorelabel; exit; exit

# 登录失败的排障
enforcing=0                                   # GRUB e 临时加
getenforce; ls -Z /etc/shadow; sudo restorecon -v /etc/shadow
sudo restorecon -Rnv /etc; sudo restorecon -Rv /etc
sudo journalctl --list-boots | tail -6
sudo journalctl --since "2026-10-28 11:30" --no-pager | grep -iE 'relabel|autorelabel|fixfiles'
sudo journalctl -b -2 --no-pager | grep -B15 -m1 'fully relabel'
sudo tail -15 /root/.bash_history
sudo rm /autorelabel

# init=/bin/bash
echo $$; mount -o remount,rw /; touch /.autorelabel; passwd; exec /sbin/init

# dracut 紧急 shell：rd.lvm.lv=rlm/rootx
ls /dev/mapper/; lvm lvs; lvm lvchange -ay rlm/root; exit
sudo journalctl -b 0 --no-pager | grep -iE 'rootx|Could not boot'
cat /proc/cmdline; sudo grubby --info=DEFAULT | grep args

# rescue 内核
uname -r; cat /proc/cmdline
ls -lh /boot/initramfs-*; sudo cmp /boot/vmlinuz-0-rescue-* /boot/vmlinuz-$(uname -r)
```

## 验证

| 检查项 | 命令 / 现象 | 预期结果 | 结果 |
|---|---|---|---|
| rescue.target 要密码 | 控制台 | Give root password for maintenance | ✅ |
| rescue.target 挂 fstab | `findmnt /backup` | rw 已挂 | ✅ 只有 4 个服务在运行 |
| rd.break 停在 switch_root 前 | 截图 | `Mounted sysroot.mount`、`Break before switch_root` | ✅ 且要 root 密码 |
| 漏做 relabel 的后果 | 登录 | root、fqh 都"密码错误" | ✅ 复现（见踩坑） |
| 原因是 SELinux | `enforcing=0` 后登录、`ls -Z` | 能登录；shadow 是 unlabeled_t | ✅ |
| 修复 | `restorecon -v /etc/shadow` | 改为 shadow_t，正常开机可登录 | ✅ |
| init=/bin/bash 是 PID 1 | `echo $$` | 1 | ✅ |
| relabel 执行 | `--list-boots`、journal | 多一次短开机，`selinux-autorelabel.service` 运行 | ✅ 14:15:31 开始，约 20 秒后自动重启 |
| relabel 完成 | `ls /.autorelabel`、`ls -Z /etc/shadow` | 文件已删；shadow_t | ✅ |
| 根 LV 找不到 → dracut shell | 截图 | 约 209 秒超时，`Could not boot` | ✅ `/dev/mapper` 只有 rlm-swap |
| 现场修复 | `lvm lvchange -ay rlm/root` → `exit` | 继续启动 | ✅ |
| `e` 的修改不持久 | `/proc/cmdline` vs `grubby` | rootx vs root | ✅ |
| rescue 内核 | `BOOT_IMAGE`、`cmp`、initramfs 大小 | vmlinuz-0-rescue，内核相同，initramfs 79M vs 39M | ✅ |

## 踩坑

- **rd.break 要 root 密码**（RHEL 10，initramfs 里的 systemd 紧急 shell 也走 sulogin）→ 课上说"一般不要"是旧版本的情况 → 忘记密码用 `init=/bin/bash`。
- **重置密码后 root 和 fqh 都登录不了** → 按证据查：`enforcing=0` 能登录（是 SELinux）→ shadow 是 `unlabeled_t` → `journalctl --list-boots` 显示没有多一次重启、没有 autorelabel 服务记录 → setroubleshoot 日志：`unix_chkpwd` 读 shadow 被拒 → `/root/.bash_history` 里是 **`touch ./autorelabel`** → chroot 后当前目录是 `/`，建成了 `/autorelabel`（没有点）。中间我两次猜错原因（以为在 chroot 外执行），证据推翻了。
- `ls -l ./autorelabel` 检查时同样写错 → 要查 `/.autorelabel`。
- `restorecon -Rnv /etc` 列出的旧偏差：`cp -a` 做的备份保留了原标签、LVM devices 文件、W2 的 drop-in → `restorecon -Rv /etc` 统一修正。
- init=/bin/bash 里 `load_policy: command not found`，但 `/usr/sbin/load_policy` 存在（policycoreutils）；dracut shell 的 PATH 含 `/usr/sbin` → 推测 PID 1 bash 的 PATH 不同，下次 `echo $PATH` 确认，用绝对路径。
- dracut shell 第一次输错 root 密码 → `Login incorrect`，再输一次。
- `systemctl --list-units` → `list-units` 是子命令，不加 `--`。
- `journalctl -b -1` 查实验 4 → 查错了开机，实验 4 当时是 `-b 0`。
- `rpm -rf` 笔误（`-r` 是 `--root`）。
- 开机信息刷屏看不清 → 不靠看屏幕，用 `journalctl --list-boots` 和 `-b` 查。

## 卡点 / 没想明白的

- 达标题 5：emergency.target 的根是**只读**、fstab 其他条目不挂（答成了"完全挂载"）；rd.break 时根**已经**只读挂在 `/sysroot`；rescue.target 的场景没想起来；缺驱动更适合 rescue 内核。
- 达标题 6：只答了改 GRUB 参数和 grubby，漏了 fstab、`/etc/default/grub` 和重新生成 initramfs（见回忆题 3）。

## 回忆题

1. （概念）RHEL 10 上忘记 root 密码，为什么用 `init=/bin/bash` 而不用 `rd.break`？改完为什么要 `exec /sbin/init` 和 `touch /.autorelabel`？
<details><summary>答案</summary>

RHEL 10 的 rd.break 进入的是 initramfs 里 systemd 的紧急 shell，要 root 密码；init=/bin/bash 让内核直接运行 bash，跳过 systemd，没人检查密码。bash 是 PID 1，exit 会 kernel panic，exec 原地换成 systemd。passwd 新建 shadow 时策略没加载 → unlabeled_t → 所有人登录失败，所以要全盘重打标签（或先 `/usr/sbin/load_policy -i`，待验证）。
</details>

2. （操作）写出 init=/bin/bash 重置 root 密码的完整步骤，并说出开机后怎么确认 relabel 执行了。
<details><summary>答案</summary>

GRUB 选条目按 `e` → linux 行末尾加 `init=/bin/bash` → `Ctrl+X` → `mount -o remount,rw /` → `passwd` → `touch /.autorelabel` → `exec /sbin/init`。确认：`journalctl --list-boots` 多一次短开机；`journalctl -b -1 | grep autorelabel` 有 `selinux-autorelabel.service`；`ls /.autorelabel` 不存在；`ls -Z /etc/shadow` 是 shadow_t。
</details>

3. （排障）根 LV 从 `rlm/root` 改名为 `rlm/sys` 但没改任何配置，重启后停在哪？现场怎么启动？怎么永久修好？
<details><summary>答案</summary>

停在 dracut 紧急 shell（`/dev/mapper/rlm-root` 不存在）。现场：GRUB `e` 把 `root=/dev/mapper/rlm-sys`、`rd.lvm.lv=rlm/sys` 都改掉（只在 dracut shell 里激活 rlm/sys 不够，`root=` 指的还是 rlm-root）。永久：`grubby --update-kernel=ALL --remove-args="rd.lvm.lv=rlm/root" --args="root=/dev/mapper/rlm-sys rd.lvm.lv=rlm/sys"`；`/etc/default/grub` 的 `GRUB_CMDLINE_LINUX`（影响以后装的内核）；`/etc/fstab` 里 `/` 那一行；`dracut -f` 重新生成 initramfs。
</details>

4. （排障）重置密码后所有用户都登录不了，按什么顺序查？
<details><summary>答案</summary>

GRUB 加 `enforcing=0` 登录 → 能登录则是 SELinux → `ls -Z /etc/shadow`（unlabeled_t）→ `restorecon -v /etc/shadow` → 查为什么 relabel 没生效：`journalctl --list-boots`、`journalctl | grep autorelabel`、`/root/.bash_history`。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
