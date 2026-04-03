# 開発ガイド

> **WARNING: Public Repository** — `.env` に記載する値はリポジトリにコミットしない。`.env.example` のみコミット対象。

---

## 1. 開発フロー

```
仕様定義 → ユーザー承認 → テスト実装 → 実装 → テスト実行 → CI確認 → PR → マージ
         ★必須承認      (RED確認)   (GREEN)  (カバレッジ確認)       CI全Green後マージ
```

> **Note:** ユーザーはコードレビューを行わない（PROJECT_CHARTER.md 参照）。PR のマージ条件は CI の全チェック通過 + AI クロスレビュー PASS（HARNESS_ARCHITECTURE.md §7.4 参照）であり、人によるコードレビュー承認は不要。仕様承認は開発着手前に完了している。

### 原則

- テストファースト: 実装コードより先にテストを書く
- 最小実装: テストを通すために必要な最小限のコードのみ
- 仕様との整合性: 仕様に記述されていない振る舞いは実装しない

---

## 2. 初回セットアップ

### 前提条件

- macOS + Docker Desktop がインストール済み
- Git がインストール済み

### 手順

```bash
# 1. リポジトリをクローン
git clone https://github.com/codingalone/study-rails-angular-with-harness.git
cd study-rails-angular-with-harness

# 2. pre-commit hook をインストール
cp ~/dotfiles/hooks/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit

# 3. 環境変数ファイルを作成
cp .env.example .env
# .env を編集して必要な値を設定

# 4. Docker コンテナを起動
docker compose up -d

# 5. データベースを初期化
docker compose exec api bundle exec rails db:create db:migrate db:seed
# 初回セットアップのマイグレーション実行は承認不要（新規マイグレーションファイルの作成が承認対象）

# 6. 動作確認
# API: http://localhost:3000/api/health
# Web: http://localhost:4200
```

---

## 3. Docker ローカル環境

### 設計方針

| 方針 | 説明 |
|------|------|
| 母艦への変更禁止 | Ruby, Node.js, PostgreSQL をホストにインストールしない |
| ホットリロード | ファイル変更がコンテナに即時反映 |
| 再現可能 | `docker compose up` で完全に立ち上がる |
| データ永続化 | Named Volume で DB データを保持 |

### サービス構成

| サービス | ポート | 用途 |
|---------|--------|------|
| api | 127.0.0.1:3000 | Rails API |
| web | 127.0.0.1:4200 | Angular dev server |
| postgres | 127.0.0.1:5432 | データベース |
| redis | 127.0.0.1:6379 | キャッシュ / ジョブキュー |
| e2e | — | E2E テスト (オンデマンド) |

### ポートバインド

すべて `127.0.0.1` にバインドし、外部からのアクセスを遮断する。

### よく使うコマンド

```bash
docker compose up -d              # 起動
docker compose down               # 停止
docker compose logs -f api        # ログ
docker compose exec api bundle exec rails console  # Rails コンソール
```

---

## 4. コーディング規約

### Ruby / Rails (RuboCop)

- `rubocop-rails`, `rubocop-rspec`, `rubocop-performance` を使用
- API mode（ビュー関連 gem は含めない）
- バリデーションはモデル層で完結
- シリアライザで JSON 構造を明示定義
- N+1 検知: `bullet` gem (development)

### TypeScript / Angular (ESLint + Prettier)

- Standalone Components を使用（NgModule ベースは不採用）
- `any` 型の使用禁止 (`@typescript-eslint/no-explicit-any: error`)
- HTTP 通信は Service 層に集約
- リアクティブフォームを使用

### コミットメッセージ

Conventional Commits 形式:

```
feat(api): ユーザー認証エンドポイントを追加

Refs: docs/specs/features/FEAT-0001-user-authentication.md
```

---

## 5. Git 運用

### ブランチ命名

```
feat/user-authentication
fix/login-validation
refactor/user-service
docs/api-spec
chore/docker-setup
```

### コミット粒度

- 1 コミット = 1 論理的変更
- テストと実装はセット（テストなしの実装コミットは禁止）
- 仕様書は独立コミット
- マイグレーションは独立コミット

### PR テンプレート

```markdown
## 概要
<!-- 1-2 文で説明 -->

## 関連仕様
- docs/specs/features/FEAT-XXXX-*.md

## チェックリスト
- [ ] 仕様が承認済み
- [ ] テストが先に書かれ RED → GREEN を確認
- [ ] カバレッジ基準を満たしている
- [ ] lint 違反なし
- [ ] .env.example が更新されている（変数追加時）
```

---

## 6. 仕様書管理

### 配置

```
docs/specs/
├── features/    FEAT-0001-*.md
├── api/         API-0001-*.md
├── ui/          UI-0001-*.md
└── infrastructure/ INFRA-0001-*.md
```

### ライフサイクル

`draft` → `review` → `approved` → `implemented` → `verified`

詳細は [SPECIFICATION_PROCESS.md](SPECIFICATION_PROCESS.md) を参照。

---

## 7. 環境変数管理

| ファイル | Git 管理 | 用途 |
|---------|---------|------|
| `.env.example` | する | テンプレート（値は空/ダミー） |
| `.env` | しない | ローカル実際値 |

### セキュリティルール

- `.env` の Git 管理禁止（.gitignore に含まれている）
- シークレットのハードコーディング禁止
- コミット前に pre-commit hook が秘密情報をスキャン

---

## 8. AI 自律開発の制約

### ユーザー承認必須（MUST ASK）

- 仕様の承認
- AWS リソースの変更・削除
- アーキテクチャの重要な判断
- 新しい gem / npm パッケージの追加
- DB マイグレーションファイルの新規作成（既存マイグレーションの実行は承認不要）
- デプロイ操作
- セキュリティに関わる設定変更
- 課金が発生する可能性のある操作
- 外部公開に関わる設定変更
- Docker 構成変更
- 破壊的 Git 操作
- ファイルの削除

> 完全な一覧は CLAUDE.md の Permission Model を参照。

### 自律実行可能（CAN DO）

- 承認済み仕様に基づくコード実装
- Docker 内でのテスト・lint 実行
- feature ブランチでの Git 操作
- 仕様書のドラフト起草

### エスカレーション

| レベル | 状況 | AI の行動 |
|--------|------|-----------|
| L1 | lint エラー、import 漏れ | 自動修正して続行 |
| L2 | テスト失敗、型エラー | エラー内容と修正案をユーザーに提示 |
| L3 | DB 接続失敗、Docker 異常、セキュリティ問題 | 即時停止、ユーザーに報告 |
