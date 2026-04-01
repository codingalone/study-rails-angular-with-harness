# プロジェクト憲章 (Project Charter)

> **WARNING: Public Repository** — https://github.com/codingalone/study-rails-angular-with-harness.git
> 機密情報を一切コミットしないこと。

---

## 1. プロジェクト概要

| 項目 | 内容 |
|------|------|
| プロジェクト名 | study-rails-angular-with-harness |
| 目的 | ハーネスエンジニアリング（テスト・CI/CD・IaC・セキュリティ等の基盤技術）の体系的学習 |
| 成果物 | Rails API + Angular SPA の Web アプリケーション（題材）と、それを支える基盤一式 |
| リポジトリ | https://github.com/codingalone/study-rails-angular-with-harness.git (Public) |
| デプロイ先 | AWS (prd 環境 1 つ、クローズド) |

アプリケーション自体は学習のための題材であり、**基盤技術の習得が本質的な目的**である。

---

## 2. 目的とスコープ

### 学習対象の基盤技術

1. **仕様駆動開発** — 仕様 → テスト → 実装のサイクル
2. **テスト設計** — 単体・統合・E2E テスト、カバレッジ管理
3. **CI/CD** — GitHub Actions によるパイプライン構築
4. **IaC** — Terraform による AWS インフラ管理
5. **コンテナ技術** — Docker / ECS Fargate
6. **セキュリティ** — OWASP Top 10 対策、シークレット管理、WAF
7. **クラウドアーキテクチャ** — VPC 設計、コスト最適化
8. **運用** — 監視、ログ、デプロイ、ロールバック

### スコープ外

- 高可用性設計（Multi-AZ、マルチリージョン）
- 大規模トラフィック対応
- 複数環境（staging 等）の運用

---

## 3. 基本原則

### 3.1 完全自律開発

ユーザー（プロジェクトオーナー）はコードを一切書かず、コードレビューも行わない。AI（Claude）が仕様検討から実装まで自律的に遂行する。

### 3.2 仕様駆動開発

すべての実装は承認済みの仕様に基づく。仕様に記述されていない振る舞いは存在してはならない。

### 3.3 テストによる品質保証

テストがコードレビューの代替として機能する。分岐カバレッジ 95% 以上、行カバレッジ 100% を原則とし、例外は極力作らない。

### 3.4 Docker サンドボックス

ローカル開発は Docker Compose で完結させ、母艦 MacBook への破壊的変更を禁止する。

### 3.5 セキュリティ最優先

- ハッキングリスクへの対策を最重視
- 不要なコスト発生を防止
- 不用意な外部公開を禁止
- **Public リポジトリであるため、機密情報のコミットは厳禁**
- pre-commit hook（Claude による秘密情報スキャン）を必ず使用

### 3.6 IaC による管理

AWS インフラは Terraform で管理し、手動変更を禁止する。インフラも仕様駆動でテストを伴う。

---

## 4. ステークホルダーと役割分担

| 役割 | 担当 | 責任範囲 |
|------|------|----------|
| プロジェクトオーナー | ユーザー | 要件定義、仕様承認、デプロイ承認、コスト管理判断 |
| 開発者 | AI (Claude) | 仕様起草、テスト実装、コード実装、ドキュメント作成、CI/CD構築 |

### オーナーの承認が必要な判断

- 仕様の承認
- アーキテクチャの重要な判断
- AWS リソースの作成・変更
- デプロイの実行
- セキュリティに関わる設定変更
- 新しい依存パッケージの追加

### AI が自律的に行う作業

- 仕様のドラフト起草
- テスト・コードの実装
- Docker 環境内でのテスト実行
- lint / format の実行と修正
- ドキュメントの更新
- feature ブランチでの git 操作

---

## 5. 成功基準

| 基準 | 測定方法 |
|------|----------|
| テストカバレッジ 95%+ (branch) / 100% (line) | SimpleCov (Rails) / Jest (Angular) |
| CI パイプラインが全 Green | GitHub Actions |
| セキュリティスキャン合格 | Brakeman, npm audit, tfsec, Trivy |
| AWS 環境にデプロイされ稼働 | ヘルスチェック + スモークテスト |
| 月額 AWS コスト $60 以下 | AWS Cost Explorer |
| 機密情報のコミットがゼロ | pre-commit hook + CI スキャン |

---

## 6. リスクと対策

| リスク | 影響度 | 対策 |
|--------|--------|------|
| Public リポジトリへの機密情報コミット | Critical | pre-commit hook、.gitignore、CI スキャン（多層防御） |
| AWS コスト暴走 | High | Budgets アラート、リソース停止スケジュール、SCP 制限 |
| ハッキング・不正アクセス | High | WAF、IP 制限、VPC 分離、最小権限 IAM |
| AI 生成コードの品質低下 | Medium | 静的解析（RuboCop, ESLint, Brakeman）、型チェック、テスト |
| コンテキスト消失（セッション断絶） | Medium | CLAUDE.md、ADR、仕様書による知識の永続化 |
| テストカバレッジの形骸化 | Medium | 分岐カバレッジ採用、ミューテーションテスト検討 |

---

## 7. プロジェクトフェーズ

### Phase 0: 基盤構築（現在）

- プロジェクトドキュメント整備
- Git リポジトリ設定
- .gitignore、pre-commit hook
- CLAUDE.md

### Phase 1: ローカル開発環境

- Docker Compose 構成（Rails API + Angular + PostgreSQL + Redis）
- コーディング規約設定（RuboCop, ESLint, Prettier）
- テストフレームワーク設定（RSpec, Jest, Playwright）
- カバレッジ計測設定（SimpleCov, Jest coverage）

### Phase 2: CI パイプライン

- GitHub Actions ワークフロー構築
- Lint → テスト → ビルド → セキュリティスキャン
- カバレッジゲート
- PR テンプレート

### Phase 3: アプリケーション実装

- アプリケーション仕様の策定
- 仕様駆動での機能実装（テストファースト）
- API + SPA の開発

### Phase 4: AWS インフラ構築

- Terraform による VPC / ECS / RDS / CloudFront 構築
- IaC テスト（tflint, checkov, conftest）
- セキュリティ設定（WAF, IAM, SG）
- コスト最適化

### Phase 5: CD パイプラインとデプロイ

- Docker イメージビルド → ECR プッシュ
- ECS デプロイフロー
- スモークテスト
- ロールバック戦略

---

## 8. ドキュメント管理

本ドキュメント群自体も git で管理し、変更履歴をコミットで追跡する。

| ドキュメント | 配置場所 | 更新タイミング |
|-------------|---------|---------------|
| CLAUDE.md | ルート | 規約・構成の変更時 |
| PROJECT_CHARTER | docs/ | プロジェクト方針の変更時 |
| 各設計ドキュメント | docs/ | 対象領域の設計変更時 |
| 仕様書 | docs/specs/ | 機能追加・変更時 |
| ADR | docs/adr/ | 重要な設計判断時 |
