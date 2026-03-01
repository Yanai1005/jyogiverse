# 技術スタック：jyogiverse

---

## 0️⃣ 前提定義

| 項目        | 内容                                        |
| --------- | ----------------------------------------- |
| プロダクト種別   | サークル作品ホスティングサービス（非商用・個人運用）                |
| 想定ユーザー規模  | 同時10作品以上・管理者1名（柳井）                        |
| 可用性目標     | 月間99%以上（外形監視）                             |
| セキュリティ要件  | 個人情報なし・管理者画面はCloudflare Accessで保護         |
| パフォーマンス要件 | API p95 < 500ms                           |
| チーム体制     | 1人（フルスタック担当）                              |
| リリース頻度    | 不定期（作品追加・機能追加時）                           |
| 予算制約      | 月額0円（電気代・ドメイン代除く）を厳守                      |
| 既存資産      | Lenovo System x3550 M5（自宅サーバー、32GB RAM）   |

---

# 1️⃣ 技術スタック構成

---

## 🖥 フロントエンド

| 項目      | 採用技術                          | 採用理由                           | 評価観点                     |
| ------- | ----------------------------- | ------------------------------ | ------------------------ |
| 言語      | TypeScript                    | 型安全。Workers と共通化できる            | 型安全性 / 保守性               |
| フレームワーク | Remix（Cloudflare Pages）       | SSR対応・loader/actionでServer State管理。無料枠で自動デプロイ | エコシステム / SSR可否 / コスト |
| UIライブラリ | なし（素のCSS）                     | 管理画面のみ。依存最小化                   | 開発速度 / バンドルサイズ           |
| 状態管理    | Remix loader / action         | Server State に寄せてクライアント状態を最小化   | スケール耐性 / Server State分離  |
| フォーム管理  | Remix 標準（Form / fetcher）      | ライブラリ追加不要                      | パフォーマンス / バリデーション        |
| テスト     | 未定（Phase 3以降で検討）             | -                              | カバレッジ / 実行速度             |

---

## 🧠 バックエンド

| 項目           | 採用技術                         | 採用理由                             | 評価観点             |
| ------------ | ---------------------------- | -------------------------------- | ---------------- |
| 言語           | TypeScript                   | 型安全。フロントと共通化できる                  | 生産性 / 型安全 / 採用市場 |
| 実行環境         | Cloudflare Workers           | 無料枠で常時稼働。エッジ実行でレイテンシ低            | 成熟度 / コスト        |
| フレームワーク      | Hono                         | Workers特化。軽量・高速・TypeScript親和性高    | 構造化 / 拡張性        |
| API方式        | REST                         | シンプルな CRUD 操作のみ。GraphQL不要         | 柔軟性 / 過不足        |
| ORM / DBアクセス | D1 直接クエリ（Prepared Statement） | ORM不要な規模。Workers統合でレイテンシ最小        | 型統合 / シンプルさ      |
| 認証           | Cloudflare Access（外部）        | アプリ内に認証ロジックなし。Google OAuth対応済み    | セキュリティ / OAuth対応 |
| 非同期処理        | なし（同期API）                    | 現フェーズは不要                         | -                |

---

## 🗄 データベース

| 項目    | 採用技術                    | 採用理由                      | 評価観点          |
| ----- | ----------------------- | ------------------------- | ------------- |
| メインDB | Cloudflare D1（SQLite互換） | Workers統合。無料枠で運用量は十分。UUID主キー | ACID / コスト   |
| キャッシュ | なし                      | 規模が小さく不要                  | -             |
| 検索    | なし                      | 全文検索不要                    | -             |
| 分析基盤  | なし                      | 現フェーズ対象外                  | -             |

---

## ☁ インフラ / DevOps

