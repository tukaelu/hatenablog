---
Title: 気づいたら1年間でサービスメトリックが6→238に増えていた話
Category:
- mackerel
Date: 2018-12-20T00:00:00+09:00
URL: https://blog.tukae.lu/entry/2018/12/20/mackerel-service-metrics-increased-from-6-to-238
EditURL: https://blog.hatena.ne.jp/tukaelu/tukaelu.hatenablog.jp/atom/entry/10257846132687170787
---

この記事は[Mackerelアドベントカレンダー2018](https://qiita.com/advent-calendar/2018/mackerel)の20日目のエントリーです。

本当は「Mackerelを使って働き方改革してみた」というネタを書きたかったのですが、12月になった途端に想定外なデスマと戦うことになり改革が出来てないので私個人の振り返り的なお話をしようと思います。ネタの方は改めて。。


ちなみにタイトルはアレですが、Mackerelをよく知らない人に「こんなことが出来る」という参考になれば幸いです。


<!-- more -->


## はじめに…

私は1年程前に入社した現職でひょんな事からインフラエンジニアとなり、あるサービスのインフラの管理をしています。

入社当時からMackerelは導入されていたのですが活用されてはおらず、サービスメトリックは外形監視を設定すると追加されるHTTPレスポンスタイムのメトリックが6つ程だったと記憶しています。

それを見て「いやいやサービスメトリックめっちゃ便利なのになぜ使わん！」と言うことで、あれこれサービスメトリックに集めたりした記録です。


## 作ったもの① - Googleアナリティクス連携

私が担当するサービスは `ある決まった日のある時間帯にだけイベントが発生する` ため、基本はその数分〜数十分間だけアクセスが集中するのですが、LINEやTwitterなどのSNSから突発的にアクセスが流入したりします。

どんなサービスでもそうですがアクセスが増えると負荷は高まりますし、想定外の不具合が起きたりしますよね。

当初は振り返る際にGoogleアナリティクスとMackerelのメトリクスを横断的に見ていたのですがタイムスケールが一致しないのは非常につらいと思い、AWS Lambda(node.js + Serverless)でGoogleアナリティクスのアクティブユーザー数を連携させました。


[https://github.com/tukaelu/mackerel-integrations/blob/master/handlers/analytics.ts:embed:cite]


だいぶ粗雑な作りですがチャチャッとコードを書くだけでこのように振り返りがとても楽になりましたｗ

<figure class="figure-image figure-image-fotolife" title="GoogleAnalytics with Mackerel">[f:id:tukaelu:20181219180908j:plain]<figcaption>GoogleアナリティクスとCPU使用率</figcaption></figure>

この頃から同僚の間で `サービスメトリック芸の人` と呼ばれてたとか呼ばれてないとか。。

アクセスに対して負荷がどの程度かかるかが見えることで必要なキャパシティを見積もることができ、リザーブドインスタンスの購入目安を決めることが出来て前年と比べてコストを40%ほど削減出来ることになりました。

あとは予期せずアクセスが増えた際にアラートを通知するようにして気づきを得ることが出来るようになったのも良かったです。


## 作ったもの② - DynamoDB連携

Googleアナリティクスが見れることでだいぶ楽になったと思ったのですが次に欲しくなったのはDynamoDBとの連携です。

担当するシステムではDynamoDBのテーブルを複数使用して稼働しています。

正直、私は現職までDynamoDBを全く触ったことがなかったのですが、実際にサービスを開発しているエンジニア達もDynamoDBのリソースがどのように消費されているかキチンと理解しているかと言うと全くわかっていない状況でした。。

エンジニアは面倒くさがりな生き物、 `CloudWatchとMackerelを横断的に見るのはつらたん` と言う事で再び実装しました。

（コードは割愛…）

[f:id:tukaelu:20181219183016j:plain]


CloudWatchよりも他の要素との比較が容易になり、不要なプロビジョンドキャパシティユニットも明確になって、これをきっかけにDynamoDBのコストも40%ほど削減することが出来ました。


ちなみにこれをリリースして数カ月後にDynamoDBとのインテグレーションが正式リリースされ個人的にはかなりワクワクしました。


[https://mackerel.io/ja/blog/entry/weekly/20180925:embed:cite]


がしかし、1テーブル = 1ホストの課金の壁が思ったより高く、引き続きオレオレインテグレーションで運用しております(*^^*)


## 作ったもの③ - AWS料金連携

コスト削減が上手く行くと、継続的に維持が出来ているか確認したくなります。

- 使用料金の推移
- リザーブドインスタンスのカバレッジ
- 現在〜3ヶ月前との料金推移の比較

芸人の血が騒ぎますよねー🤣

[https://github.com/tukaelu/mackerel-integrations/blob/master/handlers/aws/billing.ts:embed:cite]

サンプルコードは料金の推移だけですが、こんな感じでみることが出来ます。

<figure class="figure-image figure-image-fotolife" title="使用料金の推移">[f:id:tukaelu:20181219185934j:plain]<figcaption>使用料金の推移</figcaption></figure>

<figure class="figure-image figure-image-fotolife" title="リザーブドインスタンスのカバレッジ">[f:id:tukaelu:20181219190244j:plain]<figcaption>リザーブドインスタンスのカバレッジ</figcaption></figure>

<figure class="figure-image figure-image-fotolife" title="料金推移の過去比較">[f:id:tukaelu:20181219190628j:plain]<figcaption>料金推移の過去比較</figcaption></figure>

最後のグラフは式グラフを使って出してますが、かゆいところに手が届く柔軟さが本当に素晴らしい！！

リザーブドインスタンスのカバレッジをメトリックとしたことで `n%を切ったらアラートを上げる` といった運用もできるため更新漏れ等にも気づけるようになりました。


[https://twitter.com/tukaelu/status/1047797868780650496:embed]


## そして…

ここまでやって気づいたらサービスメトリックの数が200を超えてました！


[f:id:tukaelu:20181219191340p:plain]


[https://twitter.com/tukaelu/status/1055283921464619008:embed]


200を超過したことで1ホストぶん課金は増えましたが、それ以上に恩恵を受けたインパクトが大きいです。

数値であれば何でも集めることが出来るのがいいところだと思うのでもっと活用していきたいなと思いますし、ぜひまだ活用されていない方もサービスメトリックに色々集めてみてはいかがでしょうか！？
