# sso-gateway（SSO 网关）

给**自己不会讲 OIDC** 的服务加一道 Zitadel 登录（https://id.xanne.cloud），不改服务代码。
oauth2-proxy 替服务完成 OIDC 登录，Caddy 按域名分站点、用 `forward_auth` 问 oauth2-proxy「这个请求登录了吗」。
一个 cookie（`_xanne_sso`，域 `.xanne.cloud`）覆盖所有受保护站点：登录一次，处处可用。

## 受保护的站点

| 域名 | 上游 | 放行（不登录）的路径 | 为什么放行 |
|---|---|---|---|
| okx.xanne.cloud | 127.0.0.1:8801 | `/bark-okx.png` | Bark 推送图标要被手机直接拉；Gatus 存活探针 |
| market.xanne.cloud | 127.0.0.1:5173 | `/api/health` | Gatus 探针 |
| console.xanne.cloud | 127.0.0.1:13010 | `/healthz` | Gatus 探针 |

⚠️ okx 的 `/api/daemon-state` **不放行**（含策略参数、权益、持仓）。发版检查改在 sealan 上 `curl 127.0.0.1:8801/api/daemon-state`。

## 谁能进

Zitadel 组织 `xanne` 的项目 **`praxis-gateway`** 开了「登录时校验授权」：用户必须被授予角色 **`access`** 才能通过。
放人进来 = 在 Zitadel 控制台给他这个项目的 `access` 授权；收回 = 删授权（已有会话最长 168h 后失效）。
实例的自助注册已关闭（2026-10-21）。

## 加一个新站点

1. `caddy/Caddyfile` 照抄一段站点块，改域名、上游端口、放行路径
2. Zitadel：项目 `praxis-gateway` → 应用 `oauth2-proxy` → 加回调 `https://<域名>/oauth2/callback`
3. sealan 隧道 ingress：该域名改指 `http://localhost:13030`（全量 PUT，先备份、改后逐个域名回归）
4. `sudo docker compose restart caddy`（admin API 已关，不会热加载）

## 凭证

`.env`（600）三项，值在 agent-vault `infra`：`SSO_GATEWAY_CLIENT_ID`、`SSO_GATEWAY_CLIENT_SECRET`、`SSO_GATEWAY_COOKIE_SECRET`。

## 回滚

把隧道里这几个域名改回直连原端口即可（网关容器可以留着不动）。原映射：okx → 8801，market → 5173，console → 13010。