| 項目     | 採用技術                                   | 採用理由                                       | 評価観点          |
| ------ | -------------------------------------- | ------------------------------------------ | ------------- |
| ホスティング | Proxmox VE 8.x（自宅サーバー）                 | LXCを公式サポート。クラウドコストゼロ                       | 可用性 / コスト     |
| コンテナ   | LXC（Debian 12）                         | Docker不要でオーバーヘッド最小。Proxmox公式サポート範囲内         | 再現性 / 可搬性     |
| IaC    | Terraform（bpg/proxmox + cloudflare）   | LXC・DNS・D1・Pagesをコード管理。手動設定ミスを防ぐ           | 構成管理 / 再現性    |
| CI/CD  | GitHub Actions                         | PR時Lint・型チェック。mainマージ時にwrangler/terraform自動実行 | 自動化 / 安定性     |
| 監視     | systemd OnFailure= + Discord Webhook   | プロセス死活を自動検知・通知。追加ツール不要                     | 可観測性 / アラート精度 |
| ログ管理   | Workers Logs（JSON構造化）/ systemd journal | Workers側はwrangler tailで確認。アプリ側はjournalctl    | トレーサビリティ      |
| CDN    | Cloudflare（Tunnel経由）                   | ポート開放ゼロ。DDoS対策・TLS終端込み。無料                  | レイテンシ最適化      |

---

## 🔐 セキュリティ

| 項目    | 方針                                        | 評価観点          |
| ----- | ----------------------------------------- | ------------- |
| 認証方式  | Cloudflare Access（Google OAuth）。アプリ内に認証ロジックなし | 標準準拠 / 拡張性  |
| 認可    | 現フェーズは管理者1名のみ。将来的にRBAC（ADMIN / MEMBER）     | RBAC可否        |
| 通信    | Cloudflare Tunnel経由でTLS強制。ポート開放ゼロ          | TLS強制 / 証明書管理 |
| データ保護 | 個人情報なし。Proxmox APIトークンはWorkers環境変数で管理      | 暗号化 / マスキング   |
| 脆弱性対策 | Cloudflare WAF（Tunnel経由で自動適用）             | 自動検査          |

---

## 🛠 CI/CDツールチェーン

| ツール                                             | 役割              | 実行タイミング |
| ----------------------------------------------- | --------------- | :-------: |
| `oxlint`                                        | Lint            | PR        |
| `oxc formatter`                                 | フォーマットチェック      | PR        |
| `@typescript/native-preview`（tsgo）              | 型チェック           | PR        |
| `stylelint` + `stylelint-config-standard-scss`  | SCSS Lint       | PR        |
| `knip`                                          | 未使用コード・ファイル検出   | PR        |
| `similarity-ts`                                 | 重複コード検出（警告のみ）   | PR        |
| `wrangler deploy`                               | Workers APIデプロイ  | mainマージ  |
| `wrangler pages deploy`                         | Remix UIデプロイ    | mainマージ  |
| `terraform apply`                               | インフラ適用（変更時のみ）   | mainマージ  |

CI実行順序（速い順に落とす）：
```
oxlint → oxc format → stylelint → tsgo → knip → similarity-ts
```

---

## 🌐 ホスティング対応言語・ランタイム

| 言語      | ランタイム            | テンプレートCTID | 整備フェーズ  |
| ------- | ---------------- | ----------- | ------- |
| Node.js | Node.js 22 (LTS) | CT200       | Phase 1 |
| Python  | Python 3.12      | CT201       | Phase 1 |
| Go      | Go 1.22          | CT202       | Phase 2 |

---

## 📦 IaC Provider構成

| Provider                  | バージョン | 管理対象                        |
| ------------------------- | ----- | --------------------------- |
| `bpg/proxmox`             | ~0.66 | LXCコンテナ（作品ごと）               |
| `cloudflare/cloudflare`   | ~4.0  | DNS・Tunnel・D1・Pages         |

---

## ❌ 採用しない技術・理由

| 技術               | 不採用理由                                         |
| ---------------- | --------------------------------------------- |
| LXC + Docker     | Proxmox公式非推奨。nesting設定がアップデートで壊れるリスクあり        |
| LXC + Podman     | 同上。rootless動作にUID/GIDマッピングが複雑。バグ報告あり           |
| Kubernetes       | 学習コスト極大。1台構成には過剰。期限内に完走不可能                    |
| Render / Railway | 無料枠が小さい or スリープあり。常時稼働の保証がない                  |
| OpenTofu（OSS）    | ライセンス制限を避けたい場合の代替候補。HCL互換なので切り替え容易           |
