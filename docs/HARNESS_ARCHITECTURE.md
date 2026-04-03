# ハーネスアーキテクチャ設計

> **WARNING: Public Repository** — 本ドキュメントに秘密情報を記載しないこと。

---

## 1. ハーネスエンジニアリングとは

Mitchell Hashimoto が提唱した概念。**AI エージェントがミスをしたとき、そのミスを二度と起こさないように環境側を改善する**技術体系。

本プロジェクトでは「ユーザーがコードを書かず、コードレビューも行わない」完全自律開発を掲げており、ハーネスの設計がプロジェクトの成否を左右する。

### 核心的な考え方

- ルールファイル（CLAUDE.md 等）は「お願い」にすぎず、守られる保証がない
- 本当に守らせたいことは **機械的に強制** する（hooks, CI, 構造テスト）
- 同じ違反が繰り返されたら **エスカレーション**（ルール → hook → CI ゲート）
- ハーネスの価値は「出力の最良値を上げる」ことではなく「**最低ラインを引き上げる**」こと
- **制約が信頼性を生む** — モデルの性能ではなく、環境の設計が出力品質の 80% を決定する

### 権威ある知見に基づく原則

本設計は以下の権威あるソースの原則に基づく。

| ソース | 核心原則 |
|--------|---------|
| Mitchell Hashimoto (HashiCorp 創業者) | ミスのたびに環境側を改善する累積的改善プロセス |
| OpenAI Codex チーム (100万行実験) | 5つの柱: コンテキスト、アーキテクチャ制約、フィードバックループ、ドキュメント、エントロピー管理 |
| Martin Fowler | 5パターン: Knowledge Priming, Design-First, Encoding Standards, Context Anchoring, Feedback Flywheel |
| Kent Beck | TDD が AI エージェント制約の最有効手段。テストがフィードバックループの中核 |
| Stripe (Minions) | 決定論的ゲートと LLM ステップの交互配置。「信頼性はモデルサイズではなく制約の質でスケールする」 |
| Vercel (v0) | ツール 80% 削減で精度 80%→100%。選択肢の制限がモデル出力品質を向上させる |
| GitHub (Copilot Agent) | ドラフト PR フロー、自己承認禁止、構造化アーティファクトによるスコープ制御 |
| Ankit Jain (Aviator) | 5層スイスチーズモデル: 不完全なフィルターを積み重ね、穴が一直線にならないようにする |
| Anthropic | サンドボックス化、プログラム可能な安全性、保守的な権限モデル |

### 定量的根拠

| データ | 出典 |
|--------|------|
| SWE-bench: ハーネス設計の違いで **22 ポイント差**（モデル差は約 1 ポイント） | SWE-Bench Pro |
| 検証ループ追加でタスク完了率 **83%→96%** | Harness Engineering AI |
| ツールフォーマット変更のみで **6.7%→68.3%** | The Harness Problem |
| 複雑なタスクほど効果大: Expert タスクで **+36.2 ポイント** | Harness Engineering AI |
| ハーネス変更は出力品質の **80%** を決定、モデル変更は 10-15% | 複数ソース |

---

## 2. ハーネスの5層アーキテクチャ

本プロジェクトのハーネスを5層で設計する。Ankit Jain の「スイスチーズモデル」に倣い、各層は不完全なフィルターだが積み重ねることで穴を塞ぐ。

```
┌─────────────────────────────────────────────────┐
│  L5: エージェント・MCP 層                         │
│      マルチエージェント連携、外部ツール統合          │
├─────────────────────────────────────────────────┤
│  L4: スキル層                                    │
│      繰り返し作業の型化（カスタムスラッシュコマンド）  │
├─────────────────────────────────────────────────┤
│  L3: CI ゲート層                                 │
│      GitHub Actions による自動検証・マージブロック    │
├─────────────────────────────────────────────────┤
│  L2: フック層                                    │
│      ローカル実行時の自動検査・ガードレール           │
├─────────────────────────────────────────────────┤
│  L1: ルール層                                    │
│      CLAUDE.md + .claude/rules/ による行動指針     │
└─────────────────────────────────────────────────┘
```

> **用語の注意**: DEVELOPMENT_GUIDE.md で定義されている L1/L2/L3 は「AI の行動判断レベル（自動修正/ユーザー提示/即時停止）」であり、本ドキュメントの L1-L5 は「ハーネスの層」を指す。異なる概念なので混同しないこと。

### 各層の特性

| 層 | 強制力 | フィードバック速度 | コスト | 用途 |
|----|--------|-------------------|--------|------|
| L1 ルール | 弱（お願い） | 即時（コンテキスト内） | 低 | 方針・規約・スタイル |
| L2 フック | 強（実行ブロック） | 即時（ローカル） | 低 | 危険操作の防止、lint/format |
| L3 CI ゲート | 強（マージブロック） | 数分 | 中 | テスト、カバレッジ、セキュリティ |
| L4 スキル | 中（手順の標準化） | 即時 | 低 | 定型作業の品質均一化 |
| L5 エージェント | 中（自動レビュー） | 数分 | 中 | マルチ視点の品質検証 |

