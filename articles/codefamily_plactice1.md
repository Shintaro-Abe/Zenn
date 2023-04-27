---
title: "CodeCommitとローカル環境の連携 【AWS Code Family Plactice 1/9】" 
emoji: "🪺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codecommit","git"]
published: false
---

# 開発手法について
組織内での情報共有・連携が取れないサイロ化は問題であり、障害を取り除くための開発手法を用いてリリースを迅速化。

## Agile
開発のプロセスにフォーカス。
開発のサイクルを細分化し、コードなどの更新とテストを頻繁に行うことで、迅速にソフトウェアをリリース。
顧客からの仕様変更や追加などへ柔軟に対応するため、ソフトウェアの更新を迅速に行う手法。
開発チームと運用チームの連携が必要なため、DevOpsの文化が必要。
## CI/CD
継続的インテクレーション/継続的デリバリーの略。
ツールを活用した開発の自動化にフォーカス。
コードの頻繁な更新によるテストやデプロイ、本番環境へのリリースのプロセスを仕組み化。
ツールを用いて自動化することで、開発に集中することが可能。
統合開発環境、Git、構成管理、ビルド、テストなど、様々なツールを組み合わせて自動化。
## DevOps
組織の文化や役割にフォーカス。
開発者と運用者のスキルを互いに習得し協働することで、障害点の無い効率的な開発・運用を実施。
開発の段階からセキュリティ基盤を導入するDevSecOpsがある。

| 用語 | 意味 |
| --- | ---|
| DevSecOps | DevOpsサイクルにセキュリティーを組み込んだソフトウェアの開発手法。<br>DevOpsによるアジリティを活用した短期的かつ頻繁な開発サイクルにより、早期にセキュリティ基盤を導入する必要性から生まれた。<br>自動化にセキュリティを統合し、コードの保管からパイプラインまでのプロセスを保護。|

# 概念
* __システム__
プライベートGitリポジトリをホストする、マネージド型のソースコントロールサービス。 ハードウェアとソフトウェアの管理は不要で、転送時と保管時の暗号化をサポート。 開発のニーズに合わせて自動的にスケールアップし、大量のファイル、バージョン、ブランチの管理などに対応。 リポジトリのサイズ保存できるファイルのタイプに制限が無い。 
* __操作__
IAMでHTTPS接続とGit認証情報を使用し、ユーザー名とパスワードを生成。HTTPS接続で操作。 SSHキーをIAMユーザーに紐付け、SSH接続による操作も可能。ローカルにリモートリポジトリを作成し、GitコマンドとAWS CLIコマンドを使用。
* __AWSサービスとの統合__
他のAWSサービスと統合が可能で、Cloud9でコードの作成とコミットを行い、EventBridgeでリポジトリの変更をトリガー、CodePipelineがコードを読み取りビルドとデプロイを実行。

| 用語 |意味 |
| --- |
| ソースコントロール | 開発プロセスにおける要素。 <br>ソースコード管理システムを使用してコードの変更を履歴として残し、必要に応じて過去のバージョンに戻すことが可能。 <br>開発者ごとに作業を切り離したり、コードのレビューと承認作業を行い反映させることが可能。<br>個人、もしくはチームに関わらず、コードを一元化して管理することで開発を効率化。<br>Gitはその代表的なシステム。|

#CodeCommitの開始方法
##AWS CodeCommit の HTTPS Git 認証情報
IAMのセキュリティー認証情報を選択。

![image][cc1.png]

HTTTPSのGit認証情報を生成をクリック。

![image][cc2.png]

認証情報のCSVをダウンロード。

![image][cc3.png]

IAMに認証情報が登録される。

![image][cc4.png]

## VSCodeの設定

ローカル環境で初めてクローンを作成する場合、Git認証を求められる。

![image][cc8.png]

IAMで取得したユーザネームの情報を入力。

![image][cc9.png]

Git認証のパスワードを入力。

![image][cc10.png]


## Codecommitコンソール操作
CodeCommitコンソールにあるリポジトリを作成を選択。

![image][cc5.png]

リポジトリ名と説明を入力。

![image][cc6.png]

ローカルにリポジトリのクローンを作成するため、コマンドをコピー。

