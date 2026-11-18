# Nginx HTTPS：自签证书、443、HTTP 跳转 HTTPS（B3-2）

- 日期：2026-11-17
- 考点：RHCSA EX200 —— SELinux 文件上下文/端口标签、firewalld 服务放行（Nginx HTTPS 为运维岗扩展内容）
- 环境：Rocky Linux 10（VirtualBox），node1 192.168.56.11 服务端 / node2 192.168.56.12 客户端
- 状态：完成

## 目标

在 node1 用 openssl 生成自签证书，Nginx 在 443 提供 HTTPS，并用 8083 演示 HTTP → HTTPS 301 跳转（80 被 httpd 占用）。

## 关键概念

- HTTPS（HyperText Transfer Protocol Secure）= HTTP 跑在 TLS（Transport Layer Security）之上，HTTP 报文不变；TLS 提供加密、完整性、身份认证。
- 私钥只留服务器（root 600）；证书 = 公钥 + 身份信息（CN/SAN、有效期）+ 签发者签名，公开下发。
- 自签证书与 CA（Certificate Authority）证书加密强度一样，区别只在签发者是否在客户端预装的受信根 CA 列表里。
- 现代客户端校验 SAN（Subject Alternative Name），不看 CN（Common Name）。
- 证书时间用 GMT 记录：`notBefore=Nov 16 23:26:11 2026 GMT` = 北京时间 11-17 07:26。

## 关键步骤

```bash
# node1：确认现场（443 空闲，80 是 httpd；openssl 命令行工具未装）
sudo ss -lntp | grep -E ':443|:80 '
rpm -q openssl mod_ssl
sudo dnf install -y openssl
openssl version                       # OpenSSL 3.5.8
ls -ld /etc/pki/tls/certs /etc/pki/tls/private

# 生成 2048 位 RSA 私钥 + 自签证书（365 天，私钥不加密码）
sudo openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout /etc/pki/tls/private/node1.key \
  -out /etc/pki/tls/certs/node1.crt \
  -subj "/CN=192.168.56.11" \
  -addext "subjectAltName=IP:192.168.56.11"
sudo ls -l /etc/pki/tls/certs/node1.crt /etc/pki/tls/private/node1.key   # crt 644，key 600
sudo openssl x509 -in /etc/pki/tls/certs/node1.crt -noout -subject -dates -ext subjectAltName

# 测试站点 + HTTPS server 块
sudo mkdir -p /srv/https-test
echo "hello https from node1" | sudo tee /srv/https-test/index.html
sudo tee /etc/nginx/conf.d/https-test.conf >/dev/null <<'CONF'
server {
    listen 443 ssl;
    server_name 192.168.56.11;
    root /srv/https-test;

    ssl_certificate     /etc/pki/tls/certs/node1.crt;
    ssl_certificate_key /etc/pki/tls/private/node1.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
}
CONF
sudo nginx -t
sudo systemctl reload nginx
sudo ss -lntp | grep ':443'
sudo semanage port -l | grep -w http_port_t       # 443 默认已在 http_port_t
sudo firewall-cmd --add-service=https --permanent && sudo firewall-cmd --reload   # ALREADY_ENABLED

# 403 排障与修复
sudo tail -n 3 /var/log/nginx/error.log
ls -ldZ /srv/https-test /srv/https-test/index.html
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/https-test(/.*)?"
sudo restorecon -Rv /srv/https-test

# HTTP → HTTPS 跳转（8083 作为 HTTP 入口）
sudo semanage port -a -t http_port_t -p tcp 8083
sudo firewall-cmd --add-port=8083/tcp --permanent && sudo firewall-cmd --reload
sudo tee -a /etc/nginx/conf.d/https-test.conf >/dev/null <<'CONF'
server {
  listen 8083;
  server_name 192.168.56.11;
  return 301 https://$host$request_uri;
}
CONF
sudo nginx -t && sudo systemctl reload nginx

# node2：测试
curl https://192.168.56.11/          # (60) self-signed certificate
curl -k https://192.168.56.11/
curl -I http://192.168.56.11:8083/abc?x=1
curl -kL http://192.168.56.11:8083/

# 面试题第 2 题的实测：用 vim 临时去掉 listen 443 后的 ssl → reload → curl → 改回 → reload
sudo vim /etc/nginx/conf.d/https-test.conf
sudo nginx -t && sudo systemctl reload nginx
```

## 验证

