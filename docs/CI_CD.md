# CI/CD 設計

> **WARNING: Public Repository** — ワークフロー内に秘密情報をハードコードしないこと。
> `pull_request_target` トリガーは使用禁止（fork 経由の攻撃防止）。

---

## 1. 設計原則

| 原則 | 説明 |
|------|------|
| Security First | 全ステージにセキュリティチェック。OIDC 認証、SHA ピン留め |
| Fail Fast | 軽いチェック(lint)を先に実行。paths-filter で不要ジョブをスキップ |
| Immutable Artifacts | ビルド成果物は 1 度だけ生成、全環境で同一 |
| Coverage Gate | カバレッジ基準未達でマージブロック |

---

## 2. CI パイプライン

### トリガー

| イベント | 対象 | 実行内容 |
|---------|------|---------|
| pull_request | main 向け PR | フル CI |
| push | main | フル CI + CD |
| schedule | 毎日 00:00 UTC | セキュリティスキャン |

### ジョブ構成

```
paths-filter
  ├─ [api変更] Rails Lint → Rails Test → Rails Security ─┐
  ├─ [web変更] Angular Lint → Angular Test → Build ──────┤
  └─ [infra変更] Terraform Lint → Plan ──────────────────┤
                                                         ▼
                                                 Coverage Gate
                                                         │
                                              Docker Build + Trivy
                                                         │
                                                    CD (deploy)
```

### キャッシュ戦略

| 対象 | 方法 |
|------|------|
| Ruby gems | `ruby/setup-ruby` の `bundler-cache: true` |
| Node modules | `actions/setup-node` の `cache: npm` |
| Docker layers | BuildKit + ECR cache |
| Terraform plugins | `actions/cache` |

### 並列実行

- Rails 系と Angular 系は独立なので並列実行
- `concurrency` で同一ブランチの古い実行をキャンセル

---

## 3. CD パイプライン

### デプロイフロー

1. CI 全 Green → Docker Build → ECR Push
2. **承認ゲート** (GitHub Environment Protection)
3. DB マイグレーション (ECS ワンショットタスク)
4. ECS サービス更新 (force-new-deployment)
5. ヘルスチェック + スモークテスト
6. 失敗時: 自動ロールバック

### 承認プロセス

- GitHub Environments の `production` に Required Reviewers を設定
- デプロイブランチ: main のみ

### ロールバック

- ECS デプロイメントサーキットブレーカー（自動ロールバック有効）
- スモークテスト失敗時: 前回のタスク定義リビジョンに自動復帰
- 手動: `workflow_dispatch` で任意コミットを指定

---

## 4. セキュリティスキャン

| ツール | 対象 | タイミング |
|--------|------|-----------|
| Brakeman | Rails セキュリティ | PR / push |
| Bundler Audit | Ruby gems 脆弱性 | PR / daily |
| npm audit | Node packages 脆弱性 | PR / daily |
| gitleaks | シークレット検出 | PR |
| Trivy | Docker イメージ | ビルド後 |
| tfsec / checkov | Terraform | IaC 変更時 |
| Dependabot | 全依存関係 | 自動 PR |

---

## 5. GitHub Actions 設計

### ワークフロー構成

```
.github/
├── workflows/
│   ├── ci.yml              # メイン CI
│   ├── cd.yml              # デプロイ
│   ├── security.yml        # 定期セキュリティスキャン
│   └── dependabot-merge.yml
├── actions/
│   ├── setup-rails/action.yml
│   └── setup-angular/action.yml
├── dependabot.yml
└── CODEOWNERS
```

### 認証 (OIDC)

```yaml
permissions:
  id-token: write
  contents: read

- uses: aws-actions/configure-aws-credentials@<SHA>
  with:
    role-to-assume: arn:aws:iam::<ACCOUNT>:role/github-actions-deploy
    aws-region: ap-northeast-1
```

OIDC trust policy の subject: `repo:codingalone/study-rails-angular-with-harness:ref:refs/heads/main`

### コスト最適化

- `ubuntu-latest` のみ使用
- 各ジョブに `timeout-minutes` 設定
- paths-filter で変更対象のみ実行
- Dependabot patch は CI パス後に自動マージ

---

## 6. ブランチ戦略

**Trunk-Based Development** を採用。

```
main (protected)
  └── feat/user-authentication    ← 短寿命 (1-3日)
  └── fix/login-error
  └── chore/docker-setup
```

| ルール | 設定 |
|--------|------|
| 命名 | `{type}/{description}` (kebab-case) |
| マージ方式 | Squash merge |
| main 直接 push | 禁止 |
| Required status checks | lint, test, security |
| 寿命 | 最大 3 営業日 |

---

## 7. 通知

| イベント | 通知先 | 優先度 |
|---------|--------|--------|
| CI 失敗 (PR) | GitHub PR Status | 通常 |
| CD 失敗 (main) | Slack / Email | 緊急 |
| セキュリティ検出 | Slack / GitHub Issue | 緊急 |
| デプロイ成功 | Slack (簡潔) | 通常 |
