# アーキテクチャ設計

> **WARNING: Public Repository** — 具体的な AWS アカウント ID、IP アドレス、内部ドメイン名をこのファイルに記載しないこと。

---

## 1. システムアーキテクチャ概要

```
[Browser] --> [CloudFront + WAF (IP制限)]
                    |
              +-----+-----+
              |           |
        [S3 Bucket]  [ALB (Private)]
        (Angular SPA)     |
                    [ECS Fargate]
                    (Rails API)
                          |
                    [RDS PostgreSQL]
                    (Private Subnet)
```

- **フロントエンド**: Angular SPA を S3 + CloudFront で静的配信
- **バックエンド**: Rails API を ECS Fargate で実行
- **通信方式**: REST API (JSON)、将来的に OpenAPI で契約を管理
- **認証**: JWT (RS256)

---

## 2. ローカル開発環境

Docker Compose で全サービスを起動。母艦 MacBook への変更は一切行わない。

### サービス構成

| サービス | イメージ | ポート | 役割 |
|---------|---------|--------|------|
| api | カスタム (Ruby 3.3) | 127.0.0.1:3000 | Rails API |
| web | カスタム (Node 20) | 127.0.0.1:4200 | Angular dev server |
| postgres | postgres:16-alpine | 127.0.0.1:5432 | データベース |
| redis | redis:7-alpine | 127.0.0.1:6379 | キャッシュ / ジョブキュー |

### ボリューム戦略

| 対象 | 方式 | 理由 |
|------|------|------|
| ソースコード | Bind mount | ホスト側で編集、コンテナに即時反映 |
| gem (bundle) | Named volume | コンテナ再作成時の install 高速化 |
| node_modules | Named volume | OS 差異による問題を回避 |
| DB データ | Named volume | コンテナ再作成でもデータ保持 |

---

## 3. 本番環境アーキテクチャ (AWS)

### 構成選定理由

| コンポーネント | 選択 | 理由 |
|-------------|------|------|
| コンピュート | ECS Fargate | コンテナ管理不要、タスク数 0 で停止可能、学習価値が高い |
| DB | RDS PostgreSQL (db.t4g.micro) | Free Tier 対象、Rails との親和性 |
| CDN | CloudFront + S3 | SPA 配信に最適、Free Tier 1TB/月 |
| LB | ALB | HTTPS 終端、ヘルスチェック |
| WAF | AWS WAF | IP 制限によるクローズド環境の実現 |
| ネットワーク | VPC (3 層サブネット) | Public / Private / Isolated で分離 |

### ネットワーク設計

```
VPC: 10.0.0.0/16
├── Public Subnets (ALB)
│   ├── 10.0.1.0/24 (ap-northeast-1a)
│   └── 10.0.2.0/24 (ap-northeast-1c)
├── Private Subnets (ECS Fargate)
│   ├── 10.0.11.0/24 (ap-northeast-1a)
│   └── 10.0.12.0/24 (ap-northeast-1c)
└── Isolated Subnets (RDS) — インターネットアクセス不可
    ├── 10.0.21.0/24 (ap-northeast-1a)
    └── 10.0.22.0/24 (ap-northeast-1c)
```

---

## 4. データベース設計方針

- RDBMS: PostgreSQL 16（Rails との親和性、JSON サポート、実績）
- マイグレーション: Rails 標準 (`rails db:migrate`)、`strong_migrations` gem で破壊的変更を CI で検出
- Expand-Contract パターン: 破壊的変更は段階的に実施（新カラム追加 → 切替 → 旧カラム削除）

---

## 5. セキュリティアーキテクチャ

### 認証 (JWT)

| 項目 | 設定 |
|------|------|
| アルゴリズム | RS256（HS256 は拒否） |
| アクセストークン有効期限 | 15 分 |
| リフレッシュトークン有効期限 | 7 日 |
| リフレッシュトークン保存 | HttpOnly, Secure, SameSite=Strict Cookie |
| アクセストークン保存 | メモリのみ（localStorage/sessionStorage 禁止） |

### CORS

- 許可オリジン: 本番ドメインのみ（`origins '*'` は全環境で禁止）
- credentials: true

### CSP

- `default-src 'none'`、必要な src のみ明示許可
- `unsafe-eval` 禁止（Angular production build では不要）

---

## 6. 技術スタック詳細

| Layer | Technology | Version Policy |
|-------|-----------|---------------|
| Ruby | 3.3.x | .ruby-version で固定 |
| Rails | 7.x (API mode) | Gemfile.lock で固定 |
| Node.js | 20.x LTS | .node-version で固定 |
| Angular | 18.x+ | package-lock.json で固定 |
| PostgreSQL | 16 | Docker image tag で固定 |
| Redis | 7 | Docker image tag で固定 |
| Terraform | ~> 1.9 | versions.tf で制約 |

---

## 7. ADR 運用方針

重要なアーキテクチャ判断は `docs/adr/` に ADR (Architecture Decision Record) として記録する。

ファイル名: `ADR-NNNN-<title>.md`

記載項目:
- Status (Proposed / Accepted / Deprecated / Superseded)
- Context（なぜこの判断が必要か）
- Decision（何を決めたか）
- Consequences（結果として何が起こるか）
