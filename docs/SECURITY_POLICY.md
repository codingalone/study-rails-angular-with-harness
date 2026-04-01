# セキュリティポリシー

> **WARNING: Public Repository** — 本ドキュメントに具体的な AWS アカウント ID、IP アドレス、ARN、内部ドメインを記載しないこと。

---

## 1. セキュリティ原則

| 原則 | 説明 |
|------|------|
| 最小権限 | すべてのユーザー・サービスに必要最小限の権限のみ付与 |
| 多層防御 | 単一の防御層に依存しない（WAF + SG + 認証 + 入力検証） |
| フェイルセキュア | 障害時はアクセスを拒否する方向に倒す |
| セキュア・バイ・デフォルト | 新規リソースはデフォルトで最も制限的な設定 |

---

## 2. Public リポジトリの機密情報管理

**本リポジトリは Public であり、すべてのコミット履歴が公開される。**

### 絶対にコミットしてはならないもの

- API キー、トークン、シークレット
- パスワード、認証情報
- 秘密鍵（SSH, GPG, JWT, SSL）
- データベース接続文字列（パスワード入り）
- AWS アカウント ID、IAM ロール ARN
- IP アドレス、内部ドメイン名
- `.env` ファイルの内容
- `terraform.tfstate`、`terraform.tfvars`
- `config/master.key`、`config/credentials/*.key`

### 多層防御による漏洩防止

| レイヤー | ツール | タイミング |
|---------|--------|-----------|
| ローカル | pre-commit hook (Claude CLI) | コミット前 |
| ローカル | .gitignore | 常時 |
| CI | gitleaks / TruffleHog | PR / push 時 |
| GitHub | Secret Scanning (GitHub 標準) | 常時 |

### pre-commit hook

`~/dotfiles/hooks/pre-commit` を `.git/hooks/` にコピーして使用する。staged diff を Claude に渡し、秘密情報の有無を PASS/FAIL 判定する。

- `--no-verify` によるスキップは**原則禁止**
- Claude CLI 未インストール環境ではフォールバックとして警告を出力

### 万が一漏洩した場合の対応

1. 即座に該当シークレットをローテーション（新しいキーを発行、旧キーを無効化）
2. `git filter-branch` または BFG Repo-Cleaner で履歴から除去
3. GitHub にキャッシュ削除を依頼
4. 影響範囲を調査し、不正利用の有無を確認

---

## 3. アプリケーションセキュリティ

### OWASP Top 10 対策

| # | リスク | 対策 |
|---|--------|------|
| A01 | アクセス制御の不備 | Pundit で全コントローラーに認可チェック |
| A02 | 暗号化の失敗 | TLS 1.2+、bcrypt (cost 12+)、KMS 暗号化 |
| A03 | インジェクション | Active Record パラメータバインディング、Strong Parameters |
| A04 | 安全でない設計 | 脅威モデリング、仕様レビュー |
| A05 | 設定ミス | 本番デバッグモード無効、不要エンドポイント閉塞 |
| A06 | 脆弱なコンポーネント | Dependabot、Bundler Audit、npm audit |
| A07 | 認証の失敗 | JWT RS256、Rack::Attack（ブルートフォース対策） |
| A08 | データ整合性の不備 | ロックファイル必須コミット、Actions SHA ピン留め |
| A09 | ログ監視の不備 | 構造化ログ、CloudWatch、セキュリティアラート |
| A10 | SSRF | 外部 URL ホワイトリスト、プライベート IP レンジブロック |

### 入力バリデーション

- **Rails**: Strong Parameters + モデルバリデーション（サーバーが唯一の信頼源）
- **Angular**: クライアント側バリデーションは UX 目的のみ。`bypassSecurityTrust*` は禁止

### CORS

- `origins '*'` は全環境で禁止
- 本番ドメインのみ明示許可
- `credentials: true` 使用時はワイルドカード不可

### CSP

```
default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline';
img-src 'self' data:; font-src 'self'; connect-src 'self';
frame-ancestors 'none'; base-uri 'self'; form-action 'self';
```

---

## 4. インフラセキュリティ

### IAM

