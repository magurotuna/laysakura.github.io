---
title: 真・SSEを使って8flops/clockを実現する
id: sse-8flops-clock
tags:
  - High Performance Computing
date: 2012-01-06 12:43:46
---

<a href="http://atnd.org/events/21910">カーネル／VM Advent Calendar</a> の34日目として書きます．
記事の内容自体はこのイベントと関係ありませんので，どなたでもお楽しみ下さいませ．

今回は <a href="http://d.hatena.ne.jp/laysakura/20111113/1321155826">自分の前回の記事</a> で見つけた誤りを訂正しつつ，
Intelの一部CPUがサポートするSSE(Streaming SIMD Extensions)命令により1clockで8個の単精度浮動小数点演算を行なう方法を紹介します．

なるべく前回の記事とは独立した記述を心がけます．

<!-- more -->

<!-- toc -->

## 関連した記事
@takeoka さんが，カーネル／VM Advent Calendarの24日目の記事として，<a href="http://ameblo.jp/takeoka/entry-11116148541.html">AVXの効き目について</a>書かれています．
SSEが128bitレジスタを使って4つの単精度浮動小数をパックするのに対し，AVXは256bitで8個パックできます．
とはいえ，まだまだAVX命令はないけどSSE命令を積んでいるCPUも現役ですので，この記事も多少お役に立てるかと思います．

## 使用したCPU
この記事の内容は，

- Intel Xeon CPU E7540 @ 2.00GHz
- Intel Core2 Duo CPU P9400 @ 2.40GHz

で調査・実験しています．

それ以外の環境では，記述が当てはまらないことも考えられます．
もしお気づきの点がございましたらコメント頂けたら嬉しいです．

## 乗算と加算を並列に行なって 8flops/clock を実現
SSEでは，128-bitのxmmレジスタを使って計算を進めます．
例えば，

```c
   float a[4], b[4];
   b[0] += a[0];
   b[1] += a[1];
   b[2] += a[2];
   b[3] += a[3];
```

という計算であれば，

1. a[0]をxmm0番レジスタの0-31bit目に， ... , a[3]をxmm0番レジスタの96-127bit目に格納
2. b[0]をxmm1番レジスタの0-31bit目に， ... , b[3]をxmm1番レジスタの96-127bit目に格納
3. xmm1 += xmm0

とします．この xmm1 += xmm0 が1clockでできてしまうのがSSEの強みです．

さて，SSEでは掛算と足し算は同時にできてしまいます．
つまり，

```asm
   xmm1 += xmm0; 4つの単精度浮動小数の足し算
   xmm3 *= xmm2; 4つの単精度浮動小数の掛算
```

が1clockでできるので，8flops/clockが実現できます．
(※ flops: FLoating-point OPerationS の意で使っています．
似ていますがFLOPSと大文字で書くときは，FLoating-point Operations Per Secondの意ですので，
ごっちゃにしないように注意してください．)

SSEの実際の命令を使って掛算と足し算を表現すると，

```asm
   addps %xmm0, %xmm1
   mulps %xmm2, %xmm3
```

と書けます．

(この記事でのアセンブリの表記は，GCCのインラインアセンブラと同じです．
"Op Reg1, Reg2" だと，結果は Reg2 に格納されると考えてください．)

## 8flops/clock を出すための制約
<a href="http://sc.tamu.edu/systems/eos/nehalem.pdf">The Architecture of the Nehalem Processor and Nehalem-EP SMP Platforms</a> (PDF)
の14ページ目に，Nehalemアーキテクチャにおけるアウトオブオーダ実行の概要が図示されています．
これによると，SSE関係の命令については，

- ロード/ストア
- 足し算
- 掛算
- シャッフル

の4種類の命令は1clockで行えるようです．
実際のアプリケーションでSSE命令を使うとなると，xmmレジスタとメモリのやりとりでロード/ストア命令は使いますし，
xmmレジスタ内部の浮動小数の並び順を入れ替えるシャッフル命令がほしくなるときもあります．

SSEで高速な処理を記述するには，まず1clockで同時に行える命令を明確にしましょう．

