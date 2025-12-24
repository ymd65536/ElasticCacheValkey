# 【AWS】検証！〜

## はじめに

この記事では「この前リリースされた機能って実際に動かすとどんな感じなんだろう」とか 「もしかしたら内容次第では使えるかも？？」などAWSサービスの中でも特定の機能にフォーカスして検証していく記事です。

主な内容としては実践したときのメモを中心に書きます。（忘れやすいことなど） 誤りなどがあれば書き直していく予定です。

今回はAmazon ElastiCacheで利用できるAmazon ElastiCache for Valkey（以下、本文ではValkeyと表記し、Amazon ElastiCache for Redisの場合においては「Redis」と表記）を検証してみます。

[参考：Amazon ElastiCache for Valkeyの開始方法](https://aws.amazon.com/jp/blogs/news/get-started-with-amazon-elasticache-for-valkey/)

## Amazon ElastiCache for Valkeyとは

まずは今回検証するValkeyについて見ていきましょう。`What is Valkey?`では以下のように説明されています。

> Valkey is an open source, in-memory, high performance, key-value datastore. It is a drop-in replacement for Redis OSS. It can be used for a variety of workloads such as caching, session stores, and message queues, and can act as a primary database. Valkey can run as either a standalone daemon or in a cluster, with options for replication and high availability.

[引用：What is Valkey? - Valkey Explained - AWS](https://aws.amazon.com/jp/elasticache/what-is-valkey/)

どんなものか簡単にまとめると「Redis OSSの代替として開発されたオープンソースのインメモリ型キーバリューデータストア」ということになります。

## なんでValkeyなんですか？Redisじゃいけないんですか

Valkeyが登場したとき、背景を知らない人からすると「なんで？これが必要なの？」
「今までずっとRedisで上手くいっていたのに、なぜ変える必要があるのか？」という疑問が浮かんだと思います。

技術的にも経済的にもValkeyのほうが優秀に見えるのでそれが代替（Alternative）の理由となったと勘違いしそうですが
実際は技術的な背景ではなく、非常に複雑なライセンス事情が絡んでいます。

### 複雑なライセンス事情の話

長年愛されているRedisはBSDライセンスとして無料で商用利用可能な状態で提供されていましたが
2024年3月に開発元のRedis社が方針を転換し、ライセンスを変更することになりました。

なお、2025年5月のRedis8においては方針を一部修正してライセンスも違う形で提供するというアナウンスがされており
現在においてはOSI(オープンソースイニシアチブ)承認済みであり、再びOSSとして利用できるようになっています。

ただし、こういった形で利用に不信感をもたせたことや変更後のライセンス(AGPL)の縛りが強いことなどあり、Valkeyが注目を浴びることになりました。
※この件はについて、視点はさまざまでAWSやGoogleが強力に推進していることやLinux Foundationという中立な組織傘下で管理されているからといった見方もあります。

また、すでにValkeyへの開発にシフトしてしまってもとに戻ることができないといった開発者の事情もあるでしょう。（筆者が同じ立場、開発者ならそう考えます）

## Amazon ElastiCache for Valkeyの料金

このあとハンズオンしていきますが、そのまえに料金表を確認しましょう。
今回はServerlessでハンズオンするのでServerlessの料金に注目します。
細かい料金計算は難しいと思いますので割愛しますが、理解しておくべき主な課金要素は以下の2つです。

- データの保存
- ElastiCache Processing Units (ECPU)

[引用：Amazon ElastiCache の料金](https://aws.amazon.com/jp/elasticache/pricing/)

Serverlessなのでユニット利用時に料金がかかります。計算が終了してもデータ保存で料金が発生するので注意しましょう。

## キャッシュの操作方法

操作方法はいくつかあります。具体的には以下のとおりです。

- valkey-cli

## ハンズオン

どんなものか理解できたところでハンズオンしていきましょう。

おおまかな手順

- AWSマネジメントコンソールを開いてAWS Cloud Shellを起動
- ユーザーの作成
- キャッシュの作成
- デフォルトVPCの作成
- Cloud Shellの起動
- valkey-cli のセットアップ
- キーバリューストアのテスト
- キャッシュの削除

### キャッシュの作成

まずはAWS Cloud Shellを起動し、以下のコマンドでValkeyのキャッシュを作成します。
今回はAWS IAM Identity Center (SSO)でログインしていることを前提としています。権限はAdministratorAccessです。

```bash
aws elasticache create-user --user-id valkey-default-user --user-name default --engine valkey --passwords "YourStrongPassword123!" --access-string "on ~* +@all"
```

グループを作成してユーザーを追加します。以下のコマンドを実行してください。

```bash
aws elasticache create-user-group --user-group-id my-user-group --engine valkey --user-ids valkey-default-user
```

もとからあるユーザーグループにユーザーを追加する場合は以下のコマンドを実行します。

```bash
aws elasticache modify-serverless-cache --serverless-cache-name ec-valkey-serverless --user-group-id my-user-group
```

次にキャッシュを作成します。以下のコマンドを実行してください。ユーザーグループIDは先ほど作成したものを指定します。

```bash
aws elasticache create-serverless-cache --serverless-cache-name ec-valkey-serverless --engine valkey --user-group-id my-user-group --region ap-northeast-1
```

### valkey-cli のセットアップ

次にvalkey-cliをセットアップします。以下のコマンドを実行してください。

```bash
sudo yum install gcc jemalloc-devel openssl-devel tcl tcl-devel -y 
wget https://github.com/valkey-io/valkey/archive/refs/tags/7.2.7.tar.gz 
tar xvzf 7.2.7.tar.gz 
cd valkey-7.2.7/ 
make BUILD_TLS=yes install
```

### キャッシュの確認

キャッシュが作成されたか確認しましょう。以下のコマンドを実行してください。

```bash
aws elasticache describe-serverless-caches --serverless-cache-name ec-valkey-serverless --region ap-northeast-1
```

実行結果

```json
{
    "ServerlessCaches": [
        {
            "ServerlessCacheName": "ec-valkey-serverless",
            "Description": " ",
            "CreateTime": "2025-12-24T07:43:57.288000+00:00",
            "Status": "available",
            "Engine": "valkey",
            "MajorEngineVersion": "8",
            "FullEngineVersion": "8.1",
            "SecurityGroupIds": [
                "sg-XXX"
            ],
            "Endpoint": {
                "Address": "ec-valkey-serverless-XXXX.serverless.apne1.cache.amazonaws.com",
                "Port": 6379
            },
            "ReaderEndpoint": {
                "Address": "ec-valkey-serverless-XXXX.serverless.apne1.cache.amazonaws.com",
                "Port": 6380
            },
            "ARN": "arn:aws:elasticache:ap-northeast-1:0000000:serverlesscache:ec-valkey-serverless",
            "SubnetIds": [
                "subnet-XXX1",
                "subnet-XXX2",
                "subnet-XXX3"
            ],
            "SnapshotRetentionLimit": 0,
            "DailySnapshotTime": "17:30"
        }
    ]
}
```

出力されることを確認したら以下のコマンドでエンドポイントとポート番号を取得しましょう。

```bash
export ENDPOINT=`aws elasticache describe-serverless-caches --serverless-cache-name ec-valkey-serverless --region ap-northeast-1 --query "ServerlessCaches[0].Endpoint.Address" --output text` && echo $ENDPOINT
export PORT=`aws elasticache describe-serverless-caches --serverless-cache-name ec-valkey-serverless --region ap-northeast-1 --query "ServerlessCaches[0].Endpoint.Port" --output text` && echo $PORT
```

### キーバリューストアのテスト

```bash
valkey-cli -h ${ENDPOINT} -p ${PORT} -c --tls -t 15 --askpass
```

## 参考

- [Linux Foundation Launches Open Source Valkey Community](https://www.linuxfoundation.org/press/linux-foundation-launches-open-source-valkey-community)
- [Valkey 用 Amazon ElastiCache の発表 - AWS](https://aws.amazon.com/jp/about-aws/whats-new/2024/10/amazon-elasticache-valkey/)
- [Valkey 互換キャッシュ、Memcached 互換キャッシュ、Redis OSS 互換キャッシュ – Amazon ElastiCache – AWS](https://aws.amazon.com/jp/elasticache/)

# AWS CLI インストールと SSO ログイン手順 (Linux環境)

このガイドでは、Linux環境でAWS CLIをインストールし、AWS SSOを使用してログインするまでの手順を説明します。

## 前提条件

- Linux環境（Ubuntu、CentOS、Amazon Linux等）
- インターネット接続
- 管理者権限（sudoが使用可能）
- AWS SSO が組織で設定済み
- Python 3.12.1

## AWS CLI のインストール

### 公式インストーラーを使用（推奨）

最新版のAWS CLI v2を公式インストーラーでインストールします。

```bash
# 1. インストーラーをダウンロード
curl "https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip" -o "awscliv2.zip"

# 2. unzipがインストールされていない場合はインストール
sudo apt update && sudo apt install unzip -y  # Ubuntu/Debian系
# または
sudo yum install unzip -y                     # CentOS/RHEL系

# 3. ダウンロードしたファイルを展開
unzip awscliv2.zip

# 4. インストール実行
sudo ./aws/install

# 5. インストール確認
aws --version

# ダウンロードしたzipファイルと展開したディレクトリを削除してクリーンアップします。
rm  "awscliv2.zip"

# 解凍したディレクトリを削除
rm -rf aws
```

## AWS SSO の設定とログイン

### 1. AWS SSO の設定

AWS SSOを使用するための初期設定を行います。

```bash
aws configure sso
```

設定時に以下の情報の入力が求められます：

- **SSO start URL**: 組織のSSO開始URL（例：`https://my-company.awsapps.com/start`）
- **SSO Region**: SSOが設定されているリージョン（例：`us-east-1`）
- **アカウント選択**: 利用可能なAWSアカウントから選択
- **ロール選択**: 選択したアカウントで利用可能なロールから選択
- **CLI default client Region**: デフォルトのAWSリージョン（例：`ap-northeast-1`）
- **CLI default output format**: 出力形式（`json`、`text`、`table`のいずれか）
- **CLI profile name**: プロファイル名（`default`とします。）

### 2. AWS SSO ログイン

設定完了後、以下のコマンドでログインを実行します。

```bash
aws sso login
```

ログイン時の流れ：
1. コマンド実行後、ブラウザが自動的に開きます
2. AWS SSOのログインページが表示されます
3. 組織のIDプロバイダー（例：Active Directory、Okta等）でログイン
4. 認証が成功すると、ターミナルに成功メッセージが表示されます

### 3. ログイン状態の確認

認証情報を確認します。

```bash
aws sts get-caller-identity
```

正常にログインできている場合、以下のような情報が表示されます：

```json
{
    "UserId": "AROAXXXXXXXXXXXXXX:username@company.com",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/RoleName/username@company.com"
}
```

## トラブルシューティング

### よくある問題と解決方法

#### 1. ブラウザが開かない場合

```bash
# 手動でブラウザを開く場合のURL確認
aws sso login --no-browser
```

表示されたURLを手動でブラウザで開いてください。

#### 2. セッションが期限切れの場合

```bash
# 再ログイン
aws sso login
```

#### 4. プロキシ環境での設定

プロキシ環境の場合、以下の環境変数を設定してください：

```bash
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
export NO_PROXY=localhost,127.0.0.1,.company.com
```

## セキュリティのベストプラクティス

1. **定期的な認証情報の更新**: SSOセッションには有効期限があります。定期的に再ログインを行ってください。

2. **最小権限の原則**: 必要最小限の権限を持つロールを使用してください。

3. **プロファイルの分離**: 本番環境と開発環境で異なるプロファイルを使用してください。

4. **ログアウト**: 作業終了時は適切にログアウトしてください：
   ```bash
   aws sso logout --profile <プロファイル名>
   ```

## 参考リンク

- [AWS CLI ユーザーガイド](https://docs.aws.amazon.com/cli/latest/userguide/)
- [AWS SSO ユーザーガイド](https://docs.aws.amazon.com/singlesignon/latest/userguide/)
- [AWS CLI インストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
