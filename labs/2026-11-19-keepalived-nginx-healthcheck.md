# Keepalived：Nginx 健康检查与自动切换

- 日期：2026-11-19
- 考点：运维扩展 —— Keepalived `vrrp_script` 跟踪 Nginx（RHCSA EX200 不考）
- 环境：Rocky Linux 10（VirtualBox），node1 192.168.56.11 / node2 192.168.56.12，VIP 192.168.56.50/24，`enp0s8`。配置字段沿用 [11-18 HA 基础](2026-11-18-keepalived-ha-vip.md)
- 状态：完成（enable 之后没有再重启）
- 来源：课堂终端记录。时间都是北京时间（Asia/Shanghai）。归档时没有重跑

## 目标

给两台 Keepalived 加上本机 8088 检查。Nginx 停了要自动把 VIP 让走，Nginx 恢复后 priority 高的 node1 再抢回来。

## 关键概念

- 脚本的返回码就是最后一条命令的返回码。这里最后一条是 curl，所以 curl 的退出码就是检查结果。
- `weight 0`：检查失败让这个 VRRP 实例进入 FAULT，不是只把优先级调低。
- 日志里的 `sent 0 priority` 是退出 MASTER 时发出的通告。配置里的 priority 150 没有被改成 0。这是课堂讲解，不是事后改文件对过的记录。
- 手动跑检查脚本，退出码只回到当前终端，不会因此移动 VIP。日志里的脚本返回码是 Keepalived 自己按 interval 调度的。
- `sudo` 只管紧跟着的那一条命令，不穿过 `&&`。
- `systemctl enable` 只设置开机启动，不等于 `start`。
- curl 退出码 2 属于初始化失败。初稿那次没有留下 curl 的报错原文，不能补写。

## 关键步骤

重启宿主机之后：node1 的 nginx 是 active，keepalived 是 inactive；node2 两项都是 inactive；两台都没有 VIP。接着启动 node2 的 nginx，两台访问本机 8088，各自返回自己的 `ha test from nodeN`。

```bash
# 两台
command -v curl                         # /usr/bin/curl
getent passwd keepalived_script         # 无结果

useradd --system --no-create-home --shell /sbin/nologin keepalived_script
```

`/usr/local/bin/check-nginx-ha.sh`，属主 root:root，模式 755。最终内容：

```bash
#!/bin/bash
/usr/bin/curl --noproxy '*' -fsS \
  --connect-timeout 1 --max-time 2 \
  http://127.0.0.1:8088/ >/dev/null 2>&1
```

```bash
# 两台，在 /etc/keepalived/（11-18 的备份也在这个目录）
cp -a keepalived.conf keepalived.conf.before-health
```

在原 VI_1 配置上增加下面几块。VI_1 原有字段这天没有改，只在里面加了 `track_script`。

```text
global_defs {
    script_user keepalived_script
    enable_script_security
}

vrrp_script chk_nginx {
    script "/usr/local/bin/check-nginx-ha.sh"
    interval 3
    timeout 3
    fall 2
    rise 2
    weight 0
}
```

```text
track_script {
    chk_nginx
}
```

```bash
# 两台。node1 曾把 weight 写成 weigth，改完后才是 0
keepalived -t -f                        # 返回 0
```

启动后 node1 持有 VIP，node2 是 BACKUP。

第一轮含手动诊断：停 node1 的 nginx，同时手动跑了检查脚本，退出码 7。keepalived 仍是 active。这次手动执行只把退出码交给终端，不会触发 VIP 切换。下面的时间是 Keepalived 自己写进日志的调度记录。

- 09:08:08 日志里脚本返回 7
- 09:08:11 失败，进入 FAULT，`sent 0 priority`，removing VIPs
- node2 接管，响应 node2
- 恢复 nginx 后：09:20:26 脚本返回 0；09:20:29 成功，状态 BACKUP；09:20:32 MASTER
- node2 去掉 VIP，响应 node1

第二轮用来排除「是不是手动跑脚本才切换」的疑问：整轮没有手动运行检查脚本，也没有操作 Keepalived，只停、再启动 node1 的 nginx，VIP 仍然自动让出并抢回。

- 09:31:41 node1 日志 return 7
- 09:31:44 失败，进入 FAULT，移除 VIP
- node2 有 192.168.56.50，响应 node2
- 09:37:35 return 0
- 09:37:38 成功，退出 FAULT，进入 BACKUP
- 09:37:41 MASTER
- node1 有 .50，node2 没有 .50，响应 node1

恢复时用户敲的是 `sudo nginx -t && systemctl start nginx`。后面那条没有 sudo，弹出 polkit 认证，然后再执行 `sudo systemctl start nginx`。没有证据说第一次 `systemctl start nginx` 一定失败。

