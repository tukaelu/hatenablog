---
Title: graftedなgitコミットとは何者か・・・
Category:
- git
Date: 2018-04-10T23:49:19+09:00
URL: https://tukaelu.hatenablog.jp/entry/2018/04/10/git-grafted-commit
EditURL: https://blog.hatena.ne.jp/tukaelu/tukaelu.hatenablog.jp/atom/entry/17391345971633745972
---

前回の記事にちょっと出てきたのですが、gitのコミットログを見た時にハッシュ値の横に`grafted`というものを初めて見たので調べてみました。

[f:id:tukaelu:20180409224423p:plain]

<!-- more -->

## ここに至る経緯

[https://tukaelu.hatenablog.jp/entry/2018/04/07/solve-unknown-type-name-UnicodeString:embed]

ざっくりまとめると

* Homebrewで1つ前のバージョンのものをインストールしたかった
* `brew switch`が出来なかったので`git checkout`でFormulaを古いバージョンに戻そうとした
* 該当ファイルの`git log`を見たらコミット履歴が1つしかなくて戻せなかった
* そのコミットのハッシュ値の隣に`(grafted)`の表記があった

という感じ。

## ググってみた

率直に`git grafted`をキーワードでググって幾つか記事を見てみます。

[https://stackoverflow.com/questions/27296188/what-exactly-is-a-grafted-commit-in-a-shallow-clone:embed:cite]

タイトルの「shallow clone した際の "grafted" なコミットとは一体何ですか？」のコレ感がパない＼(^o^)／

このStackOverflowの質問の要約は

* gitで`--depth`オプションを使ってshallow cloneすると`grafted`マークが付く
* ググっても納得行く情報が見つからなかった
* `git grafts`とは違いそうだけど同じ意味をするもの？
* コミット履歴を省略しているフラグに過ぎないのか？それとももっと特別な意味がある？

という感じですかね。

ここでわかったのは

* `git clone --depth`すると`grafted`マークが付く
* その他にも`grafts`という概念がgitにはある

ということです。

StackOverflowの回答でも触れられていましたけど、上記の記事の質問に以下のページへのリンクがありました。

[https://git.wiki.kernel.org/index.php/GraftPoint:title]

超絶ざっくりまとめると

* 異なる開発ラインを結合することが出来る
* 別のSCMからインポートした場合など別リポジトリとの履歴と結合するのに便利
* Git 1.6.5で追加された`git replace`でこれが出来る

ということですかね。なんか間違っていそうですが。。

## 試してみる

### git clone --depth

近々試そうとしている`deployer`を使わせてもらって試してみます。

[https://github.com/deployphp/deployer:embed:cite]

普通に`git clone`してみます。

```zsh
$ git clone git@github.com:deployphp/deployer.git
Cloning into 'deployer'...
remote: Counting objects: 8810, done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 8810 (delta 0), reused 0 (delta 0), pack-reused 8805
Receiving objects: 100% (8810/8810), 1.85 MiB | 578.00 KiB/s, done.
Resolving deltas: 100% (5649/5649), done.

$ cd deployer
$ git log --graph

* commit 006e0b16f3a92025759fc1da30481000e00c0320 (HEAD -> master, origin/master, origin/HEAD)
| Author: Martin Supiot <martin@webaaz.com>
| Date:   Tue Apr 10 09:49:32 2018 +0200
|
|     fix typo (#1585)
|
* commit 35027de18ba2eadd6205336961910f0f400e557c
| Author: Keith Bremner <kmbremner@gmail.com>
| Date:   Sat Mar 17 17:39:37 2018 +0000
|
|     Update shared.php (#1571)
|
|     fix typos
|
:
```

ついでに`.git`ディレクトリ配下を確認してみます。
```zsh
$ ls .git
HEAD        description index       logs        packed-refs
config      hooks       info        objects     refs
```

ふむ。

では次は`git clone --depth=1`をしてみます。

```zsh
$ git clone --depth=1 git@github.com:deployphp/deployer.git
Cloning into 'deployer'...
remote: Counting objects: 218, done.
remote: Compressing objects: 100% (199/199), done.
remote: Total 218 (delta 36), reused 50 (delta 4), pack-reused 0
Receiving objects: 100% (218/218), 119.99 KiB | 8.57 MiB/s, done.
Resolving deltas: 100% (36/36), done.

$ git log --graph

* commit 006e0b16f3a92025759fc1da30481000e00c0320 (grafted, HEAD -> master, origin/master, origin/HEAD)
  Author: Martin Supiot <martin@webaaz.com>
  Date:   Tue Apr 10 09:49:32 2018 +0200

      fix typo (#1585)
```

`--depth=1`とオプションを指定したのでログにはコミットが1つだけとなり`HEAD`に`grafted`がつきました。

そして普通に`clone`した場合と比較してオブジェクト数が1/40まで減っています！！

先程と同様に`.git`配下も確認してみます。

```zsh
$ ls .git
HEAD        description index       logs        packed-refs shallow
config      hooks       info        objects     refs
```

直下に`shallow`というファイルが増えましたね。

中身を確認してみます。

```zsh
$ cat .git/shallow
006e0b16f3a92025759fc1da30481000e00c0320
```

`grafted`なポジションのコミットのハッシュ値が記録されているようです。

ローカルのHomebrewのコアリポジトリ`homebrew-core`を確認してみたら同じようになっていました。

[f:id:tukaelu:20180410002444p:plain]

あまり意識して`--depth`オプションをつけることがないですが、
大規模なリポジトリを扱う際（かつ過去を振り返らない場合）は有効なオプションですね。

### git replace --graft

正直、今回のことがなければ`git replace`を使うことはなかったかなと。。

初めて見たのでマニュアルを確認して先程と同様に`deployer`リポジトリを使って叩いてみます。

[https://git-scm.com/docs/git-replace:title]

```zsh
$ git log --graph

* commit 006e0b16f3a92025759fc1da30481000e00c0320 (HEAD -> master, origin/master, origin/HEAD)
| Author: Martin Supiot <martin@webaaz.com>
| Date:   Tue Apr 10 09:49:32 2018 +0200
|
|     fix typo (#1585)
|
* commit 35027de18ba2eadd6205336961910f0f400e557c
| Author: Keith Bremner <kmbremner@gmail.com>
| Date:   Sat Mar 17 17:39:37 2018 +0000
|
|     Update shared.php (#1571)
|
|     fix typos
|
:
```

コマンドが`git replace --graft {commit}`となるので`HEAD`を指定してみます。

```zsh
$ git replace --graft HEAD
$ git log

* commit 006e0b16f3a92025759fc1da30481000e00c0320 (HEAD -> master, replaced, origin/master, origin/HEAD)
  Author: Martin Supiot <martin@webaaz.com>
  Date:   Tue Apr 10 09:49:32 2018 +0200

      fix typo (#1585)
```

今度は`grafted`ではなく`replaced`とマークされましたね。

マニュアルに`Adds a replace reference in refs/replace/ namespace.`と記載があったので確認してみます。

```zsh
$ ls .git
HEAD        description index       logs        packed-refs
config      hooks       info        objects     refs

$ ls .git/refs/replace/
006e0b16f3a92025759fc1da30481000e00c0320
```

`git clone --depth`の時と違って`.git/shallow`ではなく`.git/refs/replace/{commit}`が作成されていました。

## まとめ

正直ちょっとよくわからずｗ

`grafted`のマークは`clone`する時点で対象のオブジェクトが絞られているのでリポジトリ容量はだいぶ軽くなるので、
別リポジトリを参照のためにネストする場合は最新のコミットだけ知っておけば事足りるので活用出来る気がしますが`replaced`の使いみちがピンと来ず。。

業務でgitを使っている間に`grafted`や`replaced`なリポジトリを作ることはなさそうだなーというのが正直なところ。

最近ポッドキャストの`soussune`を一ヶ月遅れで聴いていてその中でgitの話があったのですが、内部構造とかもっと詳しくなるとこの辺ピンとくるのかな。

[https://soussune.com/episode/38:embed:cite]

入門Git買って読むかー(´•ω•`)

[asin:4798023809:detail]

またわかったら別記事として書こうと思います。眠いので寝るー(ヽ´ω`)

## 追記

寝ぼけててまとめの文章が超絶おかしいけど、`git clone --depth`自体はわりと普通にやる行為なんですよね。

ただその際に`grafted`とマークが付くことを初めて知ったのとそこがイマイチしっくり来ていません。故にわからず。。

`graft`を直訳すると`接ぎ木`という意味なので別リポジトリとジョイントする際にそちら側の履歴を知っておく必要はないので`grafted`になるのかなと思うのですが、
その辺のGitの運用フローがピンときていないので`clone --depth`と`replace --graft`がふわっとしている感じなのかな。

もう少し調べてわかったら纏めたいと思います。
