---
title: Linux のスケジューリング調査と超低優先度プロセスの実装
id: linux-scheduling-low-prio-process
tags:
  - Linux Kernel
date: 2011-01-25 15:30:00
---

恩師[田浦健次朗](https://www.eidos.ic.i.u-tokyo.ac.jp/~tau/)先生の講義、「[オペレーティングシステム](https://www.eidos.ic.i.u-tokyo.ac.jp/~tau/lecture/operating_systems/)」で試験を受ける代わりにカーネルハックする課題をやりました。

目覚ましい成果とは言えない結果でしたが、調査や実装の記録を書いた発表スライドを残しておきます。

[講義資料として上がってるPDF](https://www.eidos.ic.i.u-tokyo.ac.jp/~tau/lecture/operating_systems/2010/nakatani.pdf) （なぜかはてブがいっぱいついてる）

上記のはいつ研究室のサーバから消されてもおかしくないので、ここでも配信しておきます。

[元PDF](/pdf/2011/01-25-linux-scheduling-low-prio-process.pdf)
