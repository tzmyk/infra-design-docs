# インフラテスト方針

## テストの目的

**設計書に書かれた期待値と、実際にデプロイされた AWS リソースが一致していることを確認する。**

設計書がいくら正確でも、実装がその通りになっていなければ意味がない。  
このテストは「設計書 = 仕様書」として機能させるための手段。

---

## テストの考え方

```
設計書（期待値）
    ↓
テストコード（期待値を boto3 で検証するコードに変換）
    ↓
実際の AWS リソース（テスト対象）
    ↓
差分レポート（設計と乖離している箇所を報告）
```

### 何をテストするか

設計書の中で **具体的な値が明記されている箇所** がすべてテスト対象になる。  
「適切に設定する」のような曖昧な記述はテストできない。これが設計書に具体的な値を書く理由でもある。

---

## テストカテゴリと対応する設計書

| カテゴリ | テストファイル | 参照する設計書 |
|---|---|---|
| ネットワーク | `test_network.py` | `docs/01_network/vpc-design.md` |
| セキュリティ | `test_security.py` | `docs/02_security/security-design.md` |
| IAM | `test_iam.py` | `docs/03_iam/roles.md` |
| コンピューティング | `test_compute.py` | `docs/04_compute/compute-design.md` |

---

## テストツール

| ツール | 用途 |
|---|---|
| pytest | テストフレームワーク |
| boto3 | AWS リソースへの問い合わせ |

```bash
# セットアップ
python3 -m venv .venv
.venv/bin/pip install pytest boto3

# 実行（要: AWS 認証情報の設定）
.venv/bin/pytest tests/ -v
```

### AWS 認証について

テストは読み取り専用の操作のみ行う。  
`ReadOnlyAccess` ポリシーを持つ IAM ロール / プロファイルを使うこと。

```bash
# プロファイルを指定して実行する場合
AWS_PROFILE=readonly .venv/bin/pytest tests/ -v
```

---

## テストケース一覧

### test_network.py

設計書 `docs/01_network/vpc-design.md` の内容を検証する。

| テストケース | 検証内容 | 設計書の根拠 |
|---|---|---|
| `test_vpc_exists` | VPC `order-vpc` が存在すること | サブネット一覧 |
| `test_vpc_cidr` | VPC の CIDR が `10.0.0.0/16` であること | サブネット一覧 |
| `test_subnets_exist` | 6 つのサブネットが存在し、それぞれ CIDR・AZ が正しいこと | サブネット一覧 |
| `test_db_subnet_no_internet_route` | DB サブネットのルートテーブルに IGW・NAT の経路がないこと | サブネット一覧「インターネット経路: なし」 |
| `test_app_subnet_uses_nat` | アプリサブネットのルートテーブルに NAT Gateway 経由の経路があること | サブネット一覧「NAT Gateway 経由」 |
| `test_sg_db_no_outbound` | `order-db-sg` のアウトバウンドルールが空であること | SG 一覧「なし」 |
| `test_sg_alb_inbound_443_only` | `order-alb-sg` のインバウンドが 443/tcp のみであること | SG 一覧 |
| `test_sg_app_inbound_from_alb_only` | `order-app-sg` のインバウンドが `order-alb-sg` からのみであること | SG 一覧 |
| `test_sg_uses_sg_reference_not_cidr` | `order-app-sg` → `order-db-sg` の許可が CIDR ではなく SG 参照であること | SG 間参照の原則 |

### test_security.py

設計書 `docs/02_security/security-design.md` の内容を検証する。

| テストケース | 検証内容 | 設計書の根拠 |
|---|---|---|
| `test_s3_block_public_access` | S3 バケットの BlockPublicAccess が 4 項目すべて有効であること | 禁止事項 |
| `test_s3_enforce_tls` | S3 バケットポリシーに `aws:SecureTransport: false` を拒否するステートメントがあること | 禁止事項 |
| `test_s3_versioning_enabled` | `order-receipts-*` バケットのバージョニングが有効であること | 暗号化方針 |
| `test_rds_ssl_enforced` | Aurora クラスターのパラメータグループで `rds.force_ssl=1` が設定されていること | 暗号化方針 |
| `test_cloudtrail_enabled` | CloudTrail が有効で全リージョンのログが S3 に送られていること | 証跡・ログ設計 |
| `test_secrets_rotation_enabled` | `order/db/master` のシークレットで自動ローテーションが有効であること | シークレット管理 |

