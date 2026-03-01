# 🖥️ 画面フロー設計：jyogiverse 管理UI

---

# 0️⃣ 設計前提

| 項目     | 内容                                               |
| ------ | ------------------------------------------------ |
| 対象ユーザー | サークルメンバー（申請者）/ 管理者                          |
| デバイス   | Desktop優先                                        |
| 認証要否   | `/apply` は誰でも可 / `/admin` はCloudflare Accessで外部保護 |
| 権限制御   | アプリ内に認証ロジックなし。Cloudflare Accessに委譲               |
| MVP範囲  | Phase 3 P0画面のみ                                   |

---

# 1️⃣ 画面一覧（Screen Inventory）

| ID   | 画面名          | URL                      | 主要要素                          | 認証    | 優先度 |
| ---- | ------------ | ------------------------ | --------------------------------- | ----- | --- |
| S-01 | 申請フォーム       | `/apply`                 | 作品名・リポジトリURL・言語・申請者名・説明 | 不要    | P0  |
| S-02 | 申請完了         | `/apply/done`            | 申請番号・ステータス確認メッセージ     | 不要    | P0  |
| S-03 | 管理者ダッシュボード   | `/admin`                 | 未承認申請数・稼働コンテナ数          | 管理者   | P0  |
| S-04 | 申請一覧         | `/admin/requests`        | 申請一覧テーブル（フィルタ/ソート付き）  | 管理者   | P0  |
| S-05 | 申請詳細         | `/admin/requests/:id`    | 申請内容・承認/却下ボタン            | 管理者   | P0  |
| S-06 | コンテナ一覧       | `/admin/containers`      | 稼働中コンテナ一覧・リソース使用率      | 管理者   | P0  |
| S-07 | コンテナ詳細       | `/admin/containers/:id`  | コンテナ情報・起動/停止/削除ボタン     | 管理者   | P0  |

---

# 2️⃣ 全体遷移図

```mermaid
stateDiagram-v2
    [*] --> 申請フォーム

    申請フォーム --> 申請完了 : 申請送信成功
    申請フォーム --> 申請フォーム : バリデーションエラー

    管理者ダッシュボード --> 申請一覧 : 「申請管理」クリック
    管理者ダッシュボード --> コンテナ一覧 : 「コンテナ管理」クリック

    申請一覧 --> 申請詳細 : 申請行クリック
    申請詳細 --> 申請一覧 : 承認 / 却下実行後

    コンテナ一覧 --> コンテナ詳細 : コンテナ行クリック
    コンテナ詳細 --> コンテナ一覧 : 停止 / 削除実行後
```

---

# 3️⃣ 申請フロー（メンバー視点）

```mermaid
flowchart TD
    A([メンバーが申請フォーム送信]) --> B{フロント\nバリデーション}
    B -- NG --> ERR1[エラー表示・再入力促す]
    B -- OK --> C[POST /api/requests]
    C --> D{サーバー\nバリデーション}
    D -- NG --> ERR2[400: validation_error 返却]
    D -- OK --> E{work_name\n重複チェック}
    E -- 重複あり --> ERR3[409: conflict 返却]
    E -- 重複なし --> F[D1: work_requests に pending で INSERT]
    F --> G[201: request_id 返却]
    G --> H([申請完了ページへ遷移])
```

---

# 4️⃣ 承認・デプロイフロー（管理者視点）

```mermaid
flowchart TD
    A([管理者が承認ボタンクリック]) --> B{申請が pending か？}
    B -- No --> ERR1[409: 状態不整合エラー]
    B -- Yes --> C[D1: ステータスを approved に更新]
    C --> D[Proxmox API: pct clone でコンテナ作成]
    D --> E{作成成功？}
    E -- No --> F[D1: pending に戻す]
    F --> ERR2[deploy_failed エラー返却]
    E -- Yes --> G[リソース上限設定\nCPU / メモリ]
    G --> H[Proxmox API: pct start]
    H --> I{起動成功？}
    I -- No --> J[コンテナ削除してロールバック]
    J --> F
    I -- Yes --> K[Caddy設定にサブドメインルート追加]
    K --> L[caddy reload]
    L --> M{reload成功？}
    M -- No --> N[Caddy設定を元に戻す\nコンテナ停止・削除]
    N --> F
    M -- Yes --> O[D1: ステータスを deployed に更新]
    O --> P[deploy_logs に成功ログ記録]
    P --> Q([200: subdomain 返却])
```

---

# 5️⃣ コンテナ操作フロー（管理者視点）

```mermaid
flowchart LR
    List[コンテナ一覧] --> Detail[コンテナ詳細]
    Detail --> Action{操作選択}
    Action -->|起動| Start[POST /api/containers/:id/start]
    Action -->|停止| Stop[POST /api/containers/:id/stop]
    Action -->|削除| Confirm[確認モーダル]
    Confirm -->|確定| Delete[DELETE /api/containers/:id\nD1レコードも削除]
    Confirm -->|キャンセル| Detail
```

---

# 6️⃣ URL設計

```
/apply                     # 申請フォーム（誰でもアクセス可）
/apply/done                # 申請完了
/admin                     # 管理者ダッシュボード（Cloudflare Accessで保護）
/admin/requests            # 申請一覧
/admin/requests/:id        # 申請詳細・承認/却下
/admin/containers          # コンテナ一覧
/admin/containers/:id      # コンテナ詳細・操作
```

---

# 7️⃣ 申請フォームのバリデーション仕様

| フィールド          | 必須 | 制約                                  | エラーメッセージ例              |
| -------------- | -- | ------------------------------------- | ----------------------- |
| `applicant_name` | ✅ | 1〜50字                                | 「申請者名を入力してください」         |
| `work_name`    | ✅  | 英小文字・数字・ハイフン、3〜30字、重複不可           | 「使用できない文字が含まれています」      |
| `repo_url`     | ✅  | `https://github.com/` で始まるURL        | 「GitHubリポジトリのURLを入力してください」|
| `language`     | ✅  | `nodejs` / `python` / `go` のいずれか     | 「言語を選択してください」           |
| `description`  | ❌  | 最大500字                               | 「500字以内で入力してください」        |
