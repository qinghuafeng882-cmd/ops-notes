# 第 6 周速查：Nginx

## B1：进程模型与外部访问

- master（root）：读配置、建监听 socket、fork worker、保活（SIGCHLD）、响应信号；worker（`nginx` 用户）：`accept` 并处理请求。
- worker 靠 fork 继承监听 fd，所以 `ss -p` 里 master 和 worker 都显示。
- `user nginx;` 由 master 用来给 worker 降权；`worker_processes auto;` = 按 CPU 核数。
- reload = SIGHUP：先验新配置，再起新 worker，旧 worker 处理完手头请求才退；绑定失败只写 `error.log`，旧配置继续用。

```bash
nginx -t                                  # 检查语法和引用文件（不代表运行时能绑定）
sudo nginx -T                             # -t + 打印 include 展开后的磁盘配置（不等于进程正在用的）
sudo systemctl reload nginx               # 返回 0 只表示信号发出去了
sudo tail -n 15 /var/log/nginx/error.log  # reload 后必看：[emerg] bind() ... (13: Permission denied)
ps -ef --forest | grep '[n]ginx'          # master + worker 的父子关系
sudo ss -lntp 'sport = :8081'             # 谁在监听
curl -I http://IP:PORT                    # -I 只取响应头；看 Server: 头确认打到的是谁
sudo ausearch -m avc -ts today            # recent 只查最近 10 分钟
namei -l <文件路径>                        # 逐级看普通权限
sudo semanage port -l | grep -w <端口>    # 端口当前类型
sudo semanage port -m -t http_port_t -p tcp <端口>   # 策略里已有定义用 -m；没有才用 -a
sudo semanage fcontext -a -t httpd_sys_content_t '/路径(/.*)?'   # 写规则
sudo restorecon -Rv /路径                  # 应用规则（chcon 不留规则，会被改回去）
sudo firewall-cmd --permanent --add-port=<端口>/tcp && sudo firewall-cmd --reload
```

| 现象 | 层 | 查什么 |
|---|---|---|
| `Connection refused`，本机也一样 | 没人监听 | `ss -lntp`；reload 后看 `error.log` |
| `No route to host`，几 ms，仅外部 | firewalld REJECT | `firewall-cmd --list-ports` |
| reload 后 `bind() ... Permission denied` | SELinux 端口类型 | `ausearch` 的 `name_bind` + `semanage port -l` |
| `403 Forbidden` | SELinux 标签或普通权限 | `ausearch` 的 `getattr`/`read` + `ls -Zd` + `namei -l` |

⚠️ 看到 200 不等于是目标服务，先看 `Server:` 头和端口占用。
⚠️ `nginx -t` 在自己的 shell 里跑，不受 `httpd_t` 限制，通过不代表运行时能绑定。
⚠️ 改文件标签不需要 reload（访问时检查）；改端口和配置需要。
⚠️ root 运行仍然 `Permission denied`，先想 SELinux。

## B1-3：location 匹配

| 写法 | 类型 | 命中后 |
|---|---|---|
| `= /x` | 精确（字符串完全相同） | 立即选中 |
| `^~ /x/` | 前缀，禁止正则 | 是最长前缀时立即选中，跳过正则 |
| `/x/` | 普通前缀 | 只记下，继续查正则 |
| `~` / `~*` | 正则（`~*` 不分大小写） | 按书写顺序，第一个命中胜出 |

顺序：`=` → 最长前缀（带 `^~` 就停）→ 正则按顺序 → 都不中用最长前缀。

⚠️ 前缀比**长度**，正则比**顺序**。
⚠️ 纯字符串比较：`location /api` 也吃 `/apiv2`、`/api-internal`；写 `/api/`。`= /lab` 不匹配 `/lab/`。
⚠️ 普通前缀再长也会被正则覆盖，想挡住正则用 `^~`。

```bash
sudo semanage fcontext -l -C | grep <路径>   # -C：只列自定义标签规则
sudo restorecon -Rvn <路径>                  # -n 演练；无输出 = 规则与当前标签一致
```

⚠️ 重启后标签还在 ≠ 标签持久（重启不改标签，`chcon` 也会留下），要查规则本身。端口规则和 `--permanent` 防火墙规则重启后还在，才能算证据。
