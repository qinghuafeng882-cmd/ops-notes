# 用户、组与 sudo

- 日期：2026-10-04（W2-D1，国庆停学 5 天后重新开始）
- 考点：RHCSA EX200 —— 创建/删除/修改本地用户和组；修改密码并设置有效期；配置 sudo
- 环境：Rocky Linux 10（VirtualBox）
- 状态：完成
- 来源：根据课程对话整理

## 目标

看懂 passwd/shadow/group 三个文件，会建用户、管密码有效期、配 sudo 规则。

## 关键概念

- `/etc/passwd` 7 列：用户名:x:UID:GID:注释(GECOS):家目录:Shell。所有人可读。
- `/etc/shadow`：密码哈希 + 有效期，只有 root 可读；`!` 开头 = 锁定。
- 系统只认 UID；UID 0 就是 root 权限。
- RHEL"用户私有组"：建用户自动建同名主组。
- **`usermod -G` 是替换，`-aG` 才是追加。**
- 设置密码会刷新"上次修改日期"，所以 **先设密码，再 `chage -d 0`**。
- `/etc/login.defs` 的默认值只对**之后新建**的用户生效。
- sudoers：`谁 主机=(身份) 命令`，组名前加 `%`，命令写**绝对路径**；规则放 `/etc/sudoers.d/`，用 `visudo -f` 编辑，文件权限 `0440`。

## 关键步骤

```bash
sudo groupadd sysmgrs
sudo useradd -G sysmgrs natasha
sudo useradd -s /sbin/nologin sarah
for u in natasha harry sarah; do echo "$u:Flectrag123" | sudo chpasswd; done
sudo chage -d 0 -M 90 tom                     # 首次登录强制改密 + 90 天有效
sudo usermod -aG wheel alice                  # 追加附加组
sudo visudo -f /etc/sudoers.d/ops             # %ops ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
sudo find /etc/sudoers.d -type f -exec chmod 0440 {} +
sudo visudo -c
awk -F: '$3 == 0 {print $1}' /etc/passwd      # 检查 UID 0 账号，只应有 root
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| sudoers 语法错误被拦截 | `visudo -f` 写入 `ALL:(ALL)` | 报语法错误 | ✅ `/etc/sudoers.d/ops:1:9: syntax error` |
| sudoers 全部合格 | `sudo visudo -c` | 全部 parsed OK | ✅（改权限 + 清理旧文件后） |
| UID 0 账号 | 见上 | 只有 root | ✅ root |

## 踩坑

- sudoers 里把 `ALL=` 写成 `ALL:` → visudo 报第 1 行第 9 列语法错误 → 主机字段后面是等号。
- `visudo -c` 报 `bad permissions, should be mode 0440` → `visudo -f` 新建的文件不自动设权限 → `chmod 0440`。
- `sudo chmod 0440 /etc/sudoers.d/*` 报 No such file → **通配符由普通用户的 Shell 展开**，`/etc/sudoers.d` 不让普通用户读 → `sudo sh -c '...'` 或 `sudo find ... -exec`。（和 `sudo echo > file` 同一个原理）
- `/etc/sudoers.d/` 下留着以前实验的 test、test1 等规则文件 → 多余的 sudo 规则是安全隐患 → 检查并删除。
- 热身里 awk 漏 `-F:`，当天出现两次 → 写 awk 前先 `head -3` 看分隔符。
- 达标题密码 `Welcome1` 写成 `Welcome` → 拼写类错误考试直接零分。

## 卡点 / 没想明白的

- 无

## 回忆题

1. （概念）为什么编辑 sudoers 必须用 visudo？
<details><summary>答案</summary>

保存前检查语法；sudoers 有语法错误时所有人的 sudo 都失效，禁止 root 登录的机器只能进救援模式修。
</details>

2. （操作）新员工 tom：建账号、初始密码 Welcome1、首次登录必须改密、有效期 90 天。
<details><summary>答案</summary>

`useradd tom; echo 'tom:Welcome1' | chpasswd; chage -d 0 -M 90 tom`（顺序不能反）
</details>

3. （排障）给 alice 加了 wheel 组，她说还是不能 sudo，可能原因？
<details><summary>答案</summary>

她的会话没重新登录（组信息未刷新）；用了 `-G` 而不是 `-aG`；sudoers 中 wheel 那行被改动或 sudoers.d 有语法/权限问题（`visudo -c`）。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
