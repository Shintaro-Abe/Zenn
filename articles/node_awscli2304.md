---
title: "Node.jsとAWS CLIのインストール"
emoji: "🌰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "nodejs"]
published: false
---

# Node.js
## 概要
JavaScriptをサーバーサイドで動かすための実行環境。
アプリケーションのパフォーマンスを維持するため、シングルプロセス・シングルスレッドで非同期I/Oを利用し、時間軸で処理を分散。
WEBサイトやアプリケーションなどの制作、サーバー制御をJavaScriptで実行可能。
## npm
Node Package Managerの略。
アプリケーション開発に利用できる多くのツールを、パッケージとして管理。

* __package.json__
インストールするパッケージと、その依存関係を指定。

* __node_modules__
インストールしたパッケージを保存。

* __package-lock.json__
使用されたパッケージのバージョンなどを特定。
依存関係をロックし、意図した通りに動作させる役割。

## インストール

Node.jsの[ダウンロードページ](https://nodejs.org/ja/download)へアクセス。
使用PC環境に合わせたパッケージをダウンロード。
macOS Installer ARM64を選択。

![](/images/node_awscli/node1.png =500x)

インストーラを開き、「続ける」を選択。

![](/images/node_awscli/node2.png =500x)

使用許諾契約を確認し、「続ける」を選択。

![](/images/node_awscli/node3.png =500x)

同意するを選択。

![](/images/node_awscli/node4.png =500x)

インストールを選択。

![](/images/node_awscli/node5.png =500x)

PCのパスワードを入力。

![](/images/node_awscli/node6.png =500x)

ダウンロードフォルダ内のファイルへアクセス許可。

![](/images/node_awscli/node7.png =500x)

インストール完了。

![](/images/node_awscli/node8.png =500x)

Node.jsバージョン照会コマンド

```
node -v 
```
npmパッケージバージョン照会コマンド

```
npm -v
```

コマンドの場所を確認。

```
which npm

#結果
/usr/local/bin/npm
```

コマンドにパスが通っているか確認。
```
echo $PATH

#結果
usr/local/bin
```
# AWS CLI
AWS CLI Version2を使用。

## 概要
AWS Command Line Interfceの略。
コマンドを使用して、AWSサービスを操作するツール。
Linux、mac、Windowsのコマンドラインシェルの他、AWS Systems Managerを利用してEC2インスタンスで実行も可能。

## インストール
### パッケージのインストール
curlコマンドでインターネットからパッケージをダウンロード。

```
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
```

ルートディレクトリにパッケージをインストール。

```
sudo installer -pkg ./AWSCLIV2.pkg -target /
```
パスが通っているか確認。

```
which aws

#結果　
/usr/local/bin/aws 
```
バージョンの確認。

```
aws --version

#結果
aws-cli/2.10.1 Python/3.9.11 Darwin/22.2.0 exe/x86_64 prompt/off
```
### アクセスキーのインポート
以下のコマンドでアクセスキーをPCにインポート。
アクセスキーID、シークレットキー、リージョン、出力形式を入力。

| 項目| 役割 |
| :---| :--- |
| region | デフォルトで使用するAWSリージョンを指定。<br> 他のリージョンを使用する場合は、個別のコマンドで指定。 |
| output | 結果の出力形式を指定。<br>json,yaml,text,tableから選択可能。

```
aws configure
```

![](/images/node_awscli/cl8.png =500x)

インポートされたデータは、.awsディレクトリに保管される。

| ファイル名| データ|
| :---| :---|
| credentials |アクセスキーID<br>シークレットキー|
| config | リージョン<br>出力形式|

オプションで` --profile <ユーザー名>`をつけると、個別の認証情報を保存可能。
プロファイルを指定しない場合は、デフォルトとして保存。
# AWS CLIコマンド
ローカル環境からAWSサービスを操作するコマンド例。
サービスを操作するコマンドと属性を指定するオプションを組み合わせて入力。

```
aws サービス名 実行コマンド オプション

#例
aws ec2 start-instances --instance-ids i-038f0153bb1ea073e
```
## EC2

| EC2コマンド | 役割 |
| :---| :---|
| [run-instances](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/run-instances.html) | インスタンスの起動。|
| [start-instances](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/start-instances.html) | インスタンスの開始。|
| [stop-instances](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/stop-instances.html) |インスタンスの停止。|
| [describe-instances](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-instances.html) | インスタンスの詳細。|


* __インスタンスの起動__

AMIの指定は必須。
以下は、AMI、インスタンスタイプ、キーペアを指定。
VPCとサブネットの指定がないため、` .aws `ディレクトリで指定したリージョンにあるデフォルトのVPCで起動。

```
aws ec2 run-instances \
--image-id ami-0ffac3e16de16665e \
--instance-type t2.micro \
--key-name abetest

#結果(output : text)
123456789012    r-02076396f8a95577f
INSTANCES       0       x86_64  b7e68e75-fe25-4ed5-b8ae-65aa114d5a14    False   True    xen        ami-0ffac3e16de16665e   i-0e62fa28d6d50d2b1     t2.micro        abetest 2023-02-20T19:04:47+00:00  ip-172-31-14-138.ap-northeast-1.compute.internal        172.31.14.138              /dev/xvda       ebs     True            subnet-0338acfe8d91a0c85        hvm     vpc-069b2ec37bcd4c3bb
CAPACITYRESERVATIONSPECIFICATION        open
CPUOPTIONS      1       1
ENCLAVEOPTIONS  False
MAINTENANCEOPTIONS      default
METADATAOPTIONS enabled disabled        1       optional        disabled        pending
MONITORING      disabled
NETWORKINTERFACES               interface       0a:cd:cd:67:92:65       eni-0b45ecf3d61fd18cd      123456789012    ip-172-31-14-138.ap-northeast-1.compute.internal        172.31.14.138      True    in-use  subnet-0338acfe8d91a0c85        vpc-069b2ec37bcd4c3bb
ATTACHMENT      2023-02-20T19:04:47+00:00       eni-attach-0ec9de670c5bb5cfd    True    0          0       attaching
GROUPS  sg-0ac5d80dfa99e08b1    default
PRIVATEIPADDRESSES      True    ip-172-31-14-138.ap-northeast-1.compute.internal        172.31.14.138
PLACEMENT       ap-northeast-1c         default
PRIVATEDNSNAMEOPTIONS   False   False   ip-name
SECURITYGROUPS  sg-0ac5d80dfa99e08b1    default
STATE   0       pending
STATEREASON     pending pending
```
* __インスタンスの開始__

```
aws ec2 start-instances --instance-ids i-038f0153bb1ea073e

#結果
STARTINGINSTANCES       i-038f0153bb1ea073e
CURRENTSTATE    0       pending
PREVIOUSSTATE   80      stopped
```
* __インスタンスの停止__

```
aws ec2 stop-instances --instance-ids i-038f0153bb1ea073e

#結果
STOPPINGINSTANCES       i-038f0153bb1ea073e
CURRENTSTATE    64      stopping
PREVIOUSSTATE   16      running
```
* __インスタンスの詳細__

```
aws ec2 describe-instances --instance-ids i-038f0153bb1ea073e

#結果
RESERVATIONS    123456789012    r-00d201e34bf5ab6dc
INSTANCES       0       x86_64          False   True    xen     ami-0bba69335379e17f8   i-038f0153bb1ea073e     t2.micro        abetest 2023-02-20T16:09:03+00:00    Linux/UNIX      ip-10-0-10-30.ap-northeast-1.compute.internal   10.0.10.30              /dev/xvda       ebs     True    User initiated (2023-02-20 16:09:49 GMT)     subnet-0f94a01bdaacf8038        RunInstances    2023-02-11T17:50:39+00:00       hvm     vpc-03671a70f08634fe6
BLOCKDEVICEMAPPINGS     /dev/xvda
EBS     2023-02-11T17:50:40+00:00       True    attached        vol-00b9cc3ffee05b358
CAPACITYRESERVATIONSPECIFICATION        open
CPUOPTIONS      1       1
ENCLAVEOPTIONS  False
HIBERNATIONOPTIONS      False
MAINTENANCEOPTIONS      default
METADATAOPTIONS enabled disabled        1       optional        disabled        applied
MONITORING      disabled
NETWORKINTERFACES               interface       06:f8:9f:bb:3e:87       eni-0322957cfa6bc9585   123456789012    ip-10-0-10-30.ap-northeast-1.compute.internal        10.0.10.30      True    in-use  subnet-0f94a01bdaacf8038        vpc-03671a70f08634fe6
ATTACHMENT      2023-02-11T17:50:39+00:00       eni-attach-0ff35748d0c1e537e    True    0       0       attached
GROUPS  sg-0d2bfdda0989b5be4    ansible-network
PRIVATEIPADDRESSES      True    ip-10-0-10-30.ap-northeast-1.compute.internal   10.0.10.30
PLACEMENT       ap-northeast-1a         default
PRIVATEDNSNAMEOPTIONS   False   False   ip-name
SECURITYGROUPS  sg-0d2bfdda0989b5be4    ansible-network
STATE   80      stopped
STATEREASON     Client.UserInitiatedShutdown    Client.UserInitiatedShutdown: User initiated shutdown
TAGS    Name    nginx-server
```

__outputはtableが一番視認性に優れいている。__

![](/images/node_awscli/clcf10.png =500x)

* __インスタンスの終了__

```
aws ec2 terminate-instances --instance-ids i-0e62fa28d6d50d2b1

#結果
tance-ids i-0e62fa28d6d50d2b1
TERMINATINGINSTANCES    i-0e62fa28d6d50d2b1
CURRENTSTATE    32      shutting-down
PREVIOUSSTATE   16      running
```

## CloudFormation

| CloudFormationコマンド | 役割 |
| :---| :---|
| [create-stack](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/create-stack.html) | スタックの作成。|
| [deploy](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/deploy.html) | 変更セットの作成と実行。|
| [create-change-set](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/create-change-set.html) |変更セットの作成。|
| [execute-change-set](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/execute-change-set.html) | 変更セットの実行。|
| [delete-stack ](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/delete-stack.html) | スタックの削除|
| [package](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/package.html) | ソースコードなどの依存関係データをパッケージ化。|

### スタックの作成

* __パラメーターの指定無し__

```
aws cloudformation create-stack --stack-name abetest --template-body file://event.yml

#結果
arn:aws:cloudformation:ap-northeast-1:417338593075:stack/abetest/aa96ea30-b13a-11ed-a0bd-0eea2595cf5b
```
* __パラメーターの指定あり__
 
| オプション | 役割 |
| :---| :---|
| ParameterKey | テンプレートにあるPrametersセクションの論議ID |
| ParameterValue | 論理IDの値 |

```
aws cloudformation create-stack --stack-name test \
--template-body file://subnet-test.yml \
--parameters ParameterKey=VPCCIDR,ParameterValue=10.0.0.0/16

#結果
arn:aws:cloudformation:ap-northeast-1:417338593075:stack/test/7f8e1960-d202-11ed-b292-0e97dac47123
```
---

### 変更セットの作成と実行

* __deployコマンドの使用__

変更セットを作成後、スタックを変更。
また、初回のスタック作成時で、` create-stack `コマンドの代わりに使用可能。

```
aws cloudformation deploy --template-file event.yml --stack-name test
```
パラメータを作成する場合は` parameter-overrides `オプションで指定。
`< Parametersの論理ID >=< 値 >`

```
aws cloudformation deploy --template-file subnet-test.yml --stack-name test \
--parameter-overrides VPCCIDR=10.0.0.0/16

#結果
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - test
```
CloudFormationのコンソール画面にも変更セットを作成、実行した形跡が残る。

![](/images/node_awscli/clcf6.png =500x)

---

* __create-change-setとexecute-change-setコマンドの使用__

create-change-setコマンドで変更セットを作成。

```
aws cloudformation create-change-set --stack-name test \
--change-set-name test-change-set --template-body file://subnet-test.yml \
--parameters ParameterKey=VPCCIDR,ParameterValue=172.16.0.0/16

#結果
arn:aws:cloudformation:ap-northeast-1:123456789012:changeSet/test-change-set/b675749c-54e9-47f0-97bc-c71a074c94cc       test-change-set 2023-04-03T10:59:45.723000+00:00     None    AVAILABLE       False   None    None    arn:aws:cloudformation:ap-northeast-1:123456789012:stack/test/153f8130-d20b-11ed-8f82-0a967e43cf67   test    CREATE_COMPLETE None    None
CHANGES Resource
RESOURCECHANGE  Modify  TestSubnet      subnet-070a15df116a4e8c2        True    AWS::EC2::Subnet
DETAILS VPCCIDR ParameterReference      Static
TARGET  Properties      CidrBlock       Always
DETAILS TestVPC.VpcId   ResourceAttribute       Static
TARGET  Properties      VpcId   Always
DETAILS         DirectModification      Dynamic
TARGET  Properties      CidrBlock       Always
SCOPE   Properties
CHANGES Resource
RESOURCECHANGE  Modify  TestVPC vpc-02ec090ea4e0e8cc1   True    AWS::EC2::VPC
DETAILS VPCCIDR ParameterReference      Static
TARGET  Properties      CidrBlock       Always
DETAILS         DirectModification      Dynamic
TARGET  Properties      CidrBlock       Always
SCOPE   Properties
PARAMETERS      VPCCIDR 172.16.0.0/16
```
変更内容の確認後、execute-change-setコマンドで実行。
結果は表示されない。

```
aws cloudformation execute-change-set \
--change-set-name test-change-set \
--stack-name test
```
---

### スタックの削除
delete-stackコマンドでスタックを削除。
結果は表示されない。

```
aws cloudformation delete-stack --stack-name test
```
## CloudFormation パッケージコマンド
Lambdaのデプロイは、テンプレートとソースコードなどの依存関係データをパッケージ化してS3にアップロード。
`Transform`セクションを設け、`Resources`セクションの`Type`パラメータの値に`AWS::Serverless::Function`を入力。
CodeUriパラメータの値にソースコードのパスを入力。

__s3-up.yml__

```
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::Serverless-2016-10-31
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler                               #index.pyなのでハンドラーはindex.handler
      Runtime: python3.9
      CodeUri: /Users/abeshintarou/Documents/src/Cloud-Formation/templates/index.py
```
__index.py__

```
import json

def handler(event, context):
    # TODO implement
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```
デフォルトでYAMLファイルとしてパッケージング。`--use-yaml`は不要。

* __パッケージング__ 

```
aws cloudformation package \
--template s3-up.yml \
--s3-bucket abetesttest \
--output-template-file packaged-s3-up.yml
```
packaged-s3-up.ymlを生成。
バケット`abetesttest`にパッケージファイル`10b6a22c6999c40909b97229d82e53f7`を生成

```
Uploading to 10b6a22c6999c40909b97229d82e53f7  245 / 245.0  (100.00%)
Successfully packaged artifacts and wrote output template to file packaged-s3-up.yml.
Execute the following command to deploy the packaged template
aws cloudformation deploy --template-file /Users/abeshintarou/Documents/src/Cloud-Formation/templates/packaged-s3-up.yml --stack-name <YOUR STACK NAME>
```
バケットabetesttestに自動でパッケージファイルを生成し、格納。

![](/images/node_awscli/clcf1.png =500x)

* __デプロイ__
テンプレート内でIAMリソースを作成する必要がある場合には
`--capabilities CAPABILITY_IAM`もしくは`--capabilities CAPABILITY_NAMED_IAM`をつける。

```
aws cloudformation deploy \
--template-file /Users/abeshintarou/Documents/src/Cloud-Formation/templates/packaged-s3-up.yml \
--stack-name test \
--capabilities CAPABILITY_IAM
```
CloudFormationのコンソールも、連動して結果を表示。

![](/images/node_awscli/clcf2.png =500x)

Lambda関数を構築。

![](/images/node_awscli/clcf3.png =500x)

## S3

| S3コマンド | 役割 |
| :--- | :---|
| [mb](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/mb.html) | バケットの作成。|
| [cp](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/cp.html) | オブジェクトの作成。|
| [ls](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/ls.html) |オブジェクトもしくはバケットの一覧。|
| [rm](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/rm.html) | オブジェクトの削除。|
| [rb ](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/rb.html) | バケットの削除|

* __バケットの作成__

```
aws s3 mb s3://abetesttest

#結果
make_bucket: abetesttest
```
バケットが作成された。

![](/images/node_awscli/clcf5.png =500x)

* __オブジェクトの追加__

```
aws s3 cp '/Users/abeshintarou/Desktop/server9.png' s3://abetesttest

#結果
upload: ../../../Desktop/server9.png to s3://abetesttest/server9.png
```
バケットにオブジェクトが追加された。

![](/images/node_awscli/clcf4.png =500x)

* __オブジェクトの一覧__

```
aws s3 ls abetesttest

#結果
2023-02-21 03:38:56        245 10b6a22c6999c40909b97229d82e53f7
2023-02-21 04:51:09     419804 server9.png
```

* __オブジェクトの削除__

```
aws s3 rm s3://abetesttest/server9.png

#結果
delete: s3://abetesttest/server9.png
```
* __バケットの削除__

```
aws s3 rb s3://abetesttest

#結果
remove_bucket: abetesttest
```
