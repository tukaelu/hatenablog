---
Title: Linux Mint on ThinkPad X1 Carbonにmackerel-agentをインストールしたついでにプラグインも作ってみた
Category:
- mackerel
Date: 2019-04-22T21:56:53+09:00
URL: https://blog.tukae.lu/entry/2019/04/22/impl-mackerel-plugin-for-ubuntu-on-thinkpad-x1-carbon
EditURL: https://blog.hatena.ne.jp/tukaelu/tukaelu.hatenablog.jp/atom/entry/17680117127071926238
---

お久しぶりです。

4月に入って有給消化をしており、今までずっと出来ずにいた家事とか通院とか事務手続き的なことを消化するマンをしてます。

それはいいとして本題へ、タイトルの通り雑な記録です。

<!-- more -->

## 動機

僕は6-7年ほどプライベートも仕事もmacbookユーザーなのですが、最近のメインは昨年末に購入したThinkPad X1 Carbon(Gen 6)にLinux Mintをインストールして使っています。

特にハードな使い方をしているわけではないのでリソースモニタリングは必須ではないけど、リアルタイムなリソース状況把握にConklyというシステムモニターを導入してます。

[https://github.com/brndnmtthws/conky:embed:cite]

こんな感じ。

[f:id:tukaelu:20190422214929p:plain]

デスクトップ上にレイヤーとして表示されるタイプのやつです。

正直Conklyだけで十分ではあるのですが、後から見直せたらいいなというノリでmackerel-agentをインストールしてみました。

監視するわけではないのでアラートは不要ですが、気付きとしてあったらいいのはファイルシステムとバッテリー容量くらいですかね（CPUとかメモリは使っていれば体感できそうだし…）

ファイルシステムはmackerel-agentの標準メトリックとして取得されていますがバッテリーは収集出来ないのでこちらを収集するプラグインを実装してみました。


## 作ったもの

[https://github.com/tukaelu/mackerel-plugin-thinkpad-x1-carbon-ubuntu:embed:cite]


`mkr`コマンドがインストールされていれば以下でインストール出来ます。

```
$ sudo mkr plugin install tukaelu/mackerel-plugin-thinkpad-x1-carbon-ubuntu
```

お断りを入れてしまうと手元にあるThinkPad X1 Carbon & Linux Mintに合わせて実装しているので他のマシンでのメトリック取得は保証していません（追々そこは修正する予定）


### こんな感じ（上の4つのグラフ）

[f:id:tukaelu:20190422172434p:plain]

上の4つが今回Goで実装したプラグインで下3つはその前にシェルで実装したいたやつになります（削除予定）

プラグインが取得しているのはざっくり以下。

- バッテリー関連
  - キャパシティ(%)
  - 放電回数
  - 容量
    - 規格容量
    - フル充電容量
    - 残量
- CPU温度

## どうやっているか

デバイス周りの情報は `/sys/devices/` 配下にファイルが展開されており、ファイルから各種センサーが取得した値を参照できます。

ただそちらより分類分けされて `/sys/devices/` にシンボリックリンクされている `/sys/class/` 配下を参照するほうがわかりやすい場合もあるのでそちらを見るようにしています（[sysfs](https://ja.wikipedia.org/wiki/Sysfs)参照）


### 電源周り

`/sys/class/power_supply/` 配下に電源毎のディレクトリ（実際はシンボリックリンク）が作成されています。

```
$ ls -ltr /sys/class/power_supply                                                                        合計 0
lrwxrwxrwx 1 root root 0  4月 22 20:34 BAT0 -> ../../devices/LNXSYSTM:00/LNXSYBUS:00/PNP0A08:00/device:19/PNP0C09:00/PNP0C0A:00/power_supply/BAT0
lrwxrwxrwx 1 root root 0  4月 22 20:34 AC -> ../../devices/LNXSYSTM:00/LNXSYBUS:00/PNP0A08:00/device:19/PNP0C09:00/ACPI0003:00/power_supply/AC
```

実体は `/sys/devices/` にあって、自明ですが上記の場合 `BAT0` がバッテリーで `AC` がACアダプターになります。

```
$ ls -ltr /sys/class/power_supply/BAT0/
合計 0
-rw-r--r-- 1 root root 4096  4月 22 20:51 uevent
-r--r--r-- 1 root root 4096  4月 22 20:51 serial_number
-r--r--r-- 1 root root 4096  4月 22 20:51 technology
-r--r--r-- 1 root root 4096  4月 22 20:51 power_now
-r--r--r-- 1 root root 4096  4月 22 20:51 present
drwxr-xr-x 2 root root    0  4月 22 20:51 power
-r--r--r-- 1 root root 4096  4月 22 20:51 manufacturer
lrwxrwxrwx 1 root root    0  4月 22 20:51 device -> ../../../PNP0C0A:00
-r--r--r-- 1 root root 4096  4月 22 20:51 energy_now
-r--r--r-- 1 root root 4096  4月 22 20:51 type
-r--r--r-- 1 root root 4096  4月 22 20:51 capacity
-r--r--r-- 1 root root 4096  4月 22 20:51 cycle_count
-r--r--r-- 1 root root 4096  4月 22 20:51 voltage_now
lrwxrwxrwx 1 root root    0  4月 22 20:51 subsystem -> ../../../../../../../../../class/power_supply
-r--r--r-- 1 root root 4096  4月 22 20:51 status
-rw-r--r-- 1 root root 4096  4月 22 20:51 alarm
-r--r--r-- 1 root root 4096  4月 22 20:51 model_name
-r--r--r-- 1 root root 4096  4月 22 20:51 voltage_min_design
-r--r--r-- 1 root root 4096  4月 22 20:51 capacity_level
-r--r--r-- 1 root root 4096  4月 22 20:51 energy_full_design
-r--r--r-- 1 root root 4096  4月 22 20:51 energy_full
```
`serial_number` とか `power_now` とか項目ごとにファイルがあるのですが `uevent` の中に `key=value` 形式でまとまっているのでそちらを参照しています。

ちなみにBluetoothのマウスを接続した場合、（恐らく？）対応した機種であれば `/sys/class/power_supply/` 配下にシンボリックリンクが作成されて同様にマウスの電池残量をメトリックとして取得することも可能です。手元にあるロジクールのM570tではできそうでした。


### CPU周り

`/sys/devices/platform/coretemp.{X}/hwmon/hwmon{Y}/` 配下にコア毎の温度センサーが検知した情報を取得できます。

`{X}` はCPU番号のようでCPUが1つ搭載されている場合は `{X}` が `0` のディレクトリが1つだけ存在するはずです。

`{Y}` は僕の環境では `3` だけがあったのですが意味がちょっとわかりませんでした。。

配下は以下のようになってます（ちなみに `/sys/class/hwmon/hwmon{Y}/` も同様です）

```
$ ls -l /sys/devices/platform/coretemp.0/hwmon/hwmon3
合計 0
lrwxrwxrwx 1 root root    0  4月 22 21:04 device -> ../../../coretemp.0
-r--r--r-- 1 root root 4096  4月 22 21:04 name
drwxr-xr-x 2 root root    0  4月 22 21:04 power
lrwxrwxrwx 1 root root    0  4月 22 21:04 subsystem -> ../../../../../class/hwmon
-r--r--r-- 1 root root 4096  4月 22 21:04 temp1_crit
-r--r--r-- 1 root root 4096  4月 22 21:04 temp1_crit_alarm
-r--r--r-- 1 root root 4096  4月 22 21:04 temp1_input
-r--r--r-- 1 root root 4096  4月 22 21:04 temp1_label
-r--r--r-- 1 root root 4096  4月 22 21:04 temp1_max
-r--r--r-- 1 root root 4096  4月 22 21:04 temp2_crit
-r--r--r-- 1 root root 4096  4月 22 21:04 temp2_crit_alarm
-r--r--r-- 1 root root 4096  4月 22 21:04 temp2_input
-r--r--r-- 1 root root 4096  4月 22 21:04 temp2_label
-r--r--r-- 1 root root 4096  4月 22 21:04 temp2_max
-r--r--r-- 1 root root 4096  4月 22 21:04 temp3_crit
-r--r--r-- 1 root root 4096  4月 22 21:04 temp3_crit_alarm
-r--r--r-- 1 root root 4096  4月 22 21:04 temp3_input
-r--r--r-- 1 root root 4096  4月 22 21:04 temp3_label
-r--r--r-- 1 root root 4096  4月 22 21:04 temp3_max
-r--r--r-- 1 root root 4096  4月 22 21:04 temp4_crit
-r--r--r-- 1 root root 4096  4月 22 21:04 temp4_crit_alarm
-r--r--r-- 1 root root 4096  4月 22 21:04 temp4_input
-r--r--r-- 1 root root 4096  4月 22 21:04 temp4_label
-r--r--r-- 1 root root 4096  4月 22 21:04 temp4_max
-r--r--r-- 1 root root 4096  4月 22 21:04 temp5_crit
-r--r--r-- 1 root root 4096  4月 22 21:04 temp5_crit_alarm
-r--r--r-- 1 root root 4096  4月 22 21:04 temp5_input
-r--r--r-- 1 root root 4096  4月 22 21:04 temp5_label
-r--r--r-- 1 root root 4096  4月 22 21:04 temp5_max
-rw-r--r-- 1 root root 4096  4月 22 21:04 uevent
```

`temp{Z}_input` に温度が記録されていて4コアCPUの場合は `{Z}` は `1-5` の5つになっていて、 `temp1_input` がCPUの温度、 `temp[2-5]_input` がコア毎の温度のようです。


## まとめ

mackerelプラグインを作ってみるのにはちょうどいい学習材料でした。

また[sysfs](https://ja.wikipedia.org/wiki/Sysfs)を知らなかったのですが、ゆるいもくもくから新たな発見があるのはいい傾向なので引き続きやっていきです。

あとはコードが冗長なのでもうちょっとスッキリさせたいのと、別機種での動作を考慮していないので、そのへんも今後対応していこうと思ってます！
