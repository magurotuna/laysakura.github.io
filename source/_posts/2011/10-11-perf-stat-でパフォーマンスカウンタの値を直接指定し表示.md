---
title: perf stat でパフォーマンスカウンタの値を直接指定し表示
id: perf-stat-performance-counters
tags:
  - High Performance Computing
  - Linux Kernel
  - Linuxオペレーション
date: 2011-10-11 13:13:30
---

## perfについて少し
最近のLinuxでは，perfという便利なパフォーマンス解析ツールが使えます．
aptやyumなどのパッケージ管理ツールで入れられると思いますが(Debian系だとlinux-tools*って名前のパッケージのはず)，詳しい使い方はここでは解説しません．

さて，このperfには色々な使い方がありますが，中でもperf statはプログラム実行中に起こるハードウェア的なイベントを取得できるものです．
使用例はこんな感じ．

<!-- more -->

<!-- toc -->

```sh
$ perf stat ./a.out 100000

 Performance counter stats for './a.out 100000':

       2.799156  task-clock-msecs         #      0.777 CPUs 
              0  context-switches         #      0.000 M/sec
              0  CPU-migrations           #      0.000 M/sec
            136  page-faults              #      0.049 M/sec
        3067818  cycles                   #   1095.980 M/sec
        1599297  instructions             #      0.521 IPC  
         522095  cache-references         #    186.519 M/sec
           6703  cache-misses             #      2.395 M/sec

    0.003604058  seconds time elapsed
```

取得できるイベントの一覧は，perf listで表示できます．

```sh
$ perf list

List of pre-defined events (to be used in -e):

  cpu-cycles OR cycles                       [Hardware event]
  instructions                               [Hardware event]
  cache-references                           [Hardware event]
  cache-misses                               [Hardware event]
  (中略)

  rNNN                                       [raw hardware event descriptor]
```

例えば，L1データキャッシュのload回数とCPUのクロック数を見たい時には，

```sh
$ perf stat -e L1-dcache-loads -e cpu-clock ./a.out 100000

 Performance counter stats for './a.out 100000':

         979394  L1-dcache-loads          #      0.000 M/sec
       2.755833  cpu-clock-msecs         

    0.003518612  seconds time elapsed
```

という風にします．

## パフォーマンスカウンタ
L1-dcache-loadsなどのイベントが実際に見ているのは， <span style="font-weight:bold;">パフォーマンスカウンタ(ハードウェアカウンタ)</span> というものです．
最近のCPUには，ハードウェアのイベント回数を記録するためのレジスタ(パフォーマンスモニタレジスタ)が備わっています．
このレジスタの値 = パフォーマンスカウンタを見るのが，perf statというツールだと言えます．

お使いのCPUが取得できるパフォーマンスカウンタの種類は，CPUベンダの発行しているCPUファミリーごとのマニュアルを見ると分かります．
例として，

- Intelの(Nehalemを含む)各種アーキテクチャのCPUのパフォーマンスカウンタは，<br /><a href="http://www.intel.com/content/dam/doc/manual/64-ia-32-architectures-software-developer-vol-3b-part-2-manual.pdf">Intel&#174; 64 and IA-32 Architectures Software Developer’s Manual Volume 3B: System Programming Guide, Part 2</a><br />のAppendix Aに載っています
- AMDのFamily 10hに属するCPUのPerformanceCounterは，<br /><a href="http://support.amd.com/us/Processor_TechDocs/31116.pdf">BIOS and Kernel Developer’s Guide (BKDG) For AMD Family 10h Processors</a><br />の "3.14 Performance Counter Events" に載っています．

## そのイベントは何を見ている?
perf statのL1-dcache-loadsなどのイベントは，名前が直感的で，どのCPUを使っていても「このイベント指定しとけばL1データキャッシュアクセス見れるな」と分かって便利です．
しかし，研究用途などで使う際には，直接見たいパフォーマンスカウンタを，上述したCPUのマニュアルと照らし合わせながら見たいようなこともあります．
或いは，perf listの一覧では提供されていないけど，CPUのマニュアルには書いてあるハードウェアイベントを取得したいこともあるでしょう．

## 見たいハードウェアイベントを生で指定する
このような要望があるので，perf statは数値で直接見たいハードウェアイベントを選択できるようなオプションを備えています．
perf listでも表示される，
"rNNN  [raw hardware event descriptor]"
というやつです．

CPUのマニュアルに記載されている， "UnitMask(Umask)" の値と "EventSelect(EventCode)" の値を使って，次のように指定します．

