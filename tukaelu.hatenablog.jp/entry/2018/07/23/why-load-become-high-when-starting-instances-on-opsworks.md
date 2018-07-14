---
Title: OpsWorksでEC2インスタンスを起動すると同一スタック上のCPU負荷が高騰するのはなぜか
Category:
- aws
- opsworks
Date: 2018-07-12T23:26:00+09:00
URL: https://tukaelu.hatenablog.jp/entry/2018/07/23/why-load-become-high-when-starting-instances-on-opsworks
EditURL: https://blog.hatena.ne.jp/tukaelu/tukaelu.hatenablog.jp/atom/entry/10257846132599226957
---

OpsWorksで管理しているインフラを運用していて、EC2インスタンスをスケールアウトした際に瞬間的に周囲のインスタンスのCPU使用率が高騰するのが気になり調べてみました。

<!--more-->

## そもそもOpsWorksとは…

[公式サイト](https://aws.amazon.com/jp/opsworks/)を見ましょう！ww

- [AWS OpsWorks for Chef Automate](https://aws.amazon.com/jp/opsworks/chefautomate/)
- [AWS OpsWorks for Puppet Enterprise](https://aws.amazon.com/jp/opsworks/puppetenterprise/)
- [AWS OpsWorks スタック](https://aws.amazon.com/jp/opsworks/stacks/)

今回の話は一番最後の`AWS OpsWorks スタック`についてなのですが、もしかしたら他の2つにも該当するかもしれません（未確認）

## 気になる点

OpsWorksスタックは`Stack > Layer > Instance`という構成となっていて、インスタンスには以下の3つのスケーリングタイプが存在しています。

|タイプ|特徴|
|:---|:---|
|24/7|常時稼働向けで手動でインスタンス起動する|
|Time（時間ベース）|スケジューリングした時間設定に応じて起動・停止する|
|Load（負荷ベース）|予め定義したしきい値や条件に応じて起動・停止する|

この何れにも当てはまるのですが、インスタンスを起動させると同じレイヤー配下の既に起動しているインスタンスの負荷が高くなることに気づきました。

#### CPU使用率

[f:id:tukaelu:20180708145012j:plain]

#### fork回数

[f:id:tukaelu:20180708145254j:plain]

#### 割り込み回数

[f:id:tukaelu:20180708145359j:plain]


予め負荷上昇が見込まれる事前にインスタンスを起動する場合については特に問題ないのですが、突発的なアクセスで負荷上昇している状況下の対応としてインスタンスを追加してスケールアウトする場合に瞬間的とはいえ負荷が高くなるのは少し気持ち悪いなと思って調べてみました。

### OpsWorksエージェント

OpsWorksはレイヤー配下のインスタンスにOpsWorksエージェントがインストールされています。

```
$ ps axu | grep opswork[s]
aws      15330  0.0  1.4 178252 59004 ?        Sl   Jun13   4:51 opsworks-agent: master 15330
aws      15333  0.0  3.1 449604 129092 ?       Sl   Jun13  33:55 opsworks-agent: keep_alive of master 15330
aws      15360  0.0  2.1 407448 87872 ?        Sl   Jun13  13:17 opsworks-agent: statistics of master 15330
aws      15364  0.0  3.5 394992 142180 ?       Sl   Jun13  15:30 opsworks-agent: process_command of master 15330
```

インスタンスが起動すると上記のようにエージェントプロセスが4つ起動しているのですが、上記から察するにそれぞれ以下のような役割を持っているのだと思います。

|プロセス|役割|
|:---|:---|
|master|エージェントの管理プロセス|
|keep_alive|KeepAliveレポートを30秒おきにサービス側に通知|
|statistics|CPUやメモリの使用率をサービス側に通知|
|process_command|サービス側からのデプロイ等のコマンドを受け付けて処理|

各プロセスのログが`/var/log/aws/opsworks`配下に保持されているのですが、例えば`opsworks-agent.process_command.log`には`process_command`のログが出力されています。

ちょっと中を見るとこんな感じです。

```
 :
[2018-07-07 09:03:17]  INFO [opsworks-agent(15364)]: process_command: Polling for command to process
[2018-07-07 09:03:48]  INFO [opsworks-agent(15364)]: process_command: Polling for command to process
[2018-07-07 09:03:49]  INFO [opsworks-agent(15364)]: Decoded RNA for command xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx: configure sen
t at 2018-07-07 00:03:22 UTC
[2018-07-07 09:03:50]  INFO [opsworks-agent(15364)]: process_command: Processing command
[2018-07-07 09:03:51]  INFO [opsworks-agent(15364)]: Chef: Deleting old chef search items
[2018-07-07 09:03:51]  INFO [opsworks-agent(15364)]: Chef: Generating chef search items
[2018-07-07 09:03:51]  INFO [opsworks-agent(15364)]: Chef: No data bags defined.
[2018-07-07 09:03:51]  INFO [opsworks-agent(15364)]: Chef: Creating search nodes
[2018-07-07 09:05:25]  INFO [opsworks-agent(15364)]: Chef: Finished running chef solo with exitcode 0. (94.076461029 sec)
[2018-07-07 09:06:49]  INFO [opsworks-agent(15364)]: Chef: Finished running chef solo with exitcode 0. (83.373945245 sec)
[2018-07-07 09:06:50]  INFO [opsworks-agent(15364)]: process_command: [success] Command "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" f
or "configure" sent at 2018-07-07 00:03:22 UTC processed with exitcode 0. (83.373945245 sec)
[2018-07-07 09:06:51]  INFO [opsworks-agent(15364)]: process_command: Polling for command to process
 :
```

ログの大半が30秒毎に出力される`Polling for command to process`かと思いますが、デプロイなど何かしらのコマンドを受け付けた場合は`Generating...`や`Creating...`のようなログが出ていますね。


### OpsWorksエージェントを調べてみる

どのように調査すべきか迷ったのですが今回はインスタンスを追加する前に各プロセスに`strace`をかけて追加インスタンスの起動完了して落ちつくまでの約10分間のシステムコールと生成される子プロセスの個数について調べてみました。

```
$ sudo strace -tt -f -p 15330 -o `hostname`.out
Process 15330 attached with 2 threads
Process 15330 detached
Process 15365 detached
```

`master`プロセスは平常時に`strace`をかけていると特にforkされるような挙動を見せないので、インスタンス追加で影響を受けていることがわかります。

ただ具体的に何をしているかまではエージェントのソースまで追っかけていないので憶測ですが、`/var/log/aws/opsworks/opsworks-agent.log`を見ているとChefを実行する際のログ出力をしているように見えるのでその影響なのかもしれません。

```
$ sudo strace -tt -f -p 15333 -o `hostname`.out
Process 15333 attached with 2 threads
Process 10771 attached
 :
```

`keep_alive`プロセスは起動するまでの間に200行程度`Process xxxxx attached`が出力されました。

なので単純に200回は`fork`がコールされているのではないかと思われます。

`keep_alive`なので頻度が上がる意図が想定できなかったのですが、よくよく見ていると30秒毎に10回forkがコールされているようで`200 / (10 * 2) = 10分間`と一致したのでこちらはインスタンス起動の影響は特に受けずに完全に独立していることがわかります。

```
$ sudo strace -tt -f -p 15360 -o `hostname`.out
Process 15360 attached with 2 threads
Process 10845 attached
 :
```

`statistics`プロセスは30回程度出力されました。

こちらも観察していると1分間に3回forkされているようなので計測した10分間と一致していて`keep_alive`と同じように独立してるようです。

```
$ sudo strace -tt -f -p 15364 -o `hostname`.out
Process 15364 attached with 2 threads
Process 10781 attached
 :
```

`process_command`プロセスが最も多く750回を超える出力がありました。

こちらも平常時は1分間に4回程度forkするように見えるのですが、明らかにインスタンス追加の影響を受けていることがわかります。

ここで少しミソなのがOpsWorksスタックのライフサイクルイベントです。

[AWS OpsWorks Stacks のライフサイクルイベント](https://docs.aws.amazon.com/ja_jp/opsworks/latest/userguide/workingcookbook-events.html)

公式ドキュメントにも記載があるのですが、`Configure`が以下の条件でスタックのすべてのインスタンスで実行される仕様となっています。

- インスタンスがオンライン状態に移行したか、オンライン状態から移行した。
- Elastic IP アドレスをインスタンスに関連付けたか、インスタンスから Elastic IP アドレスの関連付けを解除した。
- Elastic Load Balancing ロードバランサーを Layer にアタッチしたか、Layer からデタッチした。

今回の場合は1つ目に該当すると思うのですが、インスタンスが起動する度にイベントが発生して更にカスタムレシピが設定されている場合はそちらも実行されることになるので、その影響を受けていると思われます。

CPU負荷の高い処理で長く続くようなレシピを設定してしまうと引っ張られてしまいそうですね。

ちなみに私が管理しているスタックにはそこまで重めの処理はないのですが瞬間的にCPU使用率が40〜50%程度上がります。

## まとめ

ざっくりした調査しかしていないのですが、インスタンスを追加すると同一スタック上のインスタンスの負荷が高くなることが公式ドキュメントと実際のプロセスの挙動とリソースの負荷状況からふわっとですがおわかり頂けたかと思います。

とくに負荷ベースのインスタンス起動を使用している場合は特にこの辺りは注意が必要そうですね。

どんな処理が割り込んでいるのかとかシステムコールの種類や回数等はまた改めて（気が向いた時に）調べて見たいと思います。
