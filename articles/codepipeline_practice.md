---
title: "CodePipelineでシンプルなパイプラインを構築してみた 【CodeFamily Practices 4/7】"
emoji: "⏯️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codepipeline", "cicd", "devops"]
published: true
---
#  概念
DevOpsのワークフローをモデル化。ソフトウェアのリリースを自動化する、継続的デリバリーのマネージドサービス。

![](/images/codepipeline_practice/codepipline.drawio.png =500x)

* __仕組み__
ステージごとに環境を作成し、アーティファクトに対してビルドやデプロイを実行。ステージのアクションは、サードパーティのツールの呼び出し、手動の承認なども可能。
パイプラインはステージとトランジションの組み合わせにより成り立ち、リリースプロセスを可視化。ステージとトランジションは、追加と削除を自在に設定することが可能。
一つずつステージを実行。処理期間はステージをロック。エラーなどで失敗すると実行を停止し、ロックを解除後に再試行が可能。
パイプラインの実行結果をSNSで通知可能。

* __アクション__
ソース、ビルド、テスト、デプロイ、承認、Invokeがあり、それぞれプロバイダーもしくは手動を設定。
ステージを作成する前に、CodeBuildのビルドプロジェクト、CodeDeployのデプロイグループなどのアクションを予め作成。
アクションは一つのステージで並列処理することが可能。

* __ステージ__
アクションを実行する環境の単位。トランジションで繋げてパイプラインを形成。

# 使用方法

S3とCodeDployを使用して2台のEC2へアプリケーションをデプロイするパイプラインを構築。
構築後、ステージの追加と削除を実践。
※公式ドキュメントチュートリアル

![](/images/codepipeline_practice/codepipeline-tutorial.drawio.png =500x)

## アプリケーションバンドルのアップロード

CodePipelineと連携するバケットを作成。
アプリケーションバンドルの内容変えて更新をした場合にバージョンとして認識されるように、バージョニングを有効化して作成。

![](/images/codepipeline_practice/tut1.png =500x)

バケットへアプリケーションバンドルをアップロード。
バンドルはappspec.ymlを含み、Zipファイル形式。

![](/images/codepipeline_practice/tut2.png =500x)

## IAMロールの作成
インスタンスプロファイル用のロールを作成。

IAMロール作成画面で、信頼されたエンティティタイプはAWSのサービス、ユースケースはEC2を選択。

![](/images/codepipeline_practice/tut3.png =500x)

`AmazonEC2RoleforAWSCodeDeploy`で検索し、選択。

![](/images/codepipeline_practice/tut4.png =500x)

`AmazonSSMManagedInstanceCore`で検索し、選択。

![](/images/codepipeline_practice/tut5.png =500x)

任意のロール名を付与。

![](/images/codepipeline_practice/tut6.png =500x)

## EC2の起動
ウィンドウズを使用して作成したアプリケーションのサーバーを起動するため、
マシンイメージは、Windowsを選択。

![](/images/codepipeline_practice/tut7.png =500x)

インスタンスプロファイルに、先ほど作成したロールを選択。
インスタンスは、２機で起動。

![](/images/codepipeline_practice/tut8.png =500x)

# CodeDeployの設定
## IAMロールの作成
信頼されたエンティティタイプをAWSのサービス、ユースケースをCodeDeployを選択。

![](/images/codepipeline_practice/tut9.png =500x)

`AWSCodeDeployRole`で検索し、選択。

![](/images/codepipeline_practice/tut10.png =500x)

任意のロール名を付与。

![](/images/codepipeline_practice/tut11.png =500x)

## アプリケーションとデプロイグループの作成。
CodeDeployへ移動し、アプリケーションの作成を選択。

![](/images/codepipeline_practice/tut12.png =500x)

アプリケーション名を入力し、コンピューティングプラットフォームはEC2/オンプレミスを選択。

![](/images/codepipeline_practice/tut13.png =500x)

デプロイグループの作成を選択。

![](/images/codepipeline_practice/tut14.png =500x)

以下を入力。

| サービスロール | デプロイタイプ| 環境設定| エージェントのインストール | デプロイ設定 | ロードバランサー|
| --- | --- |--- |--- |--- |--- |
| 前述のロール| インプレース| インスタンス名(Name:タグ名) | 今すぐ更新し、<br>更新をスケジュール|OneAtTime|なし|

![](/images/codepipeline_practice/tut15.png =500x)

作成内容の確認。

![](/images/codepipeline_practice/tut16.png =500x)

## CodePipelineの設定

CodePipelineへ移動し、パイプラインを作成するを選択。

![](/images/codepipeline_practice/tut17.png =500x)

新しいサービスロールを選択。ロール名が自動的に入力される。