### エスカレーションルール

**同じ違反が 3 回発生したら、1 段階上の層に昇格させる。**

```
L1 ルール記載 → 3回違反 → L2 フックで強制
L2 フック → すり抜け → L3 CI ゲートで検出
```

---

## 3. L1: ルール層の設計

### 3.1 CLAUDE.md

プロジェクトルートの CLAUDE.md が最上位のコントロールパネル。OpenAI の実験では巨大な指示書は逆効果であり、簡潔な「地図」として機能させ、詳細は `docs/` への参照で補う。現行 CLAUDE.md は 184 行だが、Tech Stack・Commands・Directory Structure 等の構造化テーブルが大部分を占めるため、散文的な指示の肥大化ではない。今後も簡潔さを優先し、必要に応じて `docs/` や `.claude/rules/` への分離を行う。

既存の CLAUDE.md に記載済みの内容:
- Tech Stack
- Development Flow（仕様 → テスト → 実装）
- Coverage Requirements
- Commit Convention
- Permission Model（MUST ASK / CAN DO）
- Commands
- Directory Structure
- Security Rules / Host Protection

### 3.2 .claude/rules/ （コンテキスト別ルール）

CLAUDE.md の肥大化を防ぐため、領域別のルールを分離配置する。

```
.claude/
├── rules/
│   ├── rails.md          # Rails API 固有のコーディング規約
│   ├── angular.md        # Angular 固有のコーディング規約
│   ├── terraform.md      # Terraform / IaC 固有の規約
│   ├── testing.md        # テスト記述の規約
│   ├── security.md       # セキュリティ関連の禁止事項
│   └── git.md            # Git 操作・コミットの規約
├── commands/
│   ├── review-codex.md   # (既存) Codex レビュー
│   └── review-gemini.md  # (既存) Gemini レビュー
└── settings.json
```

### 3.3 ルール記載の原則

OpenAI の教訓: 「マークダウンで『X するな』と書くのは"提案"にすぎないが、X がビルド失敗を引き起こすなら"ルール"になる」

1. **具体的に書く**: 「良いコードを書け」ではなく「`any` 型は使うな。`@typescript-eslint/no-explicit-any: error`」
2. **理由を書く**: なぜそのルールが必要かを 1 行で
3. **検証手段を書く**: どうやって違反を検出するか（lint ルール名、テストコマンド等）
4. **違反時のエスカレーション先を書く**: L2 フックや L3 CI のどこで強制されるか

---

## 4. L2: フック層の設計

Claude Code の hooks 機能と、Git hooks を組み合わせる。

### 4.1 Claude Code Hooks (.claude/settings.json)

Claude Code hooks はイベントベースの設定形式を使用する。トリガーは `PreToolUse`、`PostToolUse` 等のイベント名をキーとし、マッチャーはツール名の正規表現で指定する。

**重要: ツール入力は stdin から JSON として渡される**（環境変数ではない）。フックのコマンドは stdin を `jq` 等で読み取り、`tool_input` オブジェクトからパラメータを取得する。

stdin JSON の構造:
```json
{
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/path/to/file",
    "old_string": "...",
    "new_string": "..."
  }
}
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "file_path=$(cat /dev/stdin | jq -r '.tool_input.file_path // empty') && if [ -n \"$file_path\" ] && echo \"$file_path\" | grep -qE '\\.rb$'; then container_path=\"/app${file_path#$PWD}\" && docker compose exec -T api bundle exec rubocop --autocorrect-all --fail-level E \"$container_path\" 2>&1 | tail -5; fi",
            "timeout": 30
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "file_path=$(cat /dev/stdin | jq -r '.tool_input.file_path // empty') && if [ -n \"$file_path\" ] && echo \"$file_path\" | grep -qE '\\.ts$'; then container_path=\"/app${file_path#$PWD}\" && docker compose exec -T web npx ng lint 2>&1 | tail -5; fi",
            "timeout": 30
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "file_path=$(cat /dev/stdin | jq -r '.tool_input.file_path // empty') && if echo \"$file_path\" | grep -qE '(\\.rubocop\\.yml|\\.eslintrc|eslint\\.config|biome\\.json|\\.tflint\\.hcl)$'; then echo 'BLOCKED: リンター/IaC設定の変更にはユーザー承認が必要です' >&2 && exit 2; fi"
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "file_path=$(cat /dev/stdin | jq -r '.tool_input.file_path // empty') && if echo \"$file_path\" | grep -qE '(docker-compose\\.yml|Dockerfile|config/initializers/cors\\.rb|config/initializers/content_security_policy\\.rb)$'; then echo 'BLOCKED: Docker構成/セキュリティ設定の変更にはユーザー承認が必要です' >&2 && exit 2; fi"
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "file_path=$(cat /dev/stdin | jq -r '.tool_input.file_path // empty') && if echo \"$file_path\" | grep -qE '(Gemfile|package\\.json)$'; then echo 'BLOCKED: パッケージ追加にはユーザー承認が必要です' >&2 && exit 2; fi"
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "file_path=$(cat /dev/stdin | jq -r '.tool_input.file_path // empty') && if echo \"$file_path\" | grep -qE '(config/credentials|config/master\\.key|\\.env)'; then echo 'BLOCKED: 認証・暗号鍵設定の変更にはユーザー承認が必要です' >&2 && exit 2; fi"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "cmd=$(cat /dev/stdin | jq -r '.tool_input.command // empty') && if echo \"$cmd\" | grep -qE '\\brm\\b|\\bgit\\s+rm\\b'; then echo 'BLOCKED: ファイル削除にはユーザー承認が必要です' >&2 && exit 2; fi"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "cmd=$(cat /dev/stdin | jq -r '.tool_input.command // empty') && if echo \"$cmd\" | grep -qE 'git\\s+push\\s+.*--force'; then echo 'BLOCKED: force push は禁止です' >&2 && exit 2; fi"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "cmd=$(cat /dev/stdin | jq -r '.tool_input.command // empty') && if echo \"$cmd\" | grep -qE 'git\\s+reset\\s+--hard'; then echo 'BLOCKED: git reset --hard にはユーザー承認が必要です' >&2 && exit 2; fi"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "cmd=$(cat /dev/stdin | jq -r '.tool_input.command // empty') && if echo \"$cmd\" | grep -qE 'rails\\s+generate\\s+migration'; then echo 'BLOCKED: DB マイグレーション新規作成にはユーザー承認が必要です' >&2 && exit 2; fi"
          }
        ]
      }
    ]
  }
}
```

