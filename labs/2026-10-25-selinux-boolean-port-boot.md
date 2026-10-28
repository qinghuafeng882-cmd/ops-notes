# SELinux（下）：端口标签、boolean；启动流程与 GRUB

- 日期：2026-10-25（W4-D5）
- 考点：RHCSA EX200 —— 管理 SELinux 端口标签；用 boolean 调整 SELinux 策略；配置系统默认启动 target；修改内核参数 / 引导配置
- 环境：Rocky Linux 10.2（VirtualBox），node1（httpd）、node2 客户端；node1 是 GPT + BIOS 引导
- 状态：完成
- 来源：根据课程对话和终端记录整理

## 目标

让 httpd 监听新端口 8888、启用用户个人主页；看懂启动的五个阶段；用 grubby 改内核参数、用 grub2-mkconfig 改菜单等待时间，并把被自动隐藏的 GRUB 菜单调出来。

## 关键概念

- SELinux 拒绝三类，看 AVC 判断：`tclass=file/dir` → 文件标签；`tclass=tcp_socket { name_bind }` → 端口标签；标签都对仍被拒 → boolean。`audit2why` 把 AVC 翻译成原因。
- **audit2why / sealert 给的是"能让这次通过的改动"，不一定是最好的**：8888 它建议 `nis_enabled`（放开所有受限服务对未登记端口的限制，推测），正确是 `semanage port`；个人主页它列了 `httpd_enable_homedirs` 和 `httpd_unified`（后者让 httpd 对所有 httpd 内容类型可读写执行），选前者。原则：选范围最小的。
- 端口类型：80/443/8008/8443 等是 `http_port_t`；8888 原来是 `unreserved_port_t`。`semanage port -a` 新增，已登记的端口用 `-m` 修改。
- 改服务端口的正确顺序：`semanage port` → `firewall-cmd` → 改 `Listen` → `httpd -t` → restart（`httpd -t` 查不出 SELinux 问题，先改 Listen 会让 httpd 起不来、整站停掉）。
- httpd 有 W2 加的 `Restart=on-failure`，失败后重试 5 次 → `Start request repeated too quickly`（启动频率限制）→ 必要时 `systemctl reset-failed httpd`。
- boolean：`setsebool` 只改运行时，`-P`（Persistent）写进策略存储。`semanage boolean -l` 括号里是 `(当前值, 持久值)`：`(off,off)` → 不加 -P `(on,off)` → 加 -P `(on,on)`。
- `~/public_html` 新建时直接是 `httpd_user_content_t`：策略里按名字的类型转换，是"继承父目录"的例外（推测）。
- 启动五阶段：BIOS/UEFI（Basic Input/Output System / Unified Extensible Firmware Interface）→ GRUB2（GRand Unified Bootloader）→ 内核 + initramfs（initial RAM filesystem，初始内存文件系统）→ systemd（PID 1）→ 默认 target（node1 是 multi-user.target）。
- initramfs 里带 lvm 工具和 `30-parse-lvm.sh`；`rd.` 开头的内核参数是给 initramfs（dracut）的：`rd.lvm.lv=rlm/root` 让它先激活根 LV，之后才有 `/dev/mapper/rlm-root`。`root=... ro` 先只读挂根。
- BLS（Boot Loader Specification）：每个内核一个条目 `/boot/loader/entries/*.conf`，内核参数写在里面，用 grubby 改；菜单等待时间等在 `/etc/default/grub`，改完要 `grub2-mkconfig -o /boot/grub2/grub.cfg`。两者都要重启生效；`/proc/cmdline` 是本次开机实际收到的参数。
- node1 有 2 个条目：正常内核 + `0-rescue`（带全部驱动的通用内核和 initramfs，用于救援）。另有 kdump 用的 initramfs（对应 `crashkernel=`）。
- 内核不认识的参数不报错，打印 `Unknown kernel command line parameters ... will be passed to user space`；带 `=` 的作为环境变量交给 init（dmesg 里和 `HOME=/`、`TERM=linux` 列在一起）→ 参数拼错往往只是不起作用。
- **GRUB 菜单自动隐藏**：grubenv 里 `menu_auto_hide=1`，且上次开机成功（`boot_success=1`）→ `timeout_style=hidden`。GRUB 每次开机先把 `boot_success` 置 0，开机成功后 systemd 再写回 1 → 上次开机失败时菜单自动出现。`menu_show_once` 时显示 60 秒。

