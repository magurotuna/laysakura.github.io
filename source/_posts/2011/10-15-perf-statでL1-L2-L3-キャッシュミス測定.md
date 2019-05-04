---
title: 'perf statでL1,L2(,L3)キャッシュミス測定'
id: perf-stat-L1-L2-L3-cache-miss
tags:
  - High Performance Computing
  - Linux Kernel
  - Linuxオペレーション
date: 2011-10-15 13:12:42
---

## はじめに
この記事は，<a href="/2011/10/11/perf-stat-performance-counters">前回のエントリ</a>の続きものです．
まだご覧になっていない方は，前回分をお読みになってからこちらの記事を見てください．

<!-- more -->

<!-- toc -->

## 対象プロセッサ
このエントリは，AMDの次のプロセッサを対象に書かれています．

- AMD Athlon 64
- AMD Opteron
- AMD Phenom

ただし， <span style="font-weight:bold;">これらのプロセッサでは本エントリの記述がそのまま当てはまる</span> という意味であって，
<span style="font-weight:bold;">他のプロセッサでも，記憶域の構成を知った上で本エントリのような考え方を適用すればキャッシュミス測定ができます．</span>

上に挙げたプロセッサをお使いの場合でも，L3キャッシュがあるモデルかどうかで読み替えてください．
本エントリではL3キャッシュがあるとして記述している箇所があります．

## 参考文献
<a href="/2011/10/11/perf-stat-performance-counters">前回のエントリ</a>でも紹介した
<a href="http://developer.amd.com/Assets/intro_to_ca_v3_final.pdf">Basic Performance Measurements for AMD Athlon 64, AMD Opteron and AMD Phenom Processors</a>
の "4.3 Memory Accesses" を参考に本エントリを書きました．

## perf stat でキャッシュミスを正確に測定するのは難しい
perf list 一覧できるイベント名は直感的ですが，実際には何のハードウェアイベントを測定しているのかが分かりにくいというのは<a href="/2011/10/11/perf-stat-performance-counters">前回のエントリ</a>でも触れました．

試しに，イベント名を使ってキャッシュミスを計測します．
その値と， <a href="http://developer.amd.com/Assets/intro_to_ca_v3_final.pdf">Basic Performance Measurements for AMD Athlon 64, AMD Opteron and AMD Phenom Processors</a> に書かれた計算式から求めた値と比較してみて，「イベント名はアテできない・・・」ということを実感してみましょう．

測定に使ったCPUは，Quad-Core AMD Opteron. Model. 8354 です．L1データキャッシュ，L1命令キャッシュ,L2キャッシュがコアごとに1つずつ,L3キャッシュが4コアに1つあります．

測定用アプリケーション(a.out)は単純なもので，1次元の領域に対して書き込みをした後読み出すだけです(下の擬似コード参照)．

```c
size = atoi(コマンドライン引数);
int *a = malloc(size * sizeof(int));

for (i = 0; i < size; ++size)
  size[i] = 1;

sum = 0;
for (i = 0; i < size; ++size)
  sum += a[i];

printf("%d\n", sum);
```

では，イベント名を使ってキャッシュミスらしきものを計測してみましょう．

```sh
$ perf stat -e L1-dcache-load-misses -e L1-dcache-store-misses -e cache-misses -e LLC-load-misses -e LLC-store-misses ./a.out 1000000000

 Performance counter stats for './a.out 1000000000':

       77557463  L1-dcache-load-misses    #      0.000 M/sec
  <not counted>  L1-dcache-store-misses
         190127  cache-misses             #      0.000 M/sec
       86401250  LLC-load-misses          #      0.000 M/sec
  <not counted>  LLC-store-misses

    7.447300257  seconds time elapsed
```

*-store-misses が計測出来ていない時点で，イベント名が疑わしい臭いがします．

(※ 'dcache' はデータキャッシュ， 'LLC' は 'Last Level Cache' の略です)

次に， <a href="http://developer.amd.com/Assets/intro_to_ca_v3_final.pdf">Basic Performance Measurements for AMD Athlon 64, AMD Opteron and AMD Phenom Processors</a> の計算式を使った測定の結果を見てみます．
測定用のスクリプト， "perf-stat-with-events.py" はこのエントリ中に掲載します．

```sh
$ perf-stat-with-events.py ./a.out 1000000000

(出力抜粋)

      186936122   Data Cache Misses
      146084145   L2 Misses
       16306069   L3 Misses

=== Elapsed Time ===
    7.371837186  seconds time elapsed
```

