# 软件包管理：rpm、dnf、仓库、Flatpak

- 日期：2026-10-06（W2-D3）
- 考点：RHCSA EX200 —— 配置 RPM 仓库；安装/删除 RPM 包；配置 Flatpak 仓库；安装/删除 Flatpak 包
- 环境：Rocky Linux 10（VirtualBox），系统已使用阿里云镜像
- 状态：完成
- 来源：根据课程对话整理

## 目标

会查包、装包、配仓库、回滚，会用 Flatpak。

## 关键概念

- 包名：`名称-上游版本-发布号.el10.架构`。红帽打补丁常只加发布号 → 查 `rpm -q --changelog`。
- rpm 主要用来查：`-qa -qi -ql -qc -qf`；`rpm -V 包名` 校验（`S` 大小、`5` 内容、`T` 时间，`c` 配置文件）。程序文件出现 `5` 要警惕。
- rpm 不解决依赖；本地 rpm 也用 `dnf install ./x.rpm`。
- Rocky/RHEL 10 用 **DNF 4**（Fedora 已是 DNF5）；`dnf module` 已弃用。
- `dnf provides '*/bin/dig'`：缺命令时查该装哪个包（包名常和命令名不同）。
- `dnf history` / `history undo ID`：回滚需要仓库里还有旧版本 → 升级前先快照。
- 仓库在 `/etc/yum.repos.d/*.repo`；dnf **会忽略拼错的配置项**。
- Flatpak：应用自带 runtime、沙箱隔离，主要用于桌面应用；仓库叫 **remote**。

## 关键步骤

```bash
rpm -qc openssh-server; rpm -qf /etc/passwd
rpm -V $(rpm -qf /etc/vimrc)          # 或 rpm -Vf 文件路径
sudo dnf install -y bind-utils        # dnf provides '*/bin/dig' 查到的
dnf history; sudo dnf history undo <ID> -y

# 本地仓库
sudo dnf download tree
sudo createrepo_c /opt/localrepo      # 生成元数据，没有它就只是普通目录
sudo dnf install -y tree --disablerepo='*' --enablerepo=localrepo
sudo dnf config-manager --set-disabled localrepo

# 切换镜像（只替换前缀，在副本上练习）
sed -i -E 's|^#?baseurl=https://mirrors.aliyun.com/rockylinux|baseurl=https://mirrors.ustc.edu.cn/rocky|' /tmp/test.repo

# 恢复被改坏的软件包配置文件
sudo rm /etc/vimrc && sudo dnf reinstall -y vim-common

# Flatpak
sudo flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
sudo flatpak install -y flathub org.gnome.Calculator
sudo flatpak uninstall -y --unused
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| vimrc 恢复 | `rpm -V vim-common` | 无输出 | ✅ |
| 仓库完好 | `dnf repolist` | appstream / baseos / extras | ✅ |
| 镜像切换（副本） | `grep '^baseurl' /tmp/test.repo` | 只换前缀，路径完整 | ✅ |
| 本地仓库 + Flatpak | 实验 6、7 | — | ✅（自述完成） |

## 踩坑

- 往 vimrc 追加测试行时 `tee` 漏了 `-a`，整个文件被覆盖成一行 → 删掉文件后 `dnf reinstall vim-common`（rpm 重装通常不覆盖已修改的配置文件，文件不存在时才会放回原版）。
- 想用 `sed '$d'` 删最后一行恢复，没加 `-i`，只打印到屏幕没改文件。
- 切换镜像的 sed 没生效：没加 `-E`；而且**没先看原始文件**——系统早已是阿里云镜像，根本没有 `mirrorlist=` 行。幸好没生效：`s|^#?baseurl=.*|新地址|` 会把所有仓库的路径全部吞掉，dnf 彻底不可用 → 只替换需要改的前缀。
- `.repo` 写成 `pgpcheck=0` → dnf 静默忽略，仍按默认校验签名 → `gpgcheck`。
- `rpm -V /usr/sbin/sshd` → `-V` 后跟包名，要 `rpm -Vf` 或 `rpm -V $(rpm -qf ...)`。
- 用 `cp -a /etc/yum.repos.d.bak.* /etc/yum.repos.d` 恢复 → 目标目录已存在时会复制成子目录，什么都没恢复 → 先删除目标目录再复制，写完整备份名。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）为什么安全扫描报 OpenSSH 版本有漏洞，不一定要升级上游版本？
<details><summary>答案</summary>

红帽常通过增加发布号回移补丁，上游版本号不变；用 `rpm -q --changelog openssh` 查是否已修复。
</details>

2. （操作）写一个 .repo：ID myrepo，地址 http://192.168.1.100/repo/，启用，不校验签名。
<details><summary>答案</summary>

```ini
[myrepo]
name=myrepo
baseurl=http://192.168.1.100/repo/
enabled=1
gpgcheck=0
```
</details>

3. （排障）`.repo` 里明明写了不校验签名，装包却报 GPG 错误，先查什么？
<details><summary>答案</summary>

配置项拼写（如 gpgcheck 拼错会被静默忽略）；改完 `dnf clean all` 再试。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