**重要な仕様:**
- **ツール入力は stdin で渡される**: `cat /dev/stdin | jq -r '.tool_input.file_path'` のように読み取る
- exit code `0` = 成功（続行）、`2` = ブロッキングエラー（阻止）
- stdout は JSON のみ。シェルプロファイルの `echo` がパースを壊す
- `PostToolUse` はツール実行済みのため取り消し不可（検査・ログ用途）
- `PreToolUse` の deny は `bypassPermissions` でも有効（フックは権限より強い）

**Docker パス変換:**
lint フックでは `docker compose exec` を使うため、ホスト側の絶対パスをコンテナ内パスに変換する必要がある。上記例では `container_path="/app${file_path#$PWD}"` でプロジェクトルートからの相対パスを `/app/` にマウントされたコンテナパスに変換している。Docker Compose の `volumes` 設定と一致させること。

### 4.2 Git Hooks (pre-commit)

pre-commit hook は Claude CLI による秘密情報スキャンを第一防御線として維持する（SECURITY_POLICY.md §2 準拠）。

DEVELOPMENT_GUIDE.md に記載の通り、pre-commit hook は `~/dotfiles/hooks/pre-commit` からコピーして `.git/hooks/` にインストールする。この手順はホスト上の `.git/hooks/` ディレクトリへのファイルコピーであり、シェル設定の変更やグローバルツールのインストールではないため「母艦への変更禁止」原則（ホスト環境のシステム設定やグローバル依存への変更禁止）には抵触しない。

```
[開発者 (AI)] → git commit
  → pre-commit hook: Claude CLI 秘密情報スキャン
  → commit 成功 or ブロック
```

> **Lefthook について**: Lefthook のホスト側グローバルインストールは「母艦への変更禁止」原則と矛盾する。Phase 1 でこの問題を解決する方式を ADR として記録する。選択肢: (a) Docker コンテナ内に Lefthook をインストールし entrypoint でフックを登録、(b) pre-commit hook スクリプト内で `docker compose exec` を直接呼び出す、(c) Claude Code hooks で代替する。

### 4.3 MUST ASK 項目のフック化

PROJECT_CHARTER.md / CLAUDE.md で定義された「ユーザー承認が必要」な操作を L2 フックで機械的に強制する。

**CLAUDE.md / PROJECT_CHARTER.md で定義された MUST ASK 項目:**

| MUST ASK 項目 | L2 フック実装 | L3 CI バックアップ |
|---------------|-------------|------------------|
| Docker 構成変更 | PreToolUse: Edit\|Write で docker-compose.yml, Dockerfile | — |
| セキュリティ設定変更（認証、暗号鍵、CORS、CSP） | PreToolUse: Edit\|Write で cors.rb, content_security_policy.rb, credentials/, master.key, .env | CORS/CSP 設定値を検証するテスト / gitleaks |
| 新規パッケージ追加 | PreToolUse: Edit\|Write で Gemfile, package.json | — |
| ファイルの削除 | PreToolUse: Bash で `rm` / `git rm` コマンド（※注） | — |
| DB マイグレーション作成 | PreToolUse: Bash で rails generate migration | strong_migrations |
| main への force push | PreToolUse: Bash で git push --force | ブランチ保護ルール |
| AWS リソースの作成・変更・削除 | L1 ルール記載（Phase 4 で Terraform hook 追加） | conftest (OPA) |
| GitHub Actions secrets / environment の設定 | L1 ルール記載 | GitHub Environment Protection Rules / 管理者権限制限 |
| 外部公開に関わる設定変更 | L1 ルール記載（Phase 4 で Terraform hook 追加） | conftest (OPA) |
| デプロイ操作 | L1 ルール記載 | GitHub Environment 承認ゲート |
| 課金が発生する可能性のある操作 | L1 ルール記載（Phase 4 で Terraform hook 追加） | conftest (OPA) |
| 仕様の承認 / 承認ステータス変更 | L1 ルール記載（Phase 3 で hook 追加検討） | — |
| アーキテクチャの重要な判断 | L1 ルール記載（ADR で記録） | — |

