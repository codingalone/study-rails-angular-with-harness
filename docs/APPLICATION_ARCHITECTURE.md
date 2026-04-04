# アプリケーションアーキテクチャ設計: 契約・請求管理システム

> **WARNING: Public Repository** — 機密情報を記載しないこと。
>
> **Note:** ドメインモデル・ビジネスルール・状態遷移の権威ある定義は [APP_REQUIREMENTS.md](APP_REQUIREMENTS.md) を参照。
> 本ドキュメントは技術的な実装設計に焦点を当てる。矛盾がある場合は APP_REQUIREMENTS.md を優先する。

---

## 1. Rails API 設計

### 1.1 ディレクトリ構造

```
api/app/
├── controllers/
│   ├── application_controller.rb
│   └── api/
│       └── v1/
│           ├── base_controller.rb        # V1 共通: 認証・認可・エラーハンドリング
│           ├── clients_controller.rb
│           ├── contracts_controller.rb
│           ├── contract_items_controller.rb
│           ├── invoices_controller.rb
│           ├── invoice_items_controller.rb
│           ├── payments_controller.rb
│           ├── usage_records_controller.rb
│           ├── users_controller.rb
│           ├── sessions_controller.rb
│           └── health_controller.rb
├── models/
│   ├── application_record.rb
│   ├── user.rb
│   ├── client.rb
│   ├── contract.rb
│   ├── contract_item.rb
│   ├── invoice.rb
│   ├── invoice_item.rb
│   ├── payment.rb
│   ├── tiered_rate.rb
│   └── usage_record.rb
├── serializers/                          # Alba
│   ├── base_serializer.rb
│   ├── client_serializer.rb
│   ├── contract_serializer.rb
│   ├── contract_item_serializer.rb
│   ├── invoice_serializer.rb
│   ├── invoice_item_serializer.rb
│   ├── payment_serializer.rb
│   ├── tiered_rate_serializer.rb
│   ├── usage_record_serializer.rb
│   └── user_serializer.rb
├── services/                             # サービスオブジェクト
│   ├── contracts/
│   │   ├── create_service.rb
│   │   ├── activate_service.rb
│   │   └── terminate_service.rb
│   ├── invoices/
│   │   ├── generate_service.rb           # 単一契約から請求書を生成
│   │   ├── issue_service.rb
│   │   └── void_service.rb
│   ├── invoice_generation/
│   │   ├── batch_service.rb              # 月次バッチ: 全契約を処理
│   │   ├── fixed_calculator.rb           # 固定額計算
│   │   ├── usage_calculator.rb           # 従量額計算
│   │   └── tiered_calculator.rb          # 段階制計算
│   └── payments/
│       ├── record_service.rb
│       └── reconciliation_service.rb     # 消込処理
├── policies/                             # Pundit
│   ├── application_policy.rb
│   ├── client_policy.rb
│   ├── contract_policy.rb
│   ├── invoice_policy.rb
│   ├── payment_policy.rb
│   └── usage_record_policy.rb
├── jobs/                                 # Sidekiq
│   ├── application_job.rb
│   ├── invoice_generation_job.rb         # 月次請求書生成
│   ├── overdue_invoice_check_job.rb      # 支払期限超過チェック
│   └── contract_renewal_job.rb           # 契約自動更新（日次）
├── validators/                           # カスタムバリデータ
│   ├── date_range_validator.rb
│   └── positive_amount_validator.rb
└── errors/                               # カスタムエラー
    ├── application_error.rb
    ├── state_transition_error.rb
    └── invoice_generation_error.rb
```

#### 設計判断の理由

