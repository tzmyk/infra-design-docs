# コントリビューションガイド

このリポジトリへの追加・変更を行う際の方針をまとめる。

---

## ディレクトリ構成方針

```
aws-infra/
├── docs/                      設計書（あるべき姿の定義）
│   ├── 01_network/
│   ├── 02_security/
│   ├── 03_iam/
│   ├── 04_compute/
│   ├── 05_cost/
│   └── 06_operations/
│
├── terraform/                 IaC（実装）
│   ├── modules/               再利用可能なモジュール
│   │   ├── network/
│   │   ├── security_groups/
│   │   ├── iam/
│   │   └── lambda/
│   └── environments/          環境別エントリーポイント
│       ├── dev/
│       └── prod/
│
└── tests/                     検証コード
    ├── README.md              テスト方針・テストケース一覧
    ├── infra/                 デプロイ済みリソースの検証（boto3）
    │   ├── conftest.py
    │   ├── test_network.py
    │   ├── test_security.py
    │   ├── test_iam.py
    │   └── test_compute.py
    └── terraform/             Terraformコード自体の検証
```

### 3ディレクトリの関係

```
docs/（あるべき姿）  →  terraform/（どう作るか）  →  tests/（作った通りか）
```

- `docs/` を先に書いてから `terraform/` を実装する（Design-First）
- `tests/infra/` は `docs/` の内容を boto3 で検証するコード
- `tests/terraform/` は Terraform コード自体のユニットテスト

---

## 各ディレクトリの詳細方針

### docs/

- フォーマット: Markdown（表・Mermaid・ASCII アート）
- YAML ファイル単体では作らない。Markdown の中にコードブロックとして埋め込む
- 各ファイルの末尾に `## 設計上の禁止事項` セクションを設ける
  - ここに書いた内容が `tests/` のテストケースになる
- 番号は追加順に採番する（既存番号を変えない）

### terraform/

```
modules/<name>/
├── main.tf        リソース定義
├── variables.tf   入力変数
├── outputs.tf     出力値
└── README.md      このモジュールが docs/ のどの設計書に対応するか明記する
```

- モジュール名は `docs/` のディレクトリ名と対応させる
  - `docs/01_network/` → `terraform/modules/network/`
  - `docs/03_iam/` → `terraform/modules/iam/`
- `environments/` は `dev` と `prod` の2環境を基本とする
- シークレットは `.tfvars` に書かない（Secrets Manager / SSM 参照）

### tests/infra/

- ツール: pytest + boto3
- テストは読み取り専用（`describe_*` / `get_*` / `list_*` のみ）
- `conftest.py` に定数（リージョン・リソース名）をまとめる
- テスト関数の docstring に `[設計書参照] docs/xx_xxx/xxx.md > セクション名` を必ず書く
- 詳細は `tests/README.md` を参照

### tests/terraform/

- ツール: `terraform test`（Terraform 1.6 以降の組み込み機能）
- モジュール単位でテストファイルを作成する
  - `terraform/modules/network/` → `tests/terraform/network.tftest.hcl`

---

## 作業の進め方（研修生向け）

```
1. docs/ の該当設計書を読む
2. terraform/modules/<name>/ にコードを書く
3. terraform/environments/dev/ から呼び出して apply する
4. tests/infra/ のテストを書いて実行する（tests/README.md 参照）
5. テストが通ったら PR を出す
```

### PR チェックリスト

- [ ] `docs/` の設計書に対応する実装になっているか
- [ ] `tests/infra/` のテストが追加・更新されているか
- [ ] `terraform plan` でエラーが出ていないか
- [ ] リソース名が設計書の命名規則に従っているか
- [ ] シークレットがコードにハードコードされていないか