- ルートアカウントは MFA 有効化の上、日常使用禁止
- サービスには IAM ロールを使用（アクセスキー発行禁止）
- `*` ワイルドカード権限は禁止

### VPC / セキュリティグループ

| SG | Inbound | Outbound |
|----|---------|----------|
| sg-alb | 許可 IP:443 | sg-app:3000 |
| sg-app | sg-alb:3000 | sg-db:5432, 443 (HTTPS) |
| sg-db | sg-app:5432 | なし |

- RDS のパブリックアクセスは絶対に許可しない
- `0.0.0.0/0` Inbound は sg-alb の 443 のみ（さらに WAF で IP 制限）

### 暗号化

- 通信: TLS 1.2+ 強制（ALB, RDS `sslmode=verify-full`）
- 保管: AES-256 (KMS CMK) — RDS, S3, EBS, Secrets Manager

### シークレット管理

| 種別 | 保管先 | コスト |
|------|--------|--------|
| DB パスワード、JWT 秘密鍵 | SSM Parameter Store (SecureString) | 無料 |
| 高頻度ローテーション対象 | Secrets Manager | $0.40/個/月 |
| 設定値（非機密） | SSM Parameter Store (String) | 無料 |

---

## 5. CI/CD セキュリティ

- AWS 認証: GitHub Actions **OIDC**（長期アクセスキー不使用）
- OIDC subject 条件: `repo:codingalone/study-rails-angular-with-harness:ref:refs/heads/main` に厳密制限
- サードパーティ Actions: **SHA でピン留め**（タグ指定禁止）
- `pull_request_target` トリガー: **使用禁止**（Public リポジトリでの fork 攻撃防止）
- シークレットのログ出力: `::add-mask::` で自動マスク

---

## 6. コスト管理セキュリティ

| 対策 | 設定 |
|------|------|
| AWS Budgets | 月額 $50 / $100 / $200 でアラート |
| Cost Anomaly Detection | 有効化（異常コスト自動検知） |
| SCP (リージョン制限) | ap-northeast-1, us-east-1 のみ許可 |
| SCP (インスタンス制限) | t3.*/t3a.*/t4g.* のみ許可 |
| ECS スケーリング上限 | 最大タスク数: 4 |
| RDS ストレージ上限 | max_allocated_storage: 50GB |
| GuardDuty | 有効化（不正アクセス・クリプトマイニング検知） |
| CloudTrail | 全リージョン有効化 |

---

## 7. アクセス制御（クローズド環境）

### 多層アクセス制限

| レイヤー | 方式 | 必須度 |
|---------|------|--------|
| L1 | WAF IP ホワイトリスト（デフォルト拒否） | 必須 |
| L2 | WAF マネージドルール (SQLi, XSS) | 必須 |
| L3 | アプリケーション認証 (JWT) | 必須 |
| L4 | VPN (将来検討) | 推奨 |

### SSH アクセス

- 直接 SSH は禁止。AWS Systems Manager Session Manager を使用
- セキュリティグループの 22 番ポートは閉じる

---

## 8. セキュリティテスト

| テスト | ツール | タイミング | ブロッキング |
|--------|--------|-----------|-------------|
| SAST (Rails) | Brakeman | PR / push | High 以上で失敗 |
| SAST (Angular) | ESLint Security | PR / push | Error で失敗 |
| 依存関係 (Ruby) | Bundler Audit | PR / daily | High 以上で失敗 |
| 依存関係 (Node) | npm audit | PR / daily | High 以上で失敗 |
| シークレット検出 | gitleaks / TruffleHog | PR | 検出で失敗 |
| コンテナスキャン | Trivy | イメージビルド時 | Critical/High で失敗 |
| IaC スキャン | checkov / tfsec | IaC 変更時 | High 以上で失敗 |

---

## 9. インシデント対応

### 分類

| 重要度 | 定義 | 初動対応 |
|--------|------|---------|
| P1 | データ漏洩確認、不正アクセス | 15 分以内 |
| P2 | 不正アクセスの疑い、サービス障害 | 1 時間以内 |
| P3 | 設定ミス発見、低リスク脆弱性 | 翌営業日 |

### 対応フロー

検知 → トリアージ → 封じ込め → 根絶 → 復旧 → ポストモーテム