先程のイベント名を使った測定の値と比較すると，どのように対応しているのかが全く分かりません．

やはり，真面目に各階層のキャッシュミスを計測しようと思うと，イベント名に頼ってもいられないというのが現状のようです．

### perf のソースを覗いて実際に測っているものを調べる
今回のエントリの本筋とは外れますので，読み飛ばして結構です．

perf のソースを見て perf stat では何を測っているのかを確認してみたいと思います．

perfのソースは，Linuxのソースツリーの中にあります．
Linuxのソースがあるディレクトリの中で，

```sh
find -name "perf_"
```

とすると，perfのソースのありかが分かります(ただし，これで列挙されるファイルで全てなのかは知りません)．

実際にハードウェアカウンタの値をとるソースは，当然プロセッサの種類に依存します．
先の測定に用いた， AMD Opteron. Model. 8354 では，
"arch/x86/kernel/cpu/perf_event_amd.c"
に， perf stat ではどのハードウェアカウンタの値をとっているのかが書いてあります．

コードについてあまり深くは触れませんが，これを見ると，

- 「AMDのプロセッサ」という風に汎用的な枠組みで定義しているので，L3キャッシュを持つプロセッサについても，L3キャッシュミスを計測するためのイベント名を提供していない
- 例えばデータキャッシュミスなどは， <a href="http://developer.amd.com/Assets/intro_to_ca_v3_final.pdf">Basic Performance Measurements for AMD Athlon 64, AMD Opteron and AMD Phenom Processors</a> で「この測り方は正確な測定としてはお勧めできません」と書いてあるやり方で計測しようとしている

といった問題点が見えてきます．

Oprofileみたいに，プロセッサの種類をもっと細かく分類しなければ，「直感的なイベント名で正しく計測」というのは難しいのでしょう．
(余談の余談ですが，そういうpatchは投げられてないんですかね?)

## キャッシュ・メインメモリの構成を正しく把握する
お使いのプロセッサで，キャッシュとメインメモリがどのような構成になっているのかを正しく知らなければ，正しいキャッシュミス測定はできません．
これはそれぞれのプロセッサのマニュアルに載っている情報のはずです．

先程の計測に使った AMD Opteron. Model. 8354 については，同様のキャッシュ・メインメモリ構成を持つプロセッサの日本語解説記事がありました:
<a href="http://pc.watch.impress.co.jp/docs/2006/1016/kaigai312.htm">後藤弘茂のWeekly海外ニュース - 大幅に強化されたAMDのクアッドコア「Barcelona」</a>

## キャッシュミスを計算する
それでは，キャッシュミスを計算します．計算するのは，

- L1データキャッシュミス
- L1命令キャッシュミス
- L2キャッシュミス
- L3キャッシュミス

です．

そのために，次のハードウェアイベントを計測しておきます．
括弧内は，

```sh
$ perf stat -e rNNN
```

の NNN に当たる， <UnitMask(16進数)><EventSelect(16進数)> を表します．

- Data Cache Accesses (40)
- Data Cache Refills from L2 (1e42)
- Data Cache Refills from System (1e43)
- Instruction Cache Fetches (c80)
- Instruction Cache Refills from L2 (c82)
- Instruction Cache Refills from System (c83)
- Requests to L2 Cache [TLB fill] (c47d)
- L2 Cache Misses [TLB fill] (c47e)
- Read Requests to L3 Cache (cf74e0)
- L3 Cache Misses (cf74e1)

### L1データキャッシュミスの計算を例に，考え方を知る
L1データキャッシュミスを直接測定するためのハードウェアイベントはありません．間接的な手法を採ります．

これはL1データキャッシュにも以外の全てのキャッシュにも言えることですが，
<span style="font-weight:bold;">キャッシュミスが起きると，自分より下のレベルのキャッシュかメインメモリからキャッシュライン分のデータを取ってきて，自分のキャッシュに載せる</span>
ということが起きます(この動作を refill と言います)．
つまり，
<span style="font-weight:bold;">自分より下のレベルのキャッシュ・メインメモリからデータを取ってきて，自分のキャッシュに載せるという一連の動作(Refill)の回数が，キャッシュミス回数</span>
となります．

従って，

```
"L1データキャッシュミス回数" = "L2からL1データキャッシュへのrefill回数" + "L3からL1データキャッシュへのrefill回数" + "メインメモリからL1データキャッシュへのrefill回数"
```

