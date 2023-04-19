---
title: "Serverless Frameworkの基本的な使い方"
emoji: "⛑️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "serverlessframewor"]
published: true
---
# Serverless Framework
## 概念
Serverless Frameworkは、LambdaをメインとするインフラリソースをデプロイするためのCLI。
functionsセクションを利用し、Lambda本体とLambdaがトリガーをするイベントを定義。
resourceセクションでは、CloudFormationの構文を利用してインフラを定義。
デプロイはCloudFormationを使用して行われるため、デプロイイベントをCloudFormationのコンソール画面で確認可能。
プラグインを使用して機能を追加、上書きが可能。
AWS以外にも、AzureやGoogleなどのクラウドプロバイダーをサポート。
デプロイは、serverless.ymlをCloudFormationのテンプレートへ変換、ソースコードや依存関係ファイルをZipファイルにして実行。
S3にスタックの状態を保存するためのファイルを格納。

## インストール
npmパッケージがインストールされている環境で、以下のコマンドを実行。

```
npm install -g serverless
```
` install `コマンドの` -g `オプションは` --global `と同じ。
パスが通った状態になり、どのディレクトリからも実行可能。

```
which serverless

#結果
/usr/local/bin/serverless
```

実行環境となるサービスを作成。
サービスはフレームワークの単位であり、複数のサービスをオーケストレーションすることも可能。

`serverless `コマンドは` sls `と省略可能。

| コマンド| 役割|
| :---| :--- |
| serverless create<br>(sls create) | サービスの作成。|
| --template (-t)| テンプレートの指定。ランタイムごとにテンプレートが用意されており、` aws-python3 `はPython3.9。<br>自動でserverless.ymlとhandlerファイルが生成されるが、後からランタイムの修正可能。|
| --name (-n)|サービス名。|
| --path (-p) | サービスを作成するパス。|

```
serverless create --template aws-python3 --name abetest-service --path abetest
```
サービスを作成すると、ディレクトリに必要なファイルを自動で生成。

| ファイル名 | 役割 |
| :---| :--- |
| handler | テンプレートで指定したランライムで作成。Lambdaのソースコードを定義。|
| .gitignore. | コミットの際に対象から外すファイルを指定。|

![](/images/serverlessframework/server1.png =500x)

## [serverless.yml](https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml)
### 役割
* サービスの宣言。
* サービスを展開するクラウドサービスの定義。
* 1つ以上の関数を定義。
* 各機能のトリガーとなるイベント(HTTPリクエストなど)の定義。
* 使用するプラグインの定義。
* 作成するAWSリソースのセットを定義。
* イベントセクションにリストされたイベントが、デプロイ時にそのイベントに必要なリソースを自動的に作成。
* 変数による柔軟な設定。

### 構成

| セクション| 役割|
| :---| :--- |
| service | サービスの宣言。| 
| frameworkVersion| バージョンの指定。|
| provider| アカウントのセットアップ。<br> **name** ：使用するクラウドサービス。` .aws `ディレクトリの認証情報を参照。<br> **runtime** ：使用するプログラミング言語を指定。<br> **stage** ： デプロイする環境。デフォルトは` dev `<br> **region** ： デプロイするリージョン。デフォルトはus-east-1。<br> **profile** ： 認証情報を参照するユーザーの切り替え。<br> **memorySize** ：Lambdaで使用するメモリの指定。デフォルトは1024MB。|
| package | ソースコードに含めるデータを` patterns `パラメータで定義。<br>含めるデータは、ファイル名をそのまま入力。含めないデータは、ファイル名の前に` ! `をつける。<br>ディレクトリ配下のデータ全ての場合は` /** `を使用。|
| function | 作成するLambdaとAPI Gatewayのパラメータ。|
| resouce | 作成するリソースのパラメータ。 CloudFormationと同じ構文を使用。|

__serverless.ymlの例__

