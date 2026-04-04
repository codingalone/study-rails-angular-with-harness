# 開発計画: 契約・請求管理システム

> **WARNING: Public Repository** — 機密情報を記載しないこと。
>
> **Note:** ドメインモデル・ビジネスルールは [APP_REQUIREMENTS.md](APP_REQUIREMENTS.md)、
> 技術設計は [APPLICATION_ARCHITECTURE.md](APPLICATION_ARCHITECTURE.md) を参照。

---

## 1. プロジェクト全体像

### 1.1 フェーズ構成

```
Phase 0: 基盤ドキュメント整備        [完了]
Phase 1: ローカル開発環境構築        ← 次に着手
Phase 2: CI パイプライン構築
Phase 3: アプリケーション実装
  ├── 3A: 共通基盤 (認証・認可・共通コンポーネント)
  ├── 3B: 顧客管理
  ├── 3C: 契約管理
  ├── 3D: 請求・入金管理
  └── 3E: ダッシュボード
Phase 4: AWS インフラ構築            [後続]
Phase 5: CD パイプライン + デプロイ   [後続]
```

### 1.2 フェーズ間の依存関係

```
Phase 0 ──→ Phase 1 ──→ Phase 2 ──→ Phase 3A ──→ Phase 3B ──→ Phase 3C ──→ Phase 3D ──→ Phase 3E
                                                                  ↑
                                                     (使用量記録は 3C と並行可)
                                         Phase 4 ──→ Phase 5
                                      (Phase 2 完了後、Phase 3 と並行可)
```

---

## 2. Phase 1: ローカル開発環境構築

### 2.1 目的

Docker Compose で Rails API + Angular SPA + PostgreSQL + Redis の開発環境を構築し、テスト・lint がコンテナ内で実行できる状態にする。

### 2.2 作業項目

| # | 作業 | PR | 成果物 |
|---|------|-----|--------|
| 1-1 | Docker Compose 基本構成 | `chore/docker-compose-setup` | `docker-compose.yml`, 各 `Dockerfile` |
| 1-2 | Rails API プロジェクト初期化 | `chore/rails-api-init` | `api/` ディレクトリ一式 |
| 1-3 | Angular SPA プロジェクト初期化 | `chore/angular-web-init` | `web/` ディレクトリ一式 |
| 1-4 | RuboCop + ESLint + Prettier 設定 | `chore/lint-config` | `.rubocop.yml`, `.eslintrc.json`, `.prettierrc` |
| 1-5 | RSpec + SimpleCov 設定 | `chore/rspec-setup` | `spec/` 初期設定, `.simplecov` |
| 1-6 | Jest + カバレッジ設定 | `chore/jest-setup` | `jest.config.ts`, カバレッジ閾値 |
| 1-7 | Playwright E2E 基盤 | `chore/playwright-setup` | `e2e/` ディレクトリ一式 |
| 1-8 | ヘルスチェックエンドポイント | `feat/health-check` | `GET /api/v1/health` |

### 2.3 成果物

- `docker compose up -d` で全サービスが起動する
- `docker compose exec api bundle exec rspec` が実行可能
- `docker compose exec web npx jest --coverage` が実行可能
- `docker compose exec api bundle exec rubocop` が実行可能
- `docker compose exec web npx ng lint` が実行可能
- `GET /api/v1/health` が 200 を返す

### 2.4 ユーザー承認が必要な項目

- Docker Compose 構成（ポート、ボリューム、ネットワーク）
- 新規 gem の追加（rspec-rails, simplecov, rubocop, factory_bot_rails, shoulda-matchers, pundit, rack-attack, bullet, strong_migrations, committee 等）
- 新規 npm パッケージの追加（jest-preset-angular, @testing-library/angular, eslint 関連）
- DB マイグレーションファイルの新規作成

---

## 3. Phase 2: CI パイプライン構築

### 3.1 目的

GitHub Actions で lint, テスト, セキュリティスキャン, カバレッジゲートを自動実行する。

### 3.2 作業項目

