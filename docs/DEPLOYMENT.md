# デプロイメント戦略

> **WARNING: Public Repository** — ECR URL、ALB DNS 名、RDS エンドポイント等の具体値を記載しないこと。

---

## 1. デプロイメント概要

```
[GitHub main merge]
    ↓
[CI: lint → test → build → security scan]
    ↓
[Docker Build → ECR Push]
    ↓
[承認ゲート (GitHub Environment)]  ★ユーザー承認必須
    ↓
[DB Migration (ECS ワンショットタスク)]
    ↓
[ECS Service 更新]
    ↓
[ヘルスチェック + スモークテスト]
    ↓
[成功: 本番稼働 / 失敗: 自動ロールバック]
```

---

## 2. Docker イメージ

### Rails API (マルチステージビルド)

- Builder: `ruby:3.3-slim` — gem install, asset compile
- Runtime: `ruby:3.3-slim` — 最小ランタイムのみ
- 非 root ユーザー (`rails`) で実行
- HEALTHCHECK: `curl -f http://localhost:3000/api/health`

### Angular SPA

- Builder: `node:20-slim` — `ng build --configuration=production`
- 本番配信は S3 + CloudFront（Docker コンテナ不要）

### イメージタグ戦略

| タグ | 形式 | 用途 |
|------|------|------|
| Git SHA | `sha-a1b2c3d` | 一意識別（デプロイで使用） |
| latest | — | **使用しない** |

### ECR ライフサイクル

- 最新 10 イメージを保持
- 未タグイメージは 7 日後に自動削除

---

## 3. デプロイフロー

### 自動デプロイ（main マージ時）

1. CI 全 Green → Docker Build → ECR Push
2. 承認ゲート（GitHub Environment Protection Rules）
3. DB マイグレーション（ECS ワンショットタスク）
4. ECS サービス更新 (`force-new-deployment`)
5. `aws ecs wait services-stable` で安定化待機
6. スモークテスト実行

### 手動デプロイ（緊急時）

`workflow_dispatch` でイメージタグとデプロイ理由を入力。承認ゲートは省略しない。

### 承認プロセス

GitHub Environments > production:
(GitHub 上の名称は `production`、AWS/Terraform リソース名のプレフィックスは `prd`)
- Required reviewers: 承認者リスト
- Deployment branches: main のみ

---

## 4. ヘルスチェック

### Rails `/api/health`

```json
{
  "status": "healthy",
  "checks": {
    "database": true,
    "migrations": true
  },
  "version": "sha-a1b2c3d"
}
```

### スモークテスト

デプロイ後に自動実行:
1. SPA が配信されること (`GET /` → 200)
2. API ヘルスチェック (`GET /api/health` → 200, status: healthy)
3. 認証が機能していること（未認証 → 401）

---

## 5. ロールバック

### 自動ロールバック

- ECS デプロイメントサーキットブレーカー: ヘルスチェック失敗で自動復帰
- スモークテスト失敗: 前回タスク定義リビジョンに自動切替

### 手動ロールバック

```bash
# タスク定義リビジョン一覧
aws ecs list-task-definitions --family-prefix prd-rails-api --sort DESC --max-items 5

# ロールバック実行
aws ecs update-service --cluster prd-cluster --service prd-rails-api \
  --task-definition prd-rails-api:<リビジョン番号>

# 安定化待機
aws ecs wait services-stable --cluster prd-cluster --services prd-rails-api
```

---

## 6. DB マイグレーション

### 実行戦略

- デプロイ前にワンショット ECS タスクで実行
- 成功 → アプリ更新、失敗 → デプロイ中止

### 破壊的マイグレーション

**Expand-Contract パターン**で段階的に実施:

1. Deploy 1 (Expand): 新カラム追加 + デュアルライト
2. Deploy 2 (Migrate): 読み取り切替
3. Deploy 3 (Contract): 旧カラム削除

`strong_migrations` gem で危険なマイグレーションを CI で検出。

---

## 7. シークレット管理

### ECS タスク定義での注入

- 非機密設定値: `environment` 直接指定
- 機密値: `secrets` で SSM Parameter Store / Secrets Manager から注入
- ECS 実行ロールに最小権限の取得ポリシーを付与

### ログへの機密情報出力防止

- Rails `filter_parameters`: `:password`, `:token`, `:secret`, `:key`, `:authorization`
- GitHub Actions: `::add-mask::` で自動マスク

---

## 8. コスト最適化デプロイ

### 最小構成

| サービス | 構成 | 月額概算 |
|---------|------|---------|
| ECS Fargate (API) | 0.25 vCPU / 512 MB × 1 | ~$9 |
| S3 + CloudFront (SPA) | 静的配信 | ~$0 |

### Fargate Spot

クローズド環境で中断許容可能なら Fargate Spot を活用（最大 70% 削減）。

### リソース停止

EventBridge Scheduler で営業時間外に ECS desired_count=0 に設定。