![image][cc7.png]

コンソール画面に戻り、リポジトリを確認。

![image][cc11.png]

##ローカル操作
* Codecommit用のディレクトリを作成し、Gitの初期化を実行。

```
git config --global init.defaultBranch main
```
* Codecommitリポジトリのクローンを作成。共有リポジトリを作成できるように、ディレクトリ名を指定可能。

```
git clone https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/abetest abetest
```
グローバルでinitをしなかった場合に実行。

```
git config --local init.defaultBranch main 
```
```
git add . 
```
```
git commit -m "first commit"
```
* オプション` u `は上流ブランチに設定。`--set-upstream`と同じ。

```
git push -u origin main
```
もしくは以下のコマンドでも可能だが、ブランチ名を指定していない状態。

```
git push
```

CodeCommitのリポジトリにファイルがアップロードされる。

![image][cc12.png]

コミットの履歴はコミットページから確認可能。

![image][cc13.png]

ブランチはブランチページより確認可能。

![image][cc14.png]

* リポジトリの削除(CLI)

```
aws codecommit delete-repository --repository-name abetest
```
* リポジトリの作成(CLI)

```
aws codecommit create-repository --repository-name hogepiyo \
--repository-description "hogehogepiyopiyo" \
--tags Team=abetest
```
結果

```
REPOSITORYMETADATA      arn:aws:codecommit:ap-northeast-1:417338593075:hogepiyo 417338593075    https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/hogepiyo        ssh://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/hogepiyo     2023-02-22T04:19:12.706000+09:00        2023-02-22T04:19:12.706000+09:00     hogehogepiyopiyo        8a5e5bfc-7057-45fc-ae9c-73f535fd6414    hogepiyo
```
## 共有リポジトリの作成
* リポジトリの作成は基本的に同じ。

```
git config --global init.defaultBranch main
```
```
git clone https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/abetest share
```

クローンを作成すると、リポジトリの最新情報を取得。

![image][cc15.png]

* Codecommitリポジトリから最新情報を受け取る。

```
git pull
```
```
git add .
git commit -m "first commit" 
git push
```

例)  lambda-condition.ymlをCodeCommitリポジトリに追加。

![image][cc16.png]

片方のリポジトリへ移動し、CodeCommitリポジトリの最新情報を取得。

```
git pull
```

例)  shareリポジトリからpushしたlambda-codition.ymlがabetestリポジトリへ反映された。

![image][cc17.png]

## ブランチの作成

```
git log
```
結果

```
commit 3e9eba5c9a8c7c18dbc691f3776c59e8dee9a9b2 (HEAD -> main, origin/main)
Author: Shintaro Abe <121922274+Shintaro-Abe@users.noreply.github.com>
Date:   Wed Feb 22 05:05:03 2023 +0900

    sns.yml
```
* ブランチを作成する構文は、

```
git checkout -b ブランチ名 分岐させたいバージョンのコミットID
```

```
git checkout -b newbranch 3e9eba5c9a8c7c18dbc691f3776c59e8dee9a9b2
```
作成と同時に新しいブランチへ切り替え。
結果

```
Switched to a new branch 'newbranch'
```
Codecommitリポジトリへ新しいブランチを送信。

```
git push origin newbranch
```
* 新しいブランチをプルインするコマンド。

```
git fetch origin
```
* リポジトリの全ブランチのリスト。最新情報を入手する場合はプルインしてから行う。

```
git branch --all
```
結果

```
  main
* newbranch
  remotes/origin/main
  remotes/origin/newbranch
```
```
git checkout newbranch
```
```
git branch 
or
git status
```
結果

```
  main
* newbranch
```
* ブランチを切り替える場合は`git checkout`コマンドを使用。

```
git checkout main
```
結果

```
Switched to branch 'main'
Your branch is up to date with 'origin/main'.
```
```
git branch 
or
git status
```
結果

```
On branch main
Your branch is up to date with 'origin/main'.
```

コミットごとに、作成者やバージョンのリンクを表示。

![image][cc18.png]

コミットIDをクリックすると、バージョンの内容を確認可能

![image][cc19.png]

ブランチページで現在使用しているブランチを確認可能。

![image][cc20.png]