| # | 作業 | PR | 成果物 |
|---|------|-----|--------|
| 2-1 | CI ワークフロー (lint + test) | `ci/github-actions-ci` | `.github/workflows/ci.yml` |
| 2-2 | セキュリティスキャンワークフロー | `ci/security-scan` | `.github/workflows/security.yml` |
| 2-3 | PR テンプレート + CODEOWNERS | `chore/pr-template` | `.github/PULL_REQUEST_TEMPLATE.md` |
| 2-4 | paths-filter + キャッシュ最適化 | `ci/optimize-workflow` | ワークフロー更新 |

### 3.3 成果物

- PR 作成時に自動で CI が実行される
- カバレッジゲートが機能する
- セキュリティスキャンが定期実行される

---

## 4. Phase 3: アプリケーション実装

### 4.1 サブフェーズの依存関係グラフ

```
Phase 3A: 共通基盤
  │
  ├── FEAT-0001 ユーザー管理 (認証)
  ├── FEAT-0002 ロールベース認可
  ├── FEAT-0003 共通エラーハンドリング
  └── FEAT-0004 共通 UI コンポーネント (レイアウト、ナビゲーション)
        │
        v
Phase 3B: 顧客管理
  │
  ├── FEAT-0010 顧客 CRUD
  └── FEAT-0011 顧客検索・一覧
        │
        v
Phase 3C: 契約管理
  │
  ├── FEAT-0020 契約 CRUD + 状態遷移
  ├── FEAT-0021 契約明細 (固定月額/従量課金/段階制)
  ├── FEAT-0022 使用量記録
  └── FEAT-0023 契約自動更新 (日次バッチ)
        │
        v
Phase 3D: 請求・入金管理
  │
  ├── FEAT-0030 請求書自動生成 (月次バッチ)
  ├── FEAT-0031 請求明細
  ├── FEAT-0032 入金管理
  └── FEAT-0033 入金消込
        │
        v
Phase 3E: ダッシュボード + 通知
  │
  ├── FEAT-0040 ダッシュボード
  └── FEAT-0041 通知 (ログ出力)
```

### 4.2 実装タイムライン (テキストガントチャート)

凡例: `[====]` = 実装期間, `S` = 仕様策定, `R` = レビュー待ち

```
機能                        Week1   Week2   Week3   Week4   Week5   Week6   Week7   Week8
─────────────────────────── ─────── ─────── ─────── ─────── ─────── ─────── ─────── ───────
Phase 1: Docker + 開発環境  [=====]
Phase 2: CI パイプライン            [===]
                                         |
3A: 認証 (FEAT-0001)               S[R][========]
3A: 認可 (FEAT-0002)                        S[R][======]
3A: エラーハンドリング (0003)                S[R][====]
3A: 共通 UI (FEAT-0004)                         S[R][======]
                                                           |
3B: 顧客 CRUD (FEAT-0010)                                 S[R][========]
3B: 顧客検索 (FEAT-0011)                                        S[R][====]
                                                                        |
3C: 契約 CRUD (FEAT-0020)                                              S[R][========]
3C: 契約明細 (FEAT-0021)                                                     S[R][======]
3C: 使用量記録 (FEAT-0022)                                                   S[R][====]
                                                                                      |
─────────────────────────── ─────── ─────── ─────── ─────── ─────── ─────── ─────── ───────
機能                        Week9   Week10  Week11  Week12
─────────────────────────── ─────── ─────── ─────── ───────
3D: 請求書生成 (FEAT-0030)  S[R][========]
3D: 請求明細 (FEAT-0031)          S[R][====]
3D: 入金管理 (FEAT-0032)                S[R][======]
3D: 入金消込 (FEAT-0033)                      S[R][====]
                                                     |
3E: ダッシュボード (0040)                            S[R][======]
3E: 通知 (FEAT-0041)                                      S[R][==]
```

> **Note:** 各機能の実装期間は仕様承認の待ち時間を含む。AI の実装速度は速いが、仕様レビュー・承認がボトルネックになる可能性がある。

---

## 5. 仕様書一覧

### 5.1 機能仕様 (FEAT)

