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
        email = 121922274+Shintaro-Abe@users.noreply.github.com
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

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "TerraformBackendAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:user/アクセスキーIDのユーザー名"
            },
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketVersioning",
                "s3:ListBucketMultipartUploads"
            ],
            "Resource": "arn:aws:s3:::abetest-terraform-deploymentbucket"
        },
        {
            "Sid": "TerraformStateAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:user/アクセスキーIDのユーザー名"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": "arn:aws:s3:::abetest-terraform-deploymentbucket/abetest-dev/terraform.tfstate"
        }
    ]
}
```

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

* __providers.tf__

```
provider "aws" {
  region = "ap-northeast-1"
}
```

* __backend.tf__

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
* __variables.tf__

```
#Eメールアドレス
variable "email_address" {
  type    = string
  default = "xxxxxxxxxxxx@icloud.com"
}

#Topicのリソースネーム
variable "topic_name" {
  type    = string
  default = "cost-notification"
}

#Lambdaのリソースネーム
variable "lambda_name" {
  type    = string
  default = "costnotification"
}

#Stepfunctionsのリソースネーム
variable "statemachine_name" {
  type    = string
  default = "cost-statemachine"
}
```
* __data.tf__

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

data "archive_file" "lambda" {
  type        = "zip"
  source_file = "cost.py"
  output_path = "cost.zip"
}

#Lambdaの実行ロールにアタッチするポリシー
data "aws_iam_policy_document" "lambda_logging" {
  statement {
    effect = "Allow"
    actions = [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ]
    resources = [ "arn:aws:logs:${local.region}:${local.account_id}:log-group:/aws/lambda/${var.lambda_name}:*" ]
  }
}

data "aws_iam_policy_document" "lambda_sns" {
  statement {
    effect = "Allow"
    actions = [
      "sns:Publish"
    ]
    resources = [ "arn:aws:sns:${local.region}:${local.account_id}:${var.topic_name}" ]
  }
}

data "aws_iam_policy_document" "lambda_cost" {
  statement {
    effect = "Allow"
    actions = [
      "ce:GetCostAndUsage"
    ]
    resources = [ "*" ]
  }
}

#Lambdaの信頼ポリシー
data "aws_iam_policy_document" "lambda_assume_role" {
  statement {
    actions = [ "sts:AssumeRole" ]
    principals {
      type = "Service"
      identifiers = [ "lambda.amazonaws.com" ]
    }
  }
}

#StepFunctionsを呼び出すポリシー
data "aws_iam_policy_document" "stepfunction_start" {
  statement {
    effect = "Allow"
    actions = [
      "states:StartExecution"
    ]
    resources = [ "arn:aws:states:${local.region}:${local.account_id}:stateMachine:${var.statemachine_name}" ]
  }
}

#EventBridgeの信頼ポリシー
data "aws_iam_policy_document" "event_assume_role" {
  statement {
    actions = [ "sts:AssumeRole" ]
    principals {
      type = "Service"
      identifiers = [ "scheduler.amazonaws.com" ]
    }
  }
}

#Stepfunctionsの実行ポリシー
data "aws_iam_policy_document" "stepfunction_exe" {
  statement {
    effect = "Allow"
    actions = [
      "lambda:InvokeFunction"
    ]
    resources = [ 
      "arn:aws:lambda:${local.region}:${local.account_id}:function:${var.lambda_name}",
      "arn:aws:lambda:${local.region}:${local.account_id}:function:${var.lambda_name}:*"
    ]
  }
}

data "aws_iam_policy_document" "stepfunction_logging" {
  statement {
    effect = "Allow"
    actions = [
      "logs:CreateLogDelivery",
      "logs:GetLogDelivery",
      "logs:UpdateLogDelivery",
      "logs:DeleteLogDelivery",
      "logs:ListLogDeliveries",
      "logs:PutResourcePolicy",
      "logs:DescribeResourcePolicies",
      "logs:DescribeLogGroups"
    ]
    resources = [ "*" ]         #StepFunctionのログを取得するポリシーはresourceをワイルドカード" * "にする
  }
}

#Stepfunctionsの信頼ポリシー
data "aws_iam_policy_document" "stepfunctions_assume_role" {
  statement {
    actions = [ "sts:AssumeRole" ]
    principals {
      type = "Service"
      identifiers = [ "states.amazonaws.com" ]
    }
  }
}
```
* __main.tf__