| 检查项 | 命令 | 预期结果 | 结果 |
|---|---|---|---|
| 证书主体/有效期/SAN | `openssl x509 -noout -subject -dates -ext subjectAltName` | CN 与 SAN 为 192.168.56.11，有效 1 年 | ✅ |
| 443 监听 | `ss -lntp \| grep :443` | nginx 监听 0.0.0.0:443 | ✅ |
| HTTPS 页面 | node2 `curl -k https://192.168.56.11/` | `hello https from node1` | ✅（修复 SELinux 后） |
| 不信任自签证书 | node2 `curl https://192.168.56.11/` | `(60) self-signed certificate` | ✅（预期报错） |
| 301 跳转保留路径参数 | `curl -I http://192.168.56.11:8083/abc?x=1` | `Location: https://192.168.56.11/abc?x=1` | ✅ |
| 跟随跳转 | `curl -kL http://192.168.56.11:8083/` | 返回首页内容 | ✅ |
| 重启后仍生效 | reboot 后重复上述 curl | 同上 | 未验证 |

## 踩坑

- `openssl req ... -newKey rsa:2048` → `req: Extra (unknown) options: "newKey" "rsa:2048"` → 选项区分大小写，应为 `-newkey` → 改正后重新执行。
- node2 `curl -k https://...` → `403 Forbidden`；error.log：`"/srv/https-test/index.html" is forbidden (13: Permission denied)`，`ls -Z` 为 `var_t` → `/srv` 下新建目录默认 `var_t`，nginx（`httpd_t`）不能读 → `semanage fcontext` 设 `httpd_sys_content_t` + `restorecon -Rv`。（实际顺序：先修复后补看证据，下次应先留证据再动手。）
- `semanage port -a ... 8083` → `Port tcp/8083 already defined, modifying instead` → 8083 原属 `us_cli_port_t`，semanage 自动改为修改；不是报错，之后 8083 同时出现在 `http_port_t` 和 `us_cli_port_t`。
- `sudo nginx -t && systemctl reload nginx` → 弹出 polkit `AUTHENTICATING FOR org.freedesktop.systemd1.manage-units` → `&&` 后的命令漏了 `sudo`（sudo 只作用于紧跟的一条命令）→ Ctrl+C 后用 `sudo systemctl reload nginx`。
- `sudo nignx -t` → `command not found` → 拼写错误 → 改为 `nginx`。

## 卡点 / 没想明白的

- `openssl x509 -in ... -noout -subject -dates -ext subjectAltName` 各参数含义 → 已解答：`-noout` 不输出 Base64 证书本体，只显示指定字段；看全部用 `-text -noout`。
- 去掉 `ssl` 后报错是什么 → 老师推测为 `wrong version number`，实测为 `curl: (35) TLS connect error: error:0A0000C6:SSL routines::packet length too long`，以实测为准。原因解释（推测）：curl 把 nginx 回的明文 HTTP 字节当 TLS 记录头解析，读出的长度异常。

## 面试题（5 道，对话中作答后批改）

1. （概念）自签证书和 CA 签发的证书，加密强度有区别吗？真正区别在哪？
<details><summary>批改要点</summary>

无区别，强度取决于算法和密钥长度；区别在签发者是否在客户端预装的受信根 CA 列表里。自签证书手动导入信任库后同样被信任（企业内网自建 CA）。
</details>

2. （排障）只写了 `listen 443;` 漏掉 `ssl`，`curl -k https://IP/` 会怎样？
<details><summary>批改要点</summary>

TLS 握手失败、拿不到页面，`-k` 也没用（`-k` 只跳过证书校验）。nginx 在 443 上讲明文 HTTP，把 ClientHello 当成坏的 HTTP 请求。实测报错：`(35) ... packet length too long`。
</details>

3. （排障/概念）为什么私钥用 `-nodes`（不加密码）？
<details><summary>批改要点</summary>

nginx 启动/重启都要读私钥，有密码会卡在等待输入，无人值守重启或自动化部署时服务起不来；私钥安全靠文件权限 root 600。
</details>

4. （操作）`return 301 https://$host$request_uri;` 漏掉 `$request_uri` 会怎样？
<details><summary>批改要点</summary>

原请求的路径和参数丢失，所有请求都跳到 HTTPS 首页。
</details>

5. （取舍）curl 报 `(60) self-signed certificate`，不用 `-k` 的正规解决方法？
<details><summary>批改要点</summary>

公网：向 CA 申请（如 Let's Encrypt）。内网 IP + 自签：让客户端信任证书——临时 `curl --cacert node1.crt URL`；系统级放入 `/etc/pki/ca-trust/source/anchors/` 后 `update-ca-trust`；企业做法是自建 CA 并分发根证书。
</details>

作答结果：4/5。第 2 题 0.5（原因对，未答出现象是握手失败）；第 5 题 0.5（只答了找 CA 申请，没想到内网场景让客户端信任证书）。

- 最薄弱：区分「`-k` 只跳过身份校验，不跳过握手」；内网自签证书的信任方式（`--cacert` / `update-ca-trust`）。

## 复习记录

- [x] D+1（11-18，1.5/2（合并复习，三篇合计 3.5/5，不是本篇单独 5 题）；`curl -k` 跳过证书校验，1 分；`--cacert` 少写一条横线，0.5 分）
- [ ] D+3
- [ ] D+7
