---
title: "CodePipelineとTerraformで、API Gatewayをビルド【CodeFamily Practices 5/7】"
emoji: "📡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codepipeline", "cloudformation", "cicd", "devops"]
published: false
---
# 構成図

:::message
構成図は、Zennの記事「CodePipelineとCloudFormationで、API Gatewayをビルド【CodeFamily Practices 5/7】」とほぼ同じです。
:::

API GatewayとLambdaの挙動を確認するための、シンプルな構成。

* ソースステージをCodeCommit、ビルドステージをCodeBuildに設定したCodePipelineを構築。
* ビルドはServerless Frameworkを使用。
* API Gatewayへメールのタイトルと本文を指定してアクセスをすると、SNSトピックのサブスクリプションへメールを送信
* 送信に成功すると、サブジェクトとメッセージの値をレスポンス。

![](/images/codefamily_serverless/api-serverless.drawio.png =500x)

* __コマンド__
オプション` -v `は、リクエストとレスポンスの詳細を返すために入力。
メールを送るだけなら不要。

```
curl -v -X POST \
'https://Your-domain-name' \
-d $'{"sub": "テスト", "mes": "動作異常なし。"}'
```
__実行結果__
実際に行われているリクエストとレスポンスを出力。

```
Note: Unnecessary use of -X or --request, POST is already inferred.
*   Trying xxx.xxx.xxx.xxx:443...
* Connected to Your-domain-name (xxx.xxx.xxx.xxx) port 443 (#0)
* ALPN: offers h2
* ALPN: offers http/1.1
*  CAfile: /Users/username/opt/anaconda3/ssl/cacert.pem
*  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=*.DomainName
*  start date: Apr 26 00:00:00 2023 GMT
*  expire date: May 25 23:59:59 2024 GMT
*  subjectAltName: host "Your-domain-name" matched cert's "*.DomainName"
*  issuer: C=US; O=Amazon; CN=Amazon RSA 2048 M01
*  SSL certificate verify ok.
* Using HTTP2, server supports multiplexing
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* h2h3 [:method: POST]
* h2h3 [:path: /]
* h2h3 [:scheme: https]
* h2h3 [:authority: Your-domain-name]
* h2h3 [user-agent: curl/7.84.0]
* h2h3 [accept: */*]
* h2h3 [content-length: 52]
* h2h3 [content-type: application/x-www-form-urlencoded]
* Using Stream ID: 1 (easy handle 0x7faf7c013200)
> POST / HTTP/2
> Host: Your-domain-name
> user-agent: curl/7.84.0
> accept: */*
> content-length: 52
> content-type: application/x-www-form-urlencoded
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
* We are completely uploaded and fine
< HTTP/2 200 
< date: Tue, 02 May 2023 06:08:45 GMT
< content-type: application/json
< content-length: 49
< x-amzn-requestid: ea5fdaa9-9d87-4746-a9f1-7d129dc5d09b
< x-amz-apigw-id: ER9U_HIxtjMFzwA=
< x-amzn-trace-id: Root=1-6450a8ec-4159b6d05aea26b77a5239ff;Sampled=0;lineage=ca71e3f6:0
< 
* Connection #0 to host Your-domain-name left intact
Subject: テスト Message: 動作異常なし。%                               
```
__メール__

![](/images/codefamily_serverless/apicf16.png =500x)

# 構築
* __buildspec.yml__

```
version: 0.2

env:
  secrets-manager:
    AWS_ACCESS_KEY_ID: access
    AWS_SECRET_ACCESS_KEY: secret

phases:
  install:
    commands:
      - yum install -y yum-utils shadow-utils
      - yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
      - yum -y install terraform
  build:
    commands:
      - echo $AWS_ACCESS_KEY_ID
      - echo $AWS_SECRET_ACCESS_KEY
      - terraform init
      - terraform apply -auto-approve 
```

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
variable "subscription_address" {
  type    = string
  default = "colorfulrhythms927@icloud.com"
}

#Topicのリソースネーム
variable "topic_name" {
  type    = string
  default = "Api-Notification"
}

#Lambdaのリソースネーム
variable "lambda_name" {
  type    = string
  default = "Api-Lambda-Terraform"
}

#ACMの識別子
variable "certificate_identifer" {
  type    = string
  default = "be64a90c-f218-4d3a-aca0-56e9c8246020"
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
  source_file = "sns.py"
  output_path = "sns.zip"
}

