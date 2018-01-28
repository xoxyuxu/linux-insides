# Interrupts and Interrupt Handling

<!---
In the following posts, we will cover interrupts and exceptions handling in the linux kernel.
--->
以下の投稿で、Linuxカーネル内の割り込みと例外処理をカバーします。

<!---
* [Interrupts and Interrupt Handling. Part 1.](interrupts-1.md) - describes interrupts and interrupt handling theory.
* [Interrupts in the Linux Kernel](interrupts-2.md) - describes stuffs related to interrupts and exceptions handling from the early stage.
* [Early interrupt handlers](interrupts-3.md) - describes early interrupt handlers.
* [Interrupt handlers](interrupts-4.md) - describes first non-early interrupt handlers.
* [Implementation of exception handlers](interrupts-5.md) - describes implementation of some exception handlers such as double fault, divide by zero etc.
* [Handling non-maskable interrupts](interrupts-6.md) - describes handling of non-maskable interrupts and remaining interrupt handlers from the architecture-specific part.
* [External hardware interrupts](interrupts-7.md) - describes early initialization of code which is related to handling external hardware interrupts.
* [Non-early initialization of the IRQs](interrupts-8.md) - describes non-early initialization of code which is related to handling external hardware interrupts.
* [Softirq, Tasklets and Workqueues](interrupts-9.md) - describes softirqs, tasklets and workqueues concepts.
* [Last part](interrupts-10.md) - this is the last part of the `Interrupts and Interrupt Handling` chapter and here we will see a real hardware driver and some interrupts related stuff.
--->

* [Interrupts and Interrupt Handling. Part 1.](interrupts-1.md) - 割り込みと割り込み処理の理論（概念）について記述します。
* [Interrupts in the Linux Kernel](interrupts-2.md) - 初期段階からの割り込みや例外処理に関することを記述します。
* [Early interrupt handlers](interrupts-3.md) - 早期の割り込みハンドラについて記述します。
* [Interrupt handlers](interrupts-4.md) - 最初の早期ではない割り込みハンドラについて記述します。
* [Implementation of exception handlers](interrupts-5.md) - 二重故障やゼロ除算などの例外ハンドラの実装について記述します。
* [Handling non-maskable interrupts](interrupts-6.md) - NMI(マスク不可割り込み)の処理と、アーキテクチャ依存部からの残りの割り込みハンドラについて記述します。
* [External hardware interrupts](interrupts-7.md) - 外部ハードウェア割り込み処理に関するコードの早期初期化に関して記述します。
* [Non-early initialization of the IRQs](interrupts-8.md) - 外部ハードウェア割り込み処理に関するコードの早期ではない初期化に関して記述します。
* [Softirq, Tasklets and Workqueues](interrupts-9.md) - softirq, tasklet, workqueueの概念について記述します。
* [Last part](interrupts-10.md) - `Interrupts and Interrupt Handling`の章の最後のパートです。本当のハードウェアドライバと、関連する割り込みを見ます。