- **controllers/api/v1/**: URL パスと一致させることでルーティングとの対応が明確になる。将来 v2 が必要になった場合も v1 を維持したまま並行運用できる。
- **services/**: ドメインごとにサブディレクトリで分割。コントローラーを薄く保ち、ビジネスロジックのテストを容易にする。
- **policies/**: Pundit の規約に従い `app/policies/` に配置。モデルと 1:1 対応。
- **errors/**: 例外クラスを専用ディレクトリに集約。rescue_from で統一的にハンドリングする。

### 1.2 API バージョニング

URL パス方式 (`/api/v1/`) を採用する。

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :clients
      resources :contracts do
        member do
          post :activate
          post :suspend
          post :resume
          post :terminate
        end
        resources :contract_items, shallow: true
      end
      resources :invoices, only: %i[index show create update] do
        collection do
          post :generate    # 月次バッチ実行 (Sidekiq ジョブをトリガー)
        end
        member do
          post :issue
          post :void
        end
        resources :invoice_items, only: %i[index show], shallow: true
      end
      resources :payments, only: %i[index show create]
      resources :usage_records, only: %i[index show create update]
      resources :users, only: %i[index show create update]
      resources :sessions, only: %i[create destroy] do
        post :refresh, on: :collection
      end
      namespace :dashboard do
        get :summary
        get :revenue
        get :receivables
      end
      get :health, to: "health#show"
    end
  end
end
```

#### 設計判断の理由

- **URL パス方式**: Header 方式 (Accept ヘッダー) より直感的でデバッグしやすい。OpenAPI 定義との対応も明確。学習プロジェクトとしては理解しやすさが重要。
- **状態遷移は member ルート**: RESTful な CRUD に加え、`POST /contracts/:id/activate` のような動詞エンドポイントで状態遷移を明示する。PATCH によるステータス直接更新を禁止し、不正な遷移を防ぐ。
- **shallow routing**: `contract_items` は `contracts` にネストするが、個別アクセス時はフラットな URL (`/api/v1/contract_items/:id`) にする。URL の深いネストを避けつつ所属関係を表現する。

### 1.3 シリアライザ選定: Alba

**Alba** を採用する。

```ruby
# app/serializers/invoice_serializer.rb
class InvoiceSerializer
  include Alba::Resource

  root_key :invoice, :invoices

  attributes :id, :invoice_number, :status, :issue_date, :due_date,
             :notes, :created_at, :updated_at

  # 金額は integer（円単位）で保存し、そのまま返す
  attribute :subtotal do |invoice|
    invoice.subtotal
  end

  attribute :tax_amount do |invoice|
    invoice.tax_amount
  end

  attribute :total_amount do |invoice|
    invoice.total_amount
  end

  one :client, serializer: ClientSerializer
  one :contract, serializer: ContractSerializer
  many :invoice_items, serializer: InvoiceItemSerializer
end
```

#### 設計判断の理由

| 候補 | 長所 | 短所 |
|------|------|------|
| **Alba** | 高速、Pure Ruby、シンプル API、活発なメンテナンス | 知名度がやや低い |
| ActiveModelSerializers | 知名度が高い | メンテナンス停滞、パフォーマンスが劣る |
| jsonapi-serializer | JSON:API 準拠 | JSON:API は本プロジェクトの SPA には過剰 |

- **パフォーマンス**: Alba は AMS の約 5-10 倍高速。請求書一覧のような大量データ返却時に差が出る。
- **シンプルさ**: DSL が直感的で学習コストが低い。
- **メンテナンス**: 活発に開発が継続されている。

### 1.4 サービスオブジェクト設計パターン

**Result オブジェクトパターン**を採用する。各サービスは `call` メソッドを持ち、成功/失敗を明示的に返す。

```ruby
# app/services/contracts/activate_service.rb
module Contracts
  class ActivateService
    attr_reader :contract, :errors

    def initialize(contract)
      @contract = contract
      @errors = []
    end

    def call
      validate!
      contract.activate! # aasm イベント
      Result.new(success: true, data: contract)
    rescue AASM::InvalidTransition => e
      @errors << "契約を有効化できません: #{e.message}"
      Result.new(success: false, errors: @errors)
    rescue ActiveRecord::RecordInvalid => e
      @errors << e.message
      Result.new(success: false, errors: @errors)
    end

    private

    def validate!
      @errors << "契約開始日が未設定です" if contract.start_date.blank?
      @errors << "契約明細が存在しません" if contract.contract_items.empty?
      raise ActiveRecord::RecordInvalid, contract if @errors.any?
    end
  end
end

# app/services/result.rb
class Result
  attr_reader :data, :errors

  def initialize(success:, data: nil, errors: [])
    @success = success
    @data = data
    @errors = errors
  end

  def success?
    @success
  end

  def failure?
    !@success
  end
end
```

コントローラー側での呼び出し:

```ruby
# app/controllers/api/v1/contracts_controller.rb
def activate
  result = Contracts::ActivateService.new(@contract).call

  if result.success?
    render json: ContractSerializer.new(result.data).serialize, status: :ok
  else
    render json: { errors: result.errors }, status: :unprocessable_entity
  end
end
```

#### 設計判断の理由

- **Result オブジェクト**: 例外に依存しないフロー制御。コントローラーが成功/失敗を分岐するだけの薄い層になる。
- **単一責任**: 1 サービス = 1 ユースケース。テスト対象が明確。
- **コンストラクタインジェクション**: 依存オブジェクト (contract) をコンストラクタで受け取ることで、テスト時のモック注入が容易。

### 1.5 状態遷移: aasm gem

**aasm gem** を採用する。

```ruby
# app/models/contract.rb
class Contract < ApplicationRecord
  include AASM

  aasm column: :status, whiny_transitions: true do
    state :draft, initial: true
    state :active
    state :suspended
    state :terminated

    event :activate do
      transitions from: :draft, to: :active, guard: :activatable?
      after do
        update!(activated_at: Time.current)
      end
    end

    event :suspend do
      transitions from: :active, to: :suspended
      after do
        update!(suspended_at: Time.current)
      end
    end

    event :resume do
      transitions from: :suspended, to: :active
      after do
        update!(suspended_at: nil)
      end
    end

    event :terminate do
      transitions from: %i[draft active suspended], to: :terminated
      after do
        update!(terminated_at: Time.current)
      end
    end
  end

  private

  def activatable?
    start_date.present? && contract_items.exists?
  end
end
```

```ruby
# app/models/invoice.rb
class Invoice < ApplicationRecord
  include AASM

  aasm column: :status, whiny_transitions: true do
    state :draft, initial: true
    state :issued
    state :paid
    state :overdue
    state :void

    event :issue do
      transitions from: :draft, to: :issued, guard: :issuable?
      after do
        update!(issued_at: Time.current)
      end
    end

    event :mark_paid do
      transitions from: %i[issued overdue], to: :paid
      guard do
        paid_amount >= total_amount
      end
      after do
        update!(paid_at: Time.current)
      end
    end

    event :mark_overdue do
      transitions from: :issued, to: :overdue
      guard do
        due_date < Date.current
      end
    end

    event :void_invoice do
      transitions from: %i[draft issued overdue], to: :void
      after do
        update!(voided_at: Time.current)
      end
    end
  end

  def remaining_amount
    total_amount - paid_amount
  end

  private

  def issuable?
    invoice_items.exists?
  end
end
```

#### 設計判断の理由

| 候補 | 長所 | 短所 |
|------|------|------|
| **aasm** | 宣言的 DSL、ガード条件、コールバック、ActiveRecord 統合 | 依存追加 |
| 手動実装 (enum + メソッド) | 依存なし | ガード・遷移のバリデーションを自前実装する手間が大きい |
| statesman | 遷移履歴の永続化 | 本プロジェクトには過剰 |

- **aasm**: 状態遷移図をそのままコードに落とし込める宣言的な DSL。`guard` で遷移条件を表現し、不正遷移を `AASM::InvalidTransition` で検知できる。
- **whiny_transitions**: 不正遷移時に例外を発生させ、サイレントな失敗を防ぐ。

### 状態遷移図

```
Contract:
  draft --> active --> suspended --> active (resume)
     |            |                    |
     |            |                    +--> terminated
     |            +--> terminated
     +--> terminated

Invoice:
  draft --> issued --> paid
     |          |        ^
     |          |        |
     |          +--> overdue --> paid
     |          |             |
     |          +--> void     +--> void
     +--> void
```

### 1.6 バックグラウンドジョブ: Sidekiq

```ruby
# app/jobs/invoice_generation_job.rb
class InvoiceGenerationJob < ApplicationJob
  queue_as :billing

  # リトライ設定: 最大 3 回、指数バックオフ
  sidekiq_options retry: 3

  def perform(billing_period = nil)
    billing_period ||= Date.current.beginning_of_month
    InvoiceGeneration::BatchService.new(billing_period:).call
  end
end

# app/jobs/overdue_invoice_check_job.rb
class OverdueInvoiceCheckJob < ApplicationJob
  queue_as :billing

  def perform
    Invoice.where(status: :issued)
           .where("due_date < ?", Date.current)
           .find_each do |invoice|
      invoice.mark_overdue! if invoice.may_mark_overdue?
    end
  end
end
```

Sidekiq のキュー構成:

| キュー | 優先度 | 用途 |
|--------|--------|------|
| `default` | 10 | 一般的なジョブ |
| `billing` | 5 | 請求書生成・消込 |
| `mailers` | 3 | メール送信（現時点ではスコープ外。将来のメール通知拡張用に予約のみ） |

#### 設計判断の理由

- **Sidekiq**: Redis を既に使用しているため追加インフラ不要。Active Job アダプタとして設定するだけで利用可能。
- **キュー分離**: 請求処理を `billing` キューに分離し、一般ジョブの遅延に影響されないようにする。

### 1.7 N+1 防止戦略

```ruby
# Gemfile (development, test)
gem "bullet"

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.raise = true        # 開発環境では例外を発生
  Bullet.bullet_logger = true
end

# config/environments/test.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.raise = true        # テスト時も例外で検知
end
```

コントローラーでの `includes` の適用:

```ruby
# app/controllers/api/v1/invoices_controller.rb
class Api::V1::InvoicesController < Api::V1::BaseController
  def index
    invoices = policy_scope(Invoice)
                 .includes(:client, :contract, :invoice_items)
                 .order(issue_date: :desc)
                 .page(params[:page])
                 .per(params[:per_page] || 25)

    render json: InvoiceSerializer.new(invoices).serialize
  end

  def show
    invoice = Invoice.includes(:client, :contract, invoice_items: :contract_item)
                     .find(params[:id])
    authorize invoice

    render json: InvoiceSerializer.new(invoice).serialize
  end
end
```

#### 設計判断の理由

- **bullet gem (raise モード)**: 開発・テスト環境で N+1 を例外として検知。CI で自動的にキャッチされ、本番に N+1 が混入しない。
- **includes vs preload**: 基本は `includes` (Rails が最適化)。JOIN が必要な場合のみ `eager_load` を明示。

### 1.8 エラーハンドリングパターン

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      include Pundit::Authorization

      before_action :authenticate_user!
      after_action :verify_authorized, except: :index
      after_action :verify_policy_scoped, only: :index

      rescue_from ActiveRecord::RecordNotFound do |e|
        render_error(status: :not_found, code: "not_found",
                     message: "リソースが見つかりません")
      end

      rescue_from ActiveRecord::RecordInvalid do |e|
        render_error(status: :unprocessable_entity, code: "validation_error",
                     message: "バリデーションエラー",
                     details: e.record.errors.full_messages)
      end

      rescue_from AASM::InvalidTransition do |e|
        render_error(status: :unprocessable_entity, code: "invalid_transition",
                     message: "この状態遷移は許可されていません")
      end

      rescue_from Pundit::NotAuthorizedError do |e|
        render_error(status: :forbidden, code: "forbidden",
                     message: "この操作を実行する権限がありません")
      end

      private

      def render_error(status:, code:, message:, details: [])
        render json: {
          error: {
            code:,
            message:,
            details:
          }
        }, status:
      end
    end
  end
end
```

#### エラーレスポンス形式

```json
{
  "error": {
    "code": "validation_error",
    "message": "バリデーションエラー",
    "details": [
      "契約開始日を入力してください",
      "契約明細が存在しません"
    ]
  }
}
```

#### 設計判断の理由

- **統一的なエラー形式**: フロントエンドが `error.code` でプログラマティックに判定し、`error.message` をユーザーに表示できる。
- **rescue_from チェーン**: 例外の種類ごとに適切な HTTP ステータスにマッピング。コントローラー個別の rescue を不要にする。

### 1.9 ページネーション: kaminari

```ruby
# コントローラー
invoices = Invoice.page(params[:page]).per(params[:per_page] || 25)

# レスポンスヘッダーで返却
response.headers["X-Total-Count"] = invoices.total_count.to_s
response.headers["X-Total-Pages"] = invoices.total_pages.to_s
response.headers["X-Current-Page"] = invoices.current_page.to_s
response.headers["X-Per-Page"] = invoices.limit_value.to_s
```

#### 設計判断の理由

- **ヘッダー方式**: レスポンスボディのデータ構造を汚さない。Link ヘッダー (RFC 5988) も併用可能。
- **kaminari**: Rails エコシステムで最も広く使われるページネーション gem。

---

## 2. Angular フロントエンド設計

### 2.1 ディレクトリ構造 (Standalone Components)

```
web/src/app/
├── app.component.ts
├── app.config.ts
├── app.routes.ts
├── core/                                  # アプリ全体で 1 回だけ使うもの
│   ├── auth/
│   │   ├── auth.service.ts                # JWT 管理、ログイン/ログアウト
│   │   ├── auth.guard.ts                  # ルートガード
│   │   ├── auth.interceptor.ts            # JWT 自動付与
│   │   └── token-storage.service.ts       # トークン管理 (メモリ保持)
│   ├── error/
│   │   ├── error-handler.service.ts       # グローバルエラーハンドリング
│   │   └── error.interceptor.ts           # HTTP エラーインターセプター
│   ├── api/
│   │   └── api.service.ts                 # HTTP クライアント基底クラス
│   └── layout/
│       ├── header.component.ts
│       ├── sidebar.component.ts
│       └── main-layout.component.ts
├── features/                              # 機能別モジュール (遅延ロード)
│   ├── dashboard/
│   │   ├── dashboard.component.ts
│   │   ├── dashboard.routes.ts
│   │   └── widgets/
│   │       ├── revenue-summary.component.ts
│   │       └── overdue-invoices.component.ts
│   ├── clients/
│   │   ├── client-list.component.ts       # Smart Component
│   │   ├── client-detail.component.ts     # Smart Component
│   │   ├── client-form.component.ts       # Smart Component (作成/編集)
│   │   ├── client.service.ts
│   │   ├── client.model.ts
│   │   ├── clients.routes.ts
│   │   └── components/                    # Dumb Components
│   │       ├── client-card.component.ts
│   │       └── client-info.component.ts
│   ├── contracts/
│   │   ├── contract-list.component.ts
│   │   ├── contract-detail.component.ts
│   │   ├── contract-form.component.ts
│   │   ├── contract.service.ts
│   │   ├── contract.model.ts
│   │   ├── contracts.routes.ts
│   │   └── components/
│   │       ├── contract-status-badge.component.ts
│   │       ├── contract-item-table.component.ts
│   │       └── contract-timeline.component.ts
│   ├── invoices/
│   │   ├── invoice-list.component.ts
│   │   ├── invoice-detail.component.ts
│   │   ├── invoice.service.ts
│   │   ├── invoice.model.ts
│   │   ├── invoices.routes.ts
│   │   └── components/
│   │       ├── invoice-status-badge.component.ts
│   │       ├── invoice-summary.component.ts
│   │       └── invoice-line-items.component.ts
│   ├── payments/
│   │   ├── payment-list.component.ts
│   │   ├── payment-form.component.ts
│   │   ├── payment.service.ts
│   │   ├── payment.model.ts
│   │   ├── payments.routes.ts
│   │   └── components/
│   │       └── payment-reconciliation.component.ts
│   ├── usage-records/
│   │   ├── usage-record-list.component.ts
│   │   ├── usage-record-form.component.ts
│   │   ├── usage-record.service.ts
│   │   ├── usage-record.model.ts
│   │   └── usage-records.routes.ts
│   └── settings/
│       ├── user-profile.component.ts
│       └── settings.routes.ts
└── shared/                                # 複数の feature で再利用
    ├── components/
    │   ├── data-table/
    │   │   ├── data-table.component.ts    # 汎用テーブル
    │   │   └── data-table.model.ts
    │   ├── confirm-dialog.component.ts
    │   ├── loading-spinner.component.ts
    │   ├── empty-state.component.ts
    │   ├── status-badge.component.ts
    │   └── pagination.component.ts
    ├── forms/
    │   ├── form-field.component.ts        # ラベル + 入力 + エラー表示
    │   ├── date-picker.component.ts
    │   ├── currency-input.component.ts    # 金額入力
    │   └── select-field.component.ts
    ├── pipes/
    │   ├── currency.pipe.ts               # 金額フォーマット
    │   ├── date-format.pipe.ts
    │   └── status-label.pipe.ts
    ├── directives/
    │   └── role.directive.ts              # *appRole="'admin'" 条件表示
    └── models/
        ├── api-response.model.ts
        ├── pagination.model.ts
        └── error.model.ts
```

#### 設計判断の理由

- **core / features / shared の 3 層構造**: Angular のベストプラクティスに従う。Standalone Components でも論理的な構造は維持する。
- **features 配下は遅延ロード**: 初期バンドルサイズを抑える。ログイン後に必要な画面だけロードする。
- **Smart / Dumb コンポーネント分離**: features 直下が Smart (データ取得・状態管理)、components/ 配下が Dumb (Input/Output のみ)。テスタビリティとの再利用性の向上。

### 2.2 ルーティング設計

```typescript
// app.routes.ts
export const APP_ROUTES: Routes = [
  {
    path: "login",
    loadComponent: () =>
      import("./features/auth/login.component").then((m) => m.LoginComponent),
  },
  {
    path: "",
    component: MainLayoutComponent,
    canActivate: [authGuard],
    children: [
      {
        path: "dashboard",
        loadComponent: () =>
          import("./features/dashboard/dashboard.component").then(
            (m) => m.DashboardComponent
          ),
      },
      {
        path: "clients",
        loadChildren: () =>
          import("./features/clients/clients.routes").then(
            (m) => m.CLIENT_ROUTES
          ),
      },
      {
        path: "contracts",
        loadChildren: () =>
          import("./features/contracts/contracts.routes").then(
            (m) => m.CONTRACT_ROUTES
          ),
      },
      {
        path: "invoices",
        loadChildren: () =>
          import("./features/invoices/invoices.routes").then(
            (m) => m.INVOICE_ROUTES
          ),
      },
      {
        path: "payments",
        loadChildren: () =>
          import("./features/payments/payments.routes").then(
            (m) => m.PAYMENT_ROUTES
          ),
      },
      {
        path: "usage-records",
        loadChildren: () =>
          import("./features/usage-records/usage-records.routes").then(
            (m) => m.USAGE_RECORD_ROUTES
          ),
      },
      {
        path: "settings",
        loadChildren: () =>
          import("./features/settings/settings.routes").then(
            (m) => m.SETTINGS_ROUTES
          ),
      },
      { path: "", redirectTo: "dashboard", pathMatch: "full" },
    ],
  },
  { path: "**", redirectTo: "login" },
];

// features/clients/clients.routes.ts
export const CLIENT_ROUTES: Routes = [
  { path: "", component: ClientListComponent },
  { path: "new", component: ClientFormComponent },
  { path: ":id", component: ClientDetailComponent },
  { path: ":id/edit", component: ClientFormComponent },
];
```

### 2.3 状態管理: Signal ベース

**Angular Signals** を採用する。

```typescript
// features/clients/client.service.ts
@Injectable({ providedIn: "root" })
export class ClientService {
  private readonly http = inject(ApiService);

  // Signal による状態管理
  private readonly clientsSignal = signal<Client[]>([]);
  private readonly loadingSignal = signal<boolean>(false);
  private readonly errorSignal = signal<string | null>(null);

  // 読み取り専用の公開 Signal
  readonly clients = this.clientsSignal.asReadonly();
  readonly loading = this.loadingSignal.asReadonly();
  readonly error = this.errorSignal.asReadonly();

  // Computed Signal
  readonly activeClients = computed(() =>
    this.clientsSignal().filter((c) => c.active)
  );

  readonly clientCount = computed(() => this.clientsSignal().length);

  loadClients(params?: ClientSearchParams): void {
    this.loadingSignal.set(true);
    this.errorSignal.set(null);

    this.http.get<PaginatedResponse<Client>>("/clients", { params }).subscribe({
      next: (response) => {
        this.clientsSignal.set(response.data);
        this.loadingSignal.set(false);
      },
      error: (err) => {
        this.errorSignal.set(err.message);
        this.loadingSignal.set(false);
      },
    });
  }

  createClient(data: CreateClientRequest): Observable<Client> {
    return this.http.post<Client>("/clients", data).pipe(
      tap((client) => {
        this.clientsSignal.update((clients) => [...clients, client]);
      })
    );
  }
}
```

#### 設計判断の理由

| 候補 | 長所 | 短所 |
|------|------|------|
| **Angular Signals** | Angular 公式、軽量、学習コスト低、Zone.js 不要の方向 | エコシステムがまだ発展途上 |
| NgRx | 強力な DevTools、大規模向け | 学習コスト高、ボイラープレート多、本プロジェクトには過剰 |
| BehaviorSubject | シンプル | 手動 unsubscribe 管理、変更検知が Signal より非効率 |

- **Signals**: Angular 19 の推奨アプローチ。学習プロジェクトとして最新のパターンを習得する意義がある。
- **NgRx を不採用とした理由**: ドメインモデルが 8 つ程度の中規模アプリでは NgRx のボイラープレート (Action/Reducer/Effect/Selector) が過剰。Signal + Service で十分に状態管理できる。

### 2.4 HTTP インターセプター

```typescript
// core/auth/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const tokenStorage = inject(TokenStorageService);
  const accessToken = tokenStorage.getAccessToken();

  if (accessToken && !req.url.includes("/sessions")) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${accessToken}` },
    });
  }

  return next(req);
};

// core/error/error.interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const tokenStorage = inject(TokenStorageService);
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401 && !req.url.includes("/sessions/refresh")) {
        // トークンリフレッシュを試行
        return authService.refreshToken().pipe(
          switchMap((newToken) => {
            tokenStorage.setAccessToken(newToken.access_token);
            const retryReq = req.clone({
              setHeaders: {
                Authorization: `Bearer ${newToken.access_token}`,
              },
            });
            return next(retryReq);
          }),
          catchError(() => {
            authService.logout();
            router.navigate(["/login"]);
            return throwError(() => error);
          })
        );
      }
      return throwError(() => error);
    })
  );
};

// app.config.ts での登録
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor])),
    provideRouter(APP_ROUTES),
  ],
};
```

#### 設計判断の理由

- **関数型インターセプター**: Angular 19 の推奨パターン。クラスベースの `HttpInterceptor` より簡潔。
- **トークンリフレッシュ**: 401 応答時に自動でリフレッシュトークンを使って再取得。ユーザー体験を損なわない。
- **リフレッシュエンドポイント除外**: `/sessions/refresh` への 401 は無限ループになるため除外。

### 2.5 フォーム設計

```typescript
// features/contracts/contract-form.component.ts
@Component({
  selector: "app-contract-form",
  standalone: true,
  imports: [ReactiveFormsModule, FormFieldComponent, DatePickerComponent],
  template: `...`,
})
export class ContractFormComponent implements OnInit {
  private readonly fb = inject(FormBuilder);
  private readonly contractService = inject(ContractService);
  private readonly route = inject(ActivatedRoute);

  form = this.fb.group({
    clientId: ["", [Validators.required]],
    contractNumber: [
      "",
      [Validators.required, Validators.pattern(/^CTR-\d{8}-\d{5}$/)],
    ],
    startDate: ["", [Validators.required]],
    endDate: ["", [Validators.required]],
    billingDay: [28, [Validators.required, Validators.min(1), Validators.max(28)]],
    paymentTermDays: [30, [Validators.required, Validators.min(1)]],
    notes: [""],
    items: this.fb.array<FormGroup>([]),
  });

  get items(): FormArray {
    return this.form.get("items") as FormArray;
  }

  addItem(): void {
    this.items.push(
      this.fb.group({
        name: ["", [Validators.required]],
        unitPrice: [0, [Validators.required, Validators.min(0)]],
        quantity: [1, [Validators.required, Validators.min(1)]],
        billingType: ["fixed", [Validators.required]],
      })
    );
  }

  removeItem(index: number): void {
    this.items.removeAt(index);
  }

  onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }
    // submit logic
  }
}
```

#### 設計判断の理由

- **Reactive Forms**: Template-driven Forms よりテスタビリティが高い。バリデーションロジックをコード側で完全に制御できる。
- **FormArray**: 契約明細のような可変長リストに対応。行の追加・削除が容易。

### 2.6 Smart / Dumb コンポーネントパターン

```
Smart Component (Container)           Dumb Component (Presentational)
┌──────────────────────────┐          ┌──────────────────────────┐
│ - Service を inject       │          │ - @Input() でデータ受取   │
│ - データ取得・更新         │  ──────> │ - @Output() でイベント発火 │
│ - ルーティングパラメータ    │          │ - 純粋な表示ロジックのみ   │
│ - エラーハンドリング       │          │ - テストが容易             │
└──────────────────────────┘          └──────────────────────────┘
```

```typescript
// Dumb Component: 入力と出力だけ
@Component({
  selector: "app-contract-status-badge",
  standalone: true,
  template: `
    <span
      class="badge"
      [class]="'badge--' + status()"
      (click)="statusClicked.emit(status())"
    >
      {{ status() | statusLabel }}
    </span>
  `,
})
export class ContractStatusBadgeComponent {
  status = input.required<ContractStatus>();
  statusClicked = output<ContractStatus>();
}
```

### 2.7 共通コンポーネント

```typescript
// shared/components/data-table/data-table.component.ts
@Component({
  selector: "app-data-table",
  standalone: true,
  template: `
    <table>
      <thead>
        <tr>
          @for (col of columns(); track col.key) {
          <th (click)="onSort(col)" [class.sortable]="col.sortable">
            {{ col.label }}
            @if (sortKey() === col.key) {
            <span>{{ sortDirection() === "asc" ? "^" : "v" }}</span>
            }
          </th>
          }
        </tr>
      </thead>
      <tbody>
        @for (row of data(); track trackByFn()(row)) {
        <tr (click)="rowClicked.emit(row)">
          @for (col of columns(); track col.key) {
          <td>
            <ng-container
              *ngTemplateOutlet="
                cellTemplate() || defaultCell;
                context: { $implicit: row, column: col }
              "
            />
          </td>
          }
        </tr>
        } @empty {
        <tr>
          <td [attr.colspan]="columns().length">
            <app-empty-state [message]="emptyMessage()" />
          </td>
        </tr>
        }
      </tbody>
    </table>
    <app-pagination
      [currentPage]="currentPage()"
      [totalPages]="totalPages()"
      (pageChanged)="pageChanged.emit($event)"
    />
  `,
})
export class DataTableComponent<T> {
  columns = input.required<ColumnDef[]>();
  data = input.required<T[]>();
  currentPage = input<number>(1);
  totalPages = input<number>(1);
  emptyMessage = input<string>("データがありません");
  trackByFn = input<(item: T) => unknown>(() => (item: unknown) => item);
  cellTemplate = input<TemplateRef<unknown>>();
  sortKey = input<string>("");
  sortDirection = input<"asc" | "desc">("asc");

  rowClicked = output<T>();
  pageChanged = output<number>();
  sortChanged = output<{ key: string; direction: "asc" | "desc" }>();
}
```

---

## 3. データベース設計

### 3.1 ER 図

```
users ──────────────────────────────────────────────┐
  |                                                  |
  v                                                  v
clients ─────> contracts ─────> contract_items    invoices ──> invoice_items
                   |                                  |
                   |                                  v
                   +──────────> usage_records       payments
```

### 3.2 テーブル定義

#### users

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| email | varchar(255) | NOT NULL, UNIQUE | ログイン用メールアドレス |
| password_digest | varchar(255) | NOT NULL | bcrypt ハッシュ |
| name | varchar(100) | NOT NULL | 表示名 |
| role | varchar(20) | NOT NULL | admin / accountant / sales (DEFAULT なし、作成時に必須指定) |
| active | boolean | NOT NULL, DEFAULT true | アカウント有効フラグ |
| last_sign_in_at | timestamptz | | 最終ログイン日時 |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_users_on_email` (UNIQUE)
- `index_users_on_role`

#### clients

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| code | varchar(20) | NOT NULL, UNIQUE | 顧客コード (例: C-XXXXX) |
| name | varchar(255) | NOT NULL | 顧客名 |
| name_kana | varchar(255) | | フリガナ |
| postal_code | varchar(10) | | 郵便番号 |
| address | text | | 住所 |
| phone | varchar(20) | | 電話番号 |
| email | varchar(255) | | 連絡先メール |
| contact_person | varchar(100) | | 担当者名 |
| notes | text | | 備考 |
| active | boolean | NOT NULL, DEFAULT true | 有効フラグ |
| created_by_id | bigint | FK(users), NOT NULL | 作成者 |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_clients_on_code` (UNIQUE)
- `index_clients_on_name`
- `index_clients_on_active`
- `index_clients_on_created_by_id`

#### contracts

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| contract_number | varchar(30) | NOT NULL, UNIQUE | 契約番号 (例: CTR-YYYYMMDD-XXXXX) |
| client_id | bigint | FK(clients), NOT NULL | 顧客 |
| status | varchar(20) | NOT NULL, DEFAULT 'draft' | draft/active/suspended/terminated |
| start_date | date | NOT NULL | 契約開始日 |
| end_date | date | NOT NULL | 契約終了日 |
| billing_day | integer | NOT NULL, DEFAULT 28 | 締め日 (1-28) |
| payment_due_days | integer | NOT NULL, DEFAULT 30 | 支払期限 (日数) |
| auto_renew | boolean | NOT NULL, DEFAULT false | 自動更新フラグ |
| renewal_period_months | integer | NOT NULL, DEFAULT 12 | 自動更新期間 (月数) |
| notes | text | | 備考 |
| activated_at | timestamptz | | 有効化日時 |
| suspended_at | timestamptz | | 停止日時 |
| terminated_at | timestamptz | | 解約日時 |
| created_by_id | bigint | FK(users), NOT NULL | 作成者 |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_contracts_on_contract_number` (UNIQUE)
- `index_contracts_on_client_id`
- `index_contracts_on_status`
- `index_contracts_on_start_date_and_end_date`
- `index_contracts_on_created_by_id`
- `index_contracts_on_status_and_start_date` (月次バッチ用の複合インデックス)

#### contract_items

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| contract_id | bigint | FK(contracts), NOT NULL | 契約 |
| name | varchar(255) | NOT NULL | 品目名 |
| billing_type | varchar(20) | NOT NULL | fixed / usage / tiered |
| unit_price | decimal(10,2) | | 単価（fixed/usage 用） |
| unit_name | varchar(50) | | 単位名（usage/tiered 用、例: "GB"） |
| quantity | decimal(10,2) | NOT NULL, DEFAULT 1.0 | 数量 (fixed のデフォルト) |
| sort_order | integer | NOT NULL, DEFAULT 0 | 表示順 |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_contract_items_on_contract_id`
- `index_contract_items_on_billing_type`

#### invoices

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| invoice_number | varchar(20) | NOT NULL, UNIQUE | 請求書番号 (例: INV-YYYYMM-XXXXX) |
| contract_id | bigint | FK(contracts) | 契約（手動作成時は NULL） |
| client_id | bigint | FK(clients), NOT NULL | 顧客 (非正規化: 高速表示用) |
| status | varchar(20) | NOT NULL, DEFAULT 'draft' | draft/issued/paid/overdue/void |
| billing_period_start | date | NOT NULL | 請求対象期間 開始日 |
| billing_period_end | date | NOT NULL | 請求対象期間 終了日 |
| issue_date | date | NOT NULL | 発行日 |
| due_date | date | NOT NULL | 支払期限 |
| subtotal | integer | NOT NULL, DEFAULT 0 | 小計 (税抜、円) |
| tax_amount | integer | NOT NULL, DEFAULT 0 | 消費税額 (円) |
| total_amount | integer | NOT NULL, DEFAULT 0 | 合計 (税込、円) |
| paid_amount | integer | NOT NULL, DEFAULT 0 | 入金済額 (円) |
| notes | text | | 備考 |
| issued_at | timestamptz | | 発行日時 |
| paid_at | timestamptz | | 入金完了日時 |
| voided_at | timestamptz | | 無効化日時 |
| idempotency_key | varchar(100) | NOT NULL, UNIQUE | べき等性キー。バッチ: `inv-{contract_id}-{YYYYMM}`、手動作成: `manual-{user_id}-{timestamp}` |
| created_by_id | bigint | FK(users) | 作成者 (バッチ時は null) |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_invoices_on_invoice_number` (UNIQUE)
- `index_invoices_on_idempotency_key` (UNIQUE)
- `index_invoices_on_contract_id`
- `index_invoices_on_client_id`
- `index_invoices_on_status`
- `index_invoices_on_due_date`
- `index_invoices_on_issue_date`
- `index_invoices_on_status_and_due_date` (支払期限超過チェック用)
- `index_invoices_on_contract_id_and_billing_period_start` (UNIQUE, 重複防止用。手動作成請求書 contract_id = NULL は対象外。冪等性キーと合わせた二重防止)

#### invoice_items

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| invoice_id | bigint | FK(invoices), NOT NULL | 請求書 |
| contract_item_id | bigint | FK(contract_items) | 元の契約明細 (参照用) |
| name | varchar(255) | NOT NULL | 品目名 (スナップショット) |
| unit_price | decimal(10,2) | NOT NULL | 単価 (スナップショット) |
| quantity | decimal(10,2) | NOT NULL | 数量 |
| amount | integer | NOT NULL | 金額 (円) |
| sort_order | integer | NOT NULL, DEFAULT 0 | 表示順 |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_invoice_items_on_invoice_id`
- `index_invoice_items_on_contract_item_id`

#### payments

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| invoice_id | bigint | FK(invoices), NOT NULL | 請求書 |
| amount | integer | NOT NULL | 入金額 (円) |
| payment_date | date | NOT NULL | 入金日 |
| payment_method | varchar(20) | NOT NULL | bank_transfer / other |
| reference_number | varchar(100) | | 振込番号・参照番号 |
| notes | text | | 備考 |
| recorded_by_id | bigint | FK(users), NOT NULL | 記録者 |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_payments_on_invoice_id`
- `index_payments_on_payment_date`
- `index_payments_on_recorded_by_id`

#### tiered_rates

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| contract_item_id | bigint | FK(contract_items), NOT NULL | 対象の契約明細（billing_type = tiered） |
| lower_bound | decimal(10,2) | NOT NULL | 下限値（inclusive） |
| upper_bound | decimal(10,2) | | 上限値（exclusive, NULL = 上限なし） |
| unit_price | decimal(10,2) | NOT NULL | この区間の単価 |
| sort_order | integer | NOT NULL, DEFAULT 0 | 表示順 |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_tiered_rates_on_contract_item_id`

#### usage_records

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | bigint | PK, NOT NULL | |
| contract_item_id | bigint | FK(contract_items), NOT NULL | 対象の契約明細 |
| quantity | decimal(10,2) | NOT NULL | 使用量 |
| period_start | date | NOT NULL | 集計開始日 |
| period_end | date | NOT NULL | 集計終了日 |
| recorded_at | timestamptz | NOT NULL | 記録日時 |
| source | varchar(20) | NOT NULL | manual / api |
| recorded_by_id | bigint | FK(users) | 記録者（API 経由の場合は NULL） |
| created_at | timestamptz | NOT NULL | |
| updated_at | timestamptz | NOT NULL | |

インデックス:
- `index_usage_records_on_contract_item_id`
- `index_usage_records_on_period_start_and_period_end`
- `index_usage_records_on_contract_item_id_and_period_start` (期間集計用)
- `index_usage_records_on_item_period` (UNIQUE: `contract_item_id, period_start, period_end`) — 同一品目・同一期間の重複登録防止

### 3.3 設計判断の理由

#### 金額カラム: integer（円単位）

**integer（円単位）** を採用する。JPY は小数点以下がないため cents 変換は不要。

| 候補 | 長所 | 短所 |
|------|------|------|
| **integer（円単位）** | 浮動小数点誤差なし、計算が高速、比較が正確、JPY では変換不要 | - |
| decimal(15,2) | 直感的な値 | DB 間の挙動差、Rails の BigDecimal 変換コスト |

- 円単位の整数で保存し、シリアライザでそのまま integer として返す。
- フロントエンドでの浮動小数点誤差を完全に排除する。
- 通貨計算では **切り捨て (floor)** を使用する。端数は常に切り捨てとし、顧客に不利な丸めを行わない。

#### 論理削除 vs 物理削除

**物理削除を基本とし、ビジネス上保持が必要なレコードは状態フラグで管理する。**

| エンティティ | 方式 | 理由 |
|-------------|------|------|
| User | `active` フラグ | 監査証跡。作成者参照を維持する必要がある |
| Client | `active` フラグ | 契約・請求書から参照される。削除は不可 |
| Contract | 状態遷移 (terminated) | 法的に一定期間保持が必要 |
| Invoice | 状態遷移 (void) | 法的に一定期間保持が必要。物理削除は禁止 |
| Payment | 物理削除禁止 | 会計上、削除不可。取消は逆仕訳 (将来対応) |
| ContractItem | 物理削除 (draft 契約のみ) | active 契約の明細は削除不可 |
| UsageRecord | 物理削除 (未請求分のみ) | 請求済み分は削除不可 |

- **paranoia / discard gem は不採用**: `default_scope` での論理削除は JOIN やユニーク制約で問題を起こしやすい。明示的な状態管理のほうが安全。

#### 外部キー制約

全ての FK に `ON DELETE RESTRICT` を適用する。参照先が存在するレコードの削除を DB レベルで防止する。

```ruby
# マイグレーション例
add_foreign_key :contracts, :clients, on_delete: :restrict
add_foreign_key :invoices, :contracts, on_delete: :restrict
add_foreign_key :invoice_items, :invoices, on_delete: :restrict
add_foreign_key :payments, :invoices, on_delete: :restrict
```

#### インデックス戦略

| カテゴリ | 対象 | 理由 |
|---------|------|------|
| ユニーク | 各種番号 (contract_number, invoice_number) | ビジネスキーの一意性保証 |
| 外部キー | 全 FK カラム | JOIN パフォーマンス |
| ステータス | status カラム | フィルタリング頻度が高い |
| 日付範囲 | start_date, due_date, issue_date | 範囲検索・ソート |
| 複合 | (status, due_date), (contract_id, billing_period_start) | バッチ処理・重複防止 |

---

## 4. API エンドポイント一覧

### 4.1 認証

| メソッド | パス | 認証 | 認可 | 説明 |
|---------|------|------|------|------|
| POST | /api/v1/sessions | 不要 | - | ログイン |
| DELETE | /api/v1/sessions/:id | 必要 | 全ロール | ログアウト |
| POST | /api/v1/sessions/refresh | Cookie | - | トークンリフレッシュ |

### 4.2 ヘルスチェック

| メソッド | パス | 認証 | 認可 | 説明 |
|---------|------|------|------|------|
| GET | /api/v1/health | 不要 | - | ヘルスチェック |

### 4.3 顧客 (Clients)

| メソッド | パス | 認証 | 認可 | クエリパラメータ | 説明 |
|---------|------|------|------|----------------|------|
| GET | /api/v1/clients | 必要 | 全ロール | `q`, `active`, `sort`, `direction`, `page`, `per_page` | 一覧取得 |
| POST | /api/v1/clients | 必要 | admin, sales | - | 作成 |
| GET | /api/v1/clients/:id | 必要 | 全ロール | - | 詳細取得 |
| PATCH | /api/v1/clients/:id | 必要 | admin, sales | - | 更新 |
| DELETE | /api/v1/clients/:id | 必要 | admin | - | 削除 (契約なしの場合のみ) |

### 4.4 契約 (Contracts)

| メソッド | パス | 認証 | 認可 | クエリパラメータ | 説明 |
|---------|------|------|------|----------------|------|
| GET | /api/v1/contracts | 必要 | 全ロール | `client_id`, `status`, `sort`, `direction`, `page`, `per_page` | 一覧取得 |
| POST | /api/v1/contracts | 必要 | admin, sales | - | 作成 |
| GET | /api/v1/contracts/:id | 必要 | 全ロール | - | 詳細取得 |
| PATCH | /api/v1/contracts/:id | 必要 | admin, sales | - | 更新 (draft のみ) |
| DELETE | /api/v1/contracts/:id | 必要 | admin | - | 削除 (draft のみ) |
| POST | /api/v1/contracts/:id/activate | 必要 | admin, sales | - | 有効化 |
| POST | /api/v1/contracts/:id/suspend | 必要 | admin | - | 一時停止 |
| POST | /api/v1/contracts/:id/resume | 必要 | admin | - | 再開 |
| POST | /api/v1/contracts/:id/terminate | 必要 | admin, sales | - | 解約 (draft からの解約は sales も可) |

### 4.5 契約明細 (Contract Items)

| メソッド | パス | 認証 | 認可 | 説明 |
|---------|------|------|------|------|
| GET | /api/v1/contracts/:contract_id/contract_items | 必要 | 全ロール | 一覧取得 |
| POST | /api/v1/contracts/:contract_id/contract_items | 必要 | admin, sales | 作成（draft/suspended 契約のみ） |
| GET | /api/v1/contract_items/:id | 必要 | 全ロール | 詳細取得 |
| PATCH | /api/v1/contract_items/:id | 必要 | admin, sales | 更新（draft/suspended 契約のみ） |
| DELETE | /api/v1/contract_items/:id | 必要 | admin | 削除（draft/suspended 契約のみ） |

> **ガード条件**: active 状態の契約の明細は作成・更新・削除不可。suspended にしてから編集する（APP_REQUIREMENTS.md §5.1）。

### 4.6 請求書 (Invoices)

| メソッド | パス | 認証 | 認可 | クエリパラメータ | 説明 |
|---------|------|------|------|----------------|------|
| GET | /api/v1/invoices | 必要 | 全ロール | `client_id`, `contract_id`, `status`, `issue_date_from`, `issue_date_to`, `due_date_from`, `due_date_to`, `sort`, `direction`, `page`, `per_page` | 一覧取得 |
| POST | /api/v1/invoices | 必要 | admin, accountant | - | 手動作成 |
| POST | /api/v1/invoices/generate | 必要 | admin, accountant | `billing_period` (YYYY-MM) | 月次バッチ実行 (Sidekiq ジョブをトリガー) |
| GET | /api/v1/invoices/:id | 必要 | 全ロール | - | 詳細取得 |
| PATCH | /api/v1/invoices/:id | 必要 | admin, accountant | - | 更新 (draft のみ) |
| POST | /api/v1/invoices/:id/issue | 必要 | admin, accountant | - | 発行 |
| POST | /api/v1/invoices/:id/void | 必要 | admin | - | 無効化 |

### 4.7 請求明細 (Invoice Items)

| メソッド | パス | 認証 | 認可 | 説明 |
|---------|------|------|------|------|
| GET | /api/v1/invoices/:invoice_id/invoice_items | 必要 | 全ロール | 一覧取得 |
| GET | /api/v1/invoice_items/:id | 必要 | 全ロール | 詳細取得 |

### 4.8 入金 (Payments)

| メソッド | パス | 認証 | 認可 | クエリパラメータ | 説明 |
|---------|------|------|------|----------------|------|
| GET | /api/v1/payments | 必要 | admin, accountant | `invoice_id`, `payment_date_from`, `payment_date_to`, `sort`, `direction`, `page`, `per_page` | 一覧取得 |
| POST | /api/v1/payments | 必要 | admin, accountant | - | 入金記録 + 消込 |
| GET | /api/v1/payments/:id | 必要 | admin, accountant | - | 詳細取得 |

### 4.9 使用量記録 (Usage Records)

| メソッド | パス | 認証 | 認可 | クエリパラメータ | 説明 |
|---------|------|------|------|----------------|------|
| GET | /api/v1/usage_records | 必要 | 全ロール | `contract_item_id`, `period_start_from`, `period_start_to`, `period_end_from`, `period_end_to`, `sort`, `direction`, `page`, `per_page` | 一覧取得 |
| POST | /api/v1/usage_records | 必要 | admin, accountant, sales | - | 使用量登録 |
| GET | /api/v1/usage_records/:id | 必要 | 全ロール | - | 詳細取得 |
| PATCH | /api/v1/usage_records/:id | 必要 | admin, accountant, sales | - | 更新 |

### 4.10 ユーザー (Users)

| メソッド | パス | 認証 | 認可 | クエリパラメータ | 説明 |
|---------|------|------|------|----------------|------|
| GET | /api/v1/users | 必要 | admin | `role`, `active`, `page`, `per_page` | 一覧取得 |
| POST | /api/v1/users | 必要 | admin | - | ユーザー作成 |
| GET | /api/v1/users/:id | 必要 | admin, 本人 | - | 詳細取得 |
| PATCH | /api/v1/users/:id | 必要 | admin | - | 更新 (ロール変更等) |

### 4.11 ダッシュボード (Dashboard)

| メソッド | パス | 認証 | 認可 | 説明 |
|---------|------|------|------|------|
| GET | /api/v1/dashboard/summary | 必要 | 全ロール | サマリー情報 |
| GET | /api/v1/dashboard/revenue | 必要 | 全ロール | 売上情報 |
| GET | /api/v1/dashboard/receivables | 必要 | 全ロール | 売掛金情報 |

> **sales ロールのスコープ制限**: sales ユーザーは自身が `created_by_id` として紐づく顧客・契約に関連するデータのみ表示する（APP_REQUIREMENTS.md §6「自担当のみ」）。admin / accountant は全データを閲覧可能。

### 4.12 共通クエリパラメータ

| パラメータ | 型 | デフォルト | 説明 |
|-----------|-----|-----------|------|
| `page` | integer | 1 | ページ番号 |
| `per_page` | integer | 25 | 1 ページあたりの件数 (最大 100) |
| `sort` | string | エンドポイント依存 | ソートキー |
| `direction` | string | `desc` | ソート方向 (`asc` / `desc`) |
| `q` | string | - | 全文検索キーワード |

---

## 5. 請求書自動生成アーキテクチャ

### 5.1 処理フロー

```
┌─────────────────────────────────────────────────────────────────┐
│                     月次バッチ処理                                │
│                                                                 │
│  InvoiceGenerationJob (Sidekiq)                                 │
│       |                                                         │
│       v                                                         │
│  InvoiceGeneration::BatchService                                │
│       |                                                         │
│       ├── 1. 対象契約の抽出                                       │
│       │     active 状態の契約                                     │
│       │                                                         │
│       ├── 2. 重複チェック (idempotency_key)                       │
│       │     同一契約 × 同一請求期間の請求書が存在しないことを確認       │
│       │                                                         │
│       ├── 3. 契約ごとの請求書生成 (トランザクション内)               │
│       │     ├── Invoices::GenerateService                       │
│       │     │     ├── FixedCalculator (固定額明細)                │
│       │     │     └── UsageCalculator (従量明細)                  │
│       │     ├── Invoice レコード作成                               │
│       │     ├── InvoiceItem レコード作成                           │
│       │     └── 金額集計 (subtotal, tax, total)                   │
│       │                                                         │
│       └── 4. 結果ログ出力                                        │
│             成功件数、失敗件数、スキップ件数                         │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 バッチサービス実装

```ruby
# app/services/invoice_generation/batch_service.rb
module InvoiceGeneration
  class BatchService
    attr_reader :billing_period, :results

    def initialize(billing_period:)
      @billing_period = billing_period
      @results = { success: 0, skipped: 0, failed: 0, errors: [] }
    end

    def call
      target_contracts.find_each do |contract|
        process_contract(contract)
      end

      log_results
      Result.new(success: results[:failed].zero?, data: results)
    end

    private

    def target_contracts
      Contract
        .where(status: :active)
        .where("start_date <= ?", billing_period.end_of_month)
        .where("end_date >= ?", billing_period)
        .includes(:contract_items)
    end

    def process_contract(contract)
      idempotency_key = build_idempotency_key(contract)

      # べき等性チェック: 同一キーの請求書が既に存在する場合はスキップ
      if Invoice.exists?(idempotency_key:)
        @results[:skipped] += 1
        return
      end

      # 個別トランザクション: 1 契約の失敗が他に影響しない
      ActiveRecord::Base.transaction do
        result = Invoices::GenerateService.new(
          contract:,
          billing_period:,
          idempotency_key:
        ).call

        if result.success?
          @results[:success] += 1
        else
          raise ActiveRecord::Rollback
        end
      end
    rescue StandardError => e
      @results[:failed] += 1
      @results[:errors] << {
        contract_id: contract.id,
        contract_number: contract.contract_number,
        error: e.message
      }
      Rails.logger.error(
        "Invoice generation failed for contract #{contract.contract_number}: #{e.message}"
      )
    end

    def build_idempotency_key(contract)
      # フォーマット: "inv-{contract_id}-{billing_period_start}"
      "inv-#{contract.id}-#{billing_period.strftime('%Y%m')}"
    end

    def log_results
      Rails.logger.info(
        "Invoice generation completed: " \
        "success=#{results[:success]}, " \
        "skipped=#{results[:skipped]}, " \
        "failed=#{results[:failed]}"
      )
    end
  end
end
```

### 5.3 個別請求書生成サービス

```ruby
# app/services/invoices/generate_service.rb
module Invoices
  class GenerateService
    def initialize(contract:, billing_period:, idempotency_key:)
      @contract = contract
      @billing_period = billing_period
      @idempotency_key = idempotency_key
    end

    def call
      invoice = build_invoice
      generate_items(invoice)
      calculate_totals(invoice)
      invoice.save!

      Result.new(success: true, data: invoice)
    rescue ActiveRecord::RecordInvalid => e
      Result.new(success: false, errors: [e.message])
    end

    private

    def build_invoice
      Invoice.new(
        contract: @contract,
        client: @contract.client,
        invoice_number: generate_invoice_number,
        status: :draft,
        billing_period_start: @billing_period,
        billing_period_end: @billing_period.end_of_month,
        issue_date: @billing_period.end_of_month,
        due_date: @billing_period.end_of_month + @contract.payment_due_days.days,
        idempotency_key: @idempotency_key
      )
    end

    def generate_items(invoice)
      @contract.contract_items.each do |item|
        calculator = calculator_for(item)
        quantity = calculator.calculate_quantity(item, @billing_period)
        amount = calculator.calculate_amount(item, quantity)

        invoice.invoice_items.build(
          contract_item: item,
          name: item.name,
          unit_price: item.unit_price,
          quantity:,
          amount:,
          sort_order: item.sort_order
        )
      end
    end

    def calculator_for(item)
      case item.billing_type
      when "fixed"
        InvoiceGeneration::FixedCalculator.new
      when "usage"
        InvoiceGeneration::UsageCalculator.new
      when "tiered"
        InvoiceGeneration::TieredCalculator.new
      else
        raise InvoiceGenerationError, "Unknown billing_type: #{item.billing_type}"
      end
    end

    def calculate_totals(invoice)
      invoice.subtotal = invoice.invoice_items.sum(&:amount)
      invoice.tax_amount = (invoice.subtotal * 0.10).floor
      invoice.total_amount = invoice.subtotal + invoice.tax_amount
    end

    def generate_invoice_number
      prefix = "INV-#{@billing_period.strftime('%Y%m')}"
      last_number = Invoice.where("invoice_number LIKE ?", "#{prefix}%")
                           .order(invoice_number: :desc)
                           .pick(:invoice_number)

      if last_number
        seq = last_number.split("-").last.to_i + 1
        "#{prefix}-#{seq.to_s.rjust(5, '0')}"
      else
        "#{prefix}-00001"
      end
    end
  end
end
```

### 5.4 計算モジュール

```ruby
# app/services/invoice_generation/fixed_calculator.rb
module InvoiceGeneration
  class FixedCalculator
    def calculate_quantity(contract_item, _billing_period)
      contract_item.quantity
    end

    def calculate_amount(contract_item, quantity)
      (contract_item.unit_price * quantity).floor
    end
  end
end

# app/services/invoice_generation/usage_calculator.rb
module InvoiceGeneration
  class UsageCalculator
    def calculate_quantity(contract_item, billing_period)
      UsageRecord
        .where(contract_item:)
        .where(period_start: billing_period.., period_end: ..billing_period.end_of_month)
        .sum(:quantity)
    end

    def calculate_amount(contract_item, quantity)
      (contract_item.unit_price * quantity).floor
    end
  end
end
```

### 5.5 べき等性の確保

| 手段 | 実装 | 目的 |
|------|------|------|
| **idempotency_key** | `"inv-{contract_id}-{YYYYMM}"` (UNIQUE 制約) | 同一契約 x 同一月の重複生成防止 |
| **事前チェック** | `Invoice.exists?(idempotency_key:)` でスキップ | DB レベルの UNIQUE に頼る前にアプリレベルで回避 |
| **DB UNIQUE 制約** | `index_invoices_on_idempotency_key` (UNIQUE) | 競合時のフェイルセーフ |

ジョブが途中で失敗して再実行された場合:
1. 既に生成済みの請求書は idempotency_key で検出されスキップ
2. 未生成の契約のみ処理される
3. 全体の結果は初回実行と同一になる

### 5.6 トランザクション管理

```
全体バッチ (トランザクションなし)
  └── 契約 A (個別トランザクション)
  │     ├── Invoice 作成
  │     ├── InvoiceItem 作成 (複数)
  │     └── 金額集計 → COMMIT
  └── 契約 B (個別トランザクション)
  │     ├── Invoice 作成
  │     ├── エラー発生 → ROLLBACK
  │     └── エラーログ記録
  └── 契約 C (個別トランザクション)
        ├── Invoice 作成 (契約 B の失敗に影響されない)
        └── COMMIT
```

- **バッチ全体をトランザクションで囲まない**: 1 契約の失敗で全契約の請求書生成がロールバックされるのを防ぐ。
- **契約単位のトランザクション**: Invoice と InvoiceItem の整合性は保証する。

### 5.7 エラーリカバリ

| シナリオ | 挙動 | リカバリ |
|---------|------|---------|
| 個別契約でバリデーションエラー | 当該契約のみロールバック、他は継続 | エラーログ確認後、データ修正して再実行 |
| Sidekiq ワーカーがクラッシュ | Sidekiq のリトライ機構 (最大 3 回, 指数バックオフ) | idempotency_key により安全に再実行 |
| DB 接続断 | ジョブ全体が失敗、Sidekiq がリトライ | 接続回復後に自動リトライ |
| 全契約で失敗 | results[:failed] > 0 | 管理者に通知 (将来: Slack/メール連携) |

### 5.8 スケジューリング

```ruby
# config/initializers/sidekiq_cron.rb (sidekiq-cron gem 使用)
Sidekiq::Cron::Job.create(
  name: "Monthly invoice generation",
  cron: "0 2 1 * *",  # 毎月 1 日 AM 2:00 JST
  class: "InvoiceGenerationJob"
)

Sidekiq::Cron::Job.create(
  name: "Daily overdue check",
  cron: "0 9 * * 1-5",  # 平日 AM 9:00 JST
  class: "OverdueInvoiceCheckJob"
)
```

---

## 6. 認可設計 (Pundit)

### 6.1 ロール定義

| ロール | 説明 |
|--------|------|
| admin | 全操作可能。ユーザー管理、契約の状態遷移 |
| accountant | 請求書の承認・送付、入金記録、閲覧全般 |
| sales | 顧客・契約の作成・編集、使用量記録、閲覧全般 |

### 6.2 権限マトリクス

| リソース | admin | accountant | sales |
|---------|-------|-----------|-------|
| Client CRUD | 全て | 閲覧 | 作成/編集/閲覧 |
| Contract CRUD | 全て | 閲覧 | 作成/編集/閲覧 |
| Contract 状態遷移 | 全て | - | activate/terminate (from draft) |
| Invoice 閲覧 | 全て | 全て | 閲覧 |
| Invoice 発行/無効化 | 発行: admin,accountant / 無効化: admin | - | - |
| Invoice 無効化 | 全て | - | - |
| Payment | 全て | 記録/閲覧 | - |
| UsageRecord | 全て | 作成/編集/閲覧 | 作成/閲覧 |
| User 管理 | 全て | - | - |

### 6.3 Pundit ポリシー例

```ruby
# app/policies/invoice_policy.rb
class InvoicePolicy < ApplicationPolicy
  def index?
    true  # 全ロール閲覧可能
  end

  def show?
    true
  end

  def create?
    user.admin? || user.accountant?
  end

  def update?
    user.admin? || user.accountant?
  end

  def issue?
    user.admin? || user.accountant?
  end

  def void?
    user.admin?
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      scope.all  # 全ロールが全請求書を閲覧可能
    end
  end
end
```

---

## 7. OpenAPI 3.1 / committee gem 連携

### 7.1 スキーマ定義の配置

```
docs/openapi/
├── openapi.yaml              # ルート定義
├── paths/
│   ├── clients.yaml
│   ├── contracts.yaml
│   ├── invoices.yaml
│   ├── payments.yaml
│   └── usage_records.yaml
└── schemas/
    ├── client.yaml
    ├── contract.yaml
    ├── invoice.yaml
    ├── error.yaml
    └── pagination.yaml
```

### 7.2 committee gem によるテスト時スキーマ検証

```ruby
# spec/support/committee.rb
RSpec.configure do |config|
  config.add_setting :committee_options
  config.committee_options = {
    schema_path: Rails.root.join("../docs/openapi/openapi.yaml").to_s,
    parse_response_by_content_type: false
  }
end

# Request Spec 内での使用
RSpec.describe "GET /api/v1/invoices", type: :request do
  it "returns invoices conforming to OpenAPI schema" do
    get "/api/v1/invoices", headers: auth_headers
    assert_schema_conform(200)
  end
end
```

- **CI で自動検証**: Request Spec 実行時にレスポンスが OpenAPI 定義に適合しているかチェック。API 仕様と実装の乖離を防ぐ。

---

## 8. 横断的関心事

### 8.1 ログ設計

```ruby
# 構造化ログ (JSON)
# config/environments/production.rb
config.log_formatter = ::Logger::Formatter.new
config.logger = ActiveSupport::TaggedLogging.new(
  ActiveSupport::Logger.new($stdout)
)
```

### 8.2 Rack::Attack (レートリミット)

```ruby
# config/initializers/rack_attack.rb
Rack::Attack.throttle("api/v1/sessions", limit: 5, period: 60) do |req|
  req.ip if req.path == "/api/v1/sessions" && req.post?
end

Rack::Attack.throttle("api/general", limit: 300, period: 60) do |req|
  req.ip if req.path.start_with?("/api/")
end
```

### 8.3 CORS 設定

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV.fetch("CORS_ORIGINS", "http://localhost:4200")
    resource "/api/*",
             headers: :any,
             methods: %i[get post put patch delete options head],
             credentials: true,
             expose: %w[X-Total-Count X-Total-Pages X-Current-Page X-Per-Page]
  end
end
```
