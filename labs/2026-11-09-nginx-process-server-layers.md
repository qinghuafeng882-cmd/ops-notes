# Nginx（一）：进程模型，新建 server 并逐层打通外部访问

- 日期：2026-11-09（B1-1、B1-2）
- 考点：面试高频"Nginx 进程模型""reload 做了什么"；RHCSA —— SELinux 端口类型与文件标签、firewalld 永久规则
- 环境：Rocky Linux 10.2（VirtualBox）。node1（192.168.56.11）装 Nginx 1.26.3，firewalld 开启，SELinux Enforcing，1 CPU；node2（192.168.56.12）做客户端。node1 上原有 httpd 占 80、8090，Nginx 原有 server 监听 8080
- 状态：完成（重启后持久性已在 B1-3 课前验证，见验证表）
- 来源：根据课程对话和终端记录整理

## 目标

看清 Nginx 的 master / worker 结构；新建一个监听 8081 的 server，让 node2 能访问，过程中按"监听 → SELinux 端口 → SELinux 文件标签 → 防火墙"逐层定位问题。

## 关键概念

- master（root）：读配置、建监听 socket、fork worker、保活、响应信号。worker（`nginx` 用户）：`accept` 并处理请求。worker 靠 fork 继承监听 socket 的 fd，所以 `ss -p` 里 master 和 worker 的 pid 同时出现。
- `reload` = 给 master 发 SIGHUP：先检查新配置，再起新 worker，旧 worker 处理完手头请求后退出；绑定新端口失败时，master 只在 `error.log` 记 `[emerg]`，继续用旧配置。
- 这台的 `ExecReload=/usr/sbin/nginx -s reload`，命令发出信号就返回 0，不等 master 处理完。
- 症状与层的对应：

| 症状 | 层 |
|---|---|
| `Connection refused`（几 ms，本机也一样） | 没人监听 |
| `No route to host`（几 ms，仅外部） | firewalld REJECT |
| reload 后 `[emerg] bind() ... (13: Permission denied)` | SELinux 端口类型（master 在 `httpd_t`） |
| `403 Forbidden` | SELinux 文件标签或普通权限 |

## 关键步骤

```bash
# —— B1-1 进程模型 ——
sudo dnf install -y nginx                    # 已装，顺带升级 .6 → .7
sudo systemctl enable --now nginx
ps -ef --forest | grep '[n]ginx'             # master（root）+ 1 个 worker（nginx）
sudo ss -lntp | grep ':80'                   # 8080 上 master 与 worker 的 pid 同时出现
curl -I http://localhost                     # 打到 80，返回的是 Apache（httpd 占 80）
sudo nginx -T | head -40                     # 看到 worker_processes auto;
sudo kill 10025                              # 杀 worker：master 不变，新 worker 29005，8080 仍通
curl -I http://localhost:8080

# —— B1-2 新建 server ——
sudo mkdir -p /srv/site1
echo "site1 from node1" | sudo tee /srv/site1/index.html
sudo tee /etc/nginx/conf.d/site1.conf << 'EOF'
server {
  listen 8081;
  server_name _;
  root /srv/site1;
  location / {
    index index.html;
  }
}
EOF
sudo nginx -t && sudo systemctl reload nginx     # 均成功，但 ss 里没有 8081

# 定位
sudo tail -n 15 /var/log/nginx/error.log         # [emerg] bind() to 0.0.0.0:8081 failed (13: Permission denied)
sudo ausearch -m AVC | tail -10                  # denied { name_bind } httpd_t → transproxy_port_t
sudo semanage port -l | grep -w 8081             # transproxy_port_t

# 修端口（8081 在策略里已有定义，必须 -m）
sudo semanage port -m -t http_port_t -p tcp 8081
sudo systemctl reload nginx
sudo ss -lntp | grep ':8081'                     # 已监听

# 修文件标签（curl 返回 403 后）
sudo ausearch -m avc -ts today | tail -8         # denied { getattr } path=/srv/site1/index.html var_t
namei -l /srv/site1/index.html                   # 普通权限正常
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/site1(/.*)?'
sudo restorecon -Rv /srv/site1/
curl -I http://localhost:8081                    # 200，无需 reload

# 修防火墙（node2 上 curl 得到 No route to host）
sudo firewall-cmd --permanent --add-port=8081/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports                   # 8081/tcp 8090/tcp 8888/tcp
curl -I http://192.168.56.11:8081                # node2 上：200
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 8081 在监听 | `ss -lntp \| grep ':8081'` | master 与 worker 持有 | ✅ |
| 本机访问 | `curl -I http://localhost:8081` | 200 | ✅ |
| node2 访问 | `curl -I http://192.168.56.11:8081` | 200 | ✅ |
| 端口规则重启后仍在 | `sudo semanage port -l \| grep '^http_port_t'` | 8081 在 http_port_t | ✅（11-09 重启后） |
| 标签规则持久 | `sudo semanage fcontext -l -C \| grep site1` + `sudo restorecon -Rvn /srv/site1` | 有规则；restorecon 无输出 | ✅（重启后 `ls -Z` 不算证据，见 B1-3 笔记） |
| 防火墙规则重启后仍在 | `sudo firewall-cmd --list-ports` | 含 8081/tcp | ✅（11-09 重启后） |

