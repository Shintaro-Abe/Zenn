---
title: "CodeBuildでビルドプロジェクトを作ってみよう 【CodeFamily Practices 2/7】"
emoji: "🏗️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codebuild", "cicd", "devops"]
published: true
---
# CodeBuildの概念
マネージド型のビルドサービス。GitとDockerを使用してビルドを実施。ピーク時のリクエストに合わせて自動でスケーリングするため、ビルドサーバーの管理は不要。データの保管、転送、アーティファクトをデフォルトで暗号化。

* __仕組み__
ビルドプロジェクトをもとに、ビルド環境をDockerイメージとして作成。ビルドに必要なプログラミング言語の指定や、ツールのインストールなどを定義したbuildspec.ymlをもとに、ビルド環境のコンテナを自動で設定。AWSサービスもしくはサードパーティーのリポジトリからソースコードをダウンロードし、コンパイルと単体テストを実行。デプロイのためのアーティファクト生成し、S3へ格納。
Session Managerを使用してビルド環境へアクセスし、デバッグの実施が可能。
* __ローカルマシンでのビルド__
CodeBuildエージェントをローカルマシンにインストールできるため、ローカルのGitとDockerを利用してビルド可能。

| 用語 | 意味 |
| --- | ---|
| コンパイル | ソースコードを機械が読み取るためのオブジェクトコード(バイナリコード)へ変換する処理。<br>コンパイラ型言語とインタープリタ型言語の二種類に分類。<br> __コンパイラ型言語__ ：変換したオブジェクトコードをもとに実行ファイルを作成。変換と実行を分離。Java,Goなど。<br> __インタープリタ型言語__ ：プログラムの実行時にソースコードをオブジェクトコードへ変換。変換と実行が分離しない。<br>一行ずつ変換と実行するため、エラーの時点で停止。Ruby,Python,PHPなど|
| ビルド | ソースコードをオブジェクトファイルに集約する一連の処理。 <br>ソースコードに問題がないか解析後、コンパイル。<br>複数のオブジェクトコードを1つのオブジェクトファイルへ集約する、リンカと呼ばれる処理を実施。|
| 単体テスト | Unit Test(UT)。プログラムモジュール単位で正常に動作をするか確認するためのテスト。|


## サポートするリポジトリ
EventBridgeもしくはサードパーティーのWebHookを利用して、ビルド開始のトリガーを設定。
EventBridgeでは、コードのコミットなどのルールかcron式でスケジュールを指定してトリガーを設定。

| プロバイダー<br>リポジトリ| 概要 |
| --- | ---|
| CodeCommit | AWSのプライベートGitリポジトリサービス。<br>ソースコードのコミットID、ブランチを指定。|
| S3 | AWSのオブジェクトストレージサービス。<br>ソースコードとbuildspec.ymlを含むZipファイルのオブジェクト名を指定。|
| GitHub | ソフトウェア開発のプラットフォーム。 Gitのバージョン管理システムを使用してコードを管理。<br>ソースコードのコミットID、ブランチを指定。|
| BitBucket | ATLASSIAN社のCI/CDツールで、Gitベースのリポジトリ。Bitbucket Pipesでパイプラインを構築。<br>DevSecOpsツールを使用してCI/CDを実施。<br>ソースコードのコミットID、ブランチを指定。|

## ビルド環境
ビルドを実行するためのOSとプログラミング言語のランタイムを組み合わせた、CodeBuild Dockerイメージリポジトリを作成(マネージド型イメージ)。Docker Hub、Amazon ECRリポジトリに保存されているDockerイメージを使用することも可能(カスタムイメージ)。
### [Dockerイメージ](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html)
CodeBuild Dockerイメージリポジトリは大まかにAmazon Linux2、Ubuntu、Windows Server Coreをサポート。

### [ランタイム](https://docs.aws.amazon.com/codebuild/latest/userguide/available-runtimes.html)
Dockerイメージによりサポートする言語が異なるが、Node.js, Java,Python,Go,phpなど主要な言語を使用可能。

## buildspec
* __仕様__
ビルドで実行する内容を定義。
バージョンの指定、ツールのインストールコマンド、テストやアーティファクト、キャッシュの設定などについて記述。
* __設定方法__
ビルドコマンドをコンソール、もしくはYAMLファイルに記述。
YAMLファイルの場合、デフォルトではソースリポジトリのルートディレクトリに` buildspec.yml `のファイル名で格納。 同じリポジトリに複数のbuildspecがある場合、ファイル名を変えたりルート以外のディレクトリを変更可能。
ビルドプロジェクトごとに使用できるbuildspecは一つのみ。
### セクション
* __version__
現在は0.2の使用を推奨。
* __run-as__
コマンドを実行するLinuxユーザーを指定。デフォルトはrootユーザー。
Linux系の環境のみ使用可。
* __env__
環境変数を定義。

