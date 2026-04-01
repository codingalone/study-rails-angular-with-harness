# インフラストラクチャ設計

> **WARNING: Public Repository** — AWS アカウント ID、具体的な IP、ARN をこのファイルに記載しないこと。
> `terraform.tfvars` は .gitignore 対象。変数の具体値はリポジトリに含めない。

---

## 1. AWS 構成概要

### 推奨構成

| レイヤー | サービス | 構成 |
|---------|---------|------|
| CDN | CloudFront + WAF | SPA 配信 + IP 制限 |
| 静的ホスティング | S3 | Angular ビルド成果物 |
| DNS | Route 53 | 1 ゾーン |
| SSL | ACM | 無料証明書 |
| LB | ALB | HTTPS 終端、2 AZ |
| コンピュート | ECS Fargate (Spot) | 0.25 vCPU / 512 MB |
| DB | RDS PostgreSQL | db.t4g.micro、単一 AZ |
| ネットワーク | VPC + NAT Instance | t4g.nano (NAT Gateway の代替) |
| シークレット | SSM Parameter Store / Secrets Manager | SecureString (KMS 暗号化)。原則 SSM、高頻度ローテーション対象は Secrets Manager |
| ログ | CloudWatch Logs | 30 日保持 |
| コスト管理 | AWS Budgets | 月額アラート |

---

## 2. コスト見積もり

### 月額概算（Free Tier 適用前）

| サービス | 月額 (USD) |
|---------|-----------|
| ECS Fargate (0.25vCPU/512MB × 1) | ~$9 |
| RDS db.t4g.micro + 20GB | ~$8 |
| ALB | ~$16 |
| NAT Instance (t4g.nano) | ~$3 |
| WAF (ACL + ルール) | ~$6 |
| Route 53 | ~$0.50 |
| CloudWatch Logs | ~$1 |
| SSM Parameter Store | $0 |
| ECR | ~$0.50 |
| S3 + CloudFront | ~$0 |
| **合計** | **~$44/月** |

### Free Tier 適用時（新規アカウント 12 ヶ月以内）

- RDS db.t4g.micro: 無料 (-$8)
- ALB 750h: 無料 (-$16)
- → **約 $20/月**

### コスト最適化オプション

| 施策 | 削減額 | トレードオフ |
|------|--------|------------|
| Fargate Spot | -$6 | 中断リスク（学習用途なら許容可） |
| 営業時間のみ稼働 (12h/日) | -$5 | 停止中はアクセス不可 |
| ECR ライフサイクルポリシー | -$0.5 | 古いイメージ自動削除 |
| CloudWatch Logs 保持 7 日 | -$0.5 | ログ調査期間が短くなる |

---

## 3. Terraform 設計

### ディレクトリ構成

```
infra/
├── bootstrap/              # State 管理用 (初回のみ)
│   └── main.tf
├── environments/
│   └── prd/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── providers.tf
│       ├── backend.tf
│       └── versions.tf
├── modules/
│   ├── networking/         # VPC, Subnets, SG, NAT
│   ├── ecs/                # Cluster, Service, Task Definition
│   ├── rds/                # RDS Instance
│   ├── cdn/                # CloudFront, S3, OAC
│   ├── alb/                # ALB, Target Group
│   ├── waf/                # WAF WebACL, IP Set
│   ├── dns/                # Route 53, ACM
│   ├── monitoring/         # CloudWatch
│   └── secrets/            # SSM Parameter Store
└── .tflint.hcl
```

### State 管理

- S3 + DynamoDB (state lock)
- バージョニング有効
- SSE-KMS 暗号化
- パブリックアクセス完全ブロック

### モジュール設計方針

- 単一責任: 1 モジュール = 1 サービス群
- 疎結合: モジュール間は outputs/variables のみで連携
- 命名: `${var.project_name}-${var.environment}` プレフィックス
- タグ: 全リソースに Project, Environment, ManagedBy タグ

---

## 4. IaC テスト

### テストピラミッド

| レイヤー | ツール | 実行タイミング | コスト |
|---------|--------|-------------|--------|
| 静的解析 | tflint, checkov, tfsec | PR / push | 無料 |
| Plan 検証 | terraform plan + conftest (OPA) | PR / push | 無料 |
| 結合テスト | Terratest | 手動トリガー（必要時のみ） | 実リソース費用 |

### conftest ポリシー例

- セキュリティグループで `0.0.0.0/0` Ingress を禁止（sg-alb の 443 ポートのみ例外。WAF で IP 制限を併用）
- S3 バケットの暗号化必須
- RDS のパブリックアクセス禁止
- 全リソースのタグ必須

---

## 5. リソース停止スケジュール

EventBridge Scheduler + Lambda で自動停止/起動。

| タイミング | アクション |
|-----------|-----------|
| 平日 09:00 JST | ECS desired_count=1, RDS 起動 |
| 平日 21:00 JST | ECS desired_count=0, RDS 停止 |
| 週末 | 終日停止 |

注意: RDS は停止後 7 日で自動起動する仕様あり。Lambda で再停止ロジックが必要。

---

## 6. バックアップ / リカバリ

| 対象 | 方式 | 保持期間 |
|------|------|---------|
| RDS | 自動バックアップ + ポイントインタイムリカバリ | 7 日 |
| S3 (SPA) | バージョニング | 30 日 |
| Terraform State | S3 バージョニング | 無期限 |

### RTO / RPO

| シナリオ | RTO | RPO |
|---------|-----|-----|
| ECS タスク障害 | ~5 分 | 0 |
| RDS 障害 | ~30 分 | ~5 分 |
| インフラ全体再構築 | ~1 時間 | 最終バックアップ |
