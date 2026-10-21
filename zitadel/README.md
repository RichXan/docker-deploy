# zitadel（身份中心 · id.xanne.cloud）

统一身份中心：OIDC / OAuth2 / SAML / 多租户（Organization）/ SSO。替代 2026-10-20 下线的 Ory。
跑在 **tencent/tencent-bro** 的 `/opt/docker/zitadel`（本仓库其余目录多在 sealan，这一个例外）。
设计、取舍与实施记录见 `praxis-loop/servers` 的 `docs/plans/2026-10-20-ory-to-zitadel.md`，服务登记见该仓库 `SERVICES.md`。

## 组成

| 服务 | 镜像 | 作用 |
|---|---|---|
| `zitadel-api` | `ghcr.io/zitadel/zitadel` | OIDC/SAML/管理 API/控制台；`start-from-init` 首启建表建实例，之后幂等 |
| `zitadel-login` | `ghcr.io/zitadel/zitadel-login` | 新版登录页（Next.js），`/ui/v2/login` |
| `proxy` | `caddy:2.8-alpine` | 唯一对外端口 `127.0.0.1:13003`；按路径分流，并用 **h2c** 连 api |

数据库是 `shared-postgres`（postgres-net 上的 postgres:16）里的库 `zitadel`，属主角色 `zitadel`（非超级用户）。
隧道：tencent 的 cloudflared，`id.xanne.cloud → http://localhost:13003`，**不要开 `http2Origin`**（它只支持 HTTPS 源站）。

## 凭证

`.env`（600，不进 git）由 agent-vault `infra` vault 生成，键名与 `.env.example` 一一对应：
`ZITADEL_MASTERKEY`、`ZITADEL_LOGIN_COOKIE_SECRET`、`ZITADEL_DB_PASSWORD`、`ZITADEL_ADMIN_PASSWORD`、`RESEND_API_KEY`。
部署后多一个：`ZITADEL_ADMIN_PAT`（服务账号 `automation` 的 IAM_OWNER PAT，2027-10-20 到期，给 agent 走 API 用）。

⚠️ **`ZITADEL_MASTERKEY` 丢了实例就不可恢复**（库里的密钥都靠它解）。vault 之外在 Bitwarden 另存一份。
备份 = `pg_dump zitadel` + masterkey，缺一不可。

## 常用命令（在 tencent 上，`/opt/docker/zitadel`）

```bash
sudo docker compose ps
sudo docker compose logs --tail 50 zitadel-api
sudo docker compose up -d --wait          # 改了 compose 或 .env 后
# 升级：改 .env 的 ZITADEL_VERSION → pull → up -d --wait（api 启动时自动跑迁移）
```

改 `caddy/Caddyfile` 后要 `sudo docker compose restart proxy`（已关 admin API，不会热加载）。

## 坑

- **cloudflared → 源站只能 HTTP/1.1 明文或 HTTPS**，而 api 只说 h2c，所以必须经 Caddy；浏览器控制台走 gRPC-web/Connect，HTTP/1.1 能承载；原生 gRPC 客户端经公网域名不可用（用 REST `/v2/...`）
- `ZITADEL_EXTERNALDOMAIN` 写进实例数据，**事后改域名很麻烦**
- 首个实例的管理员、SMTP、服务账号只在**第一次** `start-from-init` 时生效；之后改 `.env` 里这些值不会回写，要去控制台改
- 管理员初始密码须满足默认密码策略（大写、小写、数字、符号），否则首启建管理员失败
- 本机 docker 没有全局日志轮转，每个服务都配了 `max-size 20m × 3`
