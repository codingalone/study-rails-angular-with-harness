# アプリケーション要件定義: 契約・請求管理システム

> **WARNING: Public Repository** — 機密情報を記載しないこと。

---

## 1. システム概要

顧客と契約を登録し、契約に定義された請求ルール（固定月額・従量課金・段階制）に従って請求書を自動生成する業務システム。

### 目的

- ハーネスエンジニアリング学習の題材として、一定規模・一定複雑性の toB 業務ロジックを実装する
- 「定義されたルールに従って自動で動く」パターンの実践

### 対象ユーザー

単一組織の経理・営業・管理者（マルチテナントではない）。

---

## 2. ユーザーストーリー

### 認証・認可

| ID | ストーリー |
|----|-----------|
| US-01 | As a ユーザー, I want メールとパスワードでログインしたい, so that 権限に応じた機能を利用できる |
| US-02 | As a 管理者, I want ユーザーを登録しロールを割り当てたい, so that 適切な権限で業務を遂行させられる |
| US-03 | As a 管理者, I want ユーザーのロール・有効状態を変更したい, so that 人事異動・退職に対応できる |

### 顧客管理

| ID | ストーリー |
|----|-----------|
| US-04 | As a 営業, I want 顧客情報（会社名・住所・連絡先）を登録したい, so that 契約の紐付け先を管理できる |
| US-05 | As a 営業, I want 顧客一覧を会社名・コードで検索したい, so that 目的の顧客を素早く見つけられる |
| US-06 | As a 営業, I want 顧客を論理削除したい, so that 履歴を残しつつ一覧から非表示にできる |

### 契約管理

| ID | ストーリー |
|----|-----------|
| US-07 | As a 営業, I want 顧客に紐づく契約をドラフト作成したい, so that 内容を確認してから有効化できる |
| US-08 | As a 営業, I want 契約に明細行（固定月額・従量課金・段階制）を追加したい, so that 請求ルールを定義できる |
| US-09 | As a 管理者, I want 契約のステータスを遷移させたい, so that ライフサイクルを管理できる |
| US-10 | As a 営業, I want 契約の自動更新ルールを設定したい, so that 更新時に適切な処理が行われる |
| US-11 | As a 営業, I want 契約一覧を顧客名・ステータスで検索したい, so that 管理対象を把握できる |

### 請求書管理

| ID | ストーリー |
|----|-----------|
| US-12 | As a 経理, I want 月次締め処理で請求書を一括自動生成したい, so that 手作業なく正確な請求書が作成される |
| US-13 | As a 経理, I want 自動生成された請求書ドラフトを確認・修正してから発行したい, so that 誤りを是正できる |
| US-14 | As a 経理, I want 請求書のステータスを管理したい, so that 請求の進捗を追跡できる |
| US-15 | As a 経理, I want 臨時の請求書を手動作成したい, so that スポット請求にも対応できる |

### 入金管理

| ID | ストーリー |
|----|-----------|
| US-16 | As a 経理, I want 入金を記録し請求書に消込したい, so that 未収金を正確に把握できる |
| US-17 | As a 経理, I want 部分入金を記録したい, so that 分割払いに対応できる |
| US-18 | As a 経理, I want 過入金を記録したい, so that 入金差異を管理できる |

### 従量課金

| ID | ストーリー |
|----|-----------|
| US-19 | As a 運用担当, I want 使用量データを手動入力または API で登録したい, so that 月次請求に反映される |
| US-20 | As a 経理, I want 登録済みの使用量を確認・修正したい, so that 正確性を担保できる |

### ダッシュボード・レポート

| ID | ストーリー |
|----|-----------|
| US-21 | As a 管理者, I want 売上サマリー（月別）を表示したい, so that 収益状況を俯瞰できる |
| US-22 | As a 経理, I want 未収金一覧を確認したい, so that 回収の優先度を判断できる |
| US-23 | As a 管理者, I want 顧客別売上レポートを表示したい, so that 顧客ごとの収益を分析できる |

