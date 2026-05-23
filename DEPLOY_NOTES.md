# Sub2API 部署记录

部署时间：2026-05-23
主机：vultr.guest（149.28.130.84，Rocky Linux 9.7）
部署方式：官方 install.sh + AUTO_SETUP 环境变量注入 + systemd

## 一、访问信息

- **HTTPS 入口（唯一对外）**：https://149.28.130.84:8443
  自签证书（CN=149.28.130.84，2036 年到期），浏览器首访会提示「不安全」，点「高级 → 继续访问」即可。
- **健康检查**：https://149.28.130.84:8443/health → `{"status":"ok"}`
- **HTTP 8088**：仅本机可达（`SERVER_HOST=127.0.0.1` + firewalld 已收回），供 nginx 反代用，公网无法访问。
- **管理员账号**
  - 邮箱：`admin@sub2api.local`
  - 初始密码：`6f4dd86991ab37f2831205b7b5a6bc0e`（首次启动自动生成的一次性密码，**强烈建议首次登录后立即修改**）

## 二、组件清单

| 组件 | 版本 | 监听 | 说明 |
|---|---|---|---|
| sub2api | v0.1.130 (binary) | `0.0.0.0:8088` | systemd 服务 `sub2api`，开机自启 |
| PostgreSQL | 15.17 | `127.0.0.1:5432` | systemd 服务 `postgresql`，数据 `/var/lib/pgsql/data` |
| Redis | 7 (宿主已装) | `127.0.0.1:6379` | 复用宿主已有实例，无密码 |

## 三、关键路径

```
/opt/sub2api/                          # 安装目录，属主 sub2api:sub2api
├── sub2api                            # 主二进制 (88 MB)
├── config.yaml                        # AUTO_SETUP 写入的运行配置（含 JWT/TOTP 密钥）
├── .installed                         # 防再次跑 setup 的锁文件
├── data/                              # 数据目录
├── install.sh                         # 官方安装脚本（解压时一并落地）
└── docker-compose*.yml / Dockerfile   # 部署模板

/etc/systemd/system/sub2api.service    # systemd 服务定义（含全部 ENV）
/var/lib/pgsql/data/                   # PostgreSQL 数据目录
/var/lib/pgsql/data/pg_hba.conf        # 已改为 scram-sha-256 (127.0.0.1/::1)
/tmp/sub2api-secrets.env               # 临时密钥备份 (0600)：PGPASS / JWT_SECRET / TOTP_KEY
```

## 四、systemd 服务

服务文件：`/etc/systemd/system/sub2api.service`

关键环境变量（详细密码值见 `/tmp/sub2api-secrets.env`）：

```
AUTO_SETUP=true
SERVER_HOST=127.0.0.1   # 只监听本机，nginx 反代到这里；不对外暴露
SERVER_PORT=8088
SERVER_MODE=release
DATABASE_HOST=127.0.0.1
DATABASE_PORT=5432
DATABASE_USER=sub2api
DATABASE_PASSWORD=<生成的 16 字节 hex>
DATABASE_DBNAME=sub2api
DATABASE_SSLMODE=disable
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
ADMIN_EMAIL=admin@sub2api.local
JWT_SECRET=<生成的 32 字节 hex，固定值，避免重启失效会话>
TOTP_ENCRYPTION_KEY=<生成的 32 字节 hex，固定值，避免 2FA 配置失效>
TZ=Asia/Shanghai
```

加固选项（沿用 install.sh 默认）：

```
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/opt/sub2api
User=sub2api
Group=sub2api
```

## 五、数据库

- 数据库名：`sub2api`
- 用户：`sub2api`
- 认证：`scram-sha-256`（pg_hba.conf 默认 `ident` 已替换）
- 重置密码命令：
  ```bash
  sudo -u postgres psql -c "ALTER USER sub2api WITH PASSWORD '新密码';"
  # 然后同步改 /etc/systemd/system/sub2api.service 的 DATABASE_PASSWORD 并 daemon-reload + restart
  ```

## 六、常用运维命令

```bash
# 状态 / 重启 / 停止
sudo systemctl status  sub2api
sudo systemctl restart sub2api
sudo systemctl stop    sub2api

# 实时日志
sudo journalctl -u sub2api -f

# 最近 200 行日志
sudo journalctl -u sub2api -n 200 --no-pager

# 验证登录（替换为当前 admin 密码）
curl -s -X POST http://127.0.0.1:8088/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@sub2api.local","password":"<密码>"}'

# 连数据库
PGPASSWORD=<密码> psql -h 127.0.0.1 -U sub2api -d sub2api
```

## 七、网络放行（已配置）

两层防火墙都需要放行端口：

1. **宿主机 firewalld**（已配）
   ```bash
   sudo firewall-cmd --permanent --add-port=8443/tcp   # HTTPS（唯一对外端口）
   sudo firewall-cmd --reload
   sudo firewall-cmd --list-ports   # 验证含 8443/tcp
   ```
   > 8088 不需要放行——sub2api 监听 127.0.0.1，只供 nginx 本地反代。
2. **Vultr 控制台 Firewall / Security Group**（已配）
   规则：`accept TCP 8080 - 8090  0.0.0.0/0`，已覆盖 8443。

> 排障：本机 `curl http://127.0.0.1:8088/health` 通但浏览器打不开，绝大概率就是上面两层之一漏放了。

### 7.1 HTTPS 反向代理（nginx 8443 → sub2api 8088）

宿主机已有 nginx 1.20，443 被其他业务占用，sub2api 走独立端口 **8443**：

