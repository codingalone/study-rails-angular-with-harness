# CLAUDE.md

ハーネスエンジニアリング学習のための Rails API + Angular SPA Web アプリケーション。
AI が仕様駆動で自律開発し、テストで品質を保証する。

> **WARNING: This is a PUBLIC repository.**
> Never commit secrets, credentials, AWS account IDs, IP addresses, or any sensitive values.
> pre-commit hook (Claude による秘密情報スキャン) が有効。`--no-verify` は原則禁止。

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend API | Ruby on Rails (API mode) |
| Frontend SPA | Angular (Standalone Components) |
| Database | PostgreSQL 18 |
| Cache / Job Queue | Redis 7 |
| Local Dev | Docker Compose |
| Infrastructure | AWS (Terraform ~> 1.9) |
| CI/CD | GitHub Actions |
| E2E Test | Playwright |
| Backend Test | RSpec + SimpleCov |
| Frontend Test | Jest |

---

## Development Flow: 仕様 → テスト → 実装

1. 仕様を `docs/specs/` に起草する
2. ユーザーに仕様を提示し承認を得る（**省略不可**）
3. 承認済み仕様からテストを書く（RED 確認）
4. テストが通る最小実装を行う（GREEN）
5. Docker 内でテスト実行 + カバレッジ確認
6. CI が全 Green であることを確認

---

## Coverage Requirements

- **分岐カバレッジ**: 95% 以上（主指標）
- **行カバレッジ**: 100%（補助指標）
- **関数・文カバレッジ**: 100%（Angular / Jest のみ）
- 除外対象: `db/schema.rb`, `db/migrate/`, `config/`, `environment.ts`, ルーティング定義
- カバレッジが基準を下回る PR はマージ不可

---

## Commit Convention

Conventional Commits 形式。

```
<type>(<scope>): <description>
```

type: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `ci`, `spec`
scope: `api`, `web`, `infra`, `ci`

---

## Permission Model

### ユーザー承認が必要 (MUST ASK)

- AWS リソースの作成・変更・削除
- アーキテクチャの重要な判断
- GitHub Actions secrets / environment の設定
- 外部公開に関わる設定変更（ドメイン、DNS、SG、公開ポート）
- デプロイ操作
- セキュリティに関わる設定変更（認証、暗号鍵、CORS、CSP）
- 課金が発生する可能性のある操作
- `main` ブランチへの force push
- DB マイグレーションファイルの新規作成（既存マイグレーションの実行は承認不要）
- 新しい gem / npm パッケージの追加
- ファイルの削除（ソースコード、テスト、仕様書）
- Docker 構成の変更（ポート、ボリューム、ネットワーク）
- 仕様書の承認ステータス変更

### 自律実行可能 (CAN DO)

- Docker 環境内でのコード編集・ファイル作成
- `docker compose exec` 経由のテスト・lint 実行
- ドキュメントの作成・更新
- `git commit` / `git push`（feature ブランチのみ）
- 仕様書のドラフト起草
- カバレッジレポートの確認

---

## Commands

```bash
# Docker
docker compose up -d
docker compose down
docker compose build

# Rails API
docker compose exec api bundle exec rspec
docker compose exec api bundle exec rubocop
docker compose exec api bundle exec rubocop -A
docker compose exec api bundle exec rails db:migrate
# ★ 新規マイグレーション作成は要承認。既存マイグレーション実行（初回セットアップ・CD）は承認不要
docker compose exec api bundle exec rails console
docker compose exec api bundle exec brakeman --no-pager -q

# Angular
docker compose exec web npx jest --coverage
docker compose exec web npx ng lint
docker compose exec web npx ng build --configuration=production

# E2E
docker compose exec e2e npx playwright test

# Terraform (要ユーザー承認)
cd infra/environments/prd && terraform plan
cd infra/environments/prd && terraform apply
```

---

## Directory Structure

