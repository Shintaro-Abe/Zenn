---
title: "CodeDeployでEC2にアプリケーションをデプロイ 【CodeFamily Practices 3/7】"
emoji: "🎯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codedeploy", "cicd", "devops"]
published: true
---
# 概念
EC2、オンプレミス、Lambda、ECSへのアプリケーションのデプロイを自動化するサービス。

* __仕組み__
デプロイターゲットへCodeDeployエージェントをインストール。エージェントはS3からアプリケーションデータを取得し、ターゲットへインストール。
ライフサイクルイベントをappspec.ymlに記述してデプロイを管理。
インスタンス上のアプリケーションを中断してデプロイを行うインプレースデプロイ。 古いバージョンとは別に新しいバージョンをデプロイすることで、アプリケーションの中断を最小限に抑えるBlue/Greenデプロイがある。

| デプロイタイプ | コンピューティング<br>プラットフォーム |
| --- | --- |
| インプレース | EC2<br>オンプレミス |
| Blue/Green | EC2<br>オンプレミス<br>Lambda<br>ECS |

## デプロイ設定
CodeDeployがデプロイ中に使用するルール、成功条件、失敗条件。

* __EC2、オンプレミス__
インプレースとBlue/Greenで条件が異なる。

| デプロイ設定 | インプレース | Blue/Green|
| --- | --- | --- |
| CodeDeployDefault.AllAtOnce | 一度にできる限り多くのインスタンスへデプロイ。<br>1つでもデプロイできると成功。| 一度にできる限り多くのインスタンスへデプロイ。<br>すべてのインスタンスへルーティングし、一つでも正常に再ルーティングできれば成功。|
| CodeDeployDefault.HalfAtATime | 一度に最大で半分のインスタンスへデプロイ。<br>少なくとも半分デプロイできると成功。 | 一度に最大で半分のインスタンスへデプロイ。<br>一度に最大半分のインスタンスへルーティングし、少なくとも半分のインスタンスへ再ルーティングできれば成功。|
| CodeDeployDefault.OneAtATime | 一度に一つのインスタンスへデプロイ。<br>すべてのインスタンスへデプロイできると成功。|一度に一つのインスタンスへデプロイ。<br>一度に一つのインスタンスへトラフィックをルーティングし、すべてのインスタンスへ再ルーティングできると成功。 |

* __ECS__
決められた時間ごとに10%ずつトラフィックを移行する線形(Liner)、10%だけトラフィックを移行してから時間差で残りのトラフィックを移行させるカナリア(Canary)、すべてのトラフィックを移行するオールインワンから選択。

| デプロイタイプ | 方法 |
| --- | --- |
| CodeDeployDefault.ECSLinear10PercentEvery1Minutes | 毎分10%ずつトラフィックを移行。 |
| CodeDeployDefault.ECSCanary10Percent5Minutes | 最初に10%のトラフィックを移行し、5分後に90%を移行。 |
| CodeDeployDefault.ECSAllAtOnce| すべてのトラフィックを移行。 |

* __Lambda__
ECSと同様。

| デプロイタイプ | 方法 |
| --- | --- |
| CodeDeployDefault.LambdaLinear10PercentEvery1Minute | 毎分10%ずつトラフィックを移行。 |
| CodeDeployDefault.LambdaCanary10Percent5Minutes | 最初に10%のトラフィックを移行し、5分後に90%を移行。 |
| CodeDeployDefault.LambdaAllAtOnce| すべてのトラフィックを移行。 |

## appspecファイル
YAMLもしくはJsonで記述。

* __version__
` 0.0 `のみ記述可。

* __hooks__
デプロイライフサイクルイベントで使用するスクリプトもしくはLambda関数を指定。

* __resource (Lambda,ECSのみ)__
デプロイするLambda、ECSに関する情報を指定。

* __os (EC2,オンプレミスのみ)__
デプロイするインスタンスのオペレーティングシステムを指定。
linuxもしくはwindowsを指定。

* __permissions (EC2,オンプレミスのみ)__
アクセス権限(chmodコマンドと同様)、アクセスコントロールなど、インスタンスへのコピー後にアクセス許可をディレクトリやファイルに適用。

* __files (EC2,オンプレミスのみ)__
インストールイベントで、インスタンスにコピーするファイル名を指定。

__appspec.yml__

