---
title: Pythonで色つきログを - rainbow_logging_handler をPyPIにリリースしました
id: python-rainbow-logging-handler
tags:
  - Python
date: 2013-12-05 16:52:32
---

<img src="/img/2013/12-05-python-rainbow-logging-handler.png" alt="RainbowLoggingHandler" width="auto" height="auto">

[rainbow_logging_handler](https://github.com/laysakura/rainbow_logging_handler)をPyPIパッケージとしてリリースしました．
上のスクショのようにカラフルにPythonでログが表示できます．

<!-- more -->

<!-- toc -->

## 売りポイント

### ☆簡単に使える
デフォルトの使い方でも綺麗に色付けされます．

### ☆自由度の高い色カスタマイズ
下記のように，色は(変えたければ)細かく指定することができます．

### ☆ログ内容と色付けが分離されている
標準モジュールの`logging`でフォーマットを指定し，その色を`rainbow_logging_handler.RainbowLoggingHandler`で調整(あるいはデフォルト色を利用)という風に使います．

### ☆ログのカラム(時刻，ファイル名，メッセージなど)毎に異なる色が付けられる
対抗馬と思われる[logutils.colorize.ColorizingStreamHandler](http://pythonhosted.org/logutils/colorize.html)はログ1行ごとにしか無理．

### ☆どこでも使える
Python 2.6, 2.7, 3.2, 3.3 でテストしてます．
Linux, Windowsでは色も確認済み．おそらくMac OSとかBSDとかでも動きます．
更に更に，Public Domainです．

## インストール
`pip`とか`easy_install`で入ります．

```sh
$ pip install rainbow_logging_handler
$ # または
$ easy_install rainbow_logging_handler
```

## 使い方

### 基本の使い方
```python
# -*- coding: utf-8 -*-
import sys
import logging
from rainbow_logging_handler import RainbowLoggingHandler

def main_func():
    # `logging` モジュールを使うための準備
    logger = logging.getLogger('test_logging')
    logger.setLevel(logging.DEBUG)

    # `RainbowLoggingHandler` を使う準備
    handler = RainbowLoggingHandler(sys.stderr)
    logger.addHandler(handler)

    # 多彩なログレベルで出力
    logger.debug("デバッグ")
    logger.info("インフォ")
    logger.warn("警告")
    logger.error("エラー")
    logger.critical("深刻なエラー")
    try:
        raise RuntimeError("例外も色つきます")
    except Exception as e:
        logger.exception(e)

if __name__ == '__main__':
    main_func()
```

### 色とログフォーマットのカスタマイズ
```python
# -*- coding: utf-8 -*-
import sys
import logging
from rainbow_logging_handler import RainbowLoggingHandler

def main_func():
    # `logging` モジュールを使うための準備
    logger = logging.getLogger('test_logging')
    logger.setLevel(logging.DEBUG)
    ## フォーマットのカスタマイズ
    formatter = logging.Formatter('%(pathname)s [%(module)s] - %(funcName)s:L%(lineno)d : %(message)s')

    # `RainbowLoggingHandler` を使う準備
    handler = RainbowLoggingHandler(
        sys.stderr,
        ## 色のカスタマイズ．
        ## `pathname`, `module`, ... など，`logging.Formatter()` で使用したカラムの色を調整
        color_pathname=('black', 'red'  , True), color_module=('yellow', None, False),
        color_funcName=('blue' , 'white', True), color_lineno=('green' , None, False),
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    # カスタムしたフォーマット，色で出力
    logger.debug("デバッグ")

if __name__ == '__main__':
    main_func()
```

## 開発の経緯
Pythonで色付けられそうなロガーを頑張って探した結果，一番よさげなのが
http://opensourcehacker.com/2013/03/14/ultima-python-logger-somewhere-over-the-rainbow/
でした．

でもPyPIパッケージじゃないしライセンス的にもあんまり嬉しくなかったので，
作者の[@moo9000](https://twitter.com/moo9000)さんに「パッケージにしてよ」と言ったら「自分でやれ」って言われて作りました・・・
<blockquote class="twitter-tweet" data-conversation="none" lang="ja"><p><a href="https://twitter.com/laysakura">@laysakura</a> Please haunt me if you need any help with it. Tips: <a href="http://t.co/nQ76IfUmK6">http://t.co/nQ76IfUmK6</a></p>&mdash; Mikko Ohtamaa (@moo9000) <a href="https://twitter.com/moo9000/statuses/407786775382200320">2013, 12月 3</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

パッケージ化の過程で色のカスタマイズとか小回りをよくした結果が今の`rainbow_logging_handler`です．

## 終わりに，そしてお詫び
ログはカラフルな方が見る気が起きるような気がします．
皆様の開発の助けになれば幸いです．

バグ報告やpullreqは[Github:rainbow_logging_handler](https://github.com/laysakura/rainbow_logging_handler)までお願いします．


この記事は[Python Advent Calendar 2013](http://www.adventar.org/calendars/166)の記事として書きました．

元々は「ユニットテスト, カバレッジ計測, ドキュメント生成なんかの開発ベストプラクティス書きます」と言っていたのですが，
それより喜ぶ人の多そうなカラフルなログの話にしちゃいました・・・

ベストプラクティス云々はまたの機会に!
