# 📁 ディレクトリ構成：jyogiverse

---

# 0️⃣ 設計前提

| 項目      | 内容                                       |
| ------- | ---------------------------------------- |
| リポジトリ構成 | Monorepo（単一リポジトリ）                        |
| 構成要素    | Workers API + Pages/Remix + インフラスクリプト + IaC |
| 言語      | TypeScript（Workers / Remix）/ HCL（Terraform）/ Bash（スクリプト） |
| デプロイ単位  | Workers: wrangler / Pages: wrangler pages / インフラ: terraform apply |

---

# 1️⃣ 全体構成

```
jyogiverse/
├── src/
│   ├── workers/                  # Cloudflare Workers（Hono API）
│   │   ├── index.ts              # エントリーポイント・ルーティング
│   │   ├── routes/
│   │   │   ├── requests.ts       # 申請 CRUD（/api/requests）
│   │   │   └── containers.ts     # Proxmox API プロキシ（/api/containers）
│   │   ├── middleware/
│   │   │   └── logger.ts         # 構造化ログ出力
│   │   ├── services/
│   │   │   ├── proxmox.ts        # Proxmox API クライアント
│   │   │   └── deploy.ts         # デプロイロジック（clone→start→Caddy）
│   │   └── db/
│   │       └── schema.ts         # D1スキーマ定義・クエリ
│   └── pages/                    # Cloudflare Pages（Remix）
│       ├── app/
│       │   ├── routes/
│       │   │   ├── apply.tsx              # 申請フォーム
│       │   │   ├── apply.done.tsx         # 申請完了
│       │   │   ├── admin._index.tsx       # 管理者ダッシュボード
│       │   │   ├── admin.requests.tsx     # 申請一覧
│       │   │   ├── admin.requests.$id.tsx # 申請詳細・承認
│       │   │   ├── admin.containers.tsx   # コンテナ一覧
│       │   │   └── admin.containers.$id.tsx # コンテナ詳細・操作
│       │   ├── components/
│       │   │   ├── StatusBadge.tsx
│       │   │   ├── ConfirmModal.tsx
│       │   │   └── ResourceBar.tsx
│       │   └── lib/
│       │       └── api.ts        # WorkersへのAPIクライアント
│       └── public/
├── infra/                        # IaC（Terraform）
│   ├── versions.tf               # Provider・backend定義
│   ├── variables.tf              # 変数定義
│   ├── main.tf                   # Proxmox LXCリソース
│   ├── cloudflare.tf             # DNS・Tunnel・D1・Pages
│   ├── outputs.tf                # IPアドレス等の出力
│   └── works.auto.tfvars         # 作品一覧（ここを編集してapply）
├── scripts/                      # 運用スクリプト（Phase 2で整備）
│   ├── create-container.sh       # コンテナ作成（pct clone〜Caddy更新）
│   ├── deploy-app.sh             # アプリデプロイ（git clone〜systemd起動）
│   ├── delete-container.sh       # コンテナ削除（systemd停止〜pct destroy）
│   └── health-check.sh           # 全コンテナ死活確認
├── templates/                    # LXCテンプレート設定（言語別）
│   ├── nodejs/
│   │   ├── setup.sh              # Node.js 22インストール
│   │   └── app.service.template  # systemdサービスファイルテンプレート
│   ├── python/
│   │   ├── setup.sh              # Python 3.12 + venvインストール
│   │   └── app.service.template
│   └── go/
│       ├── setup.sh              # Go 1.22インストール
│       └── app.service.template
├── .github/
│   └── workflows/
│       ├── quality.yml           # Lint / フォーマット / 型チェック（PR時）
│       ├── deploy.yml            # Workers + Pages デプロイ（mainマージ時）
│       └── iac.yml               # terraform plan（PR）/ terraform apply（mainマージ）
├── wrangler.toml                 # Workers設定
├── docs/
│   ├── 00_designdoc.md
│   ├── 01_feature-list.md
│   ├── 02_stack.md
│   ├── 03_screen-flow.md
│   ├── 04_permission-design.md
│   ├── 05_erd.md
│   ├── 06_directory.md
│   ├── 07_infrastructure.md
│   ├── 08_logging.md
│   ├── 09_schedule_and_issues.md
│   └── neoshowcase_reference.md
└── README.md
```

---

# 2️⃣ Workers APIレイヤー責務

| レイヤー          | 責務                        | 依存方向      |
| ------------- | ------------------------- | --------- |
| `routes/`     | HTTPリクエスト受付・レスポンス整形       | → services |
| `middleware/` | ログ横断処理                    | → db      |
| `services/`   | ビジネスロジック・外部API呼び出し        | → db      |
| `db/`         | D1クエリのみ。ビジネスロジックを持たない     | なし        |

---

# 3️⃣ IaC構成（infra/）

| ファイル                 | 内容                            |
| -------------------- | ------------------------------- |
| `versions.tf`        | Provider（bpg/proxmox, cloudflare）とR2 backendを定義 |
| `main.tf`            | `works` 変数をループしてLXCコンテナを作成     |
| `cloudflare.tf`      | DNS CNAME・D1・Pages・Tunnelを管理   |
| `works.auto.tfvars`  | 作品一覧（言語・メモリ・CPU）。ここだけ編集してapply |
| `outputs.tf`         | 各コンテナのIPアドレス等を出力              |

---

# 4️⃣ APIルート一覧

| メソッド     | パス                            | 説明                  |
| -------- | ------------------------------- | --------------------- |
| GET      | `/api/requests`                 | 申請一覧（全件）             |
| POST     | `/api/requests`                 | 新規申請                  |
| GET      | `/api/requests/:id`             | 申請詳細                  |
| PATCH    | `/api/requests/:id/approve`     | 承認＆デプロイ実行            |
| PATCH    | `/api/requests/:id/reject`      | 却下                    |
| GET      | `/api/containers`               | 稼働コンテナ一覧             |
| GET      | `/api/containers/:id`           | コンテナ詳細・リソース使用率       |
| POST     | `/api/containers/:id/start`     | コンテナ起動               |
| POST     | `/api/containers/:id/stop`      | コンテナ停止               |
| DELETE   | `/api/containers/:id`           | コンテナ削除（D1レコードも削除）    |
