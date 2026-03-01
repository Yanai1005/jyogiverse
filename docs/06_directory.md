# 📁 ディレクトリ構成：jyogiverse

---

# 0️⃣ 設計前提

| 項目      | 内容                                         |
| ------- | ------------------------------------------ |
| リポジトリ構成 | Monorepo（単一リポジトリ）                          |
| 構成要素    | インフラスクリプト + 管理UI（Workers + Pages）          |
| 言語      | Bash（インフラスクリプト）/ TypeScript（管理UI）          |
| デプロイ単位  | インフラ（手動SSH）/ 管理UI（Cloudflare自動デプロイ）       |

---

# 1️⃣ 全体構成

```
jyogiverse/
├── infra/                        # インフラ管理スクリプト群
│   ├── templates/                # LXCテンプレート設定（言語別）
│   │   ├── nodejs/
│   │   ├── python/
│   │   └── go/
│   ├── scripts/                  # 運用スクリプト
│   │   ├── create-container.sh
│   │   ├── deploy-app.sh
│   │   ├── delete-container.sh
│   │   ├── update-caddy.sh
│   │   └── health-check.sh
│   ├── caddy/                    # Caddy設定
│   │   └── Caddyfile
│   └── cloudflare/               # Cloudflare Tunnel設定
│       └── config.yaml
├── management-ui/                # 管理WebUI
│   ├── api/                      # Cloudflare Workers（Hono）
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── routes/
│   │   │   │   ├── applications.ts
│   │   │   │   ├── containers.ts
│   │   │   │   └── members.ts
│   │   │   ├── db/
│   │   │   │   └── schema.sql
│   │   │   └── middleware/
│   │   │       └── auth.ts
│   │   ├── wrangler.toml
│   │   └── package.json
│   └── web/                      # Cloudflare Pages（React）
│       ├── src/
│       │   ├── pages/
│       │   │   ├── index.tsx         # 作品一覧（公開）
│       │   │   ├── apply.tsx         # 申請フォーム
│       │   │   ├── status.tsx        # 申請状況確認
│       │   │   └── admin/
│       │   │       ├── index.tsx     # 管理者ダッシュボード
│       │   │       ├── applications/
│       │   │       │   ├── index.tsx # 申請一覧
│       │   │       │   └── [id].tsx  # 申請詳細・承認
│       │   │       └── containers/
│       │   │           ├── index.tsx # コンテナ一覧
│       │   │           └── [id].tsx  # コンテナ詳細・操作
│       │   ├── components/
│       │   │   ├── AppCard.tsx
│       │   │   ├── StatusBadge.tsx
│       │   │   └── ConfirmModal.tsx
│       │   └── lib/
│       │       └── api.ts            # WorkersへのAPIクライアント
│       └── package.json
├── docs/                         # 設計書
│   ├── 00_designdoc.md
│   ├── 01_feature-list.md
│   ├── 02_stack.md
│   ├── 03_screen-flow.md
│   ├── 04_permission-design.md
│   ├── 05_erd.md
│   ├── 06_directory.md
│   ├── 07_infrastructure.md
│   ├── 08_logging.md
│   └── 09_schedule_and_issues.md
└── README.md
```

---

# 2️⃣ インフラスクリプト詳細（infra/scripts/）

```
scripts/
├── create-container.sh     # テンプレートをpct cloneしてコンテナ作成・初期セットアップ
├── deploy-app.sh           # GitHubからコードをpull → 依存関係インストール → systemd起動
├── delete-container.sh     # systemd停止 → pct destroy → Caddyルート削除
├── update-caddy.sh         # Caddyfileに新規サブドメインを追加してcaddy reload
└── health-check.sh         # 全コンテナの稼働確認（pct list + curl）
```

### create-container.sh の処理フロー

```bash
#!/bin/bash
# 引数: APP_NAME LANGUAGE PORT
APP_NAME=$1
LANGUAGE=$2
PORT=$3
LXC_ID=$(get_next_lxc_id)  # 次の空きCTIDを取得

pct clone $TEMPLATE_ID $LXC_ID --hostname $APP_NAME
pct set $LXC_ID --memory 256 --cores 1
pct start $LXC_ID
bash infra/templates/$LANGUAGE/setup.sh $LXC_ID $APP_NAME $PORT
bash infra/scripts/update-caddy.sh $APP_NAME $LXC_ID $PORT
```

---

# 3️⃣ LXCテンプレート構成（例：nodejs）

```
templates/nodejs/
├── setup.sh                  # Node.js 22インストール・アプリセットアップ
├── app.service.template      # systemdサービスファイルテンプレート
└── README.md                 # テンプレートの使い方
```

```ini
# app.service.template
[Unit]
Description={{APP_NAME}}
After=network.target

[Service]
Type=simple
User=app
WorkingDirectory=/opt/{{APP_NAME}}
ExecStart=node server.js
Restart=always
RestartSec=5
MemoryLimit=200M

[Install]
WantedBy=multi-user.target
```

---

# 4️⃣ Caddy設定構成

```
caddy/
├── Caddyfile                 # 全サブドメインのルーティング定義
└── snippets/
    └── common.caddy          # 共通ヘッダー設定（CORS等）
```

---

# 5️⃣ 管理UI APIルート設計（api/src/routes/）

| ファイル              | エンドポイント                        | 説明            |
| ----------------- | -------------------------------- | ------------- |
| applications.ts   | GET /applications                | 申請一覧取得        |
|                   | POST /applications               | 申請送信          |
|                   | GET /applications/:id            | 申請詳細取得        |
|                   | PATCH /applications/:id          | 承認 / 却下       |
| containers.ts     | GET /containers                  | コンテナ一覧取得      |
|                   | POST /containers/:id/action      | 起動/停止/再起動/削除  |
| members.ts        | GET /members                     | メンバー一覧（管理者）   |
|                   | POST /members                    | メンバー追加        |
