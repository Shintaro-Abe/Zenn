---
title: "Terraformの基本的な使い方"
emoji: "🌎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "terraform"]
published: true
---
# Terraform
## 概念
クラウドとオンプレミスのインフラをプロビジョニングするために、人間が読める設定ファイルでリソースを定義。
バージョンアップ、再利用、共有が可能なInfrastructure as Codeツール。
AWS、Azure、Google Cloud Platformなどのクラウドプロバイダーとオンプレミスの、リソースやサービスを一貫した方法で管理。
プロバイダーを使用してプラットフォームと連携し、APIを通じてリソースを作成、管理。
リソースの定義には独自のプログラム言語である、HCL（HashiCorp Configuration Language)を使用。
ファイルの拡張子は` tf `。


## インストール

Terraformはバージョンの更新が頻繁に行われるため、任意のバージョン指定可能な環境であるtfenvをインストール。

homebrewがインストールされている環境で、以下のコマンドを入力。

```
brew install tfenv
```
tfenvのバージョン確認。

```
tfenv --version

#結果
tfenv 3.0.0
```
インストール可能なバージョンのリストを出力。

```
tfenv list-remote

#結果
1.4.0-rc1
1.4.0-beta2
1.4.0-beta1
1.4.0-alpha20221207
1.4.0-alpha20221109
1.3.9
1.3.8
1.3.7
1.3.6
1.3.5
1.3.4
1.3.3
1.3.2
1.3.1

＜中略＞

0.3.1
0.3.0
0.2.2
0.2.1
0.2.0
0.1.1
0.1.0
```
最新版のバージョンをインストール。

```
tfenv install 1.3.9
```
最新版のバージョンを選択。

```
tfenv use 1.3.9
```

## Gitセキュリティ対策
Gitを利用して作業をする際にAWSの認証情報を誤ってコミットしないようにするため、git-secretsをインストール。

```
brew install git-secrets
```
すべてのリポジトリに適用。

```
git secrets --register-aws --global
```
設定ファイルの中身を確認。
コミット防止のため、認証情報の正規表現が記載されている。

```
cat ~/.gitconfig

#結果
[alias]
        st = status --short
[user]
        name = Shintaro Abe
        email = 1234567890+Shintaro-Abe@users.noreply.github.com
[init]
        defaultBranch = main
[secrets]
        providers = git secrets --aws-provider
        patterns = (A3T[A-Z0-9]|AKIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16}
        patterns = (\"|')?(AWS|aws|Aws)?_?(SECRET|secret|Secret)?_?(ACCESS|access|Access)?_?(KEY|key|Key)(\"|')?\\s*(:|=>|=)\\s*(\"|')?[A-Za-z0-9/\\+=]{40}(\"|')?
        patterns = (\"|')?(AWS|aws|Aws)?_?(ACCOUNT|account|Account)_?(ID|id|Id)?(\"|')?\\s*(:|=>|=)\\s*(\"|')?[0-9]{4}\\-?[0-9]{4}\\-?[0-9]{4}(\"|')?
        allowed = AKIAIOSFODNN7EXAMPLE
        allowed = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```
新規リポジトリ作成時にgit-secretsがインストールされるように設定。

```
git secrets --install ~/.git-templates/secrets
```
```
git config --global init.templatedir '~/.git-templates/secrets'
```
## セキュリティ確認
リポジトリを作成し、アクセスキーが入ったデータのコミットを防止できるか確認。

```
mkdir aws-secrets-test
```
テスト用ディレクトリへ移動。

```
cd aws-secrets-test
```
Gitリポジトリとして初期化。

```
git init
```
アクセスキーIDとシークレットキーに類似したデータを作成。

```
echo "aws_access_key_id = \"AKIAIOSFODNN8EXAMPLE\"\naws_secret_access_key = \"wJalrXUtnFEMI/K8MDENG/bPxRfiCYEXAMPLEKEY\"" > main.tf
```
ステージに追加。

```
git add .
```
コミットに対してエラーを返し、防止。

```
git commit -m 'secrets test'

#結果
main.tf:1:aws_access_key_id = "AKIAIOSFODNN8EXAMPLE"
main.tf:2:aws_secret_access_key = "wJalrXUtnFEMI/K8MDENG/bPxRfiCYEXAMPLEKEY"

[ERROR] Matched one or more prohibited patterns

Possible mitigations:
- Mark false positives as allowed using: git config --add secrets.allowed ...
- Mark false positives as allowed by adding regular expressions to .gitallowed at repository's root directory
- List your configured patterns: git config --get-all secrets.patterns
- List your configured allowed patterns: git config --get-all secrets.allowed
- List your configured allowed patterns in .gitallowed at repository's root directory
- Use --no-verify if this is a one-time false positive
```
## tfファイル
インフラの管理に` tf `を識別子にしたファイルを使用。

* __providers.tf__
クラウドプロバイダーやAPIの操作を可能にするための設定。

```
provider "aws" {
  region = "ap-northeast-1"
}
```