| パラメータ | 役割 |
| --- | ---|
| variables | 環境変数をキーと値で定義。|
| parameter-store | SSMパラメータストアに保管しているパラメータを参照。キーと値(パラメータストアに保管している名前)で定義。|
| secrets-manager | SSMシークレットマネージャーに保管しているパラメータを参照。キーと値(シークレットの名前)で定義。|

```
env:
  variables: 
    REGION: "ap-northeast-1"
  parameter-store:
    Account_Id: "ACCOUNT_ID"
  secrets-manager:
    AWS_ACCESS_KEY_ID: "access"
    AWS_SECRET_ACCESS_KEY: "secret"
```
* __proxy__

明示的なプロキシサーバーを使用する場合に使用。
` HTTP_PROXY `環境変数と、` HTTPS_PROXY `環境変数の設定が必要。
透過的にプロキシサーバーを使用する場合、このセクションは不要。

| パラメータ | 役割 |
| --- | ---|
| upload-artifacts | アーティファクトのアップロードにプロキシサーバーを使用するかyes,noで選択。|
| logs | CloudWatch Logsへのログ送信時にプロキシサーバーを使用するかyes,noで選択。|

* __batch__
並列でビルドする場合に使用。

| パラメータ | 役割 |
| --- | ---|
| build-graph | タスクの依存関係を記述し、逐次処理を実行。|
| build-list | 同時に実行するタスクを記述。|
| build-matrix | 異なる環境のタスク記述し、同時に実行。 |

* __phases__
CodeBuildが実行するコマンドを記述。

| パラメータ | 役割 |
| --- | ---|
| install | ビルドに必要なパッケージのインストールや、テストのコードフレームワークをインストールするコマンドを記述。<br>runtime-versionsセクションでランタイムを指定可能。|
| pre_build | ログインや依存関係のインストールなど、ビルド前に実行するコマンドを記述。|
| build | ビルドをするためにCodeBuildが実行するコマンドを記述。|
| post_build | ビルドの後に、CodeBuildが実行するコマンドを記述。<br>アーティファクトのパッケージ化、Dockerイメージをプッシュなどを行える。<br>buildにエラーが出ても実行。 |
| commands | pre_build,build,post_buildの各フェーズに記述。上から順番に実行される。 |

* __reports__
テストレポートを予め作成したレポートグループへ出力し、 codeBuildが行ったテストを可視化。
テストグループでは、テストの成功・失敗、総数、実行時間を表示。
コードカバレッジのレポートも生成可能。

```
reports:
  arn:aws:codebuild:$REGION:$ACCOUNT_ID:report-group/api-serverless:     #グループ名かarnで指定。
    files:
      - "**/*"           #ファイルのパスを指定。
    file-format: CUCUMBERJSON
```

| 用語 | 意味 |
| --- | ---|
| コードカバレッジ | コード網羅率とも呼ぶ。 ソフトウェアテストで用いられ品質を評価に使用される。 <br>プログラム全体のうち、テストされたの割合を指す。テストの抜け漏れを把握でき、品質の確保に役立つ。<br> CodeBuildでは、ラインカバレッジとブランチカバレッジをサポート。 <br> **ラインカバレッジ(C0/命令網羅率)**  ：全体の内、テストされた割合。 <br> **ブランチカバレッジ(C1/分岐網羅率)**  ：全体でテストされた分岐の割合。ラインカバレッジより基準の強度が強い。|

* __artifucts__
アーティファクトの出力先や名前について記述。
複数のアーティファクトを出力可能。

```
artifacts:
  files:
    - '**/*'
  name: api-serverless-$CODEBUILD_START_TIME
```

* __cache__
キャッシュを置く場所を、S3もしくはローカル(CodeBuildのホストマシン)から選択可能。
プロジェクトの設定でキャッシュタイプを「キャッシュなし」「S3」「ローカル」から選択しておく。
ローカル：カスタムキャッシュを選択した場合にbuildspec.ymlで指定が必要。
キャッシュを用いることで、ビルドの実行時間を短縮。

| キャッシュの種類| 概要|
| --- | ---|
| ソースキャッシュ|Gitのメタデータをキャッシュ。<br>キャッシュをした後に再度ビルドを行った場合は、コミット間の変更のみPullされる仕組み。|
| Dockerレイヤーキャッシュ | Dockerのレイヤーをキャッシュ。Linux環境のみ使用可能。 |
| カスタムキャッシュ | 上記に該当しないキャッシュ。 |

