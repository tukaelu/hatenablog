---
Title: TerraformでS3+CloudFront+ACMなブログ環境を作ってみる
Category:
- terraform
Date: 2018-04-05T00:03:54+09:00
URL: https://tukaelu.hatenablog.jp/entry/2018/04/05/create-blog-env-with-terraform
EditURL: https://blog.hatena.ne.jp/tukaelu/tukaelu.hatenablog.jp/atom/entry/17391345971632129092
---

半年くらい前にTerraformを触り始めたときにS3+CloudFront+ACMな構成でブログ書く環境を作ったときのメモです。

コードもGithubで公開しているので同じような環境がこの記事を見ながら構築できるかもしれません。

<!-- more -->

## ざっくり構成

[f:id:tukaelu:20180404064855p:plain]

コンテンツをS3にアップロード、フロントにCloudFront（以下CF）を配置してアクセスはCFで受け付けるよくあるパターンですね。

もともとドメインをRoute53で管理していたので既存のゾーンに対してレコードを追加し、ワイルドカードなSSL証明書もACMで取得していたのでそちらをCFに設置して使用します。

コンテンツ自体はサイトジェネレーターのJekyllで生成してGithubにプッシュしたタイミングでCircleCIが回ってS3にsyncする感じです。

今回はTerraformでの構築の部分にのみざくっり触れます。

## さっそく構築してみる

### 前提事項

- 今回はRoute53で管理しているゾーン内に新たにレコードを追加する想定なので予めゾーンを追加してください
- ACMはTerraformから証明書発行が現時点では出来ないので予め作成してください

### 事前準備

#### Terraformのインストール

Terraformにさくっと触れると、インフラ構成をコード管理するためのフレームワークです。

所謂`Infrastructure as Code`ってやつですね。

コードで構成を表現出来るのと、構築されている状態を保持出来るのでとても管理が楽です。

（と言っても私も最近触り始めたのでまだまだ手に馴染んでませんが…）

```
$ brew install terraform
```

公式サイトからダウンロードする場合は以下のリンクからインストーラーをダウンロードしてください。

[https://www.terraform.io/downloads.html:embed]

またTerraformでインフラ管理するにあたりS3とDynamoDBに状態を管理するためのバケットとテーブルを作る必要があるのでAWC CLIも合わせてインストールします。

（必須ではないのですが今回は状態管理をAWS側に寄せる前提で進めます）

ここではインストール方法や設定については割愛します（マネジメントコンソールでも対応は出来ます）

#### 状態管理用のS3バケットとDynamoDBのテーブルを作成

Terraformは `init`, `plan`, `apply` と構築までにざっくり3フェーズあるのですが `apply` で実際にAWS上にリソースが構築されます。

そのタイミングでリソースに関する情報が生成されてデフォルトだと `terraform.tfstate` というファイルにJSONでローカルに出力されますが、これをS3上で管理することで複数人で同じリソースを管理することが出来るようになります。

仮に `tf-state` というバケットを作ってみます。

```
$ aws s3 mb s3://tf-state
```

同様にDynamoDBにもテーブルを作成します。

```
$ aws dynamodb create-table \
    --table-name TerraformLockState \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1
```

`TerraformLockState` というテーブルに `LockID` というカラムをPKにして作成してます。

またRead/Writeのキャパシティユニットを`1`にすることで多少ですが料金を下げるようにしています。

デフォルト`5`で作成されると思うのですがそもそもそんなにアクセスするものではないので。

ここまでで事前準備は完了です。

### 構築編

GithubにTerraformのコードを公開してありますのでCloneしてください。

[https://github.com/tukaelu/tf-aws-serverless-blog-boilerplate:embed]
    - 最近Terraformに触り始めたばかりなのでPRとかもらえると嬉しい

```
$ git clone https://github.com/tukaelu/tf-aws-serverless-blog-boilerplate.git
$ cd tf-aws-serverless-blog-boilerplate
```

Terraformは可変になる値を変数としてコード外に定義して置けるのですがある程度汎用的に使えるように変数化してあります。

`terraform.tfvars.sample` というファイルがベースのファイルになっているのでコピーして `terraform.tfvars` を作成します。

```
$ cp terraform.tfvars{.sample,}
```

作成した `terraform.tfvars` を編集しましょう。

```
apex_domain = "hoge.com"
site_domain = "blog.hoge.com"

access_key = "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
secret_key = "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"

acm_certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
```

| 項目                | 説明                                                            |
|---------------------|-----------------------------------------------------------------|
| apex_domain         | Route53で管理しているゾーン名を設定                             |
| site_domain         | 公開するドメインを設定（`apex_domain`配下にAレコードとして追加）|
| access_key          | AWSアカウントのアクセスキーを設定                               |
| secret_key          | AWSアカウントのシークレットキーを設定                           |
| acm_certificate_arn | 発行済のACMのARNを指定                                          |

ACMで証明書を作成する際は`us-east-1`で作成してください。うっかり`ap-northeast-1`で作成するとCFに設定することが出来ません。。

設定後に `init` します。

```
$ terraform init

Initializing the backend...
bucket
  The name of the S3 bucket

  Enter a value: tf-state.hoge.com

key
  The path to the state file inside the bucket

  Enter a value: blog.hoge.com/terraform.tfstate
  :
Terraform has been successfully initialized!
  :
```

`terraform.tfstate` を保存するバケットとキーを聞かれるので先程作成したバケット名とキー（バケット配下のファイルのパス）を入力してください。

無事に成功すると `Terraform has been successfully initialized!` というメッセージが表示されるはずです。

続けて `plan` を実行します。

```
$ terraform plan

Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.aws_route53_zone.primary: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:
  :
```

`plan` は所謂DryRunでどの様に構築されるか確認できます。省略しますが長々と表示されているのがリソースに関する情報です。

それでは `apply` を実行して実際に構築してみましょう。

```
$ terraform apply
 :
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
```

`5 added`となっていて以下のリソースが追加されています。

- aws_cloudfront_distribution.distribution
- aws_cloudfront_origin_access_identity.origin_access_identity
- aws_route53_record.a
- aws_s3_bucket.s3_bucket
- aws_s3_bucket_policy.s3_bucket_policy

CFのディストリビューションが作成されているか一覧画面を開いてみます。

[f:id:tukaelu:20180404064923p:plain]

作成されていますね！

S3にコンテンツがないのでアクセスしても何も表示されないですが、S3にコンテンツを配置してInvalidationしてキャッシュを作り直すとコンテンツがCFから返却されるようになります。

サイトジェネレーター等で生成してS3にデプロイする流れが出来れば完璧ですね！

（そこは省略...）