### test_iam.py

設計書 `docs/03_iam/roles.md` の内容を検証する。

| テストケース | 検証内容 | 設計書の根拠 |
|---|---|---|
| `test_roles_exist` | 設計書に記載された 5 つのロールが存在すること | ロール一覧 |
| `test_lambda_create_sqs_resource_scoped` | `order-lambda-create-role` の `sqs:SendMessage` が `order-queue` のみに限定されていること | roles.md 詳細 |
| `test_lambda_create_no_wildcard_resource` | `order-lambda-create-role` のポリシーに `"Resource": "*"` が含まれないこと（X-Ray・CloudWatch Logs を除く）| 禁止事項 |
| `test_lambda_process_s3_prefix_scoped` | `order-lambda-process-role` の `s3:PutObject` がバケット全体ではなくプレフィックス指定であること | roles.md 詳細 |
| `test_deploy_role_oidc_condition` | `order-deploy-role` の信頼ポリシーに main ブランチの条件が設定されていること | roles.md 詳細 |
| `test_no_admin_access_policy` | どのロールにも `AdministratorAccess` ポリシーがアタッチされていないこと | 禁止事項 |

### test_compute.py

設計書 `docs/04_compute/compute-design.md` の内容を検証する。

| テストケース | 検証内容 | 設計書の根拠 |
|---|---|---|
| `test_lambda_runtime` | 各 Lambda 関数のランタイムが `python3.12` であること | 共通設定 |
| `test_lambda_architecture` | 各 Lambda 関数のアーキテクチャが `arm64` であること | 共通設定 |
| `test_lambda_reserved_concurrency_set` | 各 Lambda 関数に Reserved Concurrency が設定されていること（値は 100 以下）| 禁止事項 |
| `test_lambda_timeout` | 各 Lambda 関数のタイムアウトが設計書の値と一致すること | 関数一覧 |
| `test_lambda_dlq_configured` | 各 Lambda 関数に DLQ が設定されていること | 共通設定 |
| `test_lambda_xray_enabled` | 各 Lambda 関数で X-Ray トレーシングが `Active` になっていること | 共通設定 |
| `test_ecs_desired_count` | ECS サービス `order-admin-api` の desired count が 2 以上であること | タスク定義 |
| `test_ecs_task_architecture` | ECS タスク定義のアーキテクチャが `ARM64` であること | タスク定義 |
| `test_ecs_container_not_root` | ECS タスク定義のコンテナが root ユーザーで実行されていないこと | 禁止事項 |

---

## テストを書く際のガイドライン

### 1. テスト名と設計書を対応させる

```python
def test_db_subnet_no_internet_route():
    """
    [設計書参照] docs/01_network/vpc-design.md > サブネット一覧
    DB サブネットのインターネット経路: なし（完全プライベート）
    """
```

テストが失敗したとき、どの設計書のどの記述に違反しているかをすぐに追えるようにする。

### 2. テストは読み取り専用にする

`describe_*` / `get_*` / `list_*` のみ使うこと。  
リソースを変更・削除する操作は絶対に書かない。

### 3. リソース名はハードコードせず定数化する

```python
# conftest.py に定義する
REGION = "ap-northeast-1"
VPC_NAME = "order-vpc"
LAMBDA_FUNCTIONS = ["order-create", "order-process", "receipt-generate"]
```

環境（dev/prod）によってリソース名が変わる場合は環境変数で切り替える。

### 4. 1 テスト = 1 つの設計上の制約

複数の検証を 1 テストに詰め込まない。  
失敗時にどの制約が破られているか明確にするため。

---

## よくある詰まりポイント（ヒント）

| 詰まりポイント | ヒント |
|---|---|
| SG のアウトバウンドが「空」のはずなのに通らない | AWS はデフォルトで全許可のアウトバウンドを作る。Terraform で明示的に削除しているか確認 |
| SG 参照かどうかの判定 | `IpRanges` が空で `UserIdGroupPairs` に値があれば SG 参照 |
| IAM ポリシーの `Resource` が `*` か判定 | インラインポリシーと管理ポリシーの両方をチェックすること |
| Lambda の Reserved Concurrency が「未設定」と「0」は別物 | 未設定 = 制限なし / 0 = 実行不可。`get_function_concurrency` が空を返す場合は未設定 |
| ECS の「root で実行されていない」の確認方法 | タスク定義の `containerDefinitions[].user` を確認。未設定の場合は root 扱い |