![](/images/codepipeline_practice/tut18.png =500x)

ソースステージの追加。 アプリケーションパッケージの場所を指定する。

| ソースプロバイダー| バケット| S3オブジェクトキー| 提出オプションを変更する|
| --- | --- | --- | --- |
| Amazon S3 | アプリケーションzipファイルをアップロードしたバケット| zipファイル名|Amazon CloudWatch Events|

EventBridgeを消してオプションに選択することで、アプリケーションファイルのバージョンをアップロードをトリガーにし、自動的にパイプラインがデプロイを実行。

![](/images/codepipeline_practice/tut19.png =500x)

CodeBuildを使用しないためスキップ。

![](/images/codepipeline_practice/tut20.png =500x)

| デプロイプロバイダー| リージョン| アプリケーション名| デプロイグループ|
| ---| --- | --- | --- |
| CodeDeploy| 東京|前述で作成したアプリケーション| 前述で作成したデプロイグループ|

![](/images/codepipeline_practice/tut21.png =500x)

内容の確認。

![](/images/codepipeline_practice/tut22.png =500x)

パイプラインの完成と同時にデプロイを開始。

![](/images/codepipeline_practice/tut23.png =500x)

実行状況を可視化。

![](/images/codepipeline_practice/tut24.png =500x)

すべてのステージが成功すると完了。

![](/images/codepipeline_practice/tut25.png =500x)

実行履歴で、それぞれのステージの実行結果の詳細を確認可能。

![](/images/codepipeline_practice/tut26.png =500x)

CodeDeployのデプロイと画面から、各リソースのデプロイステータスを確認可能。

![](/images/codepipeline_practice/tut27.png =500x)

EC2へアクセスすると、サンプルアプリケーションが呼び出され無事デプロイが完了したことを表示。

![](/images/codepipeline_practice/tut28.png =500x) 

# 別のステージをパイプラインに追加
CodeDeployへ移り、デプロイグループの作成を選択。

![](/images/codepipeline_practice/tutop1.png =500x)

デプロイグループ名のみ変更し、残りは前述で作成した内容と同一。

![](/images/codepipeline_practice/tutop2.png =500x)

作成内容を確認。

![](/images/codepipeline_practice/tutop3.png =500x)

CodePipelineへ移動。

![](/images/codepipeline_practice/tutop4.png =500x)

編集ボタンをクリック。

![](/images/codepipeline_practice/tutop5.png =500x)

追加するステージのステージ名を入力。

![](/images/codepipeline_practice/tutop6.png =500x)

保存を選択。

![](/images/codepipeline_practice/tutop7.png =500x)

内容に変更がなければ完了を選択。

![](/images/codepipeline_practice/tutop8.png =500x)

新しいステージが追加されるので、保存を選択。

![](/images/codepipeline_practice/tutop9.png =500x)

変更をリリースするを選択。

![](/images/codepipeline_practice/tutop11.png =500x)

すでにEC2にアプリケーションがデプロイされているため、わざとエラーを表示。

![](/images/codepipeline_practice/tutop12.png =500x)

エラーの確認。

![](/images/codepipeline_practice/tutop13.png =500x)

# ステージ間の移行を無効
CodePipelineへ移動。

![](/images/codepipeline_practice/tutop14.png =500x)

パイプラインを表示。
移行を無効にするを選択。

![](/images/codepipeline_practice/tutop15.png =500x)

理由を入力し、無効にするを選択。

![](/images/codepipeline_practice/tutop16.png =500x)

無効にしても、ステージは表示されたままになる。

![](/images/codepipeline_practice/tutop17.png =500x)

# まとめ
シンプルなパイプラインの構築と、追加・削除を実践。
サードパーティのツールと組み合わせて使用したり、パイプライン自体をInfrastructure as Codeで構築してみるなど、課題はまだ多い。
構築系のハンズオンと併用して、習得していきたい。

# 合わせて読みたい👀👉CodeFamily Practicesの記事

:::details CodeCommitとローカル環境の連携 【CodeFamily Practices 1/7】
https://zenn.dev/lifewithpiano/articles/codecommit_practice
:::

:::details CodeBuildでビルドプロジェクトを作ってみよう 【CodeFamily Practices 2/7】
https://zenn.dev/lifewithpiano/articles/codebuild_practice
:::

:::details CodeDeployでEC2にアプリケーションをデプロイ 【CodeFamily Practices 3/7】
https://zenn.dev/lifewithpiano/articles/codedeploy_practice
:::

:::details CodePipelineとCloudformationで、API Gatewayをビルド【CodeFamily Practices 5/7】
https://zenn.dev/lifewithpiano/articles/codefamily_cloudformation
:::