## 关键步骤

```bash
# 端口标签
sudo semanage port -l | grep -w 8888
echo "Listen 8888" | sudo tee /etc/httpd/conf.d/port8888.conf
sudo ausearch -m AVC -ts recent | audit2why
sudo semanage port -a -t http_port_t -p tcp 8888
sudo systemctl restart httpd && sudo ss -tlnp | grep 8888
sudo firewall-cmd --permanent --add-port=8888/tcp && sudo firewall-cmd --reload

# 个人主页：DAC + httpd 配置 + boolean
chmod 711 ~; mkdir -p ~/public_html; chmod 755 ~/public_html
sudo sed -i 's/^\(\s*\)UserDir disabled/\1#UserDir disabled/; s/^\(\s*\)#UserDir public_html/\1UserDir public_html/' /etc/httpd/conf.d/userdir.conf
sudo httpd -t && sudo systemctl reload httpd
sudo setsebool httpd_enable_homedirs 1
sudo setsebool -P httpd_enable_homedirs on
sudo semanage boolean -l | grep httpd_enable_homedirs

# 启动流程
systemd-analyze; systemd-analyze blame | head -5
sudo grubby --info=DEFAULT; cat /proc/cmdline; cat /etc/default/grub
sudo lsinitrd | grep -i lvm

# 内核参数与 GRUB
sudo grubby --update-kernel=DEFAULT --args="w4lab=1"
sudo cp -a /etc/default/grub /etc/default/grub.bak.$(date +%F-%H%M%S)
sudo sed -i 's/^GRUB_TIMEOUT=5/GRUB_TIMEOUT=10/' /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo dmesg | grep -B3 -A1 '^\[ .*3\..*w4lab'
sudo grub2-editenv - list
sudo grub2-editenv - unset menu_auto_hide
sudo grubby --update-kernel=DEFAULT --remove-args="w4lab=1"
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 8888 被 SELinux 拦 | `systemctl restart httpd`、ausearch | 启动失败；`name_bind` → `unreserved_port_t` | ✅ |
| 端口标签修复 | `semanage port -l`、`ss -tlnp` | 8888 在 http_port_t，httpd 监听 8888 | ✅ |
| 防火墙放行后可访问 | node2 `curl node1:8888` | 返回页面 | ✅（之前一次失败是防火墙） |
| 个人主页被拦 | node2 `curl node1/~fqh/`、audit2why | 403；建议 httpd_enable_homedirs | ✅ |
| boolean 运行时 / 持久 | `semanage boolean -l` | (on,off) → (on,on) | ✅ |
| 个人主页可访问 | node2 `curl` | fqh home page | ✅ |
| 根 LV 在 initramfs 激活 | `sudo lsinitrd \| grep lvm`、`rd.lvm.lv=` | 有 lvm 工具和参数 | ✅ |
| grubby 改参数重启前不生效 | `cat /proc/cmdline` | 没有 w4lab | ✅ |
| 重启后生效 | `cat /proc/cmdline`、dmesg | 有 w4lab，作为环境变量传给 init | ✅ |
| grub.cfg 已更新 | `grep timeout= grub.cfg` | timeout=10 | ✅ |
| 菜单被隐藏的原因 | `grub2-editenv - list` | menu_auto_hide=1、boot_success=1 | ✅ |
| 取消自动隐藏后看到菜单 | 控制台重启 | 10 秒倒计时、2 个条目 | ✅ |
| 收尾 | `grubby --info=DEFAULT` | 没有 w4lab | ✅ |

## 踩坑

- 先改 `Listen 8888` 再 restart → httpd 起不来（W2 的 drop-in 让它重试 5 次）→ 先 `semanage port`、防火墙，最后改服务。
- audit2why 建议 `setsebool -P nis_enabled 1` → 范围太大，没照做，用 `semanage port`。✅
- `setsebool` 不加 sudo → Permission denied。
- 普通用户 `lsinitrd` → Permission denied（initramfs 权限 600）。
- 实验 4 原计划去掉 `rhgb quiet`，但 node1 的参数里本来就没有 → 改成加 `w4lab=1` 做实验（出题前没看实际配置）。
- `/etc/defaults/grub` 打错 → 是 `/etc/default/grub`。
- `grubby --default-kernal` 拼错。
- `grep -B3 A1 '...'` 漏了 `-` → A1 被当成搜索模式，正则被当成文件名 → `No such file or directory`。
- 重启时看不到 GRUB 菜单 → `menu_auto_hide=1` → `grub2-editenv - unset menu_auto_hide`（应急：开机按住 Shift 或按 Esc）。
- `systemd-analyze`：用户空间 57 秒，`sdg.device` 等了 44.8 秒（md0 成员盘，推测是虚拟盘识别慢），未处理。

## 补课（10-26 ～ 10-28）：BIOS / UEFI、GRUB2、initramfs、BLS

开机是一条接力：固件（只会加载引导程序）→ GRUB2（懂文件系统和 LVM，读出内核、传参数）→ initramfs（内存里的迷你系统，挂真正的根）→ systemd。

| | BIOS（Basic Input/Output System） | UEFI（Unified Extensible Firmware Interface） |
|---|---|---|
| 找引导程序 | 执行第 0 扇区的 446 字节代码 | 从 FAT32 的 ESP（EFI System Partition）按文件名打开 `.efi` |
| 引导项 | 无，按磁盘顺序 | 存在主板 NVRAM（`efibootmgr`） |
| Secure Boot | 无 | 有（签名链：固件 → shim → grub → 内核） |
| 判断 | `/sys/firmware/efi` 不存在 | 存在 |

- MBR（Master Boot Record）= 第 0 扇区 512 字节：引导代码 446 + 分区表 64 + 签名 2。"MBR"既指这个扇区，也指 MBR（DOS）分区方案。GPT 盘第 0 扇区是"保护性 MBR"，446 字节代码区照样能用。
- node1（GPT + BIOS）实测：保护性 MBR 放 GRUB `boot.img` → sda1（1M `BIOS boot`）放 `core.img` → sda2（2G `Linux extended boot`，即 /boot）放 grub.cfg、内核、initramfs、BLS 条目 → sda3 LVM。
- "内核读不了磁盘"说法不准确：① 内核是磁盘上的文件，没进内存前无法执行 → GRUB 负责加载；② 运行后很多驱动是模块（在根分区上），LVM 激活要用户空间的 `lvm` 程序 → initramfs 提前打包。
- initramfs：gzip 压缩的 cpio 归档（node1 39M，解开约 87M），激活 `rd.lvm.lv` → 只读挂到 `/sysroot` → `switch_root` 交给真正的 systemd。坏在这里 → dracut 紧急 shell（根还没挂上）；fstab 错 → systemd 紧急模式（根已挂上）。
- dracut 不是缩写：是马萨诸塞州的一个镇名（红帽用附近地名命名项目），生成 initramfs 的工具。
- BLS 条目字段：`title` 菜单标题、`version`、`linux` 内核、`initrd`、`options` 内核参数（grubby 改的就是这行）、`grub_users`/`grub_arg --unrestricted`（RHEL 的 GRUB 扩展：设了 GRUB 密码时，不用密码也能启动这一项）、`grub_class` 菜单图标分类。`linux /vmlinuz-...` 没有 `/boot` 前缀：/boot 是独立分区，GRUB 从这个分区的根开始找。
- grub.cfg 第 138–139 行 `insmod blscfg` + `blscfg`：开机时现场读 entries 目录。
- shim：Secure Boot 下由微软签名的小引导程序，内含发行版证书，用来验证发行版签名的 grub；node1 是 BIOS，`/boot/efi/EFI/rocky` 是空目录，没装 shim（之前猜"装了 shim"是错的）。
- `sudo cat /boot/loader/entries/*.conf` 报不存在 → `*` 由 fqh 的 shell 在 sudo 之前展开，fqh 读不了目录，展开失败 → `sudo sh -c 'cat /boot/loader/entries/*.conf'`。

## 卡点 / 没想明白的

- BIOS 和 UEFI 的区别、GRUB2 / initramfs / BLS 的作用（已补课，见上节）。
- dmesg 日志看不懂（已逐行讲解，见关键概念最后两条）。
- 达标题 2 用了 `semanage port -l` 查 boolean → 应为 `semanage boolean -l`，看括号里第二个值。
- 达标题 3 只说了"打标签"，没写出命令。
- 达标题 4 命令记不住：`sudo grubby --update-kernel=ALL --remove-args="quiet"`。

## 回忆题

1. （概念）SELinux 拒绝分哪三类？各自在 AVC 里怎么认、用什么修？
<details><summary>答案</summary>

文件标签：tclass=file/dir → `restorecon` 或 `semanage fcontext -a` + `restorecon`；端口标签：tcp_socket `{ name_bind }` → `semanage port -a -t 类型 -p tcp 端口`；boolean：标签都对仍被拒，`audit2why` 提示 → `setsebool -P 名字 on`。都选范围最小的。
</details>

2. （操作）让 httpd 在 8888 端口提供服务，node2 能访问，重启后仍有效。
<details><summary>答案</summary>

`sudo semanage port -a -t http_port_t -p tcp 8888`；`sudo firewall-cmd --permanent --add-port=8888/tcp && sudo firewall-cmd --reload`；`/etc/httpd/conf.d/` 加 `Listen 8888`；`sudo httpd -t && sudo systemctl restart httpd`；`ss -tlnp | grep 8888`；node2 `curl node1:8888`。
</details>

3. （排障）改了内核参数，`cat /proc/cmdline` 看不到；重启时 GRUB 菜单也看不到。原因？
<details><summary>答案</summary>

grubby 改的是 BLS 条目，`/proc/cmdline` 是本次开机的参数，要重启。菜单：`grub2-editenv - list` 有 `menu_auto_hide=1` 且上次开机成功 → 菜单被隐藏；`grub2-editenv - unset menu_auto_hide`，或开机按 Shift/Esc；另外确认 VirtualBox 窗口是打开的。
</details>

4. （概念）GPT + BIOS 的机器，GRUB 的代码分别放在哪几个地方？为什么 UEFI 机器不需要 BIOS boot 分区？
<details><summary>答案</summary>

保护性 MBR 的 446 字节（boot.img）→ BIOS boot 分区（core.img，带文件系统驱动）→ /boot/grub2（模块和 grub.cfg）。UEFI 固件自己能读 FAT32，直接从 ESP 打开完整的 grubx64.efi（前面有 shim），不用拆段。
</details>

5. （排障）开机停在 dracut 紧急 shell 和停在 systemd 紧急模式，分别说明坏在哪一步？各举一个常见原因。
<details><summary>答案</summary>

dracut：initramfs 阶段，真正的根还没挂上 → 如根 LV 改名但 `rd.lvm.lv=`/`root=` 还是旧名、缺驱动。systemd 紧急模式：根已挂上，挂 fstab 里其他文件系统失败 → 如 fstab 写错 UUID。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