* __backend.tf__
構築したインフラの状態を保存するデータの格納先を指定。
S3を使用する場合は、バケットポリシーの追加とバージョニングの設定が必要。
バケットポリシーのプリンシパルは、使用するアクセスキーIDのユーザーarn。

```
terraform {
  backend "s3" {
    bucket = "abetest-terraform-deploymentbucket"
    key    = "abetest-dev/terraform.tfstate"
		encrypt = true
    region = "ap-northeast-1"
  }
}
```
対象のバケットのパブリックアクセスのブロックをオンにしたまま、以下のデータをバケットポリシーに設定。

https://github.com/Shintaro-Abe/terraform-practice/blob/7089a58c5602a07ba54e7f9428c0edef5f4514a8/sources/S3BucketPolicy.json

* __main.tf__
構築するリソースを定義。

```
resource "aws_sns_topic" "cost_notification" {
  name = "cost-notification"
  display_name = "Cost Notification"
}

resource "aws_sns_topic_subscription" "subscription" {
  topic_arn = aws_sns_topic.cost_notification.arn
  protocol = "email"
  endpoint = var.availability_address
}
```

* __data.tf__
main.tfが参照する値を定義。

```
data "aws_region" "current" {}
locals{
  region = data.aws_region.current.name
}
output "region" {
  value = local.region
}

data "aws_caller_identity" "current" {}
locals{
  account_id = data.aws_caller_identity.current.account_id
}
output "account_id" {
  value = local.account_id
}
```

* __variables.tf__
変数を定義。

```
variable "availability_address" {
  type    = string
  default = "xxxxxxxxxxxxx@gmail.com"
}
```

すべてのtfファイルはmain.tf内で記述可能だが、テンプレートの再利用や視認性を考慮し分けて管理。

# 構築
## フロー
__init(初期化) → plan(計画) → apply(適用) → destroy(破棄)__

__一連の流れをTerraform用に初期化された作業ディレクトリで実行。__

---

### 初期化
リポジトリにmain.tfをアタッチしてから、Terraform用に初期化。

```
terraform init
```
新しいバックエンドで初期化するには以下のコマンドを使用。

```
terraform init -reconfigure
```
以下のコマンドでも同様の結果。

```
terraform init -migrate-state
```
Terraformの設定ファイルを生成。
.terraformにモジュールやプラグインのキャッシュを管理とワークスペースの状態を記録。
terraform.tfstateにバックエンドの情報を記載。

![](/images/terraform/terraform1.png =500x)

---

### 計画
以下のコマンドで、構築するリソースを確認。
実際に構築する前に変更点を確認可能。

```
terraform plan
```
作成するリソースをプラスで表示。

![](/images/terraform/terraform2.png =500x)

---

### 適用

` plan `コマンドを実行せずに、リソースの作成可能。
計画を実行後、適用するかについて承認作業が必要。

```
terraform apply

#結果
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.
```
yesを入力。

```
Enter a value: yes

#結果
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```
承認作業を省略したい場合は、以下のコマンドで実行。

```
terraform apply -auto-approve
```
バックエンドに指定したS3バケットにterraform.tfstateの各バージョンを格納。

![](/images/terraform/terraform3.png =500x)

---

### 破棄

以下のコマンドで、作成したリソースを破棄。
承認作業が必要。

```
terraform destroy

#結果
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.
```
yesを入力。

```
Enter a value: yes

#結果
Destroy complete! Resources: 1 destroyed.
```
承認不要で削除する場合は、` -auto-approve `オプションをつける。

```
terraform destroy -auto-approve
```

破棄するリソースをマイナスで表示。

![](/images/terraform/terraform4.png =500x)
## StepFunctionsによるAWS請求額通知
毎日12時にEventbridgeでAWSの請求額を取得するLambda関数を起動させ、SNSから通知メールを送信するシステム。
学習のため、あえてStepFunctionsを使用して構築。

![](/images/terraform/EventBridge.drawio.png =500x)

__対応するコードはGitHubに公開しています！__

https://github.com/Shintaro-Abe/terraform-practice.git

* __providers.tf__

https://github.com/Shintaro-Abe/terraform-practice/blob/7089a58c5602a07ba54e7f9428c0edef5f4514a8/sources/providers.tf

* __backend.tf__

https://github.com/Shintaro-Abe/terraform-practice/blob/7089a58c5602a07ba54e7f9428c0edef5f4514a8/sources/backend.tf

* __variables.tf__

https://github.com/Shintaro-Abe/terraform-practice/blob/7089a58c5602a07ba54e7f9428c0edef5f4514a8/sources/variables.tf

* __data.tf__

https://github.com/Shintaro-Abe/terraform-practice/blob/7089a58c5602a07ba54e7f9428c0edef5f4514a8/sources/data.tf

* __main.tf__

https://github.com/Shintaro-Abe/terraform-practice/blob/7089a58c5602a07ba54e7f9428c0edef5f4514a8/sources/main.tf

# まとめ
定義する内容ごとにtfファイルを作成できるため、工夫次第でとても運用しやすくなる。
構築できるだけで良しとせず、継続して使用する場合のクオリティを求めてブラッシュアップを重ねたい。