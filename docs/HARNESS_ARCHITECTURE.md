# ハーネスアーキテクチャ設計

> **WARNING: Public Repository** — 本ドキュメントに秘密情報を記載しないこと。

---

## 1. ハーネスエンジニアリングとは

Mitchell Hashimoto が提唱した概念。**AI エージェントがミスをしたとき、そのミスを二度と起こさないように環境側を改善する**技術体系。

本プロジェクトでは「ユーザーがコードを書かず、コードレビューも行わない」完全自律開発を掲げており、ハーネスの設計がプロジェクトの成否を左右する。

### 核心的な考え方

- CLAUDE.md（ルール）は「お願い」にすぎず、守られる保証がない
- 本当に守らせたいことは **機械的に強制** する（hooks, CI, 構造テスト）
- 同じ違反が繰り返されたら **エスカレーション**（ルール → hook → CI ゲート）
- ハーネスの価値は「出力の最良値を上げる」ことではなく「**最低ラインを引き上げる**」こと

---

## 2. ハーネスの5層アーキテクチャ

本プロジェクトのハーネスを5層で設計する。

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

プロジェクトルートの CLAUDE.md が最上位のコントロールパネル。約 2,500 トークンを目安に凝縮する。

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

1. **具体的に書く**: 「良いコードを書け」ではなく「`any` 型は使うな。`@typescript-eslint/no-explicit-any: error`」
2. **理由を書く**: なぜそのルールが必要かを 1 行で
3. **検証手段を書く**: どうやって違反を検出するか（lint ルール名、テストコマンド等）
4. **違反時のエスカレーション先を書く**: L2 フックや L3 CI のどこで強制されるか

---

## 4. L2: フック層の設計

Claude Code の hooks 機能と、Git hooks (Lefthook) を組み合わせる。

### 4.1 Claude Code Hooks (.claude/settings.json)

```jsonc
{
  "hooks": {
    // ファイル編集後に lint/format を自動実行
    "post-edit-lint": {
      "trigger": "after_edit",
      "match": ["api/**/*.rb"],
      "command": "docker compose exec -T api bundle exec rubocop --autocorrect-all --fail-level E {file}"
    },
    "post-edit-lint-ts": {
      "trigger": "after_edit",
      "match": ["web/src/**/*.ts"],
      "command": "docker compose exec -T web npx eslint --fix {file}"
    },

    // リンター設定の改変を防止
    "pre-edit-guard-lint-config": {
      "trigger": "before_edit",
      "match": [".rubocop.yml", ".eslintrc.*", "eslint.config.*", "biome.json"],
      "command": "echo 'BLOCKED: リンター設定の変更にはユーザー承認が必要です' && exit 1"
    },

    // Bash 実行前のガードレール
    "pre-bash-guard": {
      "trigger": "before_bash",
      "match_command": ["rm -rf", "docker compose down", "git push.*--force", "git reset --hard"],
      "command": "echo 'BLOCKED: 破壊的コマンドはユーザー承認が必要です' && exit 1"
    },

    // Push 前の自動テスト・lint
    "pre-push-check": {
      "trigger": "before_bash",
      "match_command": ["git push"],
      "command": "docker compose exec -T api bundle exec rspec --fail-fast && docker compose exec -T web npx jest --bail"
    }
  }
}
```

### 4.2 Git Hooks (Lefthook)

Docker 環境内で実行。母艦への Lefthook インストールは不要（Docker 内で完結）。

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    # 秘密情報スキャン (既存の Claude CLI hook を補完)
    secrets-scan:
      run: |
        docker compose exec -T api bundle exec rails runner \
          'puts "secret scan passed"'
      # 実際には gitleaks を Docker 内で実行

    rubocop:
      glob: "api/**/*.rb"
      run: docker compose exec -T api bundle exec rubocop --fail-level E {staged_files}

    eslint:
      glob: "web/src/**/*.{ts,html}"
      run: docker compose exec -T web npx eslint {staged_files}

    prettier:
      glob: "web/src/**/*.{ts,html,scss}"
      run: docker compose exec -T web npx prettier --check {staged_files}