**追加のハーネスガードレール（MUST ASK 定義外だが L2 で保護する項目）:**

| 保護対象 | L2 フック実装 | 根拠 |
|---------|-------------|------|
| リンター設定の変更 | PreToolUse: Edit\|Write でファイルパスチェック | コード品質ゲートの無力化を防止 |
| git reset --hard | PreToolUse: Bash で git reset --hard | 破壊的 Git 操作の防止（DEVELOPMENT_GUIDE.md §8「破壊的 Git 操作」参照） |

> **注（※）**: 「ファイルの削除」の L2 フックは現時点で Bash ツール経由の `rm` / `git rm` コマンドのみをカバーする。Claude Code が他のツール経由でファイルを削除する場合はフックの対象外となるため、実装時に削除系ツールの網羅性を検証し、必要に応じて追加フックを設定する。
>
> **注**: MUST ASK 項目のうち AWS・デプロイ・課金・外部公開に関する操作は、現時点（Phase 0）では L1 ルール（CLAUDE.md の Permission Model 記載）で対応する。Phase 4-5 で Terraform / GitHub Actions を実装する際に L2 フックへ昇格させる。フック化されていない項目は L1 ルールの遵守に依存するため、違反が観察された場合はエスカレーションルールに従い L2 フック化する。

---

## 5. L3: CI ゲート層の設計

GitHub Actions による最終防御線。PR マージの必須条件。Stripe のブループリントパターンに倣い、決定論的ゲートと LLM ステップを交互配置する。

### 5.1 Required Status Checks（マージブロック）

> **⚠ 本一覧は Phase 2 以降の目標構成であり、現行の CI_CD.md とは差分がある。** 現行 CI_CD.md の Required Status Checks は lint, test, security のみ。以下の項目は Phase 2 で CI_CD.md に追加する: Angular SAST、committee、strong_migrations、Actions SHA ピン留め、`paths-filter` によるジョブ絞り込み（CI_CD.md は PR を「フル CI」と記載しているが、本設計では `docs/**` 変更時はドキュメント lint のみ実行）、レビューコメント存在チェック。CI_CD.md の更新は Phase 2 の最初のタスクとして実施する。

| チェック | ツール | 閾値 | ブロッキング |
|---------|--------|------|-------------|
| Rails lint | RuboCop | 違反 0 | Yes |
| Angular lint | `ng lint` (ESLint + @angular-eslint) | 違反 0 | Yes |
| Angular セキュリティ lint | ESLint Security プラグイン | Error 0 | Yes |
| Rails テスト | RSpec + SimpleCov | branch 95%, line 100% | Yes |
| Angular テスト | Jest | branch 95%, line/func/stmt 100% | Yes |
| E2E テスト | Playwright (main push 時のみ) | 失敗 0 | No（デプロイゲート） |
| Rails セキュリティ | Brakeman | High 以上 0（SECURITY_POLICY.md §8 準拠） | Yes |
| Ruby 依存関係 | Bundler Audit | High 以上 0 | Yes |
| Node セキュリティ | npm audit | High 以上 0 | Yes |
| 秘密情報検出 | gitleaks / TruffleHog | 検出 0 | Yes |
| IaC lint | tflint | 違反 0 | Yes |
| IaC セキュリティ | checkov / tfsec | High 以上 0 | Yes |
| IaC ポリシー | conftest (OPA) | 違反 0 | Yes |
| API スキーマ整合性 | committee | 不一致 0 | Yes |
| DB マイグレーション安全性 | strong_migrations | 検出 0 | Yes |
| Docker イメージ | Trivy | Critical/High 0 | Yes |
| Actions SHA ピン留め | カスタムチェック | 未ピン留め 0 | Yes |

### 5.2 カバレッジゲートの設計

```
SimpleCov (Rails)
  ├── branch coverage ≥ 95%
  └── line coverage = 100%

Jest (Angular)
  ├── branch coverage ≥ 95%
  ├── line coverage = 100%
  ├── function coverage = 100%
  └── statement coverage = 100%
```

> **注**: PROJECT_CHARTER.md の成功基準では `100% (line, functions, statements)` を Rails/Angular 区別なく記載している。SimpleCov は function/statement カバレッジをネイティブサポートしないため、Rails 側は line + branch のみで計測する。この判断は ADR として記録する。

CI でカバレッジレポートを生成し、PR コメントに差分を自動投稿。基準未達でマージブロック。

### 5.3 ワークフロー構成

CI_CD.md で定義されたワークフロー分割設計に準拠する。