```
.
├── CLAUDE.md
├── docker-compose.yml
├── .gitignore
├── .github/workflows/
├── api/                    # Rails API
│   ├── Dockerfile
│   ├── Gemfile
│   ├── app/
│   ├── spec/
│   └── .rubocop.yml
├── web/                    # Angular SPA
│   ├── Dockerfile
│   ├── package.json
│   ├── src/
│   └── jest.config.ts
├── infra/                  # Terraform
│   ├── bootstrap/           # State 管理用 (初回のみ)
│   ├── environments/prd/
│   ├── modules/
│   └── .tflint.hcl
├── docs/
│   ├── PROJECT_CHARTER.md
│   ├── ARCHITECTURE.md
│   ├── TESTING_STRATEGY.md
│   ├── SECURITY_POLICY.md
│   ├── INFRASTRUCTURE.md
│   ├── CI_CD.md
│   ├── DEVELOPMENT_GUIDE.md
│   ├── SPECIFICATION_PROCESS.md
│   ├── DEPLOYMENT.md
│   ├── openapi/             # OpenAPI 3.1 定義
│   ├── specs/              # 仕様書
│   └── adr/                # Architecture Decision Records
└── e2e/                    # Playwright E2E tests
```

---

## Security Rules

- `.env`, `config/master.key`, `config/credentials/*.key`, `*.pem`, `*.key`, `terraform.tfstate`, `terraform.tfvars` は絶対に commit しない
- API キー・パスワードは環境変数 or AWS Secrets Manager / SSM Parameter Store で管理
- Docker イメージに秘密情報を焼き込まない
- `terraform.tfstate` は S3 remote backend で管理し、リポジトリに含めない
- `--privileged` モード禁止、Docker ソケットマウント禁止
- `terraform plan` の出力は必ずユーザーに提示してから `apply`
- Free Tier を意識し、リソース作成前にコスト見積もりを提示
- シークレット管理は原則 SSM Parameter Store (SecureString)。高頻度ローテーション対象のみ Secrets Manager を使用

## AI Cross-Review Process

仕様書・設計ドキュメント・開発計画の PR マージ前に、Codex と Gemini によるクロスレビューを実施する。

### 原則

- **両レビューアーが PASS するまで修正を繰り返す**（途中で打ち切らない）
- Critical / Medium の指摘は修正必須
- Low は対応任意（justified でスキップ可）
- レビュー結果は PR コメントとして記録（監査証跡）

### レビューアー構成

| レビューアー | 観点 |
|-------------|------|
| Codex (gpt-5.4) | コード品質、パフォーマンス、一貫性 |
| Gemini (gemini-2.5-flash) | セキュリティ、エッジケース、仕様整合性 |

### 実行方法

#### Codex

```bash
codex exec --full-auto "<レビュープロンプト>"
```

- `codex exec` を使う（`codex` 単体は対話モード）
- `--full-auto` フラグ必須（`--full-context` は存在しない）
- モデル指定不要（デフォルトの gpt-5.4 を使用。ChatGPT アカウントでは他モデルは非対応）

#### Gemini

```bash
gemini -p "<レビュープロンプト>"
```

- `-p` フラグで非対話（ヘッドレス）モード実行
- 認証は OAuth（`~/.gemini/settings.json` の `oauth-personal`）。API キー不要
- `curl` + `GEMINI_API_KEY` 方式は使わない（キー管理が煩雑、レートリミットに弱い）

### 終了条件

- 両レビューアーが 0 Critical / 0 Medium を返す
- 最終レビューサマリーを PR コメントに投稿

---

## Host Protection

母艦 MacBook への破壊的変更は禁止。すべて Docker 内で完結させる。

- ホストへの gem install / npm install -g 禁止
- シェル設定 (`.zshrc`, `config.fish` 等) の変更禁止
- ポートバインドは `127.0.0.1` 限定
- Volume マウントはプロジェクトルートのみ。`~/.ssh`, `~/.aws` はマウント禁止
