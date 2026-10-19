# gatus（praxis 巡检）

17 个对外域名的存活探测 + 4 项业务心跳，异常只推飞书「praxis 巡检」群。面板 https://status.xanne.cloud （admin 登录）。
设计与取舍见 `praxis-loop/servers` 仓库 `docs/specs/2026-10-18-gatus-patrol-design.md`，服务登记见该仓库 `SERVICES.md`。

## 文件

| 文件 | 内容 |
|---|---|
| `docker-compose.yaml` | 单容器，只绑 `127.0.0.1:13020`，SQLite 存在命名卷 `gatus-data` |
| `config/config.yaml` | 全部监控定义。**改完自动重载**，不用重启（挂的是目录，见 compose 注释） |
| `.env`（600，不进 git） | 键见 `.env.example` |

`.env` 各键：

- `GATUS_ADMIN_PASSWORD_BCRYPT_B64`：admin 密码的 bcrypt（cost 9）再 base64。**明文只在 Bitwarden「status.xanne.cloud」**，agent-vault 里没有
- `FEISHU_WEBHOOK_PRAXIS_PATROL`：与 agent-vault `comm` 同名键同值
- `GATUS_PUSH_TOKEN_*`：心跳 token，部署时 `openssl rand -hex 24` 现生成，同值另存在各上报方的 `.env`，不进 vault

## 心跳

上报：`POST /api/v1/endpoints/<key>/external?success=true|false&error=...`，头 `Authorization: Bearer <token>`。

| key | 上报方 | token |
|---|---|---|
| `biz_okx-watchdog` / `biz_okx-checks` | `/home/xan/paramita/okx-trade-monitor`（cron 每 15 分钟） | `GATUS_PUSH_TOKEN_OKX_WATCHDOG` |
| `biz_qd-checkin` | `/opt/docker-deploy/qd/tools/patrol.py`（cron 每天 10:00） | `GATUS_PUSH_TOKEN_QD_CHECKIN` |
| `biz_daily-log` | 笔记本 WSL 上的 `run-daily-log.sh`，走公网 status.xanne.cloud | `GATUS_PUSH_TOKEN_DAILY_LOG` |

## 常用

```bash
cd /opt/docker-deploy/gatus
docker compose up -d
docker logs gatus --tail 50          # 每轮评估一行，失败项带 success=false
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:13020/health
```

## 告警卡片

告警以飞书交互卡片发出：标题栏红色「🚨 巡检告警」/ 绿色「✅ 巡检恢复」，正文有分组、服务、说明、错误、（存活探测才有的）网站与检查条件，底部一个「打开巡检面板」按钮直达该项。`biz` 分组（心跳）用 `overrides` 里的精简模板，只显示上报内容。

- 标题栏的颜色与标题靠 `[ALERT_TRIGGERED_OR_RESOLVED]` 补全：它的值里带了一段 JSON（`red","title":{...`），因为 Gatus 只有这一个随状态变化的占位符。
- **改模板必须先校验**：把所有占位符换成样例值后整体 `json.loads`，再直接 POST 一次到飞书看回包 `code` 是不是 0（拒收也是 HTTP 200）。
- 恢复卡片里「错误」一栏为空、检查条件全是 ✅，属正常。

## 坑

- ⚠️ **告警正文必须含「巡检」**：飞书机器人设了自定义关键词，缺了会被拒收，而飞书拒收也回 HTTP 200，Gatus 看不出来。
- ⚠️ **上报的 error 要去掉控制字符和反斜杠**：custom 告警只转义 `"`，其余原样拼进飞书 JSON，坏了同样静默丢失。
- ⚠️ **心跳「完全没跑」最坏近 2 × interval 才发现**：Gatus 的心跳定时器不随上报重置。「跑了但失败」是即时的。
- ⚠️ **改 `config.yaml` 触发的重载会重置告警状态**（2026-10-18 实测）：重载后仍在故障中的项会**再发一次「🚨 巡检告警」**，重载那一刻到达的心跳上报可能丢失、不触发「✅ 恢复」。改配置挑没有未恢复告警的时候，改完看一眼群消息。
- 心跳超时后，日志里每隔一轮会出现 `Checked heartbeat ... success=true`：那是 Gatus 把自己刚写的失败结果当成了「区间内有记录」而跳过本轮，**不会写成功结果、也不会发恢复**，不是误报。只有真正的上报才会恢复。
- ⚠️ **不要开 `metrics`**：`/metrics` 不受登录保护。`uptimes` / `response-times` / 徽章接口本来就不受保护，只能读单项数字。
- 自身由 tencent 上的 `~/gatus-watch` 盯着（连续 3 次 `/health` 不通推同一个群）。