- 配置文件：`/etc/nginx/conf.d/sub2api.conf`
- 证书：复用 `/etc/nginx/ssl/lianghua.crt`（CN=149.28.130.84 自签，2036 到期）
- 反代后端：`http://127.0.0.1:8088`
- 已处理：SSE/WebSocket（`Upgrade`/`Connection upgrade` + `proxy_buffering off`）、长超时（30 min）、大请求体（256 MB）、`X-Forwarded-*` 透传

```bash
sudo nginx -t        # 改完先校验
sudo nginx -s reload # 再 reload
```

> nginx 1.20 不认 `http2 on;` 新指令，要用老语法 `listen 8443 ssl http2;`。

## 八、已知告警与处理建议

启动日志里有 3 条非致命告警，按需处理：

1. **`mkdir /app: read-only file system`** — 程序默认想写 `/app/data/logs/sub2api.log`，被 systemd 的 `ProtectSystem=strict` 拦下，已自动降级到 stdout/journal。
   - 想恢复独立日志文件：在 service 文件加 `Environment=DATA_DIR=/opt/sub2api/data` 后重启。
2. **`server.trusted_proxies is empty in release mode`** — 没配信任的反向代理网段，`X-Forwarded-For` 不会被采信。
   - 如果前面挂了 Nginx/CDN，在 `/opt/sub2api/config.yaml` 的 `server.trusted_proxies` 里写实际网段。
3. **`CORS allowed_origins not configured`** — 跨域请求会被拒，前端同源访问无影响。
   - 需要跨域 SPA/小程序访问时，在 config.yaml 的 `cors.allowed_origins` 里加白名单。

## 八点五、Claude OAuth 账号接入（替换 aigateway）

本机的 Claude Code 订阅 OAuth credentials 由 sub2api 接管，原先的 Python
单体网关 `/root/aigateway`（FastAPI + claude-agent-sdk，监听 127.0.0.1:8090）
已退役。

### 凭据来源

```
/home/clauded/.claude/.credentials.json
└── claudeAiOauth
    ├── accessToken          → sub2api 字段 access_token
    ├── refreshToken         → sub2api 字段 refresh_token
    ├── expiresAt            → sub2api 字段 expires_at
    ├── subscriptionType: max
    └── rateLimitTier: default_claude_max_5x
```

> sub2api 的 `TokenRefreshService` 接管后会自动 refresh，**别再让 aigateway
> 或本机 `claude` CLI 同时活跃**，否则两边各自 refresh 会把对方的 token 弄废。

### 配置位置（在 Admin UI 完成）

1. 账号管理 → 新增 Anthropic OAuth 账号（贴入上述 credentials）
2. 把账号加入对应 group（默认 group 即可），确保 `schedulable=true / status=active`
3. 用户管理 → 给 admin 账号加余额（standard 计费模式必需）
4. API Key → 生成一把 key（示例：`sk-3a1b...`），归属 admin、绑定到上述 group

### 验证调用

```bash
curl -sk https://149.28.130.84:8443/v1/messages \
  -H "x-api-key: <sk-...>" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":100,
       "messages":[{"role":"user","content":"hi"}]}'
```

200 OK 返回 `content[].text` 即贯通。OpenAI 兼容路径 `/v1/chat/completions` 同样可用。

### aigateway 现状

- 进程：已 `kill 3725432 3725455`（之前是 clauded 用户在交互 shell 起的 nohup）
- 端口 8090：释放
- 代码：`/root/aigateway` 暂留，不自启。彻底归档可：
  ```bash
  tar -C /root -czf ~/aigateway-archive-$(date +%F).tar.gz aigateway
  # 确认无误后再 rm -rf /root/aigateway
  ```

### 排错路径

常见两类错：

| 报错 | 原因 | 处理 |
|---|---|---|
| `INSUFFICIENT_BALANCE` | standard 模式下账号余额为 0 | UI 给用户加余额；或改 `RUN_MODE=simple` 整个关计费 |
| `No available accounts` | OAuth 账号没加入 API Key 所属 group / status 非 active / schedulable=false | UI 检查账号-组绑定、状态、可调度标志 |

## 九、升级

```bash
# 方式 1：Web 控制台左上角「检查更新」按钮（推荐）

# 方式 2：命令行重跑 install.sh（会保留 config.yaml）
sudo bash /opt/sub2api/install.sh
```

## 十、卸载

```bash
sudo systemctl disable --now sub2api
sudo rm /etc/systemd/system/sub2api.service
sudo systemctl daemon-reload
sudo rm -rf /opt/sub2api
sudo userdel sub2api

# 数据库（可选）
sudo -u postgres psql -c "DROP DATABASE sub2api;"
sudo -u postgres psql -c "DROP USER sub2api;"

# 临时密钥文件
rm -f /tmp/sub2api-secrets.env
```

## 十一、本次部署遇到的两个坑

记录下来，下次复用环境时会快很多。

1. **`install.sh` 管道运行（curl | sudo bash）会卡住**
   原因：脚本里所有 `read -p ... < /dev/tty`，sudo 在管道里没分配伪 tty。
   解决：下载到本地，把 `is_interactive()` patch 成 `return 1`，强制非交互走默认值；端口、AUTO_SETUP 等通过事后改写 systemd unit 注入。

2. **PostgreSQL 默认 `host ... ident` 认证拒密码登录**
   原因：Rocky 9 自带 `postgresql-setup --initdb` 生成的 `pg_hba.conf` 用 ident 映射。
   解决：把 `127.0.0.1/32` 和 `::1/128` 两行从 `ident` 改成 `scram-sha-256`，`systemctl reload postgresql`。

## 十二、临时权限

部署期间在 `/etc/sudoers.d/claude-deploy` 配了 `claude ALL=(ALL) NOPASSWD: ALL`。
**部署完成后建议立即清理**：

```bash
sudo rm /etc/sudoers.d/claude-deploy
```
