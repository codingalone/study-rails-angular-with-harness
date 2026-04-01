# テスト戦略

> **WARNING: Public Repository** — テストデータに実際の認証情報やメールアドレスを使用しないこと。

---

## 1. テスト哲学

本プロジェクトではユーザーがコードレビューを行わない。**テストが品質保証の唯一の手段**であり、コードレビューの代替として機能する。

テストが保証するもの:
- 仕様通りに動作すること（受入基準の検証）
- リグレッションが発生していないこと（回帰テスト）
- セキュリティ上の問題がないこと（静的解析 + セキュリティテスト）

テストだけでは保証できないもの（静的解析で補完）:
- コードの可読性・設計品質 → RuboCop, ESLint, 型チェック
- セキュリティ脆弱性 → Brakeman, npm audit
- コードの重複・複雑度 → flog, flay, ESLint complexity

---

## 2. テストピラミッド

```
        ┌─────────┐
        │  E2E    │  10% — Playwright（主要ユーザー導線）
        │         │
       ─┼─────────┼─
       │Integration│  20% — Request Spec (Rails), TestBed (Angular)
       │           │
      ─┼───────────┼─
      │    Unit     │  70% — Model/Service (Rails), Component/Service (Angular)
      └─────────────┘
```

---

## 3. カバレッジ定義

### 主指標: 分岐カバレッジ (Branch Coverage) — 95% 以上

行カバレッジ 100% でも `if x && y` の片方だけ検証されるケースがある。分岐カバレッジで複合条件を漏れなく検証する。

### 補助指標: 行カバレッジ (Line Coverage) — 100%

「通っていないコード行」を視覚的に特定するために使用。

### 除外対象

| 対象 | 理由 |
|------|------|
| `db/schema.rb` | Rails 自動生成 |
| `db/migrate/` | マイグレーションファイル |
| `config/` | 設定ファイル |
| `environment.ts` | Angular 環境設定 |
| ルーティング定義 | フレームワーク定義 |

---

## 4. Rails バックエンド テスト戦略

### テスティングフレームワーク

| ツール | 用途 |
|--------|------|
| RSpec | テストフレームワーク |
| FactoryBot | テストデータ生成 |
| Shoulda Matchers | バリデーション・アソシエーションの簡潔なテスト |
| SimpleCov | カバレッジ計測（branch: true 有効化） |
| committee | OpenAPI スキーマとレスポンスの整合性検証 |
| Brakeman | セキュリティ静的解析 |

### テスト種別

| 種別 | 対象 | 配置場所 |
|------|------|---------|
| Model Spec | バリデーション、スコープ、ビジネスロジック | `spec/models/` |
| Request Spec | API エンドポイント（正常系 + 異常系） | `spec/requests/` |
| Service Spec | サービスオブジェクト | `spec/services/` |
| Job Spec | バックグラウンドジョブ | `spec/jobs/` |
| Policy Spec | 認可ポリシー (Pundit) | `spec/policies/` |

### データベースクリーニング

- `database_cleaner-active_record` でトランザクション方式を使用
- 各テスト後にロールバック（高速）

### SimpleCov 設定方針

```ruby
SimpleCov.start 'rails' do
  enable_coverage :branch  # 分岐カバレッジ有効化
  minimum_coverage line: 100, branch: 95
  add_filter '/spec/'
  add_filter '/config/'
  add_filter '/db/'
end
```

---

## 5. Angular フロントエンド テスト戦略

### テスティングフレームワーク

| ツール | 用途 |
|--------|------|
| Jest | ユニット・統合テスト（Karma より高速） |
| @testing-library/angular | コンポーネントテスト（ユーザー視点） |
| jest-preset-angular | Angular 向け Jest 設定 |

### テスト種別

| 種別 | 対象 | 方法 |
|------|------|------|
| Component Test | 表示・インタラクション | TestBed + @testing-library |
| Service Test | HTTP 通信、ビジネスロジック | HttpClientTestingModule |
| Pipe Test | データ変換 | 直接呼び出し |
| Guard Test | ルーティングガード | RouterTestingModule |

### Jest カバレッジ設定方針

```json
{
  "coverageThreshold": {
    "global": {
      "branches": 95,
      "functions": 100,
      "lines": 100,
      "statements": 100
    }
  }
}
```

---

## 6. E2E テスト戦略

### フレームワーク: Playwright

選定理由:
- マルチブラウザ対応（Chromium, Firefox, WebKit）
- Angular 公式サポート（Angular CLI v19+ でビルトインサポート）
- auto-wait 機構で flaky test が少ない
- API テスト機能内蔵（`request` コンテキスト）
- 高速並列実行

### テストシナリオ設計方針

- 仕様書の受入基準 (Given-When-Then) を直接テストケースに変換
- 主要ユーザー導線のみ E2E 化（全パスの網羅はユニット/統合に任せる）
- 1 テスト = 1 ユーザーストーリーの完結したフロー

### テストファイル配置

```
e2e/
├── tests/
│   ├── auth/
│   │   └── login.spec.ts        # Spec: FEAT-XXXX
│   └── resources/
│       └── crud.spec.ts         # Spec: FEAT-XXXX
├── fixtures/
├── playwright.config.ts
└── package.json
```

---

## 7. テストデータ管理

| 方式 | 用途 |
|------|------|
| FactoryBot | RSpec でのテストデータ生成（推奨） |
| seeds.rb | 開発環境の初期データ |
| Fixture (Angular) | Jest でのモックデータ |
| Playwright fixtures | E2E テストのセットアップ |

---

## 8. テスト実行環境

### ローカル (Docker 内)

```bash
# Rails
docker compose exec api bundle exec rspec
docker compose exec api bundle exec rspec --format documentation

# Angular
docker compose exec web npx jest --coverage

# E2E
docker compose exec e2e npx playwright test
```

### CI (GitHub Actions)

- PR 時: lint → unit test → integration test → coverage gate → security scan
- main push 時: 上記 + E2E test → build → deploy

---

## 9. 仕様駆動テスト開発フロー

```
仕様書 (Given-When-Then)
    |
    v
テストケース作成（RED 確認）
    |
    v
実装（GREEN 確認）
    |
    v
リファクタリング（GREEN 維持）
    |
    v
カバレッジ確認（95%+ branch / 100% line）
```

### 仕様からテストへの変換ルール

| 仕様の要素 | テストの要素 |
|-----------|-------------|
| 受入基準タイトル | `it()` / `describe` のテストケース名 |
| Given | テストのセットアップ |
| When | テストのアクション |
| Then | テストのアサーション |
| 仕様 ID | テストファイル冒頭のコメント `# Spec: FEAT-XXXX` |

---

## 10. 品質補完策（コードレビュー代替）

テストだけでは検出できない品質問題を補完する:

| ツール | 対象 | 検出対象 |
|--------|------|---------|
| RuboCop | Rails | コーディング規約、コード品質 |
| ESLint + @angular-eslint | Angular | コーディング規約、型安全性 |
| TypeScript strict mode | Angular | 型の不整合 |
| Brakeman | Rails | セキュリティ脆弱性 |
| npm audit | Angular | 依存パッケージ脆弱性 |
| committee | Rails | API スキーマ整合性 |