```
.github/
├── workflows/
│   ├── ci.yml              # メイン CI
│   ├── cd.yml              # デプロイ
│   ├── security.yml        # 定期セキュリティスキャン (毎日 00:00 UTC)
│   └── dependabot-merge.yml
├── actions/
│   ├── setup-rails/action.yml
│   └── setup-angular/action.yml
├── dependabot.yml
└── CODEOWNERS          # 通知用（ブロッキングレビューには使用しない）
```

### 5.4 paths-filter と concurrency による効率化

```yaml
# 変更されたファイルに応じて実行ジョブを絞り込む
- api/** → Rails 系ジョブのみ
- web/** → Angular 系ジョブのみ
- infra/** → Terraform 系ジョブのみ
- docs/** → ドキュメント lint のみ（マージブロックなし）

# concurrency で同一ブランチの古い実行をキャンセル
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### 5.5 定期セキュリティスキャン

PR/push 時のチェックに加え、`security.yml` で毎日 00:00 UTC に以下を定期実行する。

| ツール | 対象 |
|--------|------|
| Bundler Audit | Ruby gems 脆弱性 |
| npm audit | Node packages 脆弱性 |
| Dependabot | 全依存関係（自動 PR 作成） |
| Trivy | Docker イメージ |

### 5.6 デプロイ承認ゲート

GitHub Environments の `production` に Required Reviewers を設定し、デプロイ承認を制御する。DEPLOYMENT.md で定義されたフローに準拠する。

```
CI 全 Green → Docker Build → ECR Push
    → 承認ゲート (GitHub Environment Protection Rules)
    → DB マイグレーション → ECS 更新 → スモークテスト
```

### 5.7 OpenAPI スキーマ整合性チェック

仕様駆動開発の要。API 実装が OpenAPI 定義と一致することを CI で自動検証。

```
仕様書 (docs/specs/api/) → OpenAPI 定義 (docs/openapi/) → committee gem (テスト時検証)
                                                         → CI ゲート (マージブロック)
```

---

## 6. L4: スキル層の設計

Claude Code のカスタムスラッシュコマンドで、繰り返し作業を型化する。

### 6.1 開発フロー系スキル

```
.claude/commands/
├── spec-draft.md       # 仕様書ドラフト起草
├── implement.md        # 仕様 → テスト → 実装の一連フロー
├── pre-push-review.md  # Push 前の自己レビュー
├── review-codex.md     # (既存) Codex によるクロスレビュー
├── review-gemini.md    # (既存) Gemini によるクロスレビュー
└── coverage-check.md   # カバレッジ確認と不足箇所の特定
```

### 6.2 スキルの設計原則

| 原則 | 説明 |
|------|------|
| 冪等性 | 何度実行しても同じ結果になる |
| 自己完結 | 前提条件のチェックをスキル内で行う |
| エラー報告 | 失敗時は何が起きたかを明確にユーザーに報告 |
| トレーサビリティ | 仕様 ID を必ず参照・記録する |

### 6.3 /spec-draft スキルの実行フロー

```
1. 指定されたテーマについて仕様書ドラフトを起草する
2. SPECIFICATION_PROCESS.md のテンプレートに準拠する
3. 自己批判プロンプトを実行する:
   - この仕様で起こりうる問題を 3 つ挙げる
   - 未確認事項 (open questions) を列挙する
4. status: draft でコミットする
5. ユーザーにレビューを依頼する
```

### 6.4 /implement スキルの実行フロー

```
1. 指定された仕様書を読み込む
2. 仕様の status が approved であることを確認
   → approved でなければ即座に停止し報告
3. テストファイルを作成（仕様 ID をコメントで記載）
4. RED 確認（テストが失敗することを確認）
5. 最小実装を行う
6. GREEN 確認（テストが通ることを確認）
7. カバレッジ確認（基準を満たすことを確認）
8. lint 実行（違反がないことを確認）
9. コミット（Conventional Commits 形式、type は feat/fix 等を適切に選択）
   コミットメッセージに Refs: docs/specs/... を含める
```

### 6.5 /pre-push-review スキルの実行フロー

```
1. git diff main...HEAD で変更差分を取得
2. 以下を自己チェック:
   - 仕様に記述されていない振る舞いを実装していないか
   - テストカバレッジ基準を満たしているか（--coverage で確認）
   - セキュリティ上の問題がないか（OWASP Top 10）
   - コミットメッセージが Conventional Commits 形式か
   - .env や秘密情報がコミットされていないか
   - origins '*' や unsafe-eval 等の禁止パターンがないか
3. 問題があれば修正、なければ Push 可能と報告
```

---

## 7. L5: エージェント・MCP 層の設計

### 7.1 マルチエージェントレビュー

人間のコードレビューを代替するため、複数の AI エージェントによるクロスレビューを実施する。Ankit Jain の「作業するエージェントと判断するエージェントを分離する」原則に基づく。

```
[Claude Code (実装)]
    ↓ PR 作成
[レビューエージェント群 (並列実行)]
    ├── Claude (別セッション) — 設計・ロジックレビュー
    ├── Codex — コード品質・パフォーマンスレビュー
    └── Gemini — セキュリティ・エッジケースレビュー
    ↓