### 通知

| ID | ストーリー |
|----|-----------|
| US-24 | As a システム, I want 支払期限前通知・未払い督促をログ出力したい, so that 将来のメール通知拡張の基盤とする |

---

## 3. ドメインモデル

### エンティティ関連図

```
User (操作者)

Client (顧客)
  └── Contract (契約) ← 請求ルールを定義
        ├── ContractItem (契約明細) ← 品目・単価・数量ルール
        │     ├── TieredRate (段階料金) ← tiered のみ
        │     └── UsageRecord (使用量記録) ← usage/tiered のみ
        └── Invoice (請求書) ← 契約定義に従い自動生成
              ├── InvoiceItem (請求明細)
              └── Payment (入金) ← 消込処理
```

### エンティティ属性

#### User

| 属性 | 型 | 制約 |
|------|-----|------|
| email | string | UNIQUE, NOT NULL |
| password_digest | string | NOT NULL |
| name | string | NOT NULL |
| role | enum (admin/accountant/sales) | NOT NULL |
| active | boolean | DEFAULT true |

#### Client

| 属性 | 型 | 制約 |
|------|-----|------|
| code | string | UNIQUE, NOT NULL, 自動採番 `C-XXXXX` |
| name | string | NOT NULL |
| name_kana | string | |
| postal_code | string | |
| address | string | |
| phone | string | |
| email | string | |
| contact_person | string | 担当者名 |
| notes | text | |
| active | boolean | DEFAULT true（論理削除） |

#### Contract

| 属性 | 型 | 制約 |
|------|-----|------|
| client_id | bigint FK | NOT NULL |
| contract_number | string | UNIQUE, NOT NULL, 自動採番 `CTR-YYYYMMDD-XXXXX` |
| status | enum (draft/active/suspended/terminated) | NOT NULL |
| start_date | date | NOT NULL |
| end_date | date | NOT NULL |
| auto_renew | boolean | DEFAULT false |
| renewal_period_months | integer | DEFAULT 12 |
| billing_day | integer (1-28) | DEFAULT 28、締め日 |
| payment_due_days | integer | DEFAULT 30 |
| notes | text | |

#### ContractItem

| 属性 | 型 | 制約 |
|------|-----|------|
| contract_id | bigint FK | NOT NULL |
| name | string | NOT NULL（品目名） |
| billing_type | enum (fixed/usage/tiered) | NOT NULL |
| unit_price | decimal(10,2) | fixed の月額単価 |
| unit_name | string | usage/tiered の単位名（例: "GB"） |
| quantity | decimal(10,2) | fixed の数量、DEFAULT 1 |
| sort_order | integer | 表示順 |

#### TieredRate

| 属性 | 型 | 制約 |
|------|-----|------|
| contract_item_id | bigint FK | NOT NULL |
| lower_bound | decimal(10,2) | NOT NULL（下限、inclusive） |
| upper_bound | decimal(10,2) | NULL = 上限なし（exclusive） |
| unit_price | decimal(10,2) | NOT NULL（この区間の単価） |
| sort_order | integer | |

#### UsageRecord

| 属性 | 型 | 制約 |
|------|-----|------|
| contract_item_id | bigint FK | NOT NULL |
| period_start | date | NOT NULL |
| period_end | date | NOT NULL |
| quantity | decimal(10,2) | NOT NULL |
| recorded_at | datetime | NOT NULL |
| source | enum (manual/api) | NOT NULL |

#### Invoice

