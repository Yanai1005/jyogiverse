# 📘 ログ設計：jyogiverse

---

# 0️⃣ 設計前提

| 項目     | 内容                                                     |
| ------ | -------------------------------------------------------- |
| 対象システム | Workers API / アプリLXC（systemd）/ 管理LXC（Caddy / cloudflared）|
| Workers | JSON構造化ログ（`console.log`）→ Cloudflare Workers Logs        |
| インフラ   | systemd journal / Caddyアクセスログ                           |
| 保持期間   | Workers Logs: 7日（無料枠）/ systemd: 永続化設定を推奨              |
| 禁止事項   | Proxmox APIトークンのログ出力禁止                                  |

---

# 1️⃣ ログ分類

| 種別           | 対象                    | 確認方法                            |
| ------------ | ----------------------- | --------------------------------- |
| APIログ        | Workers（全操作）          | Cloudflare Workers Logs / wrangler tail |
| アクセスログ       | Caddy（リクエスト経路）         | `/var/log/caddy/{work_name}.log`  |
| アプリログ        | 各作品アプリ（stdout/stderr）  | `journalctl -u {work_name}.service` |
| Tunnelログ     | cloudflaredエージェント       | `journalctl -u cloudflared`       |
| 操作ログ（DB）     | deploy_logsテーブル        | D1クエリ / 管理UI（Phase 3以降）          |

---

# 2️⃣ Workers JSONログフォーマット

設計ドキュメント（2.9節）で定義したフォーマットに準拠する。

```json
{
  "timestamp": "2025-04-01T10:00:00.000Z",
  "level": "info",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "operation": "deploy.approve",
  "target": "work_request/req-uuid",
  "outcome": "success",
  "durationMs": 3420,
  "message": "Container deployed: ctid=201 subdomain=my-app.example.dev"
}
```

### ログレベル基準

| レベル     | 使用条件                    | 例                              |
| ------- | ------------------------ | ------------------------------ |
| `error` | 即座に人間が対応すべき事象           | デプロイ失敗・Proxmox API接続不能        |
| `warn`  | 注意が必要だが自動復旧する           | Caddy reload リトライ成功            |
| `info`  | 正常な操作ログ（全API操作）         | 申請作成・承認・コンテナ停止               |
| `debug` | 開発時のみ有効化               | Proxmox APIレスポンス詳細            |

### Workers実装例

```typescript
// middleware/logger.ts
export const loggerMiddleware = (): MiddlewareHandler => async (c, next) => {
  const start = Date.now()
  const requestId = crypto.randomUUID()
  c.set('requestId', requestId)

  await next()

  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level: c.res.status >= 500 ? 'error' : 'info',
    requestId,
    operation: `${c.req.method} ${c.req.path}`,
    outcome: c.res.status < 400 ? 'success' : 'failure',
    durationMs: Date.now() - start,
    statusCode: c.res.status,
  }))
}
```

---

# 3️⃣ アプリログ（systemd journal）

```bash
# リアルタイム確認
journalctl -u {work_name}.service -f

# 最新100行
journalctl -u {work_name}.service -n 100

# エラーのみ
journalctl -u {work_name}.service -p err

# 特定日時以降
journalctl -u {work_name}.service --since "2025-01-01 00:00:00"
```

### journaldの永続化設定（各LXC内）

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

---

# 4️⃣ Caddyアクセスログ設定

```caddy
{work_name}.example.dev {
    log {
        output file /var/log/caddy/{work_name}.access.log {
            roll_size 10mb
            roll_keep 5
        }
        format json
    }
    reverse_proxy 10.0.0.{n}:{port}
}
```

---

# 5️⃣ 障害通知（Discord）

systemdの `OnFailure=` でプロセス障害時にDiscordへ通知する。

```ini
# /etc/systemd/system/{work_name}.service
[Unit]
OnFailure=notify-discord@%n.service
```

```ini
# /etc/systemd/system/notify-discord@.service
[Unit]
Description=Discord通知: %i 障害

[Service]
Type=oneshot
ExecStart=/usr/local/bin/notify-discord.sh "%i"
```

```bash
# /usr/local/bin/notify-discord.sh
#!/bin/bash
SERVICE=$1
curl -s -H "Content-Type: application/json" \
  -d "{\"content\": \"⚠️ **${SERVICE}** がクラッシュしました。systemdが自動再起動します。\"}" \
  "${DISCORD_WEBHOOK_URL}"
```

---

# 6️⃣ 操作ログ（D1 deploy_logs）

管理UIでの承認・コンテナ操作はD1の `deploy_logs` テーブルに記録する。

```typescript
// services/deploy.ts 内での記録例
await db.prepare(`
  INSERT INTO deploy_logs (log_id, request_id, action, status, message, created_at)
  VALUES (?, ?, ?, ?, ?, ?)
`).bind(
  crypto.randomUUID(),
  requestId,
  'deploy',
  success ? 'success' : 'failure',
  message,
  new Date().toISOString()
).run()
```

---

# 7️⃣ Workers Logsのリアルタイム確認

```bash
# 開発・デバッグ時
wrangler tail --env production
```

Cloudflareダッシュボード → Workers & Pages → Workers API → Logs でも確認可能（7日間保持）。

---

# 8️⃣ フェーズ別導入計画

```text
Phase 1（インフラ基盤）:
- systemd journal でアプリログ確認（journalctl コマンド）
- journaldの永続化設定
- cloudflared・Caddyのシステムログ確認

Phase 2（スクリプト・IaC）:
- Caddyアクセスログを有効化
- Discord障害通知の設定

Phase 3（管理UI）:
- Workers JSONログの実装（loggerMiddleware）
- deploy_logsテーブルへの記録開始
- wrangler tail での運用監視

Phase 4以降:
- 管理UIからdeploy_logsを参照できる画面を追加
```
