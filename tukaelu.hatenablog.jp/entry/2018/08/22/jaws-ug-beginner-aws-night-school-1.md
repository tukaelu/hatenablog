---
Title: JAWS-UG 初心者支部#13「AWS Night school」に参加してきた
Category:
- aws
- jawsug
Date: 2018-08-22T23:59:59+09:00
URL: https://tukaelu.hatenablog.jp/entry/2018/08/22/jaws-ug-beginner-aws-night-school-1
EditURL: https://blog.hatena.ne.jp/tukaelu/tukaelu.hatenablog.jp/atom/entry/10257846132613556786
---

[f:id:tukaelu:20180822233612p:plain]

今日はJAWS-UG 初心者支部#13にお邪魔してきました。

<!-- more -->

[https://jawsug-bgnr.connpass.com/event/94301/:embed:cite]

twitterのフォロワーさんが応募しているのがTLに流れてて急いでエントリーしました！

二度目のAmazon新社屋です。とにかくきれいでここで働きたい…！！

超絶ざっくりですがポイントだけまとめました。ほんとざっくりで申し訳ない。。

## AWS Night school 第一回（AWSJ 亀田さん）

まずはAWSJ 亀田さんのお話。

今回は第一回目ということで「AWS Night school」で今後どんなことをやっていくかをお話しいただきました。

- AWSの基本コンセプト
- 各種サービス（EC2、EBS、S3、VPC、etc...）

とAWS初心者向けの内容を計三回でがっちりフォローしてくれるそうです。これは期待大！

今回はAWS（Amazon）のコンセプト、EC2、EBSについてお話しいただきました。

### Amazonのコンセプト

よくあるあの話です。

![](https://s3-us-west-2.amazonaws.com/amazon.job-cms-website.paperclip.prod/global_images/29/images/Cycle.jpg?1531982346)

最近、個人的にもこの図をよく見ることがあったのですが、本当に理にかなっているなと思います。

もとはECのコンセプトだと思うのですが、AWSにも通じてますよね。

### AWSについて

- 100を超えるサービス
    - building block
    - サービスを組み合わせて構築していく
- グローバルインフラストラクチャ
    - 各リージョンの話
    - 東京リージョン
        - 4つのAZ
- AZのコンセプト
    - DCレベルでSPOFをなくす
    - AZ間の通信は2ms
- EC2とEBS
    - EC2
        - 課金単位について（最初1分、以降は秒）
        - EC2の起動方法について
        - インスタンタイプの種類について
        - ユーザーデータ
        - 3つのコンポーネント
            - ホスト
                - ホストが壊れても別のH/Wで起動する（瞬断）
            - ストレージ（EBS）
            - EC2 Management Services
    - EBS
        - EBSとインスタンスストアの違い
        - ホストとはネットワーク経由
        - EBSはブロックストレージ
        - AZ内で自動的にレプリケートする
        - 作成可能なタイプ
            - マグネティック
            - 汎用SSH
            - プロビジョンドIOPS
        - スナップショット
            - 差分でスナップショットを取るので正確な費用見積もりが難しい
- インスタンスメタデータ
    - クレデンシャルのような情報を特定のURLを介して提供できる
- インスタンスのライフサイクル

次回はS3、RDSを予定

## Auroraの凄さを振り返る（クラスメソッド 大栗さん）

<script async class="speakerdeck-embed" data-id="ff989898b2c242b4bd2b0d0b7c318cd2" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

大栗さんからはAuroraの魅力あれこれをお話しいただきました。

ボリューミーでしたが気になる内容が盛り沢山でした。アーキテクチャの話を噛み砕いてして頂けるのはありがたい。


LTもあったのですが、macbookの電池切れでまとめられず。。

資料が公開されたら反映したいと思います！

（手抜きですまん…）