| 属性 | 型 | 制約 |
|------|-----|------|
| client_id | bigint FK | NOT NULL |
| contract_id | bigint FK | NULL（手動作成時） |
| invoice_number | string | UNIQUE, NOT NULL, 自動採番 `INV-YYYYMM-XXXXX` |
| status | enum (draft/issued/paid/overdue/void) | NOT NULL |
| issue_date | date | NOT NULL |
| due_date | date | NOT NULL |
| subtotal | integer | NOT NULL（税抜小計、円） |
| tax_amount | integer | NOT NULL（消費税額、円） |
| total_amount | integer | NOT NULL（税込合計、円） |
| paid_amount | integer | DEFAULT 0（入金済み額、円） |
| billing_period_start | date | 請求対象期間 開始 |
| billing_period_end | date | 請求対象期間 終了 |
| notes | text | |

#### InvoiceItem

| 属性 | 型 | 制約 |
|------|-----|------|
| invoice_id | bigint FK | NOT NULL |
| contract_item_id | bigint FK | NULL（手動作成時） |
| name | string | NOT NULL（品目名） |
| quantity | decimal(10,2) | NOT NULL |
| unit_price | decimal(10,2) | NOT NULL |
| amount | integer | NOT NULL（円） |
| sort_order | integer | |

#### Payment

| 属性 | 型 | 制約 |
|------|-----|------|
| invoice_id | bigint FK | NOT NULL |
| amount | integer | NOT NULL（円） |
| payment_date | date | NOT NULL |
| payment_method | enum (bank_transfer/other) | NOT NULL |
| notes | text | |

---

## 4. ビジネスルール

### 4.1 金額計算

- **固定月額 (fixed)**: `金額 = floor(unit_price * quantity)`
- **従量課金 (usage)**: `金額 = floor(unit_price * 使用量)`
- **段階制 (tiered)**: 累進方式。使用量を区間ごとに分割し、`floor(Σ 区間使用量 * 区間単価)` を合算
  - 例: 0-100 @10、101-500 @8、501- @5 → 使用量 600 = `floor(100*10) + floor(400*8) + floor(100*5) = 4,700`

### 4.2 消費税計算

- 税率: 10% 固定（軽減税率対象外）
- 計算方式: **請求書単位**で算出（明細ごとではない）
- `tax_amount = floor(subtotal * 0.10)`
- `total_amount = subtotal + tax_amount`
- 通貨: 日本円（JPY）、最小単位 1 円

### 4.3 請求書自動生成（月次締め処理）

1. 対象年月を指定して実行（例: 2026 年 3 月分）
2. ステータス `active` の契約で、対象年月が契約期間内のものを抽出
3. **冪等性**: 同一契約・同一対象月の請求書が存在する場合はスキップ（DB ユニーク制約）
4. 各契約明細から請求明細を生成:
   - fixed: `unit_price * quantity`
   - usage/tiered: 対象期間の UsageRecord を集計して計算
5. 使用量レコード未登録の場合は数量 0 で明細生成
6. 生成された請求書のステータスは `draft`
7. `due_date = issue_date + payment_due_days`

### 4.4 入金消込

- 1 請求書に対して複数回入金可能（部分入金対応）
- `paid_amount = SUM(payments.amount)` で Invoice に反映
- `paid_amount >= total_amount` → ステータスを `paid` に自動遷移
- `paid_amount > total_amount`（過入金）→ 差額を notes に記録（自動充当はスコープ外）

### 4.5 期限超過

- `due_date` 経過 + ステータス `issued` → `overdue` に遷移
- 日次バッチジョブで自動チェック

### 4.6 契約自動更新

- `auto_renew = true` かつ `end_date` 到来 → `renewal_period_months` 分だけ `end_date` を延長
- 日次バッチで実行

### 4.7 採番ルール

| 対象 | 形式 | 一意性保証 |
|------|------|-----------|
| 顧客コード | `C-XXXXX` | DB シーケンス |
| 契約番号 | `CTR-YYYYMMDD-XXXXX` | DB シーケンス |
| 請求書番号 | `INV-YYYYMM-XXXXX` | DB シーケンス |

---

## 5. 状態遷移

### 5.1 契約 (Contract)