```bash
# 11-19 09:57，用户贴了 is-enabled 的结果
systemctl is-enabled nginx keepalived
# 当时：node1 enabled disabled；node2 disabled disabled
sudo systemctl enable nginx keepalived
# 之后两台都是 enabled enabled
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 重启宿主机后的初态 | 看到的服务与地址 | node1 nginx active、keepalived inactive；node2 两项 inactive；都无 VIP | ✅ 这是 enable 之前的初态 |
| 本机 8088 | 启动 node2 nginx 后访问本机 | 各自标签 | ✅ |
| curl 路径 | `command -v curl` | `/usr/bin/curl` | ✅ 两台 |
| 检查用户原先不存在 | `getent passwd keepalived_script` | 无结果 | ✅。创建后没有再记一次 getent |
| 初稿脚本 | `sudo -u keepalived_script` 执行 | 退出码 2 | ✅。没有 curl 报错原文 |
| 语法 | `bash -n` | 无输出 | ✅。查不出错误的 curl 选项 |
| 修正后的脚本 | 同上，以修正后的脚本执行 | 退出码 0 | ✅ |
| 配置测试 | `keepalived -t -f` | 返回 0 | ✅ 两台 |
| 启动后角色 | | node1 持 VIP，node2 BACKUP | ✅ |
| 第一轮停 nginx | 同时手动跑了脚本，退出码 7 | 09:08 日志里 Keepalived 进入 FAULT，node2 接管；09:20 恢复后 node1 抢回。手动执行不触发 VIP 切换 | ✅ 日志是自动调度 |
| 第二轮只停/启 nginx | 整轮不跑脚本，不动 Keepalived | 09:31 自动 FAULT 并让出 VIP；09:37 自动回到 MASTER | ✅ 自动接管和恢复抢占 |
| enable 前 | `systemctl is-enabled nginx keepalived` | node1 enabled/disabled；node2 disabled/disabled | ✅ |
| enable 后 | 同上 | 两台 enabled/enabled | ✅ 09:57 |
| enable 之后再重启 | | 开机自己起来 | 未验证。enable 不等于 start |
| 断电、两台都坏、网络隔离、数据同步 | | | 未做 |

## 踩坑

- 初稿把选项写成 `--no-proxy` 和 `--fsS` → 选项名错了；`bash -n` 没有输出，shell 语法是过的；`>/dev/null 2>&1` 把 curl 的错误藏掉了，没有可引用的报错原文。`sudo -u keepalived_script` 执行，退出码 2，属于 curl 初始化失败 → 改为 `--noproxy` 和 `-fsS`，再执行退出码 0。
- node1 把 `weight` 写成 `weigth` → `keepalived -t` 报 `Unknown keyword` → 改为 `weight`。
- `sudo nginx -t && systemctl start nginx` → `sudo` 不作用到 `&&` 后面，弹出 polkit 认证 → 再执行 `sudo systemctl start nginx`。不把第一次 start 记成必然失败。

## 卡点 / 没想明白的

- 第一轮停 nginx 时手动跑过脚本，曾怀疑是这次手动执行促成了切换。手动执行只把退出码交给终端，不会移动 VIP；09:08 的日志是 Keepalived 自己的调度。第二轮整轮没有手动跑脚本，也没有操作 Keepalived，停/启 node1 的 nginx 后仍然自动切换并恢复。
- 口答把「低权限」说成「低优先级」。分没有因此扣掉，见第 5 题。
- 薄弱的是：0 是成功、非 0 是失败；`fall` 和 `rise` 两个都要记住。不是五题全会。

## 面试题（5 道，对话中作答后批改）

下面是课后实际的 5 题。总分 4/5。

1. （排障）Keepalived 还是 active，但是没有 VIP。
<details><summary>批改要点</summary>

0 是成功，非 0 是失败。用户说成返回 0 导致失败，把返回码说反了。0.5 分。
</details>

2. （操作）`fall` 和 `rise` 各是什么。
<details><summary>批改要点</summary>

`fall 2` 是连续失败 2 次，`rise 2` 是连续成功 2 次。`fall 2` 答对，`rise 2` 忘了。0.5 分。
</details>

3. （概念）检查失败之后实例处于什么状态，还算不算主。
<details><summary>批改要点</summary>

进入 FAULT，放弃主资格。1 分。讲解：`sent 0 priority` 是退出 MASTER 的通告，不是把配置里的 priority 150 永久改成 0。`weight 0` 是失败即 FAULT，不是只降低优先级。
</details>

4. （概念）检查地址为什么写 127.0.0.1。
<details><summary>批改要点</summary>

查的是本机 Nginx，不会把对端的 8088 当成自己的健康结果。1 分。
</details>

5. （取舍）检查脚本用什么用户跑，文件归谁。
<details><summary>批改要点</summary>

`keepalived_script` 这种权限低的用户来执行；脚本 root:root，执行用户不能改它。用户说成「低优先级」，纠正为「低权限」。1 分。
</details>

作答结果：4/5。薄弱点：0 成功、非 0 失败；`fall` 与 `rise`。

## 复习记录

起算日是上课日 11-19。这三项都还没做。

- [ ] D+1
- [ ] D+3
- [ ] D+7