```
service: users
frameworkVersion: '3'
provider:
  name: aws
  runtime: python3.9
  stage: prod               #デフォルトはdev。
  region: us-east-2     #デフォルトはus-east-1。
  profile: shintaro      #認証情報にあるデフォルト以外のユーザーでデプロイする場合に指定。
  memorySize: 512    #MB単位で指定。デフォルトは1024MB。Lambdaに合わせて、必要なサイズを指定可能。

package:                     #ディレクトリにあるファイルはデフォルトで含まれるため、ソースコードは指定不要。
  patterns:
    - '!node_modules/**'
    - '!buildspec.yml'
    - '!package.json'
    - '!package-lock.json'
```

# 構築

デフォルトのLambda関数をデプロイ。

__serverless.yml__

```
service: abetest

frameworkVersion: '3'

provider:
  name: aws
  runtime: python3.9
  region: ap-northeast-1

functions:
  hello:
    handler: handler.hello
```

__handler.py__

```
import json


def hello(event, context):
    body = {
        "message": "Go Serverless v1.0! Your function executed successfully!",
        "input": event
    }

    response = {
        "statusCode": 200,
        "body": json.dumps(body)
    }

    return response

    # Use this code if you don't use the http event with the LAMBDA-PROXY
    # integration
    """
    return {
        "message": "Go Serverless v1.0! Your function executed successfully!",
        "event": event
    }
    """
```

デプロイは以下のコマンドを使用。

```
serverless deploy
```
` .severless `ディレクトリを生成。

| ファイル| 役割|
| :---| :--- |
| cloudformation-template-create-stack.json | アーティファクトを格納するS3をCloudFormationで作成するテンプレート。| 
| cloudformation-template-update-stack.json| Cloudformationを使用してデプロイするスタックのテンプレート。|
| serverless-state.json| CloudFormationのテンプレートへ変換するためのデータ。|
| abetest.zip | ソースコードの圧縮ファイル。|

![](/images/serverlessframework/server2.png =300x)

Lambda関数を作成。
関数名は、` サービス-ステージ-Lambdaの論理ID `を付与。

![](/images/serverlessframework/server3.png =500x)

CloudFormationを使用しているため、コンソールでスタックを表示。

![](/images/serverlessframework/server4.png =500x)

