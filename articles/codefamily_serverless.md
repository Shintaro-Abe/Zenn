---
title: "CodePipelineとServerless Frameworkでビルド【CodeFamily Practices 6/7】"
emoji: "🥽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codepipeline", "serverlessframewor", "cicd", "devops"]
published: true
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

## プラグインの使用
API Gatewayに付与するカスタムドメインの作成に、 __Domain Managerプラグイン__ を使用。
* __Domain Managerパッケージをインストール__

```
npm install serverless-domain-manager
```
* __インストールとドメイン作成__


ドメインの作成。

```
serverless create_domain
```
Serverless frameworkのデプロイ。

```
serverless deploy
```
* __リソースの削除__

Serverless frameworkの削除。

```
serverless remove
```
ドメインの削除。

```
serverless delete_domain
```

## コード

__対応するコードはGitHubに公開しています！__

https://github.com/Shintaro-Abe/codefamily-serverless.git


* __buildspec.yml__

https://github.com/Shintaro-Abe/codefamily-serverless/blob/50e7af970926837aceede9321aae6c7c5ff3e387/sources/buildspec.yml

* __serverless.yml__

https://github.com/Shintaro-Abe/codefamily-serverless/blob/50e7af970926837aceede9321aae6c7c5ff3e387/sources/serverless.yml

* __sns.py__

https://github.com/Shintaro-Abe/codefamily-serverless/blob/50e7af970926837aceede9321aae6c7c5ff3e387/sources/sns.py

## パイプラインの構築
__ソースステージとビルドステージの二つを持つパイプラインを作成。__

CodeCommitにリポジトリ(api-serverless)を作成し、ソースコードとbuildspec.ymlをプッシュ。

![](/images/codefamily_serverless/apiserver1.png =500x)

CodeBuildへ移動し、以下の内容でビルドプロジェクトを作成。

| 項目| 設定|
| --- | --- |
| Source provider| CodeCommit|
| repository| api-serverless |
| Git clone depth | 1 (デフォルト) |
| Image | aws/codebuild/amazonlinux2-x86_64-standard:4.0 |
| Environment type | Linux|
| Artifact | No artifact |
| Cache | No cache |
| CloudWatch Logs | ENABLED |

![](/images/codefamily_serverless/apiserver2.png =500x)

CodePipelineへ移動し、パイプラインの設定を開始。

![](/images/codefamily_serverless/apiserver3.png =500x)

前述で作成したリポジトリを指定。

![](/images/codefamily_serverless/apiserver4.png =500x)

前述で作成したビルドプロジェクトを指定。

![](/images/codefamily_serverless/apiserver5.png =500x)

デプロイを行わないためスキップ。

![](/images/codefamily_serverless/apiserver6.png =500x)

設定内容を確認の上、"Create pipeline"を選択。
選択後、自動的にパイプラインを開始。

![](/images/codefamily_serverless/apiserver7.png =500x)

##  自動構築
SourceとBuild、両方のステージが成功。

![](/images/codefamily_serverless/apiserver8.png =500x)

CodeBuildのログを確認。
Amazon Linux2のDockerイメージを使用したコンテナで、CodeCommitからダウンロードしたBuildspec.ymlを参照。
Serverless FrameworkとDomain Managerをインストールし、リソースをデプロイ。

![](/images/codefamily_serverless/apiserver9.png =500x)

Domain Managerは、ホストゾーンにデフォルトでAレコードとAAAAレコードを生成。

![](/images/codefamily_serverless/apiserver10.png =500x)

REST APIプロキシ統合のAPI Gatewayを作成。

![](/images/codefamily_serverless/apiserver11.png =500x)

カスタムドメインとして登録。

![](/images/codefamily_serverless/apiserver12.png =500x)

Lambdaを作成。

![](/images/codefamily_serverless/apiserver13.png =500x)

Cloudwatch Logsのポリシーを反映。

![](/images/codefamily_serverless/apiserver14.png =500x)

CloudWatch Logsにロググループを作成。

![](/images/codefamily_serverless/apiserver17.png =500x)

SNSのポリシーを反映。

![](/images/codefamily_serverless/apiserver15.png =500x)

トリガーにAPI Gatewayを設定。

![](/images/codefamily_serverless/apiserver16.png =500x)

SNSにトピックを作成。エンドポイントは、サブスクリプションの確認メールで承認の必要あり。

![](/images/codefamily_serverless/apiserver18.png =500x)

# まとめ
サーバーレスに特化したフレームワークなので、CloudFormationやTerraformに比べて少ない記述で構築できるところが利点。
アーキテクチャに合わせてフレームワークを活用したい。

## 合わせて読みたい👀👉CodeFamily Practicesの記事

:::details CodeCommitとローカル環境の連携 【CodeFamily Practices 1/7】
https://zenn.dev/lifewithpiano/articles/codecommit_practice
:::

:::details CodeBuildでビルドプロジェクトを作ってみよう 【CodeFamily Practices 2/7】
https://zenn.dev/lifewithpiano/articles/codebuild_practice
:::

:::details CodeDeployでEC2にアプリケーションをデプロイ 【CodeFamily Practices 3/7】
https://zenn.dev/lifewithpiano/articles/codedeploy_practice
:::

:::details CodePipelineでシンプルなパイプラインを構築してみた 【CodeFamily Practices 4/7】
https://zenn.dev/lifewithpiano/articles/codepipeline_practice
:::

:::details CodePipelineとCloudformationで、API Gatewayをビルド【CodeFamily Practices 5/7】
https://zenn.dev/lifewithpiano/articles/codefamily_serverless
:::

## 👀👉Serverless Framework関連の記事

:::details Serverless Frameworkの基本的な使い方
https://zenn.dev/lifewithpiano/articles/serverlessframework2304
:::