同時に実行できる演算の種類が限られているのは演算器の都合でしょうが，レジスタの都合に起因する制約もあります．
例えば，

```asm
   addps %xmm0, %xmm8
   mulps %xmm1, %xmm8
```

のようなコードで，同じレジスタに同時に値を書き込むことはできません．

その一方で，

```asm
   addps %xmm8, %xmm0
   mulps %xmm8, %xmm1
```

のようなコードで，同じレジスタから同時に値を読み出すことはできます．

この辺りの詳しい条件は筆者は把握していません．

## ロード，シャッフル命令を使いつつも8flops/clock近く出るコード
(この記事の終わりに，コンパイルしてすぐに実行できるフルのコードも掲載します)

実際に，

- ロード
- 足し算
- 掛算
- シャッフル

を1clock(近く)で行なうコードを書いてみます．

ざっくり読む上での注意点も述べておきます．

- movupsは(今回の使い方では)メモリからxmmレジスタに値をロードする命令
- shufpsはxmmレジスタの値をシャッフルする命令
- print_GFLOPS, print_throughput は計測結果を出力する関数
- rdtsc() は，CPUのRDTSC命令を用いて，(ほぼ)CPUクロックベースで時間を計測するための関数
- mulps SRC, DEST で，ソースにメモリ上のデータを使う場合，そのデータは16バイト境界にアラインされている必要がある

```c
void
sse_mulps_addps_movups_shufps_fromL1()
{
  printf("-- sse_mulps_addps_movups_shufps_fromL1 --\n");
 
  unsigned long long clk0, clk1, cycles;
 
  zero_all_xmm();
 
  const int instructions_per_loop = 16;
  const int flops_per_instruction = 4;
  const double flops = flops_per_instruction * instructions_per_loop * LOOP;
  int i;
 
  float __attribute__ ((aligned(16))) a[4] = {0.0, 0.0, 0.0, 0.0};
  float __attribute__ ((aligned(16))) b[4] = {0.0, 0.0, 0.0, 0.0};
  float __attribute__ ((aligned(16))) c[4] = {0.0, 0.0, 0.0, 0.0};
  float __attribute__ ((aligned(16))) d[4] = {0.0, 0.0, 0.0, 0.0};
 
  clk0 = rdtsc();
  for (i = 0; i < LOOP; ++i) {
    asm volatile
      (
       "movups %0, %%xmm4\n\t"
       "shufps $0, %%xmm4, %%xmm4\n\t"
       "mulps %%xmm4, %%xmm0\n\t"
       "addps %%xmm0, %%xmm8\n\t"
 
       "movups %1, %%xmm5\n\t"
       "shufps $0, %%xmm5, %%xmm5\n\t"
       "mulps %%xmm5, %%xmm1\n\t"
       "addps %%xmm1, %%xmm9\n\t"
 
       "movups %2, %%xmm6\n\t"
       "shufps $0, %%xmm6, %%xmm6\n\t"
       "mulps %%xmm6, %%xmm2\n\t"
       "addps %%xmm2, %%xmm10\n\t"
 
       "movups %3, %%xmm7\n\t"
       "shufps $0, %%xmm7, %%xmm7\n\t"
       "mulps %%xmm7, %%xmm3\n\t"
       "addps %%xmm3, %%xmm11\n\t"
 
       "movups %0, %%xmm4\n\t"
       "shufps $0, %%xmm4, %%xmm4\n\t"
       "mulps %%xmm4, %%xmm0\n\t"
       "addps %%xmm0, %%xmm8\n\t"
 
       "movups %1, %%xmm5\n\t"
       "shufps $0, %%xmm5, %%xmm5\n\t"
       "mulps %%xmm5, %%xmm1\n\t"
       "addps %%xmm1, %%xmm9\n\t"
 
       "movups %2, %%xmm6\n\t"
       "shufps $0, %%xmm6, %%xmm6\n\t"
       "mulps %%xmm6, %%xmm2\n\t"
       "addps %%xmm2, %%xmm10\n\t"
 
       "movups %3, %%xmm7\n\t"
       "shufps $0, %%xmm7, %%xmm7\n\t"
       "mulps %%xmm7, %%xmm3\n\t"
       "addps %%xmm3, %%xmm11\n\t"
       :
       :"m"(a[0]), "m"(b[0]), "m"(c[0]), "m"(d[0])
       );
  }
  clk1 = rdtsc();
  cycles = clk1 - clk0;
  print_GFLOPS(flops, cycles);
  print_throughput(instructions_per_loop * LOOP, cycles);
}
```

