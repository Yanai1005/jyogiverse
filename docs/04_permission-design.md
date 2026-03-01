# 🔐 権限設計：jyogiverse

> **認証・ログイン機能は現フェーズのスコープ外。**
> Phase 3管理UIの初期実装では、アプリ内に認証ロジックを持たない。

---

# 0️⃣ 設計方針

| 項目      | 内容                                              |
| ------- | ------------------------------------------------ |
| 認証方式    | **アプリ内に認証ロジックなし**。`/admin` 配下はCloudflare Accessで外部保護。 |
| 権限モデル   | RBAC（将来フェーズで実装）。現フェーズはスコープ外。                    |
| 申請フォーム  | 誰でもアクセス可能（URLを知っているメンバーが使用）                    |
| 管理者画面   | Cloudflare Access（Google認証）で保護。アプリはCloudflareのヘッダーを信頼するだけ。 |

---

# 1️⃣ アクセス制御（現フェーズ）

| パス              | アクセス制御               | 備考                            |
| --------------- | ---------------------- | ------------------------------- |
| `/apply`        | 誰でもアクセス可              | サークルメンバーへURLを周知して使用           |
| `/apply/done`   | 誰でもアクセス可              |                                 |
| `/admin`        | Cloudflare Accessで保護   | Google認証。管理者メールアドレスを登録        |
| `/admin/*`      | Cloudflare Accessで保護   | 全管理画面を一括保護                    |
| `/api/requests` (GET/POST) | Cloudflare Access不要 | `POST` は申請フォームから呼ばれる |
| `/api/requests/:id/*` | Cloudflare Accessで保護 | 承認・却下APIは管理者のみ              |
| `/api/containers/*` | Cloudflare Accessで保護 | コンテナ操作APIは管理者のみ           |

---

# 2️⃣ Cloudflare Access設定方針

| 設定項目       | 内容                                    |
| ---------- | --------------------------------------- |
| 認証プロバイダー   | Google                 |
| 保護対象パス     | `/admin/*`、`/api/requests/:id/*`、`/api/containers/*` |
| 保護対象外パス    | `/apply`、`/apply/done`、`/api/requests`（POST） |
| 管理者特定      | Cloudflare Accessのポリシーで特定メールアドレスを許可    |

---

# 3️⃣ アプリ側の実装（Workers）

Cloudflare Accessを通過したリクエストにはヘッダーが付与される。アプリはそれを信頼するだけ。

```typescript
// Cloudflare Accessが付与するヘッダーを読むだけ
// アプリ内でトークン検証等は行わない
const adminEmail = c.req.header('Cf-Access-Authenticated-User-Email')
// → 管理者操作のログ記録にのみ使用
```

---

# 4️⃣ 将来フェーズでの認証実装計画

現フェーズが安定した後に以下を追加する。

| 項目         | 内容                                       |
| ---------- | ---------------------------------------- |
| メンバー認証     | Cloudflare AccessのOIDCトークンをアプリで検証        |
| RBACロール定義  | ADMIN / MEMBER の2ロールをD1のmembersテーブルで管理  |
| 申請フォーム認証   | メンバーのみ申請可能にする（現状は誰でも可）                  |
| 申請状況確認ページ  | `/status`（自分の申請のみ確認できるメンバー向けページ）        |