| ID | タイトル | サブフェーズ | 依存先 | 概要 |
|----|---------|------------|--------|------|
| FEAT-0001 | ユーザー認証 | 3A | なし | JWT 認証 (RS256)。サインアップ、ログイン、ログアウト、トークンリフレッシュ |
| FEAT-0002 | ロールベース認可 | 3A | FEAT-0001 | admin/accountant/sales ロール。Pundit による認可ポリシー |
| FEAT-0003 | 共通エラーハンドリング | 3A | なし | API の統一エラーレスポンス形式。バリデーションエラー、認証エラー、404、500 |
| FEAT-0004 | 共通 UI レイアウト | 3A | FEAT-0001 | アプリシェル、ナビゲーション、認証状態表示、ロール別メニュー |
| FEAT-0010 | 顧客管理 CRUD | 3B | FEAT-0001, FEAT-0002 | 顧客の作成・参照・更新・削除。論理削除 |
| FEAT-0011 | 顧客検索・一覧 | 3B | FEAT-0010 | 顧客名/コード検索、ページネーション、ソート |
| FEAT-0020 | 契約管理 | 3C | FEAT-0010 | 契約 CRUD。状態遷移: draft -> active -> suspended -> terminated |
| FEAT-0021 | 契約明細 | 3C | FEAT-0020 | 固定月額/従量課金/段階制の料金タイプ。契約への明細紐付け |
| FEAT-0022 | 使用量記録 | 3C | FEAT-0021 | 従量課金の使用量データ記録。API による外部連携想定 |
| FEAT-0023 | 契約自動更新 | 3C | FEAT-0020 | 日次バッチ。auto_renew=true の契約の end_date を延長 |
| FEAT-0030 | 請求書自動生成 | 3D | FEAT-0021, FEAT-0022 | 月次バッチで請求書を生成。Sidekiq ジョブ |
| FEAT-0031 | 請求明細 | 3D | FEAT-0030 | 請求書の明細行。契約明細ごとの金額計算結果 |
| FEAT-0032 | 入金管理 | 3D | FEAT-0030 | 入金データの登録・一覧。入金方法、入金日、金額 |
| FEAT-0033 | 入金消込 | 3D | FEAT-0032 | 入金と請求書の突合。全額消込/一部消込/過入金の処理 |
| FEAT-0040 | ダッシュボード | 3E | FEAT-0030, FEAT-0032 | 売上集計、未入金一覧、契約状況サマリー |
| FEAT-0041 | 通知 | 3E | FEAT-0030, FEAT-0032 | 請求書発行通知、入金期限通知。ログ出力で代替。内部ジョブ専用（API エンドポイント不要） |

### 5.2 API 仕様 (API)

| ID | タイトル | 関連 FEAT | エンドポイント概要 |
|----|---------|-----------|-------------------|
| API-0001 | 認証 API | FEAT-0001 | `POST /sessions`, `DELETE /sessions`, `POST /sessions/refresh` |
| API-0002 | ユーザー管理 API | FEAT-0001, FEAT-0002 | `GET /users`, `GET/PATCH /users/:id` |
| API-0003 | 顧客 API | FEAT-0010, FEAT-0011 | `GET/POST/PATCH/DELETE /clients` (クエリパラメータで検索) |
| API-0004 | 契約 API | FEAT-0020 | `GET/POST/PATCH /contracts`, `POST /contracts/:id/activate` 等の状態遷移 |
| API-0005 | 契約明細 API | FEAT-0021 | `GET/POST/PATCH/DELETE /contracts/:id/contract_items` |
| API-0006 | 使用量記録 API | FEAT-0022 | `POST/GET /usage_records` |
| API-0007 | 請求書 API | FEAT-0030, FEAT-0031 | `GET /invoices`, `GET /invoices/:id`, `POST /invoices`（手動作成）, `PATCH /invoices/:id`（更新）, `POST /invoices/generate`（月次バッチ） |
| API-0008 | 入金 API | FEAT-0032 | `GET/POST /payments` |
| API-0009 | ダッシュボード API | FEAT-0040 | `GET /dashboard/summary`, `GET /dashboard/revenue`, `GET /dashboard/receivables` |

### 5.3 UI 仕様 (UI)