[指摘の集約・重複排除]
    ↓
[修正 → 再レビュー → マージ]
```

### 7.2 MCP サーバー連携

| MCP サーバー | 用途 |
|-------------|------|
| GitHub MCP | PR 操作、Issue 管理、CI ステータス確認 |
| PostgreSQL MCP | DB スキーマ確認、クエリ実行（読み取り専用） |

### 7.3 レビューエージェントの指示設計

レビューエージェントには以下を明確に指示:
- プロジェクトの CLAUDE.md と仕様書を読んだ上でレビューすること
- 「良い」「問題ない」という曖昧なフィードバックは禁止
- 具体的な改善提案には必ずコード例を含めること
- 過剰な指摘を避け、**実質的な品質問題** にフォーカスすること
- レビュー前にハーネスエンジニアリングの基本概念を自ら調査すること

### 7.4 PR レビュープロセス

人間のコードレビューに代わるプロセスとして、PR を成果物評価の単位とし、レビュー結果を PR コメントとして記録する。これにより監査証跡が GitHub 上に残り、プロセスの透明性とトレーサビリティを確保する。

#### 全体フロー

```
[ローカル開発]
    ↓ lint / test / security scan（ローカルで GREEN 確認）
    ↓ 自己レビュー（/pre-push-review）
[Push → PR 作成]
    ↓
[PR レビューフェーズ]
    ├── Round N: Codex レビュー → PR コメント投稿
    ├── Round N: Gemini レビュー → PR コメント投稿
    ├── 指摘があれば修正 → コミット → 次ラウンド
    └── 両レビュアーが 0 指摘を返すまで繰り返す
    ↓
[レビュー完了サマリー投稿]
    ↓
[マージ]
```

#### ローカルフェーズ（Push 前）

ローカルでは **機械的チェック** のみ実施する。Codex / Gemini のクロスレビューは PR 上で実施する。

| チェック | ローカル | PR |
|---------|---------|-----|
| lint / format | Yes（自動修正） | Yes（CI） |
| テスト + カバレッジ | Yes | Yes（CI） |
| セキュリティスキャン | Yes | Yes（CI） |
| 秘密情報スキャン | Yes（pre-commit） | Yes（CI） |
| Codex クロスレビュー | No | **Yes** |
| Gemini クロスレビュー | No | **Yes** |
| 仕様整合性チェック | No | **Yes** |

> **ローカルレビューの例外**: ドキュメント PR など CI ゲートが効かない成果物の場合、Push 前にローカルで 1 ラウンドの Codex/Gemini レビューを実施し、明らかな不整合を除去してから PR を作成してもよい。ただしその結果はローカルで消費され PR には記録されないため、PR 上で改めて正式レビューを実施する。

#### PR レビューコメント形式

各ラウンドの結果を以下のテンプレートで PR コメントとして投稿する。

```markdown
## 📋 Review Round N: [レビュアー名]

| 項目 | 内容 |
|------|------|
| レビュアー | Codex (gpt-X.X) / Gemini (model-name) |
| ラウンド | N |
| 対象コミット | abc1234 |
| 指摘数 | X Critical, Y Medium, Z Low |

### 指摘事項

| # | 重要度 | タイトル | 関連ファイル |
|---|--------|---------|-------------|
| 1 | Critical | ... | file.md |

### 修正コミット
- abc1234: 修正内容の説明

### 判定
**PASS** / **FAIL** — 理由
```

#### マージ条件（Exit Criteria）

以下の **すべて** を満たした場合にのみマージ可能:

1. **Codex と Gemini の両方** が最新ラウンドで 0 Critical 指摘を返す
2. Medium 以下の指摘は修正済みまたは意図的な保留（理由を記載）
3. レビュー完了サマリーが PR コメントとして投稿済み

#### レビュー完了サマリー形式

```markdown
## 🏁 レビュー完了サマリー

| レビュアー | ラウンド数 | 最終指摘数 | 判定 |
|-----------|-----------|-----------|------|
| Codex     | N         | 0         | PASS |
| Gemini    | N         | 0         | PASS |

**総修正数**: XX 件
**マージ承認**: 全 Exit Criteria 充足 ✅
```

#### コンテンツ別レビュー基準

| コンテンツ | Codex の焦点 | Gemini の焦点 |
|-----------|-------------|--------------|
| ドキュメント | 文書間整合性、用語統一、完全性 | 技術的正確性、セキュリティ記述、実行可能性 |
| コード | 仕様準拠、テストカバレッジ、コード品質 | OWASP Top 10、認証/認可、アーキテクチャ整合 |
| 混合 | コード基準を適用 + ドキュメントと仕様の整合性 | コード基準を適用 |

#### セーフガード

| リスク | 対策 |
|--------|------|
| 自己評価バイアス | レビュー実行と判定を分離。レビュアーの verdict を機械的に集計 |
| レビュアー疲労 | 3 ラウンド連続 PASS 後に再度 FAIL が出ない場合、チェックリスト形式の確認を強制 |
| 無限ループ | **最大 5 ラウンド**。超過時は `review/stalled` ラベルを付与しユーザーに判断を委ねる |
| レート制限 (429) | 60 秒後にリトライ → 失敗なら代替モデル（gemini-2.5-flash → gemini-2.0-flash）→ 全滅時は Codex 2 回目で代替し、代替を PR コメントに記録 |
| レビュアー間の矛盾 | 独立拒否権方式。片方が FAIL なら FAIL。両方 PASS で初めてマージ可能 |

#### ハーネス層との接続

PR レビューは **L5 が実行し、L4 フォーマットで記録し、L3 が存在を強制する** 3 層にまたがる制御として機能する。

```
L5 エージェント層: Codex / Gemini がレビューを実行
    ↓ 構造化された PR コメントを出力
