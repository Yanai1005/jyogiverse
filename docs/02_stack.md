## 技術選定

---

## 0️⃣ 選定方針

| 方針           | 内容                                    |
| ------------ | ------------------------------------- |
| コスト最優先       | メンバー1人あたり月額0円（電気代・ドメイン代除く）を厳守        |
| Proxmox公式範囲内 | LXC直接運用。LXC内のDocker / Podmanは使わない。   |
| 運用シンプル化      | 管理者1人（柳井）で維持できる構成にする                  |

---

## 1️⃣ インフラ層

| コンポーネント    | 選定技術              | 理由                                              |
| ---------- | ----------------- | ----------------------------------------------- |
| ハイパーバイザー   | Proxmox VE 8.x    | LXCコンテナを公式サポート。無料で利用可能。                         |
| コンテナ       | LXC（Debian 12）    | Docker不要でオーバーヘッド最小。Proxmox公式サポート範囲内。            |
| プロセス管理     | systemd           | Debian標準装備。自動再起動・ログ管理が標準で使える。                   |
| リバースプロキシ   | Caddy 2.x         | 自動HTTPS（Let's Encrypt）。Caddyfileがシンプル。           |
| 外部公開トンネル   | Cloudflare Tunnel | ポート開放不要。無料。DDoS対策・TLS終端込み。                      |
| IaC        | Terraform          | bpg/proxmox + Cloudflare Providerで全リソースを管理。 |
| IaC State  | Cloudflare R2     | S3互換。無料枠内でTerraform stateを保管。                     |

---

## 2️⃣ 管理UI層

| コンポーネント  | 選定技術                     | 理由                              |
| -------- | ------------------------ | -------------------------------- |
| APIサーバー  | Cloudflare Workers（Hono） | 無料枠で常時稼働。TypeScript。軽量フレームワーク。  |
| フロントエンド  | Cloudflare Pages（Remix）  | Git pushで自動デプロイ。無料。SSR対応。        |
| データベース   | Cloudflare D1（SQLite）    | Workers統合。無料枠で運用量は十分。            |
| 管理者アクセス制限 | Cloudflare Access        | `/admin` 配下を外部保護。アプリ内に認証ロジックなし。 |

---

## 3️⃣ CI/CDツールチェーン

| ツール                          | 役割                   | 実行タイミング |
| ------------------------------ | -------------------- | :-------: |
| `oxlint`                       | Lint                 | PR        |
| `oxc formatter`                | フォーマットチェック           | PR        |
| `@typescript/native-preview`（tsgo）| 型チェック              | PR        |
| `stylelint` + `stylelint-config-standard-scss` | SCSS Lint | PR   |
| `knip`                         | 未使用コード・ファイル検出        | PR        |
| `similarity-ts`                | 重複コード検出（警告のみ）        | PR        |
| `wrangler deploy`              | Workers APIデプロイ       | mainマージ  |
| `wrangler pages deploy`        | Remix UIデプロイ         | mainマージ  |
| `terraform apply`              | インフラ適用（infra/変更時のみ） | mainマージ  |

CI実行順序（速い順に落とす）：
```
oxlint → oxc format → stylelint → tsgo → knip → similarity-ts
```

---

## 4️⃣ 言語・ランタイム（ホスティング対応言語）

| 言語      | ランタイム            | テンプレートCTID | 整備フェーズ  |
| ------- | ---------------- | ----------- | ------- |
| Node.js | Node.js 22 (LTS) | 200         | Phase 1 |
| Python  | Python 3.12      | 201         | Phase 1 |
| Go      | Go 1.22          | 202         | Phase 2 |

---

## 5️⃣ IaC Provider構成

| Provider            | バージョン  | 管理対象                   |
| ------------------- | ------ | ---------------------- |
| `bpg/proxmox`       | ~0.66  | LXCコンテナ（作品ごと）          |
| `cloudflare/cloudflare` | ~4.0 | DNS・Tunnel・D1・Pages   |

---

## 6️⃣ 採用しない技術・理由

| 技術              | 不採用理由                                          |
| --------------- | ---------------------------------------------- |
| LXC + Docker    | Proxmox公式非推奨。nesting設定がアップデートで壊れるリスクあり。       |
| LXC + Podman    | 同上。rootless動作にUID/GIDマッピングが複雑。バグ報告あり。          |
| Kubernetes      | 学習コスト極大。1台構成には過剰。期限内に完走不可能。                    |
| Render / Railway | 無料枠が小さい or スリープあり。常時稼働の保証がない。                 |
| OpenTofu（OSS）   | ライセンス制限を避けたい場合はOpenTofuに切り替え可能。HCL互換。          |