| ID | タイトル | 関連 FEAT | 画面概要 |
|----|---------|-----------|---------|
| UI-0001 | ログイン画面 | FEAT-0001 | メール/パスワード入力、バリデーション、エラー表示 |
| UI-0002 | アプリレイアウト | FEAT-0004 | サイドバー、ヘッダー、ロール別メニュー、ログアウト |
| UI-0003 | ユーザー管理画面 | FEAT-0002 | ユーザー一覧、ロール変更 (admin のみ) |
| UI-0004 | 顧客一覧・検索画面 | FEAT-0011 | テーブル、検索フィルター、ページネーション |
| UI-0005 | 顧客詳細・編集画面 | FEAT-0010 | フォーム、バリデーション、関連契約表示 |
| UI-0006 | 契約一覧画面 | FEAT-0020 | 契約テーブル、ステータスフィルター |
| UI-0007 | 契約詳細・編集画面 | FEAT-0020, FEAT-0021 | 契約フォーム、明細管理、状態遷移ボタン |
| UI-0008 | 使用量記録画面 | FEAT-0022 | 使用量入力フォーム、履歴テーブル |
| UI-0009 | 請求書一覧画面 | FEAT-0030 | 請求書テーブル、ステータスフィルター |
| UI-0010 | 請求書詳細画面 | FEAT-0031 | 請求明細テーブル、PDF プレビュー (将来) |
| UI-0011 | 入金管理画面 | FEAT-0032, FEAT-0033 | 入金一覧、消込操作 |
| UI-0012 | ダッシュボード画面 | FEAT-0040 | チャート、サマリーカード、未入金アラート |

---

## 6. PR 粒度の方針

### 6.1 基本原則

**1 仕様書 = 1 PR** を基本とする。ただし、各 FEAT 仕様は API + UI を包含するため、以下の分割ルールを適用する。

### 6.2 分割ルール

| 条件 | PR 分割方針 |
|------|------------|
| 変更ファイル数 < 30 | 1 PR (API + UI を含む) |
| 変更ファイル数 30-50 | API と UI を分離して 2 PR |
| 変更ファイル数 > 50 | API、UI、E2E で 3 PR に分離 |

### 6.3 実際の分割計画

| FEAT | 想定 PR 数 | 分割方針 |
|------|-----------|---------|
| FEAT-0001 (認証) | 3 PR | API (モデル + エンドポイント) / UI (ログイン画面) / E2E |
| FEAT-0002 (認可) | 2 PR | API (Pundit ポリシー) / UI (メニュー制御) |
| FEAT-0003 (エラー) | 1 PR | API のみ (共通エラーハンドラー) |
| FEAT-0004 (共通 UI) | 1 PR | UI のみ (レイアウトコンポーネント) |
| FEAT-0010 (顧客 CRUD) | 2 PR | API / UI |
| FEAT-0011 (顧客検索) | 1 PR | API + UI (検索は小規模) |
| FEAT-0020 (契約管理) | 2 PR | API (状態遷移含む) / UI |
| FEAT-0021 (契約明細) | 2 PR | API (料金タイプ) / UI |
| FEAT-0022 (使用量) | 1 PR | API + UI |
| FEAT-0030 (請求書生成) | 2 PR | API (バッチジョブ) / UI |
| FEAT-0031 (請求明細) | 1 PR | API + UI (FEAT-0030 の延長) |
| FEAT-0032 (入金管理) | 2 PR | API / UI |
| FEAT-0033 (入金消込) | 2 PR | API (消込ロジック) / UI |
| FEAT-0040 (ダッシュボード) | 2 PR | API (集計クエリ) / UI (チャート) |
| FEAT-0041 (通知) | 1 PR | API のみ (ログ出力) |

**合計: 約 25 PR** (Phase 3 のみ)

---

## 7. データベース設計概要

### 7.1 主要テーブルと依存関係

```
users ─────────────────────────────────────────────────────┐
  │                                                        │
  v                                                        │
clients ──→ contracts ──→ contract_items ──→ tiered_rates   │
                  │              │                          │
                  │              v                          │
                  │         usage_records                   │
                  │              │                          │
                  v              v                          │
             invoices ──→ invoice_items                     │
                  │                                        │
                  v                                        │
             payments ─────────────────────────────────────┘
```

### 7.2 テーブル一覧

