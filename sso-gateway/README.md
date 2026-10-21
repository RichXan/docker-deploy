# sso-gateway（SSO 网关）

给**自己不会讲 OIDC** 的服务加一道 Zitadel 登录（https://id.xanne.cloud），不改服务代码。
oauth2-proxy 替服务完成 OIDC 登录，Caddy 按域名分站点、用 `forward_auth` 问 oauth2-proxy「这个请求登录了吗」。
一个 cookie（`_xanne_sso`，域 `.xanne.cloud`）覆盖所有受保护站点：登录一次，处处可用。

## 受保护的站点

| 域名 | 上游 | 放行（不登录）的路径 | 为什么放行 |
|---|---|---|---|
| okx.xanne.cloud | 127.0.0.1:8801 | `/bark-okx.png` | Bark 推送图标要被手机直接拉；Gatus 存活探针 |
| market.xanne.cloud | 127.0.0.1:5173 | `/api/health` | Gatus 探针 |
| console.xanne.cloud | 127.0.0.1:13010 | `/healthz` | Gatus 探针（`/logout` 由网关接管） |

⚠️ okx 的 `/api/daemon-state` **不放行**（含策略参数、权益、持仓）。发版检查改在 sealan 上 `curl 127.0.0.1:8801/api/daemon-state`。

## 退出

- 任一站点：`https://<站点>/oauth2/sign_out`——清网关 cookie，并由 oauth2-proxy 服务端调 Zitadel `end_session` 结束会话，下次访问会看到登录框
- console 页面上的「退出登录」（`/logout`）由网关接管，效果同上，另外清掉 console 自己的 `pc_auth`
- 只清网关 cookie 而不结束 Zitadel 会话的话，再访问会被**静默登回去**（这就是 SSO），看起来像「退出没用」
- oauth2-proxy 每次退出都会记一行 `error while calling backend logout url, returned error code 404`：**这是正常的**。Zitadel 结束会话后把服务端请求转到登录页的退出页，没有浏览器上下文所以 404；会话在此之前已结束（2026-10-21 对照实验：退出前会话存在、退出后 `GET /v2/sessions/{id}` 404）

## 在别处退出也会生效（单点退出）

在 Zitadel 控制台点退出、管理员终止会话、删用户之后，**最多约 10 分钟**所有受保护站点都要求重新登录。靠三样东西配合：

1. oauth2-proxy 每 5 分钟（`COOKIE_REFRESH`）用 refresh token 向 Zitadel 续期，拿新的 ID Token
2. 会话存在 `sso-redis`（127.0.0.1:6380，不落盘）。**不能用 cookie 存会话**：forward_auth 模式下 oauth2-proxy 写回的新 cookie
   到不了浏览器，续期结果会丢，每个请求都拿旧 refresh token 去续（Zitadel 轮换 refresh token，必失败）——2026-10-21 踩过
3. Zitadel 实例的 **ID Token 有效期设为 10 分钟**（默认 12 小时）。会话结束后 Zitadel 续期返回 `invalid_request`，
   oauth2-proxy 不认它是「已吊销」（只认 `invalid_grant` / `invalid_client`），会退回检查 ID Token 是否过期——所以要短

Redis 丢了（重启）只是所有人重新登录一次。没有这套配置时，网关只认自己 7 天有效的 cookie，Zitadel 那边退出了这边照样能进（2026-10-21 实测过）。

## console 认网关身份

console 路由在认证通过后加 `X-Gateway-Secret`（`.env` 的 `CONSOLE_GATEWAY_SECRET`，与 praxis-console 的 `.env` 同值），
console 见到它和 `X-Auth-Request-Email` 就直接放行，经域名不再出现访问密钥页。所有站点都会先剥掉客户端自带的
`X-Gateway-Secret` / `X-Auth-Request-*`，外部伪造不了。轮换密钥两边 `.env` 一起改。

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

`.env`（600）四项，值在 agent-vault `infra`：`SSO_GATEWAY_CLIENT_ID`、`SSO_GATEWAY_CLIENT_SECRET`、`SSO_GATEWAY_COOKIE_SECRET`、`CONSOLE_GATEWAY_SECRET`。

## 回滚

把隧道里这几个域名改回直连原端口即可（网关容器可以留着不动）。原映射：okx → 8801，market → 5173，console → 13010。