```
[作成] → draft
            │
            ├── activate → active
            │                ├── suspend → suspended
            │                │                ├── resume → active
            │                │                └── terminate → terminated
            │                └── terminate → terminated
            └── terminate → terminated
```

| From | To | 条件 | 操作者 |
|------|----|------|--------|
| draft | active | 必須項目入力済み、明細 1 件以上 | admin, sales |
| draft | terminated | — | admin, sales |
| active | suspended | 理由を notes に記録 | admin |
| active | terminated | 理由を notes に記録 | admin |
| suspended | active | — | admin |
| suspended | terminated | — | admin |

- `terminated` は終端状態（遷移不可）
- `active` な契約の明細は編集不可（suspended にしてから編集）

### 5.2 請求書 (Invoice)

```
[生成] → draft
            ├── issue → issued
            │             ├── [入金] → paid (paid_amount >= total_amount)
            │             ├── [期限超過] → overdue
            │             │                  ├── [入金] → paid
            │             │                  └── void → void
            │             └── void → void
            └── void → void
```

| From | To | 条件 | 操作者 |
|------|----|------|--------|
| draft | issued | 明細 1 件以上 | admin, accountant |
| draft | void | 取消理由を notes に記録 | admin, accountant |
| issued | paid | `paid_amount >= total_amount` | システム自動 |
| issued | overdue | `due_date < 現在日` | システム自動 |
| issued | void | 取消理由を notes に記録 | admin |
| overdue | paid | `paid_amount >= total_amount` | システム自動 |
| overdue | void | 取消理由を notes に記録 | admin |

- `paid`, `void` は終端状態
- `issued` / `overdue` の請求書は明細編集不可

---

## 6. 認可ルール

| 操作 | admin | accountant | sales |
|------|-------|------------|-------|
| ユーザー管理 | CRUD | — | — |
| 顧客管理 | CRUD | Read | CRUD |
| 契約管理 | CRUD | Read | CRUD |
| 請求書管理 | CRUD | CRUD | Read |
| 入金管理 | CRUD | CRUD | — |
| 使用量管理 | CRUD | CRUD | Read/Create |
| ダッシュボード | Full | Full | 自担当のみ |

---

## 7. 非機能要件

### パフォーマンス

- API レスポンス: 一覧 500ms 以内、単一取得 200ms 以内（Docker ローカル基準）
- 月次締め処理: 100 契約の一括生成が 30 秒以内（Sidekiq 非同期）
- ページネーション: 1 ページ 25 件

### セキュリティ

- JWT RS256 認証（ARCHITECTURE.md 準拠）
- Pundit による全エンドポイントの認可チェック
- 金額フィールドは integer 型（浮動小数点回避）
- 金額計算はすべてサーバーサイド（クライアント計算は表示用）
- 金額変更・ステータス変更は操作ログ記録（Rails ログ）
- Rack::Attack によるレートリミット（ログイン）

### データ整合性

- 請求書の金額は DB に永続化（発行時点の金額を保持、都度計算しない）
- 同一契約・同一対象月の請求書重複防止（DB ユニーク制約）
- 入金消込の `paid_amount` 更新はトランザクション内で実行

### バックグラウンドジョブ

- Sidekiq + Redis
- ジョブ: 月次請求書生成、期限超過チェック、契約自動更新
- リトライ: 最大 3 回、指数バックオフ
- 冪等性保証

---

## 8. スコープ外

| 項目 | 理由 |
|------|------|
| マルチテナント | 単一組織利用 |
| 複数通貨 | 日本円のみ |
| 軽減税率・インボイス制度 | 学習用途では過剰 |
| メール送信 | ログ出力で代替 |
| PDF 請求書生成 | UI 表示のみ |
| 外部決済連携 | 入金は手動記録 |
| 会計ソフト連携 | スコープ外 |
| 承認ワークフロー | 少人数想定 |
| 過入金の自動充当 | 手動対応 |
| ファイル添付 | 将来拡張 |
| 国際化 (i18n) | 日本語のみ |
