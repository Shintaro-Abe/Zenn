---
title: "CodePipelineとCloudformationで、API Gatewayをビルド【CodeFamily Practices 5/7】"
emoji: "🐟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codepipeline", "cloudformation", "cicd", "devops"]
published: true
---
# 構成図
API GatewayとLambdaの挙動を確認するための、シンプルな構成。

* ソースステージをCodeCommit、ビルドステージをCodeBuildに設定したCodePipelineを構築。
* ビルドはCloudeFormation(SAM)を使用。
* API Gatewayへメールのタイトルと本文を指定してアクセスをすると、SNSトピックのサブスクリプションへメールを送信
* 送信に成功すると、サブジェクトとメッセージの値をレスポンス。

![](/images/codefamily_cloudformation/api-cf.drawio.png =500x)

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

![](/images/codefamily_cloudformation/apicf16.png =500x)

# コード
__対応するコードはGitHubに公開しています！__
https://github.com/Shintaro-Abe/codefamily-cloudformation

## Lambda
SNSと連携し、サブスクリプションにメールを送るPythonソースコード。

* __sns.py__

```
import json
import os
import boto3

snscli = boto3.client('sns')

def lambda_handler(event, context):
    Body = json.loads(event['body'])
    Sub = Body["sub"]
    Mes = Body["mes"]
    
    response = snscli.publish(
        TopicArn = os.environ["topic"],
        Subject = Sub,
        Message = Mes)
    print(response)
    
    text = "Subject: " + Sub + " Message: " + Mes
    return {
        'statusCode': 200,
        'body': text
    }
```
## CloudFormation(SAM)

* __api-sns.yml__

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Sending emails from SNS with Lambda.
Parameters:
  LambdaFunctionName:
    Type: String
    Default: Api-Lambda-CloudFormation
Resources:
  #SNSの作成
  ApiSns:
    Type: AWS::SNS::Topic
    Properties: 
      TopicName: api-notification
      DisplayName: Notification
      Subscription: 
        - Endpoint: zennzennzenn-SE@gmail.com
          Protocol: email
  #Lambdaの実行ロール
  LambdaRole:
    Type: 'AWS::IAM::Role'
    Properties:
      RoleName: api-sns
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: lambda-logging
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                Resource: !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:*  
              - Effect: Allow
                Action:
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: !Join 
                - ''
                - - !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/
                  - !Ref LambdaFunctionName
                  - :*
        - PolicyName: lambda-sns
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: 
                  - sns:Publish
                Resource:
                  - !GetAtt ApiSns.TopicArn
  #APIGatewayの作成
  ApiGatewayApi:
    Type: AWS::Serverless::Api
    Properties:
      EndpointConfiguration: 
        Type: REGIONAL
      StageName: prod
      #ホストゾーンとACM証明書はすでに存在するリソースを使用
      Domain: 
        DomainName: Your-domain-name
        CertificateArn: !Sub arn:aws:acm:${AWS::Region}:${AWS::AccountId}:certificate/z26y25x24w23v22u21
        Route53:
          HostedZoneId: 1a2b3c4d5e6f7g
  #Lambdaの作成
  ApiFunction: 
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Ref LambdaFunctionName
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /
            Method: any
            RestApiId:
              Ref: ApiGatewayApi
      Runtime: python3.9
      Handler: sns.handler
      Environment: 
        Variables:
          topic: !GetAtt ApiSns.TopicArn
      CodeUri: sns.py
      Role: !GetAtt LambdaRole.Arn
```
* __buildspec.yml__

```
version: 0.2

phases:

  build:
    commands:
      - aws cloudformation package 
        --template api-sns.yml 
        --s3-bucket layers-bucket123456789        #パッケージを格納するS3バケットを指定
        --output-template-file packaged-api-sns.yml
      - aws cloudformation deploy 
        --template-file packaged-api-sns.yml 
        --stack-name Serverless 
        --capabilities CAPABILITY_NAMED_IAM       #作成したロールを使用する場合は_NAMED_が必要
```
## パイプラインの構築
__ソースステージとビルドステージの二つを持つパイプラインを作成。__

CodeCommitにリポジトリ(api-cloudformation)を作成し、ソースコードとbuildspec.ymlをプッシュ。

![](/images/codefamily_cloudformation/apicf1.png =500x)

CodeBuildへ移動し、以下の内容でビルドプロジェクトを作成。

| 項目| 設定|
| --- | --- |
| ソースプロバイダ| CodeCommit|
| リポジトリ| api-cloudformation |
| Gitクローンの深さ | 1 (デフォルト) |
| イメージ | aws/codebuild/amazonlinux2-x86_64-standard:4.0 |
| 環境タイプ | Linux|
| アーティファクト | なし |
| キャッシュ | なし |
| CloudWatch Logs | 有効 |

![](/images/codefamily_cloudformation/apicf2.png =500x)

CodePipelineへ移動し、パイプラインの設定を開始。

![](/images/codefamily_cloudformation/apicf3.png =500x)

前述で作成したリポジトリを指定。

![](/images/codefamily_cloudformation/apicf4.png =500x)

前述で作成したビルドプロジェクトを指定。

![](/images/codefamily_cloudformation/apicf5.png =500x)

デプロイを行わないためスキップ。

![](/images/codefamily_cloudformation/apicf6.png =500x)

設定内容を確認の上、「パイプラインを作成する」を選択。
選択後、自動的にパイプラインを開始。

![](/images/codefamily_cloudformation/apicf7.png =500x)

##  自動構築
ソース、ビルド両方のステージが成功。

![](/images/codefamily_cloudformation/apicf8.png =500x)


REST APIプロキシ統合のAPI Gatewayを作成。
ホストゾーンにAレコードを生成し、カスタムドメインとして登録。

![](/images/codefamily_cloudformation/apicf10.png =500x)

Lambdaを作成。

![](/images/codefamily_cloudformation/apicf12.png =500x)

Cloudwatch Logsのポリシーを反映。
この段階ではロググループは存在しないが、Lambdaを実行すると自動的に生成。

![](/images/codefamily_cloudformation/apicf13.png =500x)

SNSのポリシーを反映。

![](/images/codefamily_cloudformation/apicf14.png =500x)

トリガーにAPI Gatewayを設定。

![](/images/codefamily_cloudformation/apicf15.png =500x)

# まとめ
コードの更新をプッシュするとパイプラインが自動でスタートするため、ビルドまでの作業が効率的になった。
デプロイステージなどを繋げて、様々なアプリケーションを作成できるように学習を進めていきたい。