* __その他__
ビルドの設定でセッション接続を有効化し、ビルドフェーズのコマンドセクションに、codebuild-breakpointを記述。
ビルドを一時停止させ、セッションマネージャーを使用してコンテナへログインが可能。
調査の終了後、` codebuild-resume `コマンドをシェルで実行し、ビルドを再開。
# 使用方法
## ビルドプロジェクトの作成

CodeBuildコンソールの「ビルドプロジェクトを作成する」を選択。

![](/images/codebuild_practice/cb1.png =500x)

| 項目| 概要|
| --- | ---|
| プロジェクトの設定| プロジェクト名の入力。 ビルド結果を表示するバッジや同時ビルド制限の有効化を設定。|
| ソース | 使用するプロバイダーとリポジトリ、参照するブランチやコミットIDを指定。バージョンの最新履歴からクローンの深さを指定することもできる。他のGitリポジトリを組み込むGitサブモジュールを有効化できる。|
| 環境 | CodeBuildによって管理されたマネージド型イメージもしくはDockerイメージを指定したLinuxのOS環境環境で使用。<br>OSは複数のバージョンから選択可能。<br>CodeBuildがビルドを行うためのサービスロールを指定。<br>サービスロールにはビルド実行に必要なポリシーをあらかじめアタッチしておく。|
| Buildspec |参照するbuildspec.ymlを指定。もしくはビルドコマンドを直接入力。 |
| バッチ設定 | バッチビルドを行う場合に設定。<br>コンピューティングタイプ、ビルド数の制限、タイムアウトなどビルドに関する設定。<br>アーティファクト、レポートなど出力に関する設定を行う。|
| アーティファクト | アーティファクトとキャッシュの出力先を設定。アーティファクトの出力先はS3のみ。キャッシュはS3かローカルを選択。|
| ログ| ビルドログの出力先をCloudwatch LogsとS3へアップロードするための設定。|

バッチを有効化にしバッチのURLコピー。

![](/images/codebuild_practice/cb6.png =500x)

リポジトリにマークダウンファイルを作成し、リンクを挿入。

```
![](バッジURL)
```
ビルドの結果をバッジで表示。

![](/images/codebuild_practice/cb7.png =500x)

以下のシンプルな設定内容を入力。
ビルドプロジェクトの作成後、「ビルドの開始」を選択するとビルドを開始。

| 項目 | 概要|
| --- | ---|
| プロジェクト名 | api-serverless |
|ソース|  **ソースプロバイダ**：CodeCommit<br>  **リポジトリ**：api-serverless <br>  **リファレンスタイプ**：ブランチ<br> **ブランチ** ：main |
| 環境|   **環境イメージ**  ：マネージド型イメージ<br>  **オペレーティングシステム** ：Amazon Linux 2<br>  **ランタイム** ：Standard<br> **イメージ** ：aws/codebuild/amazonlinux2-x86_64-standard:4.0<br> **イメージのバージョン** ：最新のイメージ<br> **環境タイプ** ：Linux| 
| アーティファクト |  **タイプ**：CodeCommit<br>  **バケット名**：api-serverless<br>  **セマンティックバージョニングの有効化**：有効<br> **パス** ：指定なし<br>  **暗号化キー**：デフォルト(AWS管理キー)<br>  **キャッシュタイプ**：S3<br>  **キャッシュバケット**：abetest-cache  |
| ログ |  CloudWatch Logs|

![](/images/codebuild_practice/cb2.png =500x)
![](/images/codebuild_practice/cb3.png =500x)

* __buildspec.ymlのアーティファクトセクション__

```
artifacts:
  files:
    - '**/*'
  name: api-serverless-$CODEBUILD_START_TIME
```

| 環境変数 | 機能 |
| --- | ---|
| CODEBUILD_START_TIME | ビルドの開始時間(Unix)。 |

指定したバケットにアーティファクト格納。

![](/images/codebuild_practice/cb4.png =500x)

* __buildspec.ymlのキャッシュセクション__

```
cache:
  paths:
    - '**/*'
```
指定したパケットにキャッシュを格納。

![](/images/codebuild_practice/cb5.png =500x)

# まとめ
今回は基本的な使い方のみの習得。
今後は単体テストやバッチビルドなどにトライし、CI/CDツールとして活用できるようにしたい。

# 合わせて読みたい👀👉CodeFamily Practicesの記事

:::details CodeCommitとローカル環境の連携 【CodeFamily Practices 1/7】
https://zenn.dev/lifewithpiano/articles/codecommit_practice
:::

:::details CodeDeployでEC2にアプリケーションをデプロイ 【CodeFamily Practices 3/7】
https://zenn.dev/lifewithpiano/articles/codedeploy_practice
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