L4 スキル層: /review-codex, /review-gemini が出力フォーマットを標準化
    ↓ PR コメントとして GitHub に記録
L3 CI ゲート層: レビューコメント存在チェック（Phase 2 で CI に統合）
```

> **Phase 2 での CI 統合**: GitHub Actions でマージ前に「レビュー完了サマリー」コメントの存在を自動チェックする。これにより「レビューなしマージ」を構造的に不可能にする。

---

## 8. フィードバックループの設計

ハーネスは固定的なものではなく、継続的に改善する。OpenAI の教訓: 「もっと頑張れ」ではなく「何の能力が不足しているか? どうすればエージェントにとって読みやすく、強制力のあるものにできるか?」を問う。

### 8.1 ミスからの学習サイクル

```
エージェントがミスをする
    ↓
原因を分析する（何のツール/ガードレール/ドキュメントが不足していたか）
    ↓
どの層で防げたかを判断する
    ↓
該当する層にルール/フック/テストを追加する
    ↓
ADR に判断を記録する (docs/adr/)
```

### 8.2 ハーネスメトリクス

| メトリクス | 計測方法 | 目標 |
|-----------|---------|------|
| CI パス率 | GitHub Actions 成功率 | 90%+ |
| フック阻止回数 | hooks のログ | 減少傾向 |
| レビュー指摘の再発率 | Issue ラベル | 0% |
| カバレッジ推移 | SimpleCov / Jest レポート | 基準維持 |
| 秘密情報インシデント | gitleaks 検出数 | 0 |

### 8.3 通知設計

CI_CD.md で定義された通知設計に準拠する。

| イベント | 通知先 | 優先度 |
|---------|--------|--------|
| CI 失敗 (PR) | GitHub PR Status | 通常 |
| CD 失敗 (main) | Slack / Email | 緊急 |
| セキュリティ検出 | Slack / GitHub Issue | 緊急 |
| デプロイ成功 | Slack (簡潔) | 通常 |

### 8.4 ハーネス改善のトリガー

| トリガー | アクション |
|---------|-----------|
| 同じミスが 3 回発生 | エスカレーション（1 段階上の層に昇格） |
| CI が頻繁に失敗 | フック層で事前検出を追加 |
| レビューで毎回同じ指摘 | ルール追記 + lint ルール化 |
| カバレッジが下がった | CI ゲートの閾値を厳格化 |
| 新しい技術を導入 | 対応するルール・フック・テストを追加 |

### 8.5 エントロピー管理

OpenAI の5番目の柱。ドキュメントの不整合やアーキテクチャ制約の違反を定期的に検出する。

- ドキュメント間の整合性を定期的にレビュー（CLAUDE.md ↔ 各 docs/ の記述一致）
- 使われなくなったルールやフックを削除
- モデルが改善されるにつれ不要になった制約を段階的に除去

---

## 9. アンチパターン

実世界の事例から学んだ「やってはいけないこと」。

| アンチパターン | 問題 | 対策 |
|--------------|------|------|
| 過剰な事前設計 | 実際の障害発生前に理想的な設定を作り込む | シンプルに始め、障害を観察してからインフラを追加 |
| コンテキストの過剰注入 | 巨大な CLAUDE.md がコンテキストを圧迫 | 100 行の地図 + docs/ への参照 |
| ドキュメントの腐敗 | 古くなった指示をエージェントが信じる | 定期的な整合性レビュー + エントロピー管理 |
| 自己評価バイアス | エージェントが自分の作業を高く評価 | Generator と Evaluator を分離 |
| 一発完了の試み | 全機能を一度に実装しようとする | 1 セッション = 1 機能の制約 |
| 失敗アクションの削除 | エラーを消すと学習信号を失う | 失敗とスタックトレースをコンテキストに残す |

---

## 10. 本プロジェクトへの適用ロードマップ

プロジェクトの Phase に合わせてハーネスを段階的に構築する。Manus の教訓に倣い、シンプルに始めて障害を観察してから追加する。

### Phase 0（現在）: 基盤構築

PROJECT_CHARTER.md §7 Phase 0 の成果物に準拠し、ハーネス固有の成果物を追加。

- [x] CLAUDE.md 作成
- [x] プロジェクトドキュメント整備
- [x] Git リポジトリ設定、.gitignore
- [x] pre-commit hook（秘密情報スキャン — DEVELOPMENT_GUIDE.md の手順でインストール済み）
- [ ] `.claude/rules/` にコンテキスト別ルールを配置
- [ ] `.claude/settings.json` に hooks 定義
- [ ] `.claude/commands/` にスキル定義

### Phase 1: ローカル開発環境

- [ ] Docker Compose 構成
- [ ] Claude Code hooks（PostToolUse lint, PreToolUse ガード）
- [ ] Lefthook 実行方式の ADR 作成と実装
- [ ] カバレッジ計測設定

### Phase 2: CI パイプライン

- [ ] GitHub Actions ワークフロー構築（ci.yml, security.yml）
- [ ] paths-filter + concurrency 設定
- [ ] カバレッジゲート設定
- [ ] セキュリティスキャン統合（Brakeman, Bundler Audit, npm audit, gitleaks）
- [ ] Dependabot 設定
- [ ] PR テンプレート + 自動チェックリスト

### Phase 3: アプリケーション実装

- [ ] /spec-draft スキルの活用開始
- [ ] /implement スキルの活用開始
- [ ] OpenAPI スキーマ整合性チェック導入（committee gem）
- [ ] マルチエージェントレビューの導入（Codex + Gemini）

### Phase 4: AWS インフラ構築

- [ ] Terraform lint（tflint）+ conftest ポリシー
- [ ] IaC セキュリティスキャン（checkov/tfsec）
- [ ] コスト見積もりチェック導入

### Phase 5: CD パイプラインとデプロイ

- [ ] cd.yml ワークフロー構築
- [ ] デプロイ承認ゲート（GitHub Environment Protection Rules）
- [ ] デプロイ前スモークテストの自動化
- [ ] ロールバック戦略の検証

---

## 11. 設計判断の記録

ハーネスに関する重要な設計判断は ADR として `docs/adr/` に記録する。

### 初期の設計判断

| 判断 | 理由 |
|------|------|
| エスカレーションの 3 回ルール | 過剰な制約を避けつつ、繰り返しミスを防止 |
| カバレッジの主指標に分岐カバレッジを採用 | 行カバレッジだけでは複合条件の検証漏れがある |
| マルチエージェントレビューを Phase 3 から導入 | まず L1-L3 の基盤を固めてから L5 に進む |
| CLAUDE.md を簡潔に維持（地図であり百科事典ではない） | OpenAI の教訓: 巨大な指示書は逆効果。構造化テーブルは許容し、散文的指示の肥大化を防ぐ |
| Brakeman 閾値を SECURITY_POLICY.md に準拠し High 以上で統一 | セキュリティポリシーが権威ドキュメントであり、閾値はそこに従う |
| Generator-Evaluator 分離原則 | 自己評価バイアスの排除 |

---

## 参考資料

### 一次ソース（AI プラットフォーム公式）

- [OpenAI — Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- [OpenAI — Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [OpenAI — Custom instructions with AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
- [Anthropic — Claude Code Hooks Documentation](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [Anthropic — Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [GitHub — Copilot Coding Agent](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)

### 著名エンジニア・研究者

- [Mitchell Hashimoto — My AI Adoption Journey](https://mitchellh.com/writing/my-ai-adoption-journey)
- [Martin Fowler — Patterns for Reducing Friction in AI-Assisted Development](https://martinfowler.com/articles/reduce-friction-ai/)
- [Martin Fowler — Harness Engineering](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [Kent Beck — TDD, AI agents and coding](https://newsletter.pragmaticengineer.com/p/tdd-ai-agents-and-coding-with-kent)
- [Simon Willison — Agentic Engineering Patterns](https://simonw.substack.com/p/agentic-engineering-patterns)
- [Ankit Jain — How to Kill the Code Review](https://www.latent.space/p/reviews-dead)
- [Addy Osmani — How to write a good spec for AI agents](https://addyosmani.com/blog/good-spec/)

### 企業事例

- [Stripe — Minions: autonomous coding agents](https://stripe.com/blog/can-ai-agents-build-real-stripe-integrations)
- [Vercel — We removed 80% of our agent's tools](https://vercel.com/blog/we-removed-80-percent-of-our-agents-tools)
- [Datadog — Closing the Verification Loop](https://www.datadoghq.com/blog/ai/harness-first-agents/)
- [LangChain — Improving Deep Agents with Harness Engineering](https://blog.langchain.com/improving-deep-agents-with-harness-engineering/)

### ベンチマーク・研究

- [SWE-Bench Pro Analysis](https://www.morphllm.com/swe-bench-pro)
- [The Harness Problem (6.7%→68.3%)](https://blog.can.ac/2026/02/12/the-harness-problem/)
- [Google DeepMind — Intelligent AI Delegation](https://arxiv.org/html/2602.11865v1)

### コミュニティ

- [theaktky — ハーネスエンジニアリングで人間のコードレビューをやめる](https://zenn.dev/theaktky/articles/1c6c3b9333117c)
- [ignission — Claude Codeでハーネスエンジニアリングを実践する](https://zenn.dev/ignission/articles/f1c15646c990f1)
- [GMO Developers — ハーネスで縛れ、AIに任せろ](https://developers.gmo.jp/technology/81389/)