```
version: 0.0
os: linux
files:
  - source: /index.html
    destination: /var/www/html/
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies
      timeout: 300
      runas: root
    - location: scripts/start_server
      timeout: 300
      runas: root
  ApplicationStop:
    - location: scripts/stop_server
      timeout: 300
      runas: root
```
## リビジョン
アプリケーションのバージョン。
コンピューティングプラットフォームにインストールされたCodeDeployエージェントが、S3からアプリケーションデータを取得してデプロイを実施。
デプロイに使用するソースデータを圧縮し、ディビジョンのバケットへアップロード
特定のCodeDeployサービスロール使用時にアクセスできるように、バケットポリシーを設定。
プリンシパルはデプロイに使用するIAMサービスロールarn。

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:role/CodeDeployRole"
            },
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": "arn:aws:s3:::bucketname/*"
        }
    ]
}
```
アプリケーションのバージョンと名前の関連付けを実施。
AWS CLIの[deploy](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/deploy/index.html)コマンドを使用。

| コマンド | 機能 |
| --- | --- |
|  create-application| アプリケーションの作成。|
| --applocation-name| アプリケーションの名前。IAMもしくはアカウント内において一意であること。<br>オプション` --application-name `は必須。|
| push| コンテンツとappspecファイルをZipファイルに圧縮してバンドルにし、S3へアップロード。<br>オプション` --application-name `と` --s3-location `は必須。|
| --s3-location | アップロードするバケットとオブジェクトキーを` s3:/// `の形式で入力。|
| --ignore-hidden-files | 隠しファイルを除いてアップロード 。 |

```
aws deploy create-application --application-name Myapplication_Name
```
Zipファイルに圧縮格納するバケットの名前とオブジェクトキーを指定。
ルートディレクトリにappspecファイルやソースコードのディレクトリなどを含めるように圧縮。

```
aws deploy push \
--application-name SampleApp_Linux \
--s3-location s3://bucketname/Myapplication_Name.zip \
--ignore-hidden-files
```

# 使用方法
EC2にApacheサーバーを起動。バンドルにあるhtmlファイルをドキュメントルートに配置。
※公式ドキュメントチュートリアル。

![](/images/codedeploy_practice/CodeDeploy.drawio.png =500x)

以下のappspec.ymlを使用。

__appspec.yml__

```
version: 0.0
os: linux
files:
  - source: /index.html
    destination: /var/www/html/
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies
      timeout: 300
      runas: root
    - location: scripts/start_server
      timeout: 300
      runas: root
  ApplicationStop:
    - location: scripts/stop_server
      timeout: 300
      runas: root
```
## サービスロールの作成
### EC2用サービスロール

「ロールを作成」を選択。

![](/images/codedeploy_practice/cdtut1.png =500x)

信頼されたエンティティタイプをAWSのサービス、
ユースケースをEC2で選択。

![](/images/codedeploy_practice/cdtut2.png =500x)

AmazonEC2RoleforAWSCodeDeployポリシーを選択。

![](/images/codedeploy_practice/cdtut3.png =500x)

AmazonSSMManagedInstanceCoreポリシーを選択。

![](/images/codedeploy_practice/cdtut4.png =500x)

任意のロール名を付与。

![](/images/codedeploy_practice/cdtut5.png =500x)

### CodeDeploy用サービスロール
信頼されたエンティティタイプをAWSのサービス、
ユースケースをCodeDeployで選択。

![](/images/codedeploy_practice/cdtut6.png =500x)

AWSCodeDeployRoleポリシーはデフォルトでアタッチ済み。

![](/images/codedeploy_practice/cdtut7.png =500x)

任意のロール名を付与。

![](/images/codedeploy_practice/cdtut8.png =500x)
## EC2の作成
以下の設定でパブリックサブネットにEC2を起動。
セキュリティグループは、ポート80番・ソース0.0.0.0/0で設定。


| 項目 | 設定 |
| --- | --- |
|  インスタンス名 | CodeDeploy-Demo |
| マシンイメージ | Amazon Linux 2023 |
| インスタンスタイプ | t2.micro |
| キーペア | キーペアなしで続行 |
| パブリックIPの自動割り当て | 有効化 |
| IAM instance profile | 前述で作成したEC2用サービスロール |

![](/images/codedeploy_practice/cdtut10.png =500x)
![](/images/codedeploy_practice/cdtut9.png =500x)

## リビジョンの作成
S3にリビジョン格納用のバケットを作成。
バージョニングを有効化にし、パブリックアクセスはブロックのまま。
アクセス許可に、以下のバケットポリシーを追加し、CodeDeployエージェントのアクセスを許可。

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:role/CodeDeployRole"
            },
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": "arn:aws:s3:::codedeploydemobucket202304011201/*"
        }
    ]
}
```
![](/images/codedeploy_practice/cdtut11.png =500x)

ローカル環境から以下のコマンドを実行し、バンドルされたアプリケーションZipファイルをアップロード。

```
aws deploy create-application --application-name SampleApp_Linux
```
```
aws deploy push \
--application-name SampleApp_Linux \
--s3-location s3://codedeploydemobucket202304011201/SampleApp_Linux.zip \
--ignore-hidden-files
```
![](/images/codedeploy_practice/cdtut12.png =500x)

## CodeDeployの設定
CodeDeployのコンソール画面から「アプリケーションの作成」」を選択。

![](/images/codedeploy_practice/cdtut13.png =500x)

任意のアプリケーション名を入力し、コンピューティングプラットフォームは「EC2/オンプレミス」を選択

![](/images/codedeploy_practice/cdtut14.png =500x)

アプリケーションを作成後、「デプロイグループの作成」を選択。

![](/images/codedeploy_practice/cdtut15.png =500x)

以下の設定でデプロイグループを作成。

| 項目 | 設定 |
| --- | --- |
|  デプロイグループ名 | MyDemoDeloymentGroup |
| サービスロール | 前述で作成したCodeDeploy用サービスロール。 |
| デプロイタイプ | インプレース |
| タググループ |キー：Name、値：CodeDeploy-Demo<br>※前述で作成たEC2のタグを入力。 |
| CodeDeployエージェントのインストール | 今すぐ更新し、更新をスケジュール。 |
| デプロイ設定 | CodeDeployDefault.OneAtTime |
| ロードバランシング | 無効 |

![](/images/codedeploy_practice/cdtut16.png =500x)

## デプロイの作成
「デプロイの作成」を選択。

![](/images/codedeploy_practice/cdtut17.png =500x)

以下の設定を入力。
デプロイを作成すると、自動的にデプロイを開始。

| 項目 | 設定 |
| --- | --- |
| デプロイグループ | MyDemoDeloymentGroup |
| リビジョンタイプ| S3 |
| リビジョンの場所 | S3://codedeploydemobudket202340111201/SampleApp_Linux.zip |
| リビジョンファイルの種類| .zip |
| デプロイグループのオーバーライド | CodeDeployDefault.OneAtTime |
| ロールバック | 無効 |

![](/images/codedeploy_practice/cdtut18.png =500x)

デプロイ成功画面。

![](/images/codedeploy_practice/cdtut19.png =500x)

EC2へアクセスし、アプリケーションのインストールを確認。

![](/images/codedeploy_practice/cdtut20.png =500x)

# まとめ
EC2へアプリケーションをインストールする、シンプルなデプロイを習得。
appspec.ymlについて掘り下げなかったので、LambdaやECSのデプロイ時にライフサイクルイベントなどをトライしてみたい。

# 合わせて読みたい👀👉CodeFamily Practicesの記事

:::details CodeCommitとローカル環境の連携 【CodeFamily Practices 1/7】
https://zenn.dev/lifewithpiano/articles/codecommit_practice
:::

:::details CodeBuildでビルドプロジェクトを作ってみよう 【CodeFamily Practices 2/7】
https://zenn.dev/lifewithpiano/articles/codebuild_practice
:::

:::details CodePipelineでシンプルなパイプラインを構築してみた 【CodeFamily Practices 4/7】
https://zenn.dev/lifewithpiano/articles/codepipeline_practice
:::

:::details CodePipelineとCloudformationで、API Gatewayをビルド【CodeFamily Practices 5/7】
https://zenn.dev/lifewithpiano/articles/codefamily_cloudformation
:::

:::details CodePipelineとServerless Frameworkでビルド【CodeFamily Practices 6/7】
https://zenn.dev/lifewithpiano/articles/codefamily_serverless
:::

:::details CodePipelineとTerraformで、API Gatewayをビルド【CodeFamily Practices 7/7】
https://zenn.dev/lifewithpiano/articles/codefamily_terraform
:::