| テーブル | 主要カラム | 備考 |
|---------|-----------|------|
| users | email, password_digest, role, name | enum: admin/accountant/sales |
| clients | code, name, address, phone, email | 論理削除 (active フラグ) |
| contracts | client_id, status, start_date, end_date | 状態遷移管理 |
| contract_items | contract_id, name, billing_type, unit_price, quantity | enum: fixed/usage/tiered |
| tiered_rates | contract_item_id, lower_bound, upper_bound, unit_price | 段階制料金定義 |
| usage_records | contract_item_id, period_start, period_end, quantity | 従量課金の使用量 |
| invoices | contract_id, issue_date, due_date, total_amount, status | enum: draft/issued/paid/overdue/void |
| invoice_items | invoice_id, contract_item_id, name, amount | 請求明細 |
| payments | invoice_id, amount, payment_date, payment_method | 入金データ (Invoice に 1:N) |

---

## 8. 各フェーズの成果物リスト

### Phase 1: ローカル開発環境

| 成果物 | ファイル/ディレクトリ |
|--------|---------------------|
| Docker Compose 設定 | `docker-compose.yml` |
| Rails API Dockerfile | `api/Dockerfile` |
| Angular Dockerfile | `web/Dockerfile` |
| E2E Dockerfile | `e2e/Dockerfile` |
| Rails プロジェクト | `api/` (Gemfile, app/, config/, spec/ 等) |
| Angular プロジェクト | `web/` (package.json, src/ 等) |
| E2E プロジェクト | `e2e/` (playwright.config.ts 等) |
| RuboCop 設定 | `api/.rubocop.yml` |
| ESLint 設定 | `web/.eslintrc.json` |
| Prettier 設定 | `web/.prettierrc` |
| SimpleCov 設定 | `api/spec/spec_helper.rb` (SimpleCov 設定含む) |
| Jest 設定 | `web/jest.config.ts` |
| 環境変数テンプレート | `.env.example` |

### Phase 2: CI パイプライン

| 成果物 | ファイル/ディレクトリ |
|--------|---------------------|
| CI ワークフロー | `.github/workflows/ci.yml` |
| セキュリティスキャン | `.github/workflows/security.yml` |
| Composite Actions | `.github/actions/setup-rails/action.yml`, `.github/actions/setup-angular/action.yml` |
| PR テンプレート | `.github/PULL_REQUEST_TEMPLATE.md` |
| Dependabot 設定 | `.github/dependabot.yml` |
| CODEOWNERS | `.github/CODEOWNERS` |

### Phase 3A: 共通基盤

| 成果物 | 主要ファイル |
|--------|------------|
| User モデル + マイグレーション | `api/app/models/user.rb`, `api/db/migrate/*_create_users.rb` |
| 認証コントローラー | `api/app/controllers/api/v1/auth_controller.rb` |
| JWT サービス | `api/app/services/jwt_service.rb` |
| Pundit ポリシー基盤 | `api/app/policies/application_policy.rb` |
| 共通エラーハンドラー | `api/app/controllers/concerns/error_handler.rb` |
| Angular 認証サービス | `web/src/app/core/services/auth.service.ts` |
| Angular 認証ガード | `web/src/app/core/guards/auth.guard.ts` |
| Angular HTTP インターセプター | `web/src/app/core/interceptors/auth.interceptor.ts` |
| ログイン画面 | `web/src/app/features/auth/login/` |
| アプリレイアウト | `web/src/app/core/layout/` |
| OpenAPI 定義 (認証) | `docs/openapi/auth.yaml` |
| 仕様書 4 件 | `docs/specs/features/FEAT-0001...0004` |
| API 仕様書 2 件 | `docs/specs/api/API-0001, API-0002` |
| UI 仕様書 3 件 | `docs/specs/ui/UI-0001...UI-0003` |

### Phase 3B: 顧客管理

| 成果物 | 主要ファイル |
|--------|------------|
| Client モデル + マイグレーション | `api/app/models/client.rb` |
| 顧客コントローラー | `api/app/controllers/api/v1/clients_controller.rb` |
| 顧客シリアライザ | `api/app/serializers/client_serializer.rb` |
| Angular 顧客サービス | `web/src/app/features/clients/services/client.service.ts` |
| 顧客一覧コンポーネント | `web/src/app/features/clients/client-list/` |
| 顧客詳細コンポーネント | `web/src/app/features/clients/client-detail/` |
| 仕様書 2 件 | `docs/specs/features/FEAT-0010, FEAT-0011` |