## 踩坑

- `curl http://localhost` 返回 200 → 打到的是 Apache（80 被 httpd 占）→ 看 `Server:` 头和 `ss`，Nginx 在 8080。
- `grep -n worker_processer` 没有输出 → 拼错 → 改成 `worker_processes`。
- `nginx -t` 通过、`reload` 无报错，8081 却没监听 → `-t` 在自己的 shell 里跑，不受 `httpd_t` 限制；reload 只发信号 → 看 `error.log` 和 AVC。
- `ausearch -m avc -ts recent` 显示 `no matches` → `recent` 只查最近 10 分钟，事件已超过 → 改用 `-ts today`。
- `semanage -m -t http_port_t -p tcp 8081` 报 `invalid choice` → 漏了 `port` 对象 → 补上 `semanage port -m ...`。
- 预测"标签不对会报 `Connection refused`"，实际是 403 → refused 是 TCP 层没人监听，标签问题发生在连接建立之后 → 症状与层要配对。

## 卡点 / 没想明白的

- reload 为什么返回成功但没有生效：原先以为它只检查语法、不绑定端口（实际 master 会绑，失败只写日志）。
- 标签问题对应 403 而不是 refused。

## 面试题（5 道，对话中作答后批改）

1. （复述）`Connection refused` 在 TCP 层面表示什么？
<details><summary>批改要点</summary>

该端口没有进程监听，内核回 RST（reset，重置）。
</details>

2. （解释原因）为什么 `systemctl reload nginx` 返回成功，新端口却没有出现？
<details><summary>批改要点</summary>

reload 只是发 SIGHUP（这台是 `nginx -s reload`），发出去就返回 0；master 之后才读配置、绑定，失败只写 `error.log`，旧配置继续用。
</details>

3. （排障）新站点放在 `/data/web`，返回 403，`namei -l` 显示权限正常，下一步看什么？
<details><summary>批改要点</summary>

SELinux：`ausearch -m avc -ts today` 看有没有 `getattr`/`read` 的拒绝；`ls -Zd /data/web` 看标签是否为 `httpd_sys_content_t`；`error.log` 里有具体路径。
</details>

4. （排障）`ausearch -m avc -ts recent` 显示 `no matches`，为什么不能证明没有 SELinux 拒绝？
<details><summary>批改要点</summary>

`recent` 只查最近 10 分钟，事件更早就查不到。（超纲）有些拒绝被 `dontaudit` 规则压掉，不写日志。
</details>

5. （反事实）当初用 `chcon` 代替 `semanage fcontext`，会有什么后果？
<details><summary>批改要点</summary>

`chcon` 只改磁盘上的标签，不留规则；之后 `restorecon` 或重新打标签（relabel）会按规则库改回去。`semanage fcontext` 写规则，`restorecon` 应用规则，两条缺一不可。
</details>

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
