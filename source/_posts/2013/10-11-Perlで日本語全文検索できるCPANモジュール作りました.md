---
title: Perlで日本語全文検索できるCPANモジュール作りました
id: perl-japanese-full-text-search
tags:
  - データベース
date: 2013-10-11 16:41:06
---

Perlで全文検索を手軽にできるようにするCPANモジュール，<a href="http://search.cpan.org/perldoc?Search::Fulltext">Search::Fulltext</a> をリリースしました．
これ単品だと英語での全文検索ができるのですが，これまた拙作の <a href="http://search.cpan.org/perldoc?Search::Fulltext::Tokenizer::MeCab">Search::Fulltext::Tokenizer::MeCab</a> と組み合わせて使うと <span style="font-weight:bold;">日本語全文検索</span> ができるようになります．

ここでは日本語全文検索をするためのインストール方法と簡単な使い方などについて記述します．
尚，この記事を書いている時点での最新版は Search::Fulltext-1.02, Search::Fulltext::Tokenizer::MeCab-1.04 です．
最新の情報についてはperldocで確認するようにしてください．

<!-- more -->

<!-- toc -->

# セールスポイントまとめ
- すごくシンプルに使える．メイン部分はこれだけ．
  ```perl
      my $fts = Search::Fulltext->new({
          docs      => \@docs,
          tokenizer => "perl 'Search::Fulltext::Tokenizer::MeCab::tokenizer'",
      });
      my $results = $fts->search($query);
      is_deeply($results, [0, 2]);
  ```
- AND, OR, NOT, NEAR 検索などをサポート
- メモリをはみ出しても大丈夫．SQLiteがバックエンドなので．
- トーカナイザがPerlだけでプラガブル開発できる．

詳しくはperldocを参照してください．この記事では導入方法までを説明します．

# まずは英文全文検索
## Search::Fulltextのインストール
```sh
$ cpanm Search::Fulltext
```

## Search::Fulltextの動作チェック (または使い方の確認)
以下のファイルを "check.t" という名前で保存して，

```sh
$ prove check.t
```

でテストしてみてください．

```perl
use strict;
use warnings;
use utf8;
use Test::More;

use Search::Fulltext;
use Search::Fulltext::TestSupport;

plan tests => 1;

my $query = 'beer';
my @docs = (
    'I like beer the best',
    'Wine makes people saticefied',  # does not include beer
    'Beer makes people happy',
);

# Common usage
{
    my $fts = Search::Fulltext->new({
        docs => \@docs,
    });
    my $results = $fts->search($query);
    is_deeply($results, [0, 2]);
}
```

この例では，'beer' という単語を含む0番目と2番目の文字列がヒットしています．

# いよいよ日本語全文検索
## MeCabのインストール
MeCabのライブラリと実行ファイルをインストールします．
もしかしたらライブラリだけでいいのかもしれませんが，よく調べてません．

```sh
$ sudo apt-get install mecab libmecab2
```

## Search::Fulltext::Tokenizer::MeCab のインストール

```sh
$ cpanm Search::Fulltext::Tokenizer::MeCab
```

## Search::Fulltext::Tokenizer::MeCab の動作確認 & 使い方
以下のファイルを "check-jp.t" という名前で <SPAN STYLE="FONT-WEIGHT:BOLD;">UTF-8</SPAN> で保存して，
```sh
$ prove check-jp.t
```

でテストしてみてください．

```perl
use strict;
use warnings;
use utf8;
use Test::More;

use Search::Fulltext;
use Search::Fulltext::Tokenizer::MeCab;

plan tests => 1;

my $query = '猫';
my @docs = (
    '我輩は猫である',
    '犬も歩けば棒に当る',
    '実家でてんちゃんって猫を飼ってまして，ものすっごい可愛いんですよほんと',
);

{
    my $fts = Search::Fulltext->new({
        docs      => \@docs,
        tokenizer => "perl 'Search::Fulltext::Tokenizer::MeCab::tokenizer'",
    });
    my $results = $fts->search($query);
    is_deeply($results, [0, 2]);
}
```

動きましたか?

# 仕組み
DBD::SQLiteを通してSQLiteのFTS4という全文検索機能を使用しています．
単なるSQLiteのラッパーだと言うこともできます．

また，日本語トークンはMeCabの形態素解析で拾ってます．
MeCab形態素解析なトーカナイザの開発は，DBD::SQLiteの機能のPerl Tokenizerを使ってPerlだけで行えました．
詳しくは，'perldoc Search::Fulltext' の 'CUSTOM TOKENIZERS' のセクションを参照してください．

# おわりに
気に入っていただけたでしょうか?
バグ報告やpullreqは

- https://github.com/laysakura/Search-Fulltext
- https://github.com/laysakura/Search-Fulltext-Tokenizer-MeCab

でウェルカムです!