### Phase 3C: 契約管理

| 成果物 | 主要ファイル |
|--------|------------|
| Contract モデル + 状態遷移 | `api/app/models/contract.rb` |
| ContractItem モデル | `api/app/models/contract_item.rb` |
| TieredRate モデル | `api/app/models/tiered_rate.rb` |
| UsageRecord モデル | `api/app/models/usage_record.rb` |
| 契約コントローラー | `api/app/controllers/api/v1/contracts_controller.rb` |
| Angular 契約画面群 | `web/src/app/features/contracts/` |
| 仕様書 3 件 | `docs/specs/features/FEAT-0020...FEAT-0022` |

### Phase 3D: 請求・入金管理

| 成果物 | 主要ファイル |
|--------|------------|
| Invoice モデル | `api/app/models/invoice.rb` |
| InvoiceItem モデル | `api/app/models/invoice_item.rb` |
| Payment モデル | `api/app/models/payment.rb` |
| 請求書生成サービス | `api/app/services/invoice_generation_service.rb` |
| 請求書生成ジョブ | `api/app/jobs/invoice_generation_job.rb` |
| 消込サービス | `api/app/services/payments/reconciliation_service.rb` |
| Angular 請求書画面群 | `web/src/app/features/invoices/` |
| Angular 入金画面群 | `web/src/app/features/payments/` |
| 仕様書 4 件 | `docs/specs/features/FEAT-0030...FEAT-0033` |

### Phase 3E: ダッシュボード + 通知

| 成果物 | 主要ファイル |
|--------|------------|
| ダッシュボードコントローラー | `api/app/controllers/api/v1/dashboard_controller.rb` |
| ダッシュボードサービス | `api/app/services/dashboard_service.rb` |
| 通知サービス | `api/app/services/notification_logger.rb` (ログ出力のみ) |
| Angular ダッシュボード画面 | `web/src/app/features/dashboard/` |
| 仕様書 2 件 | `docs/specs/features/FEAT-0040, FEAT-0041` |

---

## 9. リスクと対策

### 9.1 技術的リスク

| リスク | 影響度 | 確率 | 対策 |
|--------|--------|------|------|
| JWT RS256 の鍵管理が複雑 | High | Medium | 開発環境は固定鍵ペア (.env)、本番は SSM。初期段階で鍵生成スクリプトを用意 |
| 契約の状態遷移ロジックが複雑 | Medium | High | 状態遷移図を仕様に明記。AASM gem の採用を検討。テストで全遷移パスを網羅 |
| 請求書生成バッチの金額計算ロジック | High | Medium | 固定月額 → 従量課金 → 段階制の順に実装。各料金タイプを独立してテスト |
| 入金消込の部分消込/過入金処理 | Medium | Medium | 仕様で全パターン (全額/一部/過入金) を列挙。金額は integer (円) + BigDecimal (計算時) で扱う |
| Angular Standalone Components の知識不足 | Low | Low | 公式ドキュメント準拠。NgModule は使用しない方針を堅持 |

### 9.2 テスト戦略上の注意点

| 注意点 | 対策 |
|--------|------|
| 状態遷移のテスト漏れ | 許可・禁止の両方向を全パターンテスト。不正遷移のテストを必須化 |
| 金額計算の精度 | 浮動小数点を使わない。DB では integer (円)、計算時は BigDecimal |
| バッチジョブのテスト | Sidekiq::Testing.inline! で同期実行テスト。冪等性のテスト |
| 認証テストの漏れ | 全コントローラーで「認証なし → 401」テストを必須化 |
| 認可テストの漏れ | 全コントローラーで「権限なし → 403」テストを必須化。Pundit `after_action :verify_authorized` |
| E2E テストの不安定さ | Playwright の auto-wait 活用。リトライ設定。テストデータの独立性確保 |

### 9.3 依存関係のリスク