pre-push:
  parallel: true
  commands:
    rspec:
      run: docker compose exec -T api bundle exec rspec --fail-fast
    jest:
      run: docker compose exec -T web npx jest --bail
    brakeman:
      run: docker compose exec -T api bundle exec brakeman --no-pager -q --exit-on-warn
```

### 4.3 pre-commit hook: 秘密情報スキャン

既存の Claude CLI による秘密情報スキャンを第一防御線として維持。  
加えて `gitleaks` を Docker 内で実行し、多層防御を実現する。

```
[開発者 (AI)] → git commit
  → L2-a: Claude CLI 秘密情報スキャン (既存)
  → L2-b: gitleaks スキャン (補完)
  → L2-c: RuboCop / ESLint (コード品質)
  → commit 成功 or ブロック
```

---

## 5. L3: CI ゲート層の設計

GitHub Actions による最終防御線。PR マージの必須条件。

### 5.1 Required Status Checks（マージブロック）

| チェック | ツール | 閾値 | ブロッキング |
|---------|--------|------|-------------|
| Rails lint | RuboCop | 違反 0 | Yes |
| Angular lint | ESLint | 違反 0 | Yes |
| Rails テスト | RSpec + SimpleCov | branch 95%, line 100% | Yes |
| Angular テスト | Jest | branch 95%, line/func/stmt 100% | Yes |
| Rails セキュリティ | Brakeman | High 以上 0 | Yes |
| Node セキュリティ | npm audit | High 以上 0 | Yes |
| 秘密情報検出 | gitleaks / TruffleHog | 検出 0 | Yes |
| IaC セキュリティ | checkov / tfsec | High 以上 0 | Yes |
| API スキーマ整合性 | committee | 不一致 0 | Yes |
| Docker イメージ | Trivy | Critical/High 0 | Yes |

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

CI でカバレッジレポートを生成し、PR コメントに差分を自動投稿。基準未達でマージブロック。

### 5.3 paths-filter による効率化

```yaml
# 変更されたファイルに応じて実行ジョブを絞り込む
- api/** → Rails 系ジョブのみ
- web/** → Angular 系ジョブのみ
- infra/** → Terraform 系ジョブのみ
- docs/** → ドキュメント lint のみ（マージブロックなし）
```

### 5.4 OpenAPI スキーマ整合性チェック

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

### 6.3 /implement スキルの実行フロー

```
1. 指定された仕様書を読み込む
2. 仕様の status が approved であることを確認
3. テストファイルを作成（仕様 ID をコメントで記載）
4. RED 確認（テストが失敗することを確認）
5. 最小実装を行う
6. GREEN 確認（テストが通ることを確認）
7. カバレッジ確認（基準を満たすことを確認）
8. lint 実行（違反がないことを確認）
9. コミット（Conventional Commits 形式）
```

### 6.4 /pre-push-review スキルの実行フロー

```
1. git diff main...HEAD で変更差分を取得
2. 以下を自己チェック:
   - 仕様に記述されていない振る舞いを実装していないか
   - テストカバレッジ基準を満たしているか
   - セキュリティ上の問題がないか（OWASP Top 10）
   - コミットメッセージが Conventional Commits 形式か
   - .env や秘密情報がコミットされていないか
3. 問題があれば修正、なければ Push 可能と報告
```

---

## 7. L5: エージェント・MCP 層の設計

### 7.1 マルチエージェントレビュー

人間のコードレビューを代替するため、複数のAIエージェントによるクロスレビューを実施。

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
| Docker MCP | コンテナ状態の確認、ログ取得 |

### 7.3 レビューエージェントの指示設計

レビューエージェントには以下を明確に指示:
- プロジェクトの CLAUDE.md と仕様書を読んだ上でレビューすること
- 「良い」「問題ない」という曖昧なフィードバックは禁止
- 具体的な改善提案には必ずコード例を含めること
- 過剰な指摘を避け、**実質的な品質問題** にフォーカスすること

---

## 8. フィードバックループの設計

ハーネスは固定的なものではなく、継続的に改善する。

### 8.1 ミスからの学習サイクル

```
エージェントがミスをする
    ↓
原因を分析する
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

### 8.3 ハーネス改善のトリガー

| トリガー | アクション |
|---------|-----------|
| 同じミスが 3 回発生 | エスカレーション（1 段階上の層に昇格） |
| CI が頻繁に失敗 | フック層で事前検出を追加 |
| レビューで毎回同じ指摘 | ルール追記 + lint ルール化 |
| カバレッジが下がった | CI ゲートの閾値を厳格化 |
| 新しい技術を導入 | 対応するルール・フック・テストを追加 |

---

## 9. 本プロジェクトへの適用ロードマップ

プロジェクトの Phase に合わせてハーネスを段階的に構築する。

### Phase 0（現在）: 基盤構築

- [x] CLAUDE.md 作成
- [x] プロジェクトドキュメント整備
- [ ] `.claude/rules/` にコンテキスト別ルールを配置
- [ ] `.claude/settings.json` に hooks 定義
- [ ] `.claude/commands/` にスキル定義

### Phase 1: ローカル開発環境

- [ ] Docker Compose 構成
- [ ] Lefthook 設定（Docker 内実行）
- [ ] pre-commit hook（秘密情報スキャン + lint）
- [ ] post-edit-lint hook（Claude Code hooks）
- [ ] カバレッジ計測設定

### Phase 2: CI パイプライン

- [ ] GitHub Actions ワークフロー構築
- [ ] paths-filter による効率化
- [ ] カバレッジゲート設定
- [ ] セキュリティスキャン統合
- [ ] PR テンプレート + 自動チェックリスト

### Phase 3: アプリケーション実装

- [ ] /spec-draft スキルの活用開始
- [ ] /implement スキルの活用開始
- [ ] OpenAPI スキーマ整合性チェック導入
- [ ] committee gem によるテスト時検証

### Phase 4-5: インフラ・デプロイ

- [ ] Terraform lint + conftest ポリシー
- [ ] IaC セキュリティスキャン（checkov/tfsec）
- [ ] デプロイ前スモークテストの自動化
- [ ] マルチエージェントレビューの本格運用

---

## 10. 設計判断の記録

ハーネスに関する重要な設計判断は ADR として `docs/adr/` に記録する。

### 初期の設計判断

| 判断 | 理由 |
|------|------|
| Lefthook を Docker 内で実行 | 母艦への変更禁止の原則を遵守 |
| エスカレーションの 3 回ルール | 過剰な制約を避けつつ、繰り返しミスを防止 |
| カバレッジの主指標に分岐カバレッジを採用 | 行カバレッジだけでは複合条件の検証漏れがある |
| マルチエージェントレビューを Phase 3 以降に | まず L1-L3 の基盤を固めてから高度な層に進む |
| CLAUDE.md を 2,500 トークン目安に維持 | コンテキストウィンドウの効率的な利用 |

---

## 参考資料

- [Mitchell Hashimoto — My AI Adoption Journey](https://mitchellh.com/writing/my-ai-adoption-journey)
- [theaktky — ハーネスエンジニアリングで人間のコードレビューをやめる](https://zenn.dev/theaktky/articles/1c6c3b9333117c)
- [ignission — Claude Codeでハーネスエンジニアリングを実践する](https://zenn.dev/ignission/articles/f1c15646c990f1)
- [GMO Developers — ハーネスで縛れ、AIに任せろ](https://developers.gmo.jp/technology/81389/)
- [Speaker Deck — AIエージェント時代のハーネスエンジニアリング Claude Code実装編](https://speakerdeck.com/tame/aiezientoshi-dai-nohanesuenziniaringu-claude-codeshi-zhuang-bian)