#Lambdaの実行ロールにアタッチするポリシー
data "aws_iam_policy_document" "lambda-logging" {
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

data "aws_iam_policy_document" "lambda-sns" {
  statement {
    effect = "Allow"
    actions = [
      "sns:Publish"
    ]
    resources = [ "arn:aws:sns:${local.region}:${local.account_id}:${var.topic_name}" ]
  }
}

data "aws_iam_policy_document" "lambda-assume-role" {
  statement {
    actions = [ "sts:AssumeRole" ]
    principals {
      type = "Service"
      identifiers = [ "lambda.amazonaws.com" ]
    }
  }
}
```


* __main.tf__

```
#Lambdaモニタリングのポリシー作成
resource "aws_iam_policy" "lambda-logging" {
  name = "lambda-logging"
  path = "/"
  policy = data.aws_iam_policy_document.lambda-logging.json
}

#LambdaのSNSポリシー作成
resource "aws_iam_policy" "lambda-sns" {
  name = "lambda-sns"
  path = "/"
  policy = data.aws_iam_policy_document.lambda-sns.json
}

#Lambdaの実行ロール作成

resource "aws_iam_role" "lambda-role" {
  name = "api-sns"
  assume_role_policy = data.aws_iam_policy_document.lambda-assume-role.json
  managed_policy_arns = [aws_iam_policy.lambda-logging.arn, aws_iam_policy.lambda-sns.arn]
}

#SNSの作成
resource "aws_sns_topic" "cost_sns" {
  name = var.topic_name
  display_name = "Notification"
}

resource "aws_sns_topic_subscription" "subscription" {
  topic_arn = aws_sns_topic.cost_sns.arn
  protocol = "email"
  endpoint = var.subscription_address
}

#API Gatewayの作成
resource "aws_api_gateway_rest_api" "tfsns-api" {
  name = "api-sns"
  endpoint_configuration {
    types = [ "REGIONAL" ]
  }
}

resource "aws_api_gateway_method" "tfsns-method" {
  rest_api_id = aws_api_gateway_rest_api.tfsns-api.id
  resource_id = aws_api_gateway_rest_api.tfsns-api.root_resource_id
  http_method = "ANY"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "tfsns-integration" {
  rest_api_id = aws_api_gateway_rest_api.tfsns-api.id
  resource_id = aws_api_gateway_rest_api.tfsns-api.root_resource_id
  http_method = aws_api_gateway_method.tfsns-method.http_method
  integration_http_method = "ANY"
  type = "AWS_PROXY"
  uri = aws_lambda_function.tfsns-lambda.invoke_arn
}

resource "aws_lambda_permission" "apigw_lambda" {
  statement_id = "AllowExecutionFromAPIGateway"
  action = "lambda:InvokeFunction"
  function_name = aws_lambda_function.tfsns-lambda.function_name
  principal = "apigateway.amazonaws.com"
  source_arn = "arn:aws:execute-api:${local.region}:${local.account_id}:${aws_api_gateway_rest_api.tfsns-api.id}/*/*"
}

#Lambdaの作成
resource "aws_lambda_function" "tfsns-lambda" {
  function_name = var.lambda_name
  filename = "sns.zip"
  role = aws_iam_role.lambda-role.arn
  handler = "sns.handler"
  runtime = "python3.9"
  environment {
    variables = {
      topic: "arn:aws:sns:${local.region}:${local.account_id}:${var.topic_name}"
    }
  }
}

#API Gatewayのデプロイ
resource "aws_api_gateway_deployment" "tfsns-deploy" {
  depends_on = [
    aws_api_gateway_method.tfsns-method, aws_api_gateway_integration.tfsns-integration
  ]
  rest_api_id = aws_api_gateway_rest_api.tfsns-api.id
  stage_name = "prod"
}

#カスタムドメインの作成
resource "aws_api_gateway_domain_name" "tfsns-domain" {
  domain_name = "sns.shintaro-abe-personal.click"
  regional_certificate_arn = "arn:aws:acm:${local.region}:${local.account_id}:certificate/${var.certificate_identifer}"
  endpoint_configuration {
    types = [ "REGIONAL" ]
  }
}

resource "aws_route53_record" "tfsns-record" {
  name = aws_api_gateway_domain_name.tfsns-domain.domain_name
  type = "A"
  zone_id = "Z08412181CCQ5G40PSDXA"

  alias {
    evaluate_target_health = true
    name = aws_api_gateway_domain_name.tfsns-domain.regional_domain_name
    zone_id = aws_api_gateway_domain_name.tfsns-domain.regional_zone_id
  }
}

resource "aws_api_gateway_base_path_mapping" "domain-mapping" {
  api_id = aws_api_gateway_rest_api.tfsns-api.id
  stage_name = aws_api_gateway_deployment.tfsns-deploy.stage_name
  domain_name = aws_api_gateway_domain_name.tfsns-domain.domain_name
}
```

* __main.tf(リソースパスあり)__

```
#Lambdaモニタリングのポリシー作成
resource "aws_iam_policy" "lambda-logging" {
  name = "lambda-logging"
  path = "/"
  policy = data.aws_iam_policy_document.lambda-logging.json
}

#LambdaのSNSポリシー作成
resource "aws_iam_policy" "lambda-sns" {
  name = "lambda-sns"
  path = "/"
  policy = data.aws_iam_policy_document.lambda-sns.json
}

#Lambdaの実行ロール作成
resource "aws_iam_role" "lambda-role" {
  name = "api-sns"
  assume_role_policy = data.aws_iam_policy_document.lambda-assume-role.json
  managed_policy_arns = [aws_iam_policy.lambda-logging.arn, aws_iam_policy.lambda-sns.arn]
}

#SNSの作成。
resource "aws_sns_topic" "cost_sns" {
  name = var.topic_name
  display_name = "Notification"
}

resource "aws_sns_topic_subscription" "subscription" {
  topic_arn = aws_sns_topic.cost_sns.arn
  protocol = "email"
  endpoint = var.subscription_address
}

#ApiGatewayの作成
resource "aws_api_gateway_rest_api" "tfsns-api" {
  name = "api-sns"
  endpoint_configuration {
    types = [ "REGIONAL" ]
  }
}

resource "aws_api_gateway_resource" "tfsns-resouce" {
  path_part = "sns"
  parent_id = aws_api_gateway_rest_api.tfsns-api.root_resource_id
  rest_api_id = aws_api_gateway_rest_api.tfsns-api.id  
}

resource "aws_api_gateway_method" "tfsns-method" {
  rest_api_id = aws_api_gateway_rest_api.tfsns-api.id
  resource_id = aws_api_gateway_resource.tfsns-resouce.id
  http_method = "ANY"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "tfsns-integration" {
  rest_api_id = aws_api_gateway_rest_api.tfsns-api.id
  resource_id = aws_api_gateway_resource.tfsns-resouce.id
  http_method = aws_api_gateway_method.tfsns-method.http_method
  integration_http_method = "ANY"
  type = "AWS_PROXY"
  uri = aws_lambda_function.tfsns-lambda.invoke_arn
}

resource "aws_lambda_permission" "apigw_lambda" {
  statement_id = "AllowExecutionFromAPIGateway"
  action = "lambda:InvokeFunction"
  function_name = aws_lambda_function.tfsns-lambda.function_name
  principal = "apigateway.amazonaws.com"
  source_arn = "arn:aws:execute-api:${local.region}:${local.account_id}:${aws_api_gateway_rest_api.tfsns-api.id}/*/${aws_api_gateway_method.tfsns-method.http_method}${aws_api_gateway_resource.tfsns-resouce.path}"
}

#Lambdaの作成
resource "aws_lambda_function" "tfsns-lambda" {
  function_name = var.lambda_name
  filename = "sns.zip"
  role = aws_iam_role.lambda-role.arn
  handler = "sns.handler"
  runtime = "python3.9"
  environment {
    variables = {
      topic: "arn:aws:sns:${local.region}:${local.account_id}:${var.topic_name}"
    }
  }
}

#ApiGatewayのデプロイ
resource "aws_api_gateway_deployment" "tfsns-deploy" {
  depends_on = [
    aws_api_gateway_method.tfsns-method, aws_api_gateway_integration.tfsns-integration
  ]
  rest_api_id = aws_api_gateway_rest_api.tfsns-api.id
  stage_name = "prod"
}

#カスタムドメインの設定
resource "aws_api_gateway_domain_name" "tfsns-domain" {
  domain_name = "sns.shintaro-abe-personal.click"
  regional_certificate_arn = "arn:aws:acm:${local.region}:${local.account_id}:certificate/${var.certificate_identifer}"
  endpoint_configuration {
    types = [ "REGIONAL" ]
  }
}

resource "aws_route53_record" "tfsns-record" {
  name = aws_api_gateway_domain_name.tfsns-domain.domain_name
  type = "A"
  zone_id = "Z08412181CCQ5G40PSDXA"

  alias {
    evaluate_target_health = true
    name = aws_api_gateway_domain_name.tfsns-domain.regional_domain_name
    zone_id = aws_api_gateway_domain_name.tfsns-domain.regional_zone_id
  }
}

resource "aws_api_gateway_base_path_mapping" "domain-mapping" {
  api_id = aws_api_gateway_rest_api.tfsns-api.id
  stage_name = aws_api_gateway_deployment.tfsns-deploy.stage_name
  domain_name = aws_api_gateway_domain_name.tfsns-domain.domain_name
}
```