| リスク | 対策 |
|--------|------|
| FEAT-0001 (認証) が遅延すると全後続が停止 | Phase 1 完了直後に認証の仕様策定を開始。最優先で実装 |
| 仕様承認がボトルネック | 仕様を先行して複数起草し、まとめてレビュー依頼する |
| 契約明細の料金タイプが仕様変更される | 固定月額を最初に実装し、従量/段階制は追加実装として分離 |
| OpenAPI 定義と実装の乖離 | committee gem で自動検証。仕様書と OpenAPI は同一 PR で更新 |

### 9.4 プロジェクト運営上のリスク

| リスク | 対策 |
|--------|------|
| コンテキスト消失 (セッション断絶) | 仕様書 + ADR + CLAUDE.md で知識を永続化。各 PR に十分な説明 |
| カバレッジ 100% の維持コスト | 除外対象を明確化。カバレッジのための無意味なテストは書かない |
| Phase 3 の全体規模が大きい | サブフェーズごとに完結。各 FEAT を独立リリース可能な単位に |

---

## 10. 開発の進め方

### 10.1 各機能の開発サイクル

```
1. AI が FEAT 仕様書をドラフト起草
2. 関連する API 仕様書をドラフト起草
3. 関連する UI 仕様書をドラフト起草
4. ユーザーに仕様を提示し承認を得る (★ 必須)
5. OpenAPI 定義を作成
6. DB マイグレーションを作成 (★ ユーザー承認必要)
7. テストを先に書く (RED 確認)
8. 実装コードを書く (GREEN 確認)
9. lint + カバレッジ確認
10. PR 作成 → CI Green → マージ
```

### 10.2 次のアクション

Phase 1 に着手するために、以下の順序で進める:

1. **Docker Compose 構成の仕様を起草** し、ユーザー承認を得る
2. 承認後、`docker-compose.yml` と各 `Dockerfile` を作成
3. Rails API プロジェクトを `rails new` で初期化
4. Angular プロジェクトを `ng new` で初期化
5. lint + テストフレームワークを設定
6. ヘルスチェックエンドポイントを実装（最初の仕様駆動開発サイクル）

---

## 付録 A: 仕様書 ID の採番ルール

| タイプ | 範囲 | 用途 |
|--------|------|------|
| FEAT-0001 ~ FEAT-0009 | 共通基盤 (Phase 3A) | 認証、認可、エラーハンドリング、共通 UI |
| FEAT-0010 ~ FEAT-0019 | 顧客管理 (Phase 3B) | 顧客 CRUD、検索 |
| FEAT-0020 ~ FEAT-0029 | 契約管理 (Phase 3C) | 契約、明細、使用量 |
| FEAT-0030 ~ FEAT-0039 | 請求・入金 (Phase 3D) | 請求書生成、入金、消込 |
| FEAT-0040 ~ FEAT-0049 | ダッシュボード (Phase 3E) | ダッシュボード、通知 |
| API-0001 ~ API-0009 | API 仕様 | 各 API エンドポイント群 |
| UI-0001 ~ UI-0012 | UI 仕様 | 各画面 |
| INFRA-0001 ~ | インフラ仕様 | Phase 4 で使用 |

## 付録 B: 技術選定サマリー

| 技術領域 | 選定 | 選定理由 |
|---------|------|---------|
| 認証 | JWT (RS256) + HttpOnly Cookie | ARCHITECTURE.md で決定済み |
| 認可 | Pundit | ポリシーベース。テストしやすい |
| 状態遷移 | aasm | 契約・請求書の状態遷移管理。宣言的 DSL でガード条件を表現 |
| シリアライザ | Alba | JSON レスポンス構造の明示定義。高速、活発にメンテナンス |
| ジョブ | Sidekiq + Redis | 請求書月次生成バッチ |
| 金額計算 | integer (円) + BigDecimal (計算時) | 浮動小数点誤差を回避 |
| ページネーション | Kaminari or Pagy | 一覧表示のページネーション |
| フロントエンド状態管理 | Angular Signals (v19+) | NgRx は過剰。Signals で十分 |
| フォーム | Reactive Forms | テンプレート駆動は不採用 |
| HTTP 通信 | HttpClient + Interceptor | 認証トークン自動付与 |