という関係が成立します．

右辺の項とハードウェアイベントの対応は，

```
"L2からL1データキャッシュへのrefill回数" = "Data Cache Refills from L2"
("L3からL1データキャッシュへのrefill回数" + "メインメモリからL1データキャッシュへのrefill回数") = "Data Cache Refills from System"
```

となっています．

以上より，

```
"L1データキャッシュミス回数" = "Data Cache Refills from L2" + "Data Cache Refills from System"
```

として計算ができます．

### その他のレベルのキャシュミスについて
L2キャッシュミスが起こる状況を考えてみます．

- L1データキャッシュミスが起きて，L2キャッシュにアクセスしたところ，L2キャッシュもミスしたので，L3キャッシュまたはメインメモリから，L1データキャッシュにキャッシュラインを取ってきた
- L1命令キャッシュミスが起きて，L2キャッシュにアクセスしたところ，L2キャッシュもミスしたので，L3キャッシュまたはメインメモリから，L1命令キャッシュにキャッシュラインを取ってきた

これらの場合の他に，L2キャッシュのTLBミスが発生した場合も勘定に入れます(よく分かってないのですが，ページテーブルのキャッシュとして，TLBとの間のL2キャッシュも使用しているのでしょうかね?．

従って，

```
"L2キャッシュミス回数" = "Data Cache Refills from System" + "Instruction Cache Refills from System" + "L2 Cache Misses [TLB fill]"
```

と計算できます．

今回対象にしているプロセッサでは，(L3キャッシュを持つものならば)L3キャッシュミスは直接計測できます．

## キャッシュに関するイベントを計測するスクリプト
データキャッシュ・命令キャッシュ・L2キャッシュ・L3キャッシュに関する測定をするスクリプトを作りました．
AMD Opteron. Model. 8354 で動作を確認しています．

```sh
$ perf-stat-with-events.py ./a.out 1000000000
=== Running Environment ===
 Performance counter stats for './mopt_ref_implementation 1000000000':

=== Counted Events ===
     6122320253                                      Retired Instructions
     2123804830                                       Data Cache Accesses
       59707845                                Data Cache Refills from L2
      127228277              Data Cache Refills from System (Northbridge)
     1630510550                                 Instruction Cache Fetches
          80385                         Instruction Cache Refills from L2
          88990       Instruction Cache Refills from System (Northbridge)
       18766878                           Requests to L2 Cache [TLB fill]
        8167131                                L2 Cache Misses [TLB fill]
       32867005                                 Read Requests to L3 Cache
       16306069                                           L3 Cache Misses

=== Calculated Events ===
     6122320253   Retired Instructions
     2123804830   Data Cache Accesses
        34.690%   Data Cache Request Rate
      186936122   Data Cache Misses
         8.802%   Data Cache Miss Ratio
     1630510550   Instruction Cache Fetches
        26.632%   Instruction cache Request Rate
         169375   Instruction Cache Misses
         0.010%   Instruction Cache Miss Ratio
      205872375   L2 Requests
         3.363%   L2 Request Rate
      146084145   L2 Misses
        70.959%   L2 Miss Ratio
       32867005   Read Requests to L3 Cache
         0.537%   L3 Request Rate
       16306069   L3 Misses
        49.612%   L3 Miss Ratio

=== Elapsed Time ===
    7.371837186  seconds time elapsed
```

といった感じの出力が得られます．

改変等ご自由にどうぞ．
(print_calculated_events 関数がアレな感じですが，基本的にやっていることは上述のキャッシュミス計算などです)

```python
#!/usr/bin/env python

# This script is written referring to
#   Basic Performance Measurements for AMD Athlon 64,
#   AMD Opteron and AMD Phenom Processors
#   http://developer.amd.com/Assets/intro_to_ca_v3_final.pdf


import sys
import re
import subprocess
import types

# Modify this list as you like.
# The format of each element is:
#   select: `NNN' for "perf stat -e rNNN"
#   name:   Event name used in output
events = [
    {"select": "c0", "name": "Retired Instructions"},
    {"select": "40", "name": "Data Cache Accesses"},
    {"select": "1e42", "name": "Data Cache Refills from L2"},
    {"select": "1e43", "name": "Data Cache Refills from System (Northbridge)"},
    {"select": "c80", "name": "Instruction Cache Fetches"},
    {"select": "c82", "name": "Instruction Cache Refills from L2"},
    {"select": "c83", "name": "Instruction Cache Refills from System (Northbridge)"},
    {"select": "c47d", "name": "Requests to L2 Cache [TLB fill]"},
    {"select": "c47e", "name": "L2 Cache Misses [TLB fill]"},
    {"select": "cf74e0", "name": "Read Requests to L3 Cache"},
    {"select": "cf74e1", "name": "L3 Cache Misses"}
    ]

def get_event_with_name(name):
    return [event for event in events if event["name"] == name][0]

def event_list():
    ret = ""
    for event in events:
        ret += "-e r" + event["select"] + "   "
    return ret

def store_count_to_events(perf_output_lines):
    # Sample perf-stat output line
    #        1565650  raw 0xc0                 #      0.000 M/sec    ( +-   1.457% )  (scaled from 75.04%)
    #  <not counted>  raw 0xc0                 #      0.000 M/sec
    count_pat = re.compile("^ <span style="font-weight:bold;">([0-9]+|<not counted>).</span>")
    variance_pat = re.compile("\+- *([0-9]+\.[0-9]+)%")

    lines = perf_output_lines
    for event in events:
        search_str = "raw 0x" + event["select"]
        event_line = "".join([line for line in lines if line.find(search_str) != -1])

        count_mat = count_pat.match(event_line)
        variance_mat = variance_pat.search(event_line)

        if count_mat is None:
            print("Unexpected event format:\n  %s"
                  % event_line)
            exit(1)
        if count_mat.group(1) != "<not counted>":
            event["count"] = int(count_mat.group(1))
            if variance_mat is not None:
                event["variance"] = float(variance_mat.group(1))
            else:
                event["variance"] = -1.0
        else:
            event["count"] = -1
            event["variance"] = -1.0

def print_running_env(perf_output_lines):
    search_str = "Performance counter stats for"
    print("".join([line for line in perf_output_lines if line.find(search_str) != -1]))

def print_elapsed_time(perf_output_lines):
    search_str = "seconds time elapsed"
    print("".join([line for line in perf_output_lines if line.find(search_str) != -1]))

def print_counted_events():
    for event in events:
        if event["count"] >= 0 and event["variance"] >= 0.0:
            print("%15s   %55s   ( +- %6.2f%%)"
                  % (str(event["count"]), event["name"], event["variance"]))
        elif event["count"] >= 0 and event["variance"] < 0.0:
            print("%15s   %55s"
                  % (str(event["count"]), event["name"]))
        else:
            print("%15s   %55s"
                  % ("<not counted>", event["name"]))

def print_calculated_events():
    def _clever_calc(rhs, terms):
        """
        @returns:
        if `terms' does not have any value less than 0:
            The result of rhs
        else:
            Value less than 0
        """
        if len([term for term in terms if term < 0]) > 0:
            return -1
        else:
            return rhs

    def _clever_print(val, ev_name):
        """
        @parameters:
        val: Value to print. Must be int or float.
             Value less than 0 means that the event is <not counted>.
        """
        is_valid_val = (val >= 0)
        if is_valid_val and type(val) == types.IntType:
            print("%15d   %s" % (val, ev_name))
        elif is_valid_val and type(val) == types.FloatType:
            print("%14.3f%%   %s" % (100.0 * val, ev_name))
        elif not is_valid_val:
            print("%15s   %s" % ("<not counted>", ev_name))
        else:
            print("Unexpected calculated value: " + str(val))

    retired_instructions = get_event_with_name("Retired Instructions")["count"]
    _clever_print(retired_instructions, "Retired Instructions")

    # Data Cache
    data_cache_accesses = get_event_with_name("Data Cache Accesses")["count"]
    _clever_print(data_cache_accesses, "Data Cache Accesses")

    data_cache_request_rate = _clever_calc(float(data_cache_accesses) / float(retired_instructions),
                                           [data_cache_accesses, retired_instructions])
    _clever_print(data_cache_request_rate, "Data Cache Request Rate")

    data_cache_refills_from_L2 = get_event_with_name("Data Cache Refills from L2")["count"]
    data_cache_refills_from_system = get_event_with_name("Data Cache Refills from System (Northbridge)")["count"]
    data_cache_misses = _clever_calc(data_cache_refills_from_L2 + data_cache_refills_from_system,
                                     [data_cache_refills_from_L2, data_cache_refills_from_system])
    _clever_print(data_cache_misses, "Data Cache Misses")

    data_cache_miss_ratio = _clever_calc(float(data_cache_misses) / float(data_cache_accesses),
                                         [data_cache_misses, data_cache_accesses])
    _clever_print(data_cache_miss_ratio, "Data Cache Miss Ratio")

    # Instruction Cache
    instruction_cache_fetches = get_event_with_name("Instruction Cache Fetches")["count"]
    _clever_print(instruction_cache_fetches, "Instruction Cache Fetches")

    instruction_cache_request_rate = _clever_calc(float(instruction_cache_fetches) / float(retired_instructions),
                                                  [instruction_cache_fetches, retired_instructions])
    _clever_print(instruction_cache_request_rate, "Instruction cache Request Rate")

    instruction_cache_refills_from_L2 = get_event_with_name("Instruction Cache Refills from L2")["count"]
    instruction_cache_refills_from_system = get_event_with_name("Instruction Cache Refills from System (Northbridge)")["count"]
    instruction_cache_misses = _clever_calc(instruction_cache_refills_from_L2 + instruction_cache_refills_from_system,
                                            [instruction_cache_refills_from_L2, instruction_cache_refills_from_system])
    _clever_print(instruction_cache_misses, "Instruction Cache Misses")

    instruction_cache_miss_ratio = _clever_calc(float(instruction_cache_misses) / float(instruction_cache_fetches),
                                                [instruction_cache_misses, instruction_cache_fetches])
    _clever_print(instruction_cache_miss_ratio, "Instruction Cache Miss Ratio")

    # L2 Cache
    L2_misses_TLB = get_event_with_name("Requests to L2 Cache [TLB fill]")["count"]
    L2_requests = _clever_calc(data_cache_misses + instruction_cache_misses + L2_misses_TLB,
                               [data_cache_misses, instruction_cache_misses, L2_misses_TLB])
    _clever_print(L2_requests, "L2 Requests")

    L2_request_rate = _clever_calc(float(L2_requests) / float(retired_instructions),
                                   [L2_requests, retired_instructions])
    _clever_print(L2_request_rate, "L2 Request Rate")

    L2_misses = _clever_calc(data_cache_refills_from_system + instruction_cache_refills_from_system + L2_misses_TLB,
                             [data_cache_refills_from_system, instruction_cache_refills_from_system, L2_misses_TLB])
    _clever_print(L2_misses, "L2 Misses")

    L2_miss_ratio = _clever_calc(float(L2_misses) / float(L2_requests),
                                 [L2_misses, L2_requests])
    _clever_print(L2_miss_ratio, "L2 Miss Ratio")

    # L3 Cache
    L3_requests = get_event_with_name("Read Requests to L3 Cache")["count"]
    _clever_print(L3_requests, "Read Requests to L3 Cache")

    L3_request_rate = _clever_calc(float(L3_requests) / float(retired_instructions),
                                   [L3_requests, retired_instructions])
    _clever_print(L3_request_rate, "L3 Request Rate")

    L3_misses = get_event_with_name("L3 Cache Misses")["count"]
    _clever_print(L3_misses, "L3 Misses")

    L3_miss_ratio = _clever_calc(float(L3_misses) / float(L3_requests),
                                 [L3_misses, L3_requests])
    _clever_print(L3_miss_ratio, "L3 Miss Ratio")


def main():
    command = "perf stat " + event_list() + " ".join(sys.argv[1:])
    p = subprocess.Popen(command, shell=True,
                         stderr=subprocess.PIPE)
    perf_output_lines = p.stderr.readlines()
    store_count_to_events(perf_output_lines)

    print("=== Running Environment ===")
    print_running_env(perf_output_lines)
    print("=== Counted Events ===")
    print_counted_events()
    print("\n=== Calculated Events ===")
    print_calculated_events()
    print("\n=== Elapsed Time ===")
    print_elapsed_time(perf_output_lines)

if __name__ == "__main__":
    main()
```

### スクリプトの実装上の細かい注意点
キャッシュミスなどの計算値は，計算に必要なハードウェアイベントが正しく測定されていなければ出せません．
更に，計算値Aを利用して計算値Bを出す場合も，計算値Aが正しい値である必要があります．

つまり，「測定値が正しく採れなかった場合，影響が連鎖する」という性質があります．
スクリプトでは，
`_clever_calc` , `_clever_print`
関数によって，この連鎖関係を把握しています．
