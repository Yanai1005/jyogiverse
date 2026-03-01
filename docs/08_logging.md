# 📘 ログ設計：jyogiverse

---

# 0️⃣ 設計前提

| 項目     | 内容                                                         |
| ------ | ------------------------------------------------------------ |
| 対象システム | アプリLXC（systemd journal）/ 管理UI（Cloudflare Workers Logs）     |
| ログ方式   | systemd journal（インフラ）/ Workers console.log（管理UI）           |
| 集約方式   | 分散管理（Phase 1〜2）→ 管理UIにログ確認画面を追加（Phase 3以降）               |
| 保持期間   | systemd: デフォルト（再起動でリセットされないよう persistent 設定推奨）/ Workers: 7日 |
| 個人情報   | メールアドレスはログ出力時にマスキング                                        |

---

# 1️⃣ ログ分類

| 種別              | 対象                       | 確認方法                              | Phase |
| --------------- | ------------------------ | --------------------------------- | ----- |
| アプリログ           | 各作品アプリ（stdout / stderr）  | `journalctl -u {app}.service`     | P0    |
| Caddyアクセスログ     | リクエスト経路                  | `/var/log/caddy/{app}.log`        | P1    |
| Tunnel接続ログ      | cloudflaredエージェント        | `journalctl -u cloudflared`       | P0    |
| 操作ログ（Audit）     | 管理UIでの承認・コンテナ操作          | Workers Logs / D1 audit_logs      | P1    |
| システムログ          | Proxmox / LXC OS         | `/var/log/syslog`, Proxmox GUI    | P0    |

---

# 2️⃣ アプリログ（systemd journal）

```bash
# リアルタイム確認
journalctl -u {app_name}.service -f

# 最新100行
journalctl -u {app_name}.service -n 100

# 特定日時以降
journalctl -u {app_name}.service --since "2025-01-01 00:00:00"

# エラーのみ
journalctl -u {app_name}.service -p err
```

### journaldの永続化設定（各LXC内）

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

---

# 3️⃣ Caddyアクセスログ設定

```caddy
{app_name}.jyogiverse.dev {
    log {
        output file /var/log/caddy/{app_name}.access.log {
            roll_size 10mb
            roll_keep 5
        }
        format json
    }
    reverse_proxy 10.0.0.{n}:{port}
}
```

### ログフォーマット例

```json
{
  "ts": 1704067200.123,
  "request": {
    "method": "GET",
    "host": "myapp.jyogiverse.dev",
    "uri": "/api/data",
    "remote_addr": "xxx.xxx.xxx.xxx",
    "user_agent": "Mozilla/5.0 ..."
  },
  "status": 200,
  "size": 1234,
  "duration": 0.012
}
```

---

# 4️⃣ 操作ログ（Audit Log）設計

管理UIでの操作はCloudflare Workers Logsに出力し、Phase 2以降でD1の`audit_logs`テーブルにも記録する。

### Workers側の出力例（TypeScript）

```typescript
// 承認操作のログ
console.log(JSON.stringify({
  level: "INFO",
  action: "approve_application",
  application_id: id,
  app_name: application.app_name,
  admin_email: maskEmail(adminEmail),  // xxx@gmail.com → xxx@***
  timestamp: new Date().toISOString()
}))

// コンテナ操作のログ
console.log(JSON.stringify({
  level: "INFO",
  action: "container_stop",
  container_id: id,
  lxc_id: container.lxc_id,
  admin_email: maskEmail(adminEmail),
  timestamp: new Date().toISOString()
}))
```

### D1 audit_logsテーブル（Phase 2以降）

```sql
CREATE TABLE audit_logs (
    id TEXT PRIMARY KEY,
    admin_id TEXT NOT NULL,        -- 操作した管理者のmembers.id
    action TEXT NOT NULL,          -- "approve", "reject", "start", "stop", "delete"
    target_type TEXT NOT NULL,     -- "application", "container"
    target_id TEXT NOT NULL,       -- 操作対象のID
    detail TEXT,                   -- 補足情報（JSON文字列）
    created_at TEXT NOT NULL       -- ISO 8601
);
```

---

# 5️⃣ Workers Logsのリアルタイム確認

```bash
# wrangler tailでリアルタイム確認（開発・デバッグ時）
wrangler tail --env production
```

Cloudflareダッシュボード → Workers & Pages → {Worker名} → Logs からも確認可能（7日間保持）。

---

# 6️⃣ マスキングポリシー

| 対象          | 方針                            |
| ----------- | ------------------------------- |
| メールアドレス     | ローカル部を `xxx` に置換（`xxx@domain.com`） |
| GitHubトークン  | 絶対出力禁止                        |
| IPアドレス      | Caddyログのみ記録。Workers Logsには含めない |

---

# 7️⃣ アラート・監視方針

| 監視項目              | Phase 1（手動）              | Phase 3以降（自動化）      |
| ----------------- | ------------------------- | ------------------- |
| アプリプロセス停止         | systemdが自動再起動。失敗時はjournalに記録 | 管理UIのステータスバッジで即時確認  |
| Proxmoxサーバー死活     | 定期的に手動確認                  | Cloudflare監視の活用を検討  |
| ディスク残量            | 月1回程度手動確認                 | 管理UIに残量表示を追加        |
| Tunnel接続断         | cloudflaredの自動再接続に依存       | Workers Logsで確認     |

---

# 8️⃣ フェーズ別導入計画

```text
Phase 1（インフラ基盤）:
- systemd journal でアプリログ確認（journalctl コマンド）
- Workers console.log で管理UI操作確認
- Tunnel / Caddy のシステムログを手動確認

Phase 2（スクリプト整備）:
- Caddyアクセスログを有効化
- D1にaudit_logsテーブル追加
- Workers側でaudit_logsへの書込みを実装

Phase 3（管理UI）:
- 管理UIからジャーナルログを確認できる画面を追加
- Slack通知（承認・コンテナ障害）を実装

Phase 4以降:
- 集約ログ基盤の検討（ログ量が増えた場合）
```