```
#SNSの作成。
resource "aws_sns_topic" "cost_sns" {
  name = var.topic_name
  display_name = "Cost Notification"
}

resource "aws_sns_topic_subscription" "subscription" {
  topic_arn = aws_sns_topic.cost_sns.arn
  protocol = "email"
  endpoint = var.email_address
}

#Lambda用CloudWatch Loggroupの作成
resource "aws_cloudwatch_log_group" "lambda_log" {
  name = "/aws/lambda/${var.lambda_name}"
}

#StepFunctions用CloudWatch Loggroupの作成
resource "aws_cloudwatch_log_group" "statemachine_log" {
  name = "/aws/vendedlogs/states/${var.statemachine_name}"
}

#Lambdaの実行ロール
resource "aws_iam_policy" "lambda_logging" {
  name = "lambda_logging"
  path = "/"
  policy = data.aws_iam_policy_document.lambda_logging.json
}

resource "aws_iam_policy" "lambda_sns" {
  name = "lambda_sns"
  path = "/"
  policy = data.aws_iam_policy_document.lambda_sns.json
}

resource "aws_iam_policy" "lambda_cost" {
  name = "lambda_cost"
  path = "/"
  policy = data.aws_iam_policy_document.lambda_cost.json
}

resource "aws_iam_role" "lambda_role" {
  name = "costnotification"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume_role.json
  managed_policy_arns = [
    aws_iam_policy.lambda_logging.arn,
    aws_iam_policy.lambda_sns.arn, 
    aws_iam_policy.lambda_cost.arn
  ]
}

#StepFunctionを呼び出すロール
resource "aws_iam_policy" "stepfunction_start" {
  name = "event_statemachine"
  path = "/"
  policy = data.aws_iam_policy_document.stepfunction_start.json
}

resource "aws_iam_role" "event_role" {
  name = "event_costnotification"
  assume_role_policy = data.aws_iam_policy_document.event_assume_role.json
  managed_policy_arns = [aws_iam_policy.stepfunction_start.arn]
}

#StepFunctionsの実行ロール
resource "aws_iam_policy" "stepfunction_exe" {
  name = "stepfunction_exe"
  path = "/"
  policy = data.aws_iam_policy_document.stepfunction_exe.json
}

resource "aws_iam_policy" "stepfunction_logging" {
  name = "stepfunction_logging"
  path = "/"
  policy = data.aws_iam_policy_document.stepfunction_logging.json
}

resource "aws_iam_role" "statemachine_role" {
  name = "statemachine_role"
  assume_role_policy = data.aws_iam_policy_document.stepfunctions_assume_role.json
  managed_policy_arns = [aws_iam_policy.stepfunction_exe.arn, aws_iam_policy.stepfunction_logging.arn]
}

#Lambdaの作成
resource "aws_lambda_function" "cost_lambda" {
  function_name = var.lambda_name
  filename = "cost.zip"
  role = aws_iam_role.lambda_role.arn
  handler = "cost.handler"
  runtime = "python3.9"
  environment {
    variables = {
      topic: aws_sns_topic.cost_sns.arn
    }
  }
}

#StepFunctionsの作成
resource "aws_sfn_state_machine" "cost_statemachine" {
  name = var.statemachine_name
  role_arn = aws_iam_role.statemachine_role.arn
  definition = <<EOF
{
  "StartAt": "Lambda Invoke",
  "States": {
    "Lambda Invoke": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:${local.region}:${local.account_id}:function:${var.lambda_name}"
      },
      "End": true
    }
  }
}
EOF

  logging_configuration {
    log_destination = "${aws_cloudwatch_log_group.statemachine_log.arn}:*"
    include_execution_data = true
    level = "ALL"
  }
}

#EventBridgeの作成
resource "aws_scheduler_schedule" "cost_eventbridge" {
  name = "cost_statemachine_event"
  schedule_expression = "cron(0 3 * * ? *)"
  flexible_time_window {
    mode = "OFF"
  }
  target {
    arn = aws_sfn_state_machine.cost_statemachine.arn
    role_arn = aws_iam_role.event_role.arn
  }
}
```
* __cost.py__
[Developers IO 藤井元貴さん作成のapp.py](https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml)を参考にさせていただきました。
```
import boto3
import os 
from datetime import datetime, timedelta, date

def handler(event, context):
    ce = boto3.client('ce')
    sns = boto3.client('sns')

    # 今月の合計請求額を取得
    total_billing = get_total_billing(ce)
    # 今月の合計請求額を取得（サービス毎）
    service_billings = get_service_billings(ce)

    # Amazon SNSトピックに発行するメッセージを生成
    (subject, message) = get_message(total_billing, service_billings)

    response = sns.publish(
        TopicArn = os.environ["topic"],
        Subject = subject,
        Message = message
    )
    return response

def get_total_billing(ce):
    (start_date, end_date) = get_total_cost_date_range()

    response = ce.get_cost_and_usage(
        TimePeriod={
            'Start': start_date,
            'End': end_date
        },
        Granularity='MONTHLY',
        Metrics=[
            'AmortizedCost'
        ]
    )

    return {
        'start': response['ResultsByTime'][0]['TimePeriod']['Start'],
        'end': response['ResultsByTime'][0]['TimePeriod']['End'],
        'billing': response['ResultsByTime'][0]['Total']['AmortizedCost']['Amount'],
    }

def get_service_billings(ce):
    (start_date, end_date) = get_total_cost_date_range()

    response = ce.get_cost_and_usage(
        TimePeriod={
            'Start': start_date,
            'End': end_date
        },
        Granularity='MONTHLY',
        Metrics=[
            'AmortizedCost'
        ],
        GroupBy=[
            {
                'Type': 'DIMENSION',
                'Key': 'SERVICE'
            }
        ]
    )

    billings = []

    for item in response['ResultsByTime'][0]['Groups']:
        billings.append({
            'service_name': item['Keys'][0],
            'billing': item['Metrics']['AmortizedCost']['Amount']
        })

    return billings


def get_total_cost_date_range():
    start_date = date.today().replace(day=1).isoformat()
    end_date = date.today().isoformat()

    # get_cost_and_usage()のstartとendに同じ日付は指定不可のため、今日が1日なら「先月1日から今月1日（今日）」までの範囲にする
    if start_date == end_date:
        end_of_month = datetime.strptime(start_date, '%Y-%m-%d') + timedelta(days=-1)
        begin_of_month = end_of_month.replace(day=1)
        return begin_of_month.date().isoformat(), end_date
    return start_date, end_date


def get_message(total_billing, service_billings):
    start = datetime.strptime(total_billing['start'], '%Y-%m-%d').strftime('%Y/%m/%d')

    # Endの日付は結果に含まないため、表示上は前日にしておく
    end_today = datetime.strptime(total_billing['end'], '%Y-%m-%d')
    end_yesterday = (end_today - timedelta(days=1)).strftime('%Y/%m/%d')

    total = round(float(total_billing['billing']), 2)
    subject = f'{start}～{end_yesterday}の請求額：${total:.2f}'

    message = []
    message.append('【内訳】')
    for item in service_billings:
        service_name = item['service_name']
        billing = round(float(item['billing']), 2)

        if billing == 0.0:
            # 請求無しの場合は内訳を表示しない
            continue
        message.append(f'・{service_name}: ${billing:.2f}')

    return subject, '\n'.join(message)
```
# まとめ
定義する内容ごとにtfファイルを作成できるため、工夫次第でとても運用しやすくなる。
構築できるだけで良しとせず、継続して使用する場合のクオリティを求めてブラッシュアップを重ねたい。