xmmレジスタが8本しかないCPUではコンパイルエラーになりますので，xmm8 - xmm15を適宜置き換えてください．

## すぐに実験できるコードと実験方法
以下のコードをコピペし， "sse_8flops_per_clock.c" など適当なファイルを作ってください．
コンパイル方法は，

```sh
$ gcc -O3 -funroll-loops sse_8flops_per_clock.c -o sse_8flops_per_clock
```

辺りだと性能が出るはずです．
特にループ・アンローリングのための "-funroll-loops" は性能に直結します．

実行すると，8flops/clock程度出ていることが確認できるかと思います．

```sh
$ ./sse_8flops_per_clock
-- sse_mulps_addps_movss_shufps_fromL1 --
GFLOPS @ 2.00GHz:
  7.408 [flops/clock] = 14.816 [GFLOPS]  (134217728 flops in 18117460 clock = 0.009059 sec)
Throughput:
  1.852 [instructions/clock]   (33554432 instrucions in 18117460 clock)
```

```c
#include <time.h>
#include <sys/time.h>
#include <stdint.h>
#include <inttypes.h>
#include <stdio.h>
#include <stdlib.h>

#define GHz 2.00

static inline uint64_t
rdtsc()
{
  uint64_t ret;
#if defined _LP64
  asm volatile
    (
     "rdtsc\n\t"
     "mov $32, %%rdx\n\t"
     "orq %%rdx, %%rax\n\t"
     "mov %%rax, %0\n\t"
     :"=m"(ret)
     :
     :"%rax", "%rdx"
     );
#else
  asm volatile
    (
     "rdtsc\n\t"
     "mov %%eax, %0\n\t"
     "mov %%edx, %1\n\t"
     :"=m"(((uint32_t<span style="font-weight:bold;">)&ret)[0]), "=m"(((uint32_t</span>)&ret)[1])
     :
     :"%eax", "%edx"
     );
#endif
  return ret;
}

void
print_GFLOPS(double flops, uint64_t cycles)
{
  double GFLOPS = flops * GHz / cycles;
  double sec = (double)cycles * 1e-9 / GHz;
  printf("GFLOPS @ %.2fGHz:\n  %.3f [flops/clock] = %.3f [GFLOPS]  (%.0f flops in %"PRIu64" clock = %f sec)\n",
         GHz, flops / (double)cycles, GFLOPS, flops, cycles, sec);
}

void
print_throughput(uint64_t instructions, uint64_t cycles)
{
  printf("Throughput:\n  %.3f [instructions/clock]   (%"PRIu64" instrucions in %"PRIu64" clock)\n",
         (double)instructions / (double)cycles, instructions, cycles);
}

#define zero_all_xmm() \
  do { \
    asm volatile \
      ("xorps %xmm0, %xmm0\n\t" \
       "xorps %xmm1, %xmm1\n\t" \
       "xorps %xmm2, %xmm2\n\t" \
       "xorps %xmm3, %xmm3\n\t" \
       "xorps %xmm4, %xmm4\n\t" \
       "xorps %xmm5, %xmm5\n\t" \
       "xorps %xmm6, %xmm6\n\t" \
       "xorps %xmm7, %xmm7\n\t" \
       ); \
  } while (0)

#define LOOP (1 << 21)

void
sse_mulps_addps_movups_shufps_fromL1()
{
  printf("-- sse_mulps_addps_movups_shufps_fromL1 --\n");
 
  unsigned long long clk0, clk1, cycles;
 
  zero_all_xmm();
 
  const int instructions_per_loop = 16;
  const int flops_per_instruction = 4;
  const double flops = flops_per_instruction * instructions_per_loop * LOOP;
  int i;
 
  float __attribute__ ((aligned(16))) a[4] = {0.0, 0.0, 0.0, 0.0};
  float __attribute__ ((aligned(16))) b[4] = {0.0, 0.0, 0.0, 0.0};
  float __attribute__ ((aligned(16))) c[4] = {0.0, 0.0, 0.0, 0.0};
  float __attribute__ ((aligned(16))) d[4] = {0.0, 0.0, 0.0, 0.0};
 
  clk0 = rdtsc();
  for (i = 0; i < LOOP; ++i) {
    asm volatile
      (
       "movups %0, %%xmm4\n\t"
       "shufps $0, %%xmm4, %%xmm4\n\t"
       "mulps %%xmm4, %%xmm0\n\t"
       "addps %%xmm0, %%xmm8\n\t"
 
       "movups %1, %%xmm5\n\t"
       "shufps $0, %%xmm5, %%xmm5\n\t"
       "mulps %%xmm5, %%xmm1\n\t"
       "addps %%xmm1, %%xmm9\n\t"
 
       "movups %2, %%xmm6\n\t"
       "shufps $0, %%xmm6, %%xmm6\n\t"
       "mulps %%xmm6, %%xmm2\n\t"
       "addps %%xmm2, %%xmm10\n\t"
 
       "movups %3, %%xmm7\n\t"
       "shufps $0, %%xmm7, %%xmm7\n\t"
       "mulps %%xmm7, %%xmm3\n\t"
       "addps %%xmm3, %%xmm11\n\t"
 
       "movups %0, %%xmm4\n\t"
       "shufps $0, %%xmm4, %%xmm4\n\t"
       "mulps %%xmm4, %%xmm0\n\t"
       "addps %%xmm0, %%xmm8\n\t"
 
       "movups %1, %%xmm5\n\t"
       "shufps $0, %%xmm5, %%xmm5\n\t"
       "mulps %%xmm5, %%xmm1\n\t"
       "addps %%xmm1, %%xmm9\n\t"
 
       "movups %2, %%xmm6\n\t"
       "shufps $0, %%xmm6, %%xmm6\n\t"
       "mulps %%xmm6, %%xmm2\n\t"
       "addps %%xmm2, %%xmm10\n\t"
 
       "movups %3, %%xmm7\n\t"
       "shufps $0, %%xmm7, %%xmm7\n\t"
       "mulps %%xmm7, %%xmm3\n\t"
       "addps %%xmm3, %%xmm11\n\t"
       :
       :"m"(a[0]), "m"(b[0]), "m"(c[0]), "m"(d[0])
       );
  }
  clk1 = rdtsc();
  cycles = clk1 - clk0;
  print_GFLOPS(flops, cycles);
  print_throughput(instructions_per_loop * LOOP, cycles);
}

int
main()
{
  sse_mulps_addps_movups_shufps_fromL1();
  return 0;
}
```

## 前回の記事の欠陥
前回の記事では，足し算と掛算を1clockで行なうための条件を「フォワーディングが重要です(どやぁ」と宣っていたのですが，
色々コードを書いてみたり <a href="http://sc.tamu.edu/systems/eos/nehalem.pdf">The Architecture of the Nehalem Processor and Nehalem-EP SMP Platforms</a> を
ざっと見た感じ，もっとアーキテクチャに根ざした複雑な問題なようです．

## 測定環境についての注意
測定は，RDTSC命令を用いて行っています．
この命令で読み取るカウンタは，CPUの通常動作時の速度で更新されます(自分はそう理解しています)．
従って，CPUの動作周波数を動的に変化させる DVFS (Dynamic Voltage and Frequency Scaling) や TurboBoost が有効になっているマシンだと，正確な測定はできません(8flops/clockを超えるように見えたりする)．
どちらもBIOSでオフにできる機能なはずですので，正確な測定環境を求める方はご検討ください．
