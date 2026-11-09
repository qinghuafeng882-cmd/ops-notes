# Nginx（二）：location 匹配规则；重启后持久性验证

- 日期：2026-11-09（B1-3）
- 考点：面试高频"location 匹配优先级"；RHCSA —— SELinux / firewalld 规则持久性
- 环境：Rocky Linux 10.2（VirtualBox）。node1（192.168.56.11）Nginx 1.26.3，8081 为 site1（`/etc/nginx/conf.d/site1.conf`，root `/srv/site1`）；node2（192.168.56.12）做客户端
- 状态：完成（第 3 题的 `^~` 改法未动手验证）
- 来源：根据课程对话和终端记录整理

## 目标

1. 重启 node1，验证 B1-2 的端口、标签、防火墙三条规则是否持久。
2. 搞清楚多个 `location` 同时匹配时 Nginx 选哪一个。

## 关键概念

| 写法 | 类型 | 命中后 |
|---|---|---|
| `= /lab` | 精确匹配（字符串完全相同） | 立即选中，结束 |
| `^~ /lab/static/` | 前缀，禁止正则 | 是最长前缀时立即选中，跳过正则 |
| `/lab/img/` | 普通前缀 | 只记下来，继续查正则 |
| `~ \.png$` / `~* ...` | 正则（`~*` 不区分大小写） | 按书写顺序，第一个命中的胜出 |

选择顺序：`=` 命中即结束 → 找**最长**前缀，带 `^~` 即结束 → 按**顺序**查正则，命中即结束 → 都没命中就用最长前缀。

- 前缀比长度，正则比顺序。
- 匹配是纯字符串比较，不认识"目录"：`/lab` 也匹配 `/labx`、`/lab/`。习惯写 `location /api/`。

持久性要看"证据能不能区分两种假设"：

| 规则 | 存在哪里 | 重启后还在能否证明持久 |
|---|---|---|
| 端口 `semanage port` | SELinux 策略库 | 能 |
| 防火墙 `--permanent` | `/etc/firewalld/zones/*.xml`，重启时运行时配置从这里加载 | 能 |
| 文件标签 | 文件扩展属性 `security.selinux` | **不能**：重启不改标签，`chcon` 的标签也会留下；要查规则本身 |

## 关键步骤

```bash
# —— 重启 node1 后检查 ——
systemctl is-active nginx; systemctl is-enabled nginx    # active / enabled
sudo ss -lntp | grep nginx                    # 8081（仅 IPv4）、8080（IPv4 + IPv6）
semanage port -l | grep '^http_port_t'        # 不加 sudo：ValueError ... store cannot be accessed
sudo semanage port -l | grep '^http_port_t'   # 含 8081
ls -Zd /srv/site1 /srv/site1/index.html       # httpd_sys_content_t
sudo firewall-cmd --list-ports                # 8081/tcp 8090/tcp 8888/tcp
curl -I http://192.168.56.11:8081             # node2 上：200

# 标签规则的真正证据
sudo semanage fcontext -l -C | grep site1     # /srv/site1(/.*)? ... httpd_sys_content_t（-C 只列自定义规则）
sudo restorecon -Rvn /srv/site1               # 无输出 = 规则与当前标签一致（-n 只演练）
```

加进 site1.conf 的 server 块（终端未贴出编辑和 reload 过程；curl 结果与此配置一致）：

```nginx
location = /lab              { return 200 "A exact\n"; }
location /lab                { return 200 "B prefix /lab\n"; }
location ^~ /lab/static/     { return 200 "C ^~ static\n"; }
location /lab/img/           { return 200 "D prefix img\n"; }
location ~ \.png$            { return 200 "E regex png\n"; }
```

```bash
# node2 上逐个测试
curl http://192.168.56.11:8081/lab                 # A exact
curl http://192.168.56.11:8081/lab/                # B prefix /lab
curl http://192.168.56.11:8081/lab/img/a.png       # E regex png
curl http://192.168.56.11:8081/lab/static/a.png    # C ^~ static
curl http://192.168.56.11:8081/lab/img/a.PNG       # D prefix img
curl http://192.168.56.11:8081/labx                # B prefix /lab
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| nginx 开机自启 | `systemctl is-enabled nginx` | enabled，重启后 active | ✅ |
| 三条规则重启后仍在 | 见上 | 均保留 | ✅ |
| 六个 URI 的匹配结果 | node2 上 `curl` | A B E C D B | ✅ |
| D 改为 `^~` 后 `/lab/img/a.png` 返回 D | `curl` | D prefix img | 未验证 |

## 踩坑

- 预测 `/lab/` 命中 A → `=` 要求字符串完全相同，多一个 `/` 就不等 → 实际走前缀，命中 B。
- 预测 `/labx` 返回 404 → 前缀只比较字符串开头，`/labx` 以 `/lab` 开头 → 命中 B。
- `semanage` 不加 sudo 报 "policy is not managed" → 实际是普通用户读不了策略库 → 加 sudo。
- `crul` 敲错 → `command not found`。

## 卡点 / 没想明白的

- 课前预测"三条都不会丢"是对的，但用"重启后标签还在"来证明标签持久，证据不能区分 `chcon` 和 `semanage fcontext`。

## 面试题（5 道，对话中作答后批改）

1. （复述）两个正则 `location` 都能匹配同一个 URI，选哪个？
<details><summary>批改要点</summary>

配置文件里书写顺序靠前的那个。
</details>

2. （原因）`/lab/img/a.png` 的最长前缀是 D，为什么最后由 E 处理？
<details><summary>批改要点</summary>

普通前缀命中后只被记下，还要继续查正则；正则命中就覆盖它。
</details>

3. （应用）想让 `/lab/img/` 下的 `.png` 也由 D 处理，最小改动是什么？
<details><summary>批改要点</summary>

改服务端配置：`location ^~ /lab/img/`。它是最长前缀，带 `^~` 就跳过正则。不能靠改客户端请求的 URI。
</details>

4. （排障）`location /admin { deny all; }` 让 `/administrator.html` 也 403，原因？
<details><summary>批改要点</summary>

普通前缀是字符串比较，`/administrator.html` 以 `/admin` 开头。改成 `location /admin/`。
</details>

5. （反事实）删掉 `location = /lab`，`curl .../lab` 返回什么？
<details><summary>批改要点</summary>

B prefix /lab。
</details>

作答结果：4/5。第 3 题答成"客户端改用 `.PNG` 绕开正则"，题目问的是服务端配置。

## 复习记录

- [ ] D+1
- [ ] D+3
- [ ] D+7
