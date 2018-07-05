---
Title: 'PHPインストール時のerror: unknown type name ''UnicodeString''を解消する'
Category:
- macOS
- homebrew
Date: 2018-04-07T05:15:52+09:00
URL: https://tukaelu.hatenablog.jp/entry/2018/04/07/solve-unknown-type-name-UnicodeString
EditURL: https://blog.hatena.ne.jp/tukaelu/tukaelu.hatenablog.jp/atom/entry/17391345971632412598
---

先日、macOSをSierraからHigh Sierraにバージョンアップするタイミングで一度OSをまっさらにしてからやったのですが、その際にPHPがインストール出来なったので対処方法をまとめます。

<!-- more -->

## 環境
- macOS High Sierra 10.13.4
- homebrew 1.5.14
- phpenv v0.0.4-dev

## やったことと発生したエラー

HDD初期化→High Sierraインストール後にAnsibleでサクッと（は行かなかったけど）諸々インストールして行きました。

[https://github.com/tukaelu/setup-macos:embed]

やってることはmacの初期設定とローカルで開発するのに必要なソフト等をインストールしてます。

雰囲気でAnsible書いたのですべて思うように動かず手動で対応した部分もあり、
その際インストールした`phpenv`でPHP（ここでは`7.2.0`）をインストールした際に以下のエラーが起きました。

```zsh
$ phpenv install 7.2.0
 :
-----------------
|  BUILD ERROR  |
-----------------

Here are the last 10 lines from the log:

-----------------------------------------
                        ^
/var/tmp/php-build/source/7.2.0/ext/intl/intl_convertcpp.cpp:59:40: error: unknown type name 'UnicodeString'; did you mean 'icu_61::UnicodeString'?
zend_string* intl_charFromString(const UnicodeString &from, UErrorCode *status)
                                       ^~~~~~~~~~~~~
                                       icu_61::UnicodeString
/usr/local/Cellar/icu4c/61.1/include/unicode/unistr.h:286:20: note: 'icu_61::UnicodeString' declared here
class U_COMMON_API UnicodeString : public Replaceable
                   ^
22 warnings and 4 errors generated.
make: *** [ext/intl/intl_convertcpp.lo] Error 1
-----------------------------------------

The full Log is available at '/tmp/php-build.7.2.0.20180401082731.log'.
[Warn]: Aborting build.
```

`icu_61`というのが`icu4c`というUnicodeを扱う系のライブラリのようで、
バージョン等を確認して2018/4/1時点の最新版である`stable 61.1`がインストールされているようです。

```zsh
$ brew info icu4c
icu4c: stable 61.1 (bottled), HEAD [keg-only]
C/C++ and Java libraries for Unicode and globalization
http://site.icu-project.org/
 :
```

会社のmacが`brew update`前だったので調べたところ`stable 60.2`のバージョンでは正常に動いてました。

## 60.2が欲しい、、
こうなると当然動くことが確認できている`60.2`が欲しくなるのでここを参考にやってみます。

[https://qiita.com/honeniq/items/778cc08d2db78e6774d8:embed]


ただし以下が出来ないのは把握しているのでスキップします。

- `brew tap homebrew/versions` からの `brew search`
- `brew info`でローカル上にある旧バージョンの確認
    - OS入れ直したてなのであるわけがない

なのでFormulaファイルをGitで巻き戻すことにチャレンジします。

参考にした記事には`/usr/local/Library/Formula`とあるのですが、
バージョンに依るのか私の環境は`/usr/local/Homebrew/Library/Taps/homebrew/homebrew-core/Formula`にありました。

```
ls -l /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core/Formula/icu4c.rb
-rw-r--r--  1 tukaelu  admin  1225  4  1 20:13 /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core/Formula/icu4c.rb
```

`git checkout`で過去のFormulaのハッシュを指定して戻せばええんやろ！と思い`git log`で確認します。

```
git log icu4c.rb
commit dc57a79d6c422c2052df8b33a1782a43cc1cfd53 (grafted)
Author: BrewTestBot <brew-test-bot@googlegroups.com>
Date:   Sat Mar 31 04:28:55 2018 +0000

    dscanner: update 0.5.0 bottle.
```

アレ、コミットが1つしかない、、。

コミットハッシュにある`(grafted)`ってなんだ、、。

どういうわけかログを見ることが出来なかった（出来るんだろうけど…）ので`formulae.brew.sh`を参照します。

[http://formulae.brew.sh/formula/icu4c:embed]

ページの下の方にある`Recent formula history`で`60.2`のリンクをクリックするとgithub上のコミットを見れます。

[https://github.com/Homebrew/homebrew-core/commit/86ff03fbfcb44a6bdd3b0662854922a8185db910#diff-d376a2c16d287fd56cabf43f749d165b:embed:cite]

これをチェックアウトしてみます。

```bash
$ cd /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core/Formula/
$ git checkout 86ff03fbfcb44a6bdd3b0662854922a8185db910
fatal: reference is not a tree: 86ff03fbfcb44a6bdd3b0662854922a8185db910
```

ダメでした。。

わからんのでgithub上のコードをコピペして`icu4c.rb`を書き換えて再インストールした後にPHPのインストールをしてみます。

```
$ brew remove icu4c
$ brew install icu4c
$ phpenv install 7.2.0
 :
[Preparing]: /var/tmp/php-build/source/7.2.0
[Compiling]: /var/tmp/php-build/source/7.2.0
[xdebug]: Installing version 2.6.0
[Downloading]: http://xdebug.org/files/xdebug-2.6.0.tgz
[xdebug]: Compiling xdebug in /var/tmp/php-build/source/xdebug-2.6.0
[xdebug]: Installing xdebug configuration in /Users/nsymtks/.anyenv/envs/phpenv/versions/7.2.0/etc/conf.d/xdebug.ini
[xdebug]: Cleaning up.
[Info]: Enabling Opcache...
[Info]: Done
[Info]: The Log File is not empty, but the Build did not fail. Maybe just warnings got logged. You can review the log in /tmp/php-build.7.2.0.20180401103604.log
[Success]: Built 7.2.0 successfully.
```

今度は問題なく成功しました！

どうも`61.1`との相性が悪いようですね。他に影響を受けているものがあるのであろうか。。

過去バーションがインストールされて残っている場合は`brew switch`で戻してあげれば問題なさそうでした。
