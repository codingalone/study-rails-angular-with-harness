# 仕様駆動開発プロセス

> **WARNING: Public Repository** — 仕様書内にも機密情報（API キー、実パスワード等）を記載しないこと。

---

## 1. 仕様駆動開発とは

**仕様が唯一の権威ある情報源 (Single Source of Truth)** であり、すべての実装は仕様に基づく。

- コードは仕様の実装物
- テストは仕様の検証手段
- 仕様に記述されていない振る舞いは存在してはならない

### ユーザーがコードレビューをしない環境での意義

| 課題 | 仕様駆動開発による解決 |
|------|----------------------|
| AI が何を実装すべきか不明確 | 仕様が契約として機能 |
| 生成コードの品質担保 | 仕様から導出されたテストが自動検証 |
| ユーザー意図と AI 実装の乖離 | 仕様レビューで事前合意 |
| 変更時の影響範囲不明 | 仕様間トレーサビリティで影響分析 |

---

## 2. 仕様のライフサイクル

```
draft → review → approved → implemented → verified
  ↑        |                                  |
  └── (差し戻し)                        deprecated (後継仕様に置換)
```

| ステータス | 意味 | 遷移条件 |
|-----------|------|---------|
| draft | AI が起草した初稿 | 作成時点 |
| review | ユーザーレビュー中 | AI がレビュー依頼した時点 |
| approved | ユーザー承認済み | ユーザーが承認した時点 |
| implemented | 実装完了 | テストとコードがコミットされた時点 |
| verified | 検証完了 | CI で全テスト合格 + カバレッジ達成 |
| deprecated | 廃止 | 後継仕様に置換された時点 |

---

## 3. 仕様書テンプレート

### 3.1 機能仕様 (FEAT)

```yaml
---
spec_id: "FEAT-XXXX"
title: "機能名"
status: draft
version: "1.0.0"
created_at: "YYYY-MM-DD"
updated_at: "YYYY-MM-DD"
related_specs: []
---
```

必須セクション:
1. 概要
2. ユーザーストーリー (As a ... I want ... so that ...)
3. 機能要件
4. 受入基準 (Given-When-Then 形式)
5. 非機能要件（該当する場合）
6. テストマッピング（テストファイルとの紐付け）

### 3.2 API 仕様 (API)

必須セクション:
1. エンドポイント定義（メソッド、パス、認証、認可）
2. リクエスト（パラメータ、ボディ、バリデーションルール）
3. レスポンス（成功 + 全エラーケース）
4. テストマッピング

### 3.3 画面仕様 (UI)

必須セクション:
1. URL / ルーティング
2. コンポーネント構成
3. 表示要素・操作要素
4. 状態管理
5. API 連携
6. テストマッピング

### 3.4 インフラ仕様 (INFRA)

必須セクション:
1. リソース定義
2. セキュリティ要件
3. コスト見積もり
4. テストマッピング（IaC テスト）

---

## 4. 仕様からテストへの変換

### 受入基準 → E2E テスト

```markdown
### AC-001: ログインに成功する
- Given: ユーザーがログイン画面にいる
- When: 正しいメールアドレスとパスワードを入力して送信する
- Then: ダッシュボード画面にリダイレクトされる
```

→

```typescript
// Spec: FEAT-0001, AC-001
test('AC-001: ログインに成功する', async ({ page }) => {
  // Given
  await page.goto('/login');
  // When
  await page.fill('[data-testid="email"]', 'user@example.com');
  await page.fill('[data-testid="password"]', 'password123');
  await page.click('[data-testid="submit"]');
  // Then
  await expect(page).toHaveURL(/dashboard/);
});
```

### API 仕様 → Request Spec

```ruby
# Spec: API-0001
RSpec.describe "POST /api/v1/resources", type: :request do
  context "有効なパラメータ" do
    it "201 を返しリソースが作成される" do
      post "/api/v1/resources", params: valid_params, headers: auth_headers
      expect(response).to have_http_status(:created)
      expect(json_response["data"]["name"]).to eq("test")
    end
  end

  context "認証トークンなし" do
    it "401 を返す" do
      post "/api/v1/resources", params: valid_params
      expect(response).to have_http_status(:unauthorized)
    end
  end
end
```

### バリデーションルール → Model Spec

```ruby
# Spec: API-0001
RSpec.describe Resource, type: :model do
  it { is_expected.to validate_presence_of(:name) }
  it { is_expected.to validate_length_of(:name).is_at_most(255) }
  it { is_expected.to validate_uniqueness_of(:name).scoped_to(:user_id) }
end
```

---

## 5. 仕様書の管理

### ディレクトリ構成

```
docs/specs/
├── features/          FEAT-0001-user-authentication.md
├── api/               API-0001-create-resource.md
├── ui/                UI-0001-login-screen.md
└── infrastructure/    INFRA-0001-vpc-setup.md
```

### 命名規則

- ID: `{TYPE}-{4桁連番}` (例: FEAT-0001)
- ファイル名: `{ID}-{kebab-case-summary}.md`
- 欠番は再利用しない

### バージョン管理

- Git で管理。変更履歴はコミットで追跡
- コミットメッセージ: `spec(FEAT-0001): 受入基準を追加`
- Approved 後の変更は新バージョンとして再レビュー

---

## 6. 仕様レビュープロセス

```
AI が仕様ドラフトを起草
    ↓
AI がユーザーに仕様を提示（status: review）
    ↓
ユーザーがレビュー
    ├── 承認 → status: approved → 実装フェーズへ
    └── 修正要求 → status: draft → AI が修正して再提示
```

### 承認の記録

- 仕様書のフロントマター `status: approved` を更新
- コミットメッセージ: `spec(FEAT-0001): approved`

### AI 自己批判プロンプト

仕様を提示する際、AI は以下を自問し回答に含める:
- この仕様で起こりうる問題を 3 つ
- 未確認事項 (open questions) があれば列挙

---

## 7. トレーサビリティ

### テストファイルとの紐付け

テストファイル冒頭に仕様 ID をコメントで記載:

```ruby
# Spec: API-0001, FEAT-0001
```

```typescript
// Spec: UI-0001, FEAT-0001
```

### 仕様変更時の影響分析

1. 変更対象の仕様 ID で関連テストを検索: `grep -r "Spec: .*FEAT-0001" spec/ e2e/`
2. 関連仕様を `related_specs` フィールドで確認
3. テストを更新し、全テスト合格を確認

---

## 8. API 仕様管理 (OpenAPI)

- API 仕様書（上流）→ OpenAPI 定義（下流）の二重管理
- `docs/openapi/` に OpenAPI 3.1 定義を配置
- committee gem でテスト時にレスポンスが OpenAPI 定義に準拠しているか自動検証
- 仕様書と OpenAPI 定義は同一コミットで同期
