# 🖥️ 画面フロー設計：jyogiverse 管理UI

---

# 0️⃣ 設計前提

| 項目     | 内容                                          |
| ------ | ------------------------------------------- |
| 対象ユーザー | サークルメンバー（申請者）/ 管理者（柳井）/ 一般閲覧者              |
| デバイス   | Desktop優先（Responsive対応）                     |
| 認証要否   | 公開ページあり / 管理画面はCloudflare Access（Google認証）  |
| 権限制御   | RBAC（ADMIN / MEMBER / PUBLIC）                |
| MVP範囲  | Phase 3 P0画面のみ（S-02〜S-05）                   |

---

# 1️⃣ 画面一覧（Screen Inventory）

| ID   | 画面名            | 役割                   | 認証      | 優先度 | Phase |
| ---- | -------------- | -------------------- | ------- | --- | ----- |
| S-01 | 作品一覧（公開）       | 公開中の作品をカード形式で一覧表示    | 不要      | P1  | 3     |
| S-02 | 申請フォーム         | メンバーがホスティング申請を送信     | 不要      | P0  | 3     |
| S-03 | 管理者ダッシュボード     | 申請・コンテナ状況の概要を確認      | 管理者     | P0  | 3     |
| S-04 | 申請一覧（管理者）      | 未承認・全申請の一覧確認         | 管理者     | P0  | 3     |
| S-05 | 申請詳細・承認操作      | 申請内容確認・承認 / 却下       | 管理者     | P0  | 3     |
| S-06 | コンテナ一覧（管理者）    | 稼働中コンテナ一覧と状態確認       | 管理者     | P1  | 3     |
| S-07 | コンテナ詳細         | 個別コンテナの詳細確認・起動/停止操作  | 管理者     | P1  | 3     |
| S-08 | 申請状況確認（メンバー）   | 自分の申請の進捗・結果確認        | メンバー    | P1  | 3     |

---

# 2️⃣ 全体遷移図

```mermaid
flowchart TD
    PUBLIC[作品一覧 公開]
    FORM[申請フォーム]
    DASH[管理者ダッシュボード]
    APP_LIST[申請一覧]
    APP_DETAIL[申請詳細・承認]
    CONT_LIST[コンテナ一覧]
    CONT_DETAIL[コンテナ詳細]
    STATUS[申請状況 メンバー]

    PUBLIC --> FORM
    FORM --> STATUS
    DASH --> APP_LIST
    DASH --> CONT_LIST
    APP_LIST --> APP_DETAIL
    APP_DETAIL -->|承認| CONT_LIST
    CONT_LIST --> CONT_DETAIL
```

---

# 3️⃣ 申請フロー（メンバー視点）

```mermaid
flowchart LR
    Start[申請フォーム入力]
    Submit[送信]
    Waiting[審査待ち]
    Approved[承認・デプロイ]
    Rejected[却下・理由通知]

    Start --> Submit
    Submit --> Waiting
    Waiting -->|管理者承認| Approved
    Waiting -->|管理者却下| Rejected
    Approved --> Done[URLで即日公開]
```

---

# 4️⃣ 承認・デプロイフロー（管理者視点）

```mermaid
flowchart TD
    List[申請一覧] --> Detail[申請詳細確認]
    Detail --> Decision{承認？}
    Decision -->|承認| Clone[pct clone でコンテナ作成]
    Clone --> Setup[コード取得・依存関係インストール]
    Setup --> Deploy[systemdサービス登録・起動]
    Deploy --> Caddy[Caddyfileにサブドメイン追加]
    Caddy --> Notify[URLをメンバーに通知]
    Decision -->|却下| Reason[却下理由を入力]
    Reason --> RejectNotify[却下通知]
```

---

# 5️⃣ コンテナ操作フロー（管理者視点）

```mermaid
flowchart LR
    List[コンテナ一覧] --> Detail[コンテナ詳細]
    Detail --> Action{操作選択}
    Action -->|起動| Start[pct start]
    Action -->|停止| Stop[pct stop]
    Action -->|再起動| Restart[pct reboot]
    Action -->|削除| Confirm[確認モーダル]
    Confirm -->|確定| Delete[pct destroy + Caddyルート削除]
    Confirm -->|キャンセル| Detail
```

---

# 6️⃣ 認証・権限フロー

```mermaid
flowchart TD
    Request[リクエスト] --> PublicCheck{公開パス?}
    PublicCheck -->|Yes| Show[そのまま表示]
    PublicCheck -->|No| CFAccess[Cloudflare Access認証]
    CFAccess --> RoleCheck{ロール確認}
    RoleCheck -->|ADMIN| AdminView[全画面アクセス可]
    RoleCheck -->|MEMBER| MemberView[申請・状況確認のみ]
    RoleCheck -->|未登録| Deny[403 アクセス拒否]
```

---

# 7️⃣ URL設計

```
/                          # 作品一覧（公開）
/apply                     # 申請フォーム（公開）
/status                    # 申請状況確認（メンバー認証）
/admin                     # 管理者ダッシュボード
/admin/applications        # 申請一覧
/admin/applications/:id    # 申請詳細・承認操作
/admin/containers          # コンテナ一覧
/admin/containers/:id      # コンテナ詳細・操作
```