```sh
$ perf stat -e r<UnitMask(16進数)><EventSelect(16進数)> ./a.out 100000
```

例えば，AMDのFamily 10hに属するCPUの "Data Cache Refills from the Northbridge" を "Modified" 属性以外のラインについて見たいなら，
<a href="http://support.amd.com/us/Processor_TechDocs/31116.pdf">BIOS and Kernel Developer’s Guide (BKDG) For AMD Family 10h Processors</a> の445ページを参考にして，

```sh
$ perf stat -e rf43 ./a.out 100000

 Performance counter stats for './a.out 100000':

           2886  raw 0xf43                #      0.000 M/sec

    0.003496994  seconds time elapsed
```

とします．
先頭の 'f' がUnitMaskで，その後の '43' がEventSelectです．

"Data Cache Misses" も一緒に見たいなら，444ページも参考にして，

```sh
$ perf stat -e rf43 -e r41 ./a.out 100000

 Performance counter stats for './a.out 100000':

           2728  raw 0xf43                #      0.000 M/sec
           7624  raw 0x41                 #      0.000 M/sec

    0.003468212  seconds time elapsed
```

ですね．

## 測定の基本
これでハードウェアカウンタを直接読むことができるようになりました．
しかし，「何を読むか」，「どう解釈するか」が分からなければ意味がありません．

AMDの発行している， <a href="http://developer.amd.com/Assets/intro_to_ca_v3_final.pdf">Basic Performance Measurements for AMD Athlon 64, AMD Opteron and AMD Phenom Processors</a> は，それを理解するのにとても役立ちます．
AMDの一部のプロセッサを対象とした計測手法が詳しく書かれていますが，基本的な考え方は他のプロセッサにも応用できると思います．是非ご一読を．

## perf statの出力を分かりやすく
これはおまけです．
ハードウェアイベントを生で指定したら，perf statの出力が分かりづらくなりますね．
"2886  raw 0xf43"
とか言われても，CPUマニュアルとにらめっこしないと意味が分からないわけです．

そこで，出力を分かりやすくするための簡単なPythonスクリプトを作りましたので，良かったら改良しつつお使いください．
`events' という辞書をリストにしたデータ構造を適当に変更してください．

適当な名前で保存して，

```sh
$ chmod +x hoge.py
$ ./hoge.py ./a.out 100000
   1563561   Retired Instructions
    987532   Data Cache Accesses
      6178   Data Cache Refills from L2
      7056   Data Cache Refills from System (Northbridge)
```

と使います．

```python
#!/usr/bin/env python                                                                                                                                                                                                                        
 
# This script is written referring to                                                                                                                                                                                                        
#   Basic Performance Measurements for AMD Athlon 64,                                                                                                                                                                                        
#   AMD Opteron and AMD Phenom Processors                                                                                                                                                                                                    
#   http://developer.amd.com/Assets/intro_to_ca_v3_final.pdf                                                                                                                                                                                 
 
 
import sys
import re
import subprocess
 
# Modify this list as you like.                                                                                                                                                                                                              
# The format of each element is:                                                                                                                                                                                                             
#   select: `NNN' for "perf stat -e rNNN"                                                                                                                                                                                                    
#   name:   Event name used in output                                                                                                                                                                                                        
events = [
    {"select": "c0", "name": "Retired Instructions"},
    {"select": "40", "name": "Data Cache Accesses"},
    {"select": "1e42", "name": "Data Cache Refills from L2"},
    {"select": "1e43", "name": "Data Cache Refills from System (Northbridge)"}
    ]
 
def event_list():
    ret = ""
    for event in events:
        ret += "-e r" + event["select"] + "   "
    return ret

def store_count_to_events(fd_perf_output):
    # Sample perf-stat output line
    #        1565650  raw 0xc0                 #      0.000 M/sec
    pat = re.compile("^ *([0-9]+).*")

    lines = fd_perf_output.readlines()
    for event in events:
        search_str = "raw 0x" + event["select"]
        event_line = [line for line in lines if line.find(search_str) != -1]
        mat = pat.match("".join(event_line))
        event["count"] = int(mat.group(1))

def print_events():
    for event in events:
        print "%10s   %s" % (str(event["count"]), event["name"])

def main():
    command = "perf stat " + event_list() + " ".join(sys.argv[1:])
    p = subprocess.Popen(command, shell=True,
                         stderr=subprocess.PIPE)
    fd_perf_output = p.stderr
    store_count_to_events(fd_perf_output)
    print_events()

if __name__ == "__main__":
    main()
```