スタックの状態を保存するための[ServerlessDeploymentBucket](https://www.serverless.com/plugins/serverless-deployment-bucket)を生成。
serverless removeコマンドでサービスを削除する場合、このバケットも削除の対象。
テンプレートのproviderセクションにdeploymentBucketパラメータを設け、nameでバケットを指定可能。
バケットを自動で生成する必要がないため、cloudformation-template-create-stack.jsonが` .serverless `ディレクトリから消える。
データはデフォルトで5回のデプロイまで保存され、それより古いものは自動で削除。maxPreviousDeploymentArtifactsで変更可能。

```
provider:
  deploymentBucket:
    name: abetest-deploymentbucket
    maxPreviousDeploymentArtifacts: 10
```

![](/images/serverlessframework/server5.png =500x)

サービスの削除は以下のコマンドを使用。

```
serverless remove
```

## StepFunctionsによるAWS請求額通知
毎日12時にEventbridgeでAWSの請求額を取得するLambda関数を起動させ、SNSから通知メールを送信するシステム。
Serverless Frameworkを使用するので、あえてStepFunctionsを使用して構築。

![](/images/serverlessframework/EventBridge.drawio.png =500x)

serverless stepfunctionsプラグインをインストール。
尚、resourceセクションにCloudFormation構文で記述する方法でも作成可能。

```
sudo npm install --save-dev serverless-step-functions
```
#### serverless.yml 【part.1】

* providerセクションにLambdaの実行ロールを定義。
CloudWatch Logsのポリシーは自動的に付与。他にアタッチするポリシーを記述。
ロググループを自動で生成。
* stepFunctionsセクションでEventBridgeを定義。
stepFunctionsセクションで作成すると、実行に必要なポリシーをアタッチしたロールを自動で作成。
EventBridgeが使用するロールはresourceセクションで定義が必要。
ロググループはresourceセクションで定義が必要。stepFunctionsセクションで関連付けを行うと、必要なポリシーを自動でアタッチ。

Serverless Frameworkの機能を活用した方が、テンプレートに記述する量が少ない。

__自動で付与されるポリシー__

| サービス | ポリシー|
| :---| :--- |
| Lambda |logs:CreateLogStream<br>logs:CreateLogGroup<br>logs:TagResource<br>logs:PutLogEvents|
| StepFunctions | 実行に必要なポリシー。 |

```
service: abetest
frameworkVersion: '3'

provider:
  name: aws
  runtime: python3.9
  region: ap-northeast-1
  deploymentBucket:
    name: abetest-deploymentbucket
  #lambdaの実行ロール
  iam:
    role:
      name: costnotification
      statements:
        #SNSポリシー
        - Effect: Allow
          Action:
            - sns:Publish
          Resource:
            - !GetAtt CostSns.TopicArn
        #CostExplolerポリシー
        - Effect: Allow
          Action:
            - ce:GetCostAndUsage
          Resource:
            - '*'

#環境変数を設定
custom:
  LambdaName: cost-notification
  StatemachineName: LambdaStatemachine

#Serverless Step Functionsプラグインの実装
plugins:
  - serverless-step-functions

functions:
  cost-notification:
    name: ${self:custom.LambdaName}  #customセクションで設定した環境変数
    handler: cost.handler
    environment:
        Fn::GetAtt: [CostSns, TopicArn]

#StepFunctionsの作成
stepFunctions:
  stateMachines:
    costnotification:    #Stepfunctionsの実行に必要なポリシーをアタッチしたロールは自動で作成
      id: CostNotificationStateMachine
      name: ${self:custom.StatemachineName}
      definition:
        StartAt: Lambda Invoke
        States:
          Lambda Invoke:
            Type: Task
            Resource: arn:aws:states:::lambda:invoke
            Parameters:
              FunctionName: 
                Fn::GetAtt: [cost-notification, Arn]
            End: true
      #Eventbridgeの作成
      events:
        - schedule:
            name: cost-notification
            rate: cron(0 3 * * ? *)
            role: !GetAtt StepfunctionStartRole.Arn
      #Loggroupの関連付け
      loggingConfig: 
        level: ALL
        includeExecutionData: true
        destinations:
          - Fn::GetAtt: [StepfunctionsLoggroup, Arn]

resources:
  Resources:
    #SNSの設定
    CostSns:
      Type: AWS::SNS::Topic
      Properties: 
        TopicName: cost-notification
        DisplayName: Cost notification
        Subscription: 
          - Endpoint: xxxxxxxxxxxxxx@icloud.com
            Protocol: email

    #SteoFunctionsのロググループ作成
    StepfunctionsLoggroup:
      Type: AWS::Logs::LogGroup
      Properties: 
        LogGroupName: ${self:custom.StatemachineName}

    #StepFunctionsを呼び出すロール
    StepfunctionStartRole:
      Type: AWS::IAM::Role
      Properties:
        Path: /
        RoleName: stepfunction-eventbridge
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - events.amazonaws.com
              Action: sts:AssumeRole      
        Policies:
          - PolicyName: stepfunction-eventbridge
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
                - Effect: Allow
                  Action:
                    - states:StartExecution
                  Resource: !GetAtt CostNotificationStateMachine.Arn
```
#### serverless.yml 【part.2】

LambdaとStepFunctions以外のリソースをresourceセクションで作成。

```
service: abetest
frameworkVersion: '3'

provider:
  name: aws
  runtime: python3.9
  region: ap-northeast-1
  deploymentBucket:
    name: abetest-deploymentbucket

#環境変数を設定
custom:
  LambdaName: cost-notification
  StatemachineName: LambdaStatemachine

#Serverless Step Functionsプラグインの実装
plugins:
  - serverless-step-functions

functions:
  cost-notification:
    name: ${self:custom.LambdaName}  #customセクションで設定した環境変数
    handler: cost.handler
    environment:
        Fn::GetAtt: [CostSns, TopicArn]
    role: LambdaExeRole

#StepFunctionsの作成
stepFunctions:
  stateMachines:
    costnotification: 
      id: CostNotificationStateMachine
      name: ${self:custom.StatemachineName}
      role: !GetAtt StepfunctionExeRole.Arn
      definition:
        StartAt: Lambda Invoke
        States:
          Lambda Invoke:
            Type: Task
            Resource: arn:aws:states:::lambda:invoke
            Parameters:
              FunctionName: 
                Fn::GetAtt: [cost-notification, Arn]
            End: true
      #Loggroupの関連付け
      loggingConfig: 
        level: ALL
        includeExecutionData: true
        destinations:
          - Fn::GetAtt: [StepfunctionsLoggroup, Arn]

resources:
  Resources:
    #SNSの設定
    CostSns:
      Type: AWS::SNS::Topic
      Properties: 
        TopicName: cost-notification
        DisplayName: Cost notification
        Subscription: 
          - Endpoint: xxxxxxxxxxxxxx@icloud.com
            Protocol: email
    #SteoFunctionsのロググループ作成
    StepfunctionsLoggroup:
      Type: AWS::Logs::LogGroup
      Properties: 
        LogGroupName: ${self:custom.StatemachineName}
    #Lambdaの実行ロール
    LambdaExeRole:
      Type: AWS::IAM::Role
      Properties:
        Path: /
        RoleName: costnotification
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole 
        #Cloudwatch Logsポリシー
        Policies:
          - PolicyName: lambda-logging
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
                - Effect: Allow
                  Action:
                    - logs:CreateLogGroup
                  Resource:
                    - !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:*
                - Effect: Allow
                  Action:
                    - logs:CreateLogStream
                    - logs:PutLogEvents
                  Resource:
                    - !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:/aws/lambda/${self:custom.LambdaName}:*
          #SNSポリシー
          - PolicyName: lambda-sns
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
                - Effect: Allow
                  Action:
                    - sns:Publish
                  Resource:
                    - !GetAtt CostSns.TopicArn
          #CostExplolerポリシー
          - PolicyName: lambda-CE
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
                - Effect: Allow
                  Action:
                    - ce:GetCostAndUsage
                  Resource:
                    - '*'
    #StepFunctionsを呼び出すロール
    StepfunctionStartRole:
      Type: AWS::IAM::Role
      Properties:
        Path: /
        RoleName: stepfunction-eventbridge
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - events.amazonaws.com
              Action: sts:AssumeRole      
        Policies:
          - PolicyName: stepfunction-eventbridge
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
                - Effect: Allow
                  Action:
                    - states:StartExecution
                  Resource: !GetAtt CostNotificationStateMachine.Arn
    #Lambdを呼び出すロール
    StepfunctionExeRole:
      Type: AWS::IAM::Role
      Properties:
        Path: /
        RoleName: stepfunction-invoke-lambda
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - states.amazonaws.com
              Action: sts:AssumeRole      
        Policies:
          - PolicyName: stepfunction-invoke-lambda
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
                - Effect: Allow
                  Action:
                    - lambda:InvokeFunction
                  Resource: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:${self:custom.LambdaName}
                  
    #EventBridgeの作成
    StepfunctionSchedule:
      Type: AWS::Events::Rule
      Properties:
        Name: cost-notification
        ScheduleExpression: "cron(0 3 * * ? *)"
        Targets: 
          - Arn: !Ref CostNotificationStateMachine
            Id: ${self:custom.StatemachineName}         
            RoleArn: !GetAtt StepfunctionStartRole.Arn
```

#### cost.py
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
LambdaやStepFunctionsはServerless Frameworkで定義すると、ロールやトリガーなどの管理がしやすい。
様々なプラグインを試してみて、開発の引き出しを増やしていきたい。