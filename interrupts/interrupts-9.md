Interrupts and Interrupt Handling. Part 9.
================================================================================

Introduction to deferred interrupts (Softirq, Tasklets and Workqueues)
--------------------------------------------------------------------------------

<!---
It is the nine part of the Interrupts and Interrupt Handling in the Linux kernel [chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) and in the previous [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-8.html) we saw implementation of the `init_IRQ` from that defined in the [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irqinit.c) source code file. So, we will continue to dive into the initialization stuff which is related to the external hardware interrupts in this part.
--->
[Linuxカーネルの章](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)
割り込みと割り込みハンドリングの第九部です。
[前の部](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-8.html)

`init_IRQ`の実装は、
[arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irqinit.c) ソースコードファイルに定義されているのが見えます。
それでは、引き続きこの章で外部ハードウェア割り込みに関する初期化手順に潜っていきましょう。

<!---
Interrupts may have different important characteristics and there are two among them:

* Handler of an interrupt must execute quickly;
* Sometime an interrupt handler must do a large amount of work.

As you can understand, it is almost impossible to make so that both characteristics were valid. Because of these, previously the handling of interrupts was split into two parts:

* Top half;
* Bottom half;

--->

割り込みは異なる重要な特性を持っており、その中には2つあります

* 割り込みのハンドラは、早く実行する必要があります
* 時々、割り込みハンドラは、大量の作業を行う必要があります

皆さんも理解できるように、ほとんどの場合はどちらの特性も有効に作ることは不可能です。
このため、以前は割り込みのハンドリングは2つの部分に分けることができました。

[](
In the past there was one way to defer interrupt handling in Linux kernel. And it was called: `the bottom half` of the processor, but now it is already not actual. Now this term has remained as a common noun referring to all the different ways of organizing deferred processing of an interrupt.The deferred processing of an interrupt suggests that some of the actions for an interrupt may be postponed to a later execution when the system will be less loaded. As you can suggest, an interrupt handler can do large amount of work that is impermissible as it executes in the context where interrupts are disabled. That's why processing of an interrupt can be split on two different parts. In the first part, the main handler of an interrupt does only minimal and the most important job. After this it schedules the second part and finishes its work. When the system is less busy and context of the processor allows to handle interrupts, the second part starts its work and finishes to process remaining part of a deferred interrupt.
)


以前は、Linuxカーネルで割り込み処理を遅らせる方法がありました。
そしてそれはプロセッサーの"下半分"(bottom half)と呼ばれましたが、今はすでに実際にはありません。
今では、この用語は、割り込みの遅延処理を構成するさまざまな方法をすべて指す共通の名詞として残っています。
割り込みの遅延処理は、システムの負荷が少ない場合に、割り込みの動作の一部を後で実行するように延期する可能性があることを示しています。
あなたが示唆するように、割り込みハンドラは、割り込みが無効になっているコンテキストで実行されるため、許されない大量の作業を行うことができます。
そのため、割り込みの処理を2つの異なる部分に分割することができます。
最初の部分、割り込みのメインハンドラは、最小限で重要な仕事です。その後、2番目の部分をスケジュールし、作業を終了します。
システムのビジーが少なく、プロセッサのコンテキストが割り込みを処理することができる場合、2番目の部分は作業を開始し、遅延割り込みの残りの部分の処理を終了します。


[](
There are three types of `deferred interrupts` in the Linux kernel:
)

Linuxカーネルには、`遅延割り込み`の種類は三つあります：

* `softirqs`;
* `tasklets`;
* `workqueues`;

[](
And we will see description of all of these types in this part. As I said, we saw only a little bit about this theme, so, now is time to dive deep into details about this theme.
)

そして、この部分では、これらすべての種類の説明が表示されます。
私が言ったように、私たちはこのテーマについて少ししか見ませんでした。
だから今このテーマに関する詳細に深く掘り下げる時間です。

Softirqs
----------------------------------------------------------------------------------

[](
With the advent of parallelisms in the Linux kernel, all new schemes of implementation of the bottom half handlers are built on the performance of the processor specific kernel thread that called `ksoftirqd` (will be discussed below). Each processor has its own thread that is called `ksoftirqd/n` where the `n` is the number of the processor. We can see it in the output of the `systemd-cgls` util:
)


Linuxカーネルでの並列処理の出現により、ボトムハーフハンドラの新しい実装スキームは全て、
 `ksoftirqd` と呼ばれるプロセッサ固有のカーネルスレッドの性能の上に構築されています（後述）。
各プロセッサには `ksoftirqd / n`と呼ばれる独自のスレッドがあり、` n`はプロセッサの番号です。
`systemd-cgls`ユーティリティの出力で見ることができます：

```
$ systemd-cgls -k | grep ksoft
├─   3 [ksoftirqd/0]
├─  13 [ksoftirqd/1]
├─  18 [ksoftirqd/2]
├─  23 [ksoftirqd/3]
├─  28 [ksoftirqd/4]
├─  33 [ksoftirqd/5]
├─  38 [ksoftirqd/6]
├─  43 [ksoftirqd/7]
```

[](
The `spawn_ksoftirqd` function starts this these threads. As we can see this function called as early [initcall](http://www.compsoc.man.ac.uk/~moz/kernelnewbies/documents/initcall/index.html):
)

`spawn_ksoftirqd`関数は、これらのスレッドを起動します。
early [initcall] (http://www.compsoc.man.ac.uk/~moz/kernelnewbies/documents/initcall/index.html) として、この関数が呼ばれることを見られます：

```C
early_initcall(spawn_ksoftirqd);
```

[](
Softirqs are determined statically at compile-time of the Linux kernel and the `open_softirq` function takes care of `softirq` initialization. The `open_softirq` function defined in the [kernel/softirq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c):
)

Softirqsは、Linuxカーネルのコンパイル時に静的に決定され、 `open_softirq` 関数は `softirq`の初期化を行います。
`open_softirq` 関数は、 [kernel / softirq.c] (https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c) で定義されています。


```C
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```

[](
and as we can see this function uses two parameters:
)

そして、この関数は２つのパラメータを使用していることが見られます。
[](
* the index of the `softirq_vec` array;
* a pointer to the softirq function to be executed;
)

* `softirq_vec` 配列のインデックス
* 実行すべき softirq関数へのポインタ

[](
First of all let's look on the `softirq_vec` array:
)

まずは、`softirq_vec`配列を見てみましょう：

```C
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
```

[](
it defined in the same source code file. As we can see, the `softirq_vec` array may contain `NR_SOFTIRQS` or `10` types of `softirqs` that has type `softirq_action`. First of all about its elements. In the current version of the Linux kernel there are ten softirq vectors defined; two for tasklet processing, two for networking, two for the block layer, two for timers, and one each for the scheduler and read-copy-update processing. All of these kinds are represented by the following enum:
)

同じソースコードファイルに定義されています。
`softirq_vec` 配列は、 `softirq_action` タイプを持つ `softirqs` の `NR_SOFTIRQS` または 10のタイプから成ることがわかります。
まずはその要素について。Linuxカーネルの現在のバージョンでは、10個のsoftirqヴェクタが定義されています。そのうち２つは tasklet処理用で、２つはネットワーク用、２つはブロックレイヤー用、２つはタイマ用、そして、１つのスケジューラと１つのRCU(read-copy-update)処理用です。
これらの種類はすべて次の列挙型で表されます：

```C
enum
{
        HI_SOFTIRQ=0,
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        BLOCK_IOPOLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ,
        RCU_SOFTIRQ,
        NR_SOFTIRQS
};
```
[](All names of these kinds of softirqs are represented by the following array:)

softirqのこれらの種類のすべての名前は、以下の配列によって表されます。


```C
const char * const softirq_to_name[NR_SOFTIRQS] = {
        "HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "BLOCK_IOPOLL",
        "TASKLET", "SCHED", "HRTIMER", "RCU"
};
```

[](
Or we can see it in the output of the `/proc/softirqs`:
)
もしくは、`/proc/softirqs` の出力に見ることができます：

```
~$ cat /proc/softirqs 
                    CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7       
          HI:          5          0          0          0          0          0          0          0
       TIMER:     332519     310498     289555     272913     282535     279467     282895     270979
      NET_TX:       2320          0          0          2          1          1          0          0
      NET_RX:     270221        225        338        281        311        262        430        265
       BLOCK:     134282         32         40         10         12          7          8          8
BLOCK_IOPOLL:          0          0          0          0          0          0          0          0
     TASKLET:     196835          2          3          0          0          0          0          0
       SCHED:     161852     146745     129539     126064     127998     128014     120243     117391
     HRTIMER:          0          0          0          0          0          0          0          0
         RCU:     337707     289397     251874     239796     254377     254898     267497     256624
```

[](
As we can see the `softirq_vec` array has `softirq_action` types. This is the main data structure related to the `softirq` mechanism, so all `softirqs` represented by the `softirq_action` structure. The `softirq_action` structure consists a single field only: an action pointer to the softirq function:
)

`softirq_vec`配列が `softirq_action`型を持つことがわかります。これは、`softirq`メカニズムに関連する主なデータ構造なので、すべての`softirqs` は `softirq_action' 構造体で表されます。

```C
struct softirq_action
{
         void    (*action)(struct softirq_action *);
};
```

[](
So, after this we can understand that the `open_softirq` function fills the `softirq_vec` array with the given `softirq_action`. The registered deferred interrupt (with the call of the `open_softirq` function) for it to be queued for execution, it should be activated by the call of the `raise_softirq` function. This function takes only one parameter -- a softirq index `nr`. Let's look on its implementation:
)
この後、 `open_softirq`関数は、` softirq_vec`配列を与えられた `softirq_action`で満たすことを理解できます。
実行のためにキューイングされるために登録された遅延割り込み（ `open_softirq`関数の呼び出し）は、`raise_softirq` 関数の呼び出しによってアクティブ化されるべきです。
この関数は、softirqインデックス `nr`という1つのパラメータのみをとります。
その実装を見てみましょう：

```C
void raise_softirq(unsigned int nr)
{
        unsigned long flags;

        local_irq_save(flags);
        raise_softirq_irqoff(nr);
        local_irq_restore(flags);
}
```

<!---
  Here we can see the call of the `raise_softirq_irqoff` function between the `local_irq_save` and the `local_irq_restore` macros. The `local_irq_save` defined in the [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqflags.h) header file and saves the state of the [IF](https://en.wikipedia.org/wiki/Interrupt_flag) flag of the [eflags](https://en.wikipedia.org/wiki/FLAGS_register) register and disables interrupts on the local processor. The `local_irq_restore` macro defined in the same header file and does the opposite thing: restores the `interrupt flag` and enables interrupts. We disable interrupts here because a `softirq` interrupt runs in the interrupt context and that one softirq (and no others) will be run.
--->

ここでは、 `local_irq_save`マクロと` local_irq_restore`マクロの間の `raise_softirq_irqoff`関数の呼び出しを見ることができます。

<!--- [ja] add source code --->
```C
/*
 * This function must run with irqs disabled!
 */
inline void raise_softirq_irqoff(unsigned int nr)
{
	__raise_softirq_irqoff(nr);

	/*
	 * If we're in an interrupt or softirq, we're done
	 * (this also catches softirq-disabled code). We will
	 * actually run the softirq once we return from
	 * the irq or softirq.
	 *
	 * Otherwise we wake up ksoftirqd to make sure we
	 * schedule the softirq soon.
	 */
	if (!in_interrupt())
		wakeup_softirqd();
}
```

[include/linux/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqflags.h) ヘッダファイルで定義された `local_irq_save`は、
[eflags](https://en.wikipedia.org/wiki/FLAGS_register)レジスタの
[IF](https://en.wikipedia.org/wiki/Interrupt_flag)フラグの状態を保存し、ローカルプロセッサの割り込みを無効にします。

`local_irq_restore`マクロは同じヘッダファイルで定義され、逆のことをします。`interrupt flag`を復元し、割り込みを有効にします。
`softirq`割り込みが割り込みコンテキストで実行され、1つのsoftirqのみ（他のものは実行されない）が実行されるので、ここでは割り込みを無効にします。

<!---
The `raise_softirq_irqoff` function marks the softirq as deffered by setting the bit corresponding to the given index `nr` in the `softirq` bit mask (`__softirq_pending`) of the local processor. It does it with the help of the:
--->

`raise_softirq_irqoff`関数は、ローカルプロセッサの` softirq`ビットマスク（ `__softirq_pending`）内の与えられたインデックス` nr`に対応するビットを設定することによって、softirqを遅延（deffered）としてマークします。
マークするためには、以下のマクロを用いて行います：

```C
__raise_softirq_irqoff(nr);
```

<!---
macro. After this, it checks the result of the `in_interrupt` that returns `irq_count` value. We already saw the `irq_count` in the first [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) of this chapter and it is used to check if a CPU is already on an interrupt stack or not. We just exit from the `raise_softirq_irqoff`, restore `IF` flag and enable interrupts on the local processor, if we are in the interrupt context, otherwise  we call the `wakeup_softirqd`:
--->

その後、 `irq_count`値を返す`in_interrupt()`の結果をチェックします。
この章の最初の
[part](http:///0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) で既に `irq_count`を見ており、CPU がすでに割り込みスタックに入っているかどうかを調べるのに使われています。
`raise_softirq_irqoff()`を終了し、`IF`フラグを復元し、ローカルプロセッサ上の割り込みを有効にします。
割り込みコンテキストではない場合は `wakeup_softirqd()`を呼び出します。
<!--- 原文、otherwiseでコンテキストではない、という意味だろう。コードはそう言っている。 --->

```C
if (!in_interrupt())
	wakeup_softirqd();
```

<!---
Where the `wakeup_softirqd` function activates the `ksoftirqd` kernel thread of the local processor:
--->
`wakeup_softirqd()`関数がローカルプロセッサの`ksoftirqd`カーネルスレッドを起動する場所：

```C
static void wakeup_softirqd(void)
{
	struct task_struct *tsk = __this_cpu_read(ksoftirqd);

    if (tsk && tsk->state != TASK_RUNNING)
        wake_up_process(tsk);
}
```

<!---
Each `ksoftirqd` kernel thread runs the `run_ksoftirqd` function that checks existence of deferred interrupts and calls the `__do_softirq` function depends on result. This function reads the `__softirq_pending` softirq bit mask of the local processor and executes the deferrable functions corresponding to every bit set. During execution of a deferred function, new pending `softirqs` might occur. The main problem here that execution of the userspace code can be delayed for a long time while the `__do_softirq` function will handle deferred interrupts. For this purpose, it has the limit of the time when it must be finished:
--->
各`ksoftirqd`カーネルスレッドは、`run_ksoftirqd()`関数を実行します。この関数は、遅延割り込みの存在をチェックし、その結果によって `__do_softirq()`関数を呼び出します。
この`__do_softirq()`関数は、ローカルプロセッサの `__softirq_pending` softirqビットマスクを読み取り、すべてのビットセットに対応する遅延機能を実行します。
遅延関数の実行中に、新しい保留中の `softirqs`が発生する可能性があります。
ここでの主な問題は、ユーザ空間コードの実行を長時間遅らせ、 `__do_softirq`関数が遅延割り込みを処理することができることです。この問題を解決するため、関数の処理時間は、終了する必要がある時間の限界を持っています：

```C
unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
...
...
...
restart:
while ((softirq_bit = ffs(pending))) {
	...
	h->action(h);
	...
}
...
...
...
pending = local_softirq_pending();
if (pending) {
	if (time_before(jiffies, end) && !need_resched() &&
		--max_restart)
            goto restart;
}
...
```

<!---
   Checks of the existence of the deferred interrupts performed periodically and there are some points where this check occurs. The main point where this situation occurs is the call of the `do_IRQ` function that defined in the [arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irq.c) and provides main possibilities for actual interrupt processing in the Linux kernel. When this function will finish to handle an interrupt, it calls the `exiting_irq` function from the [arch/x86/include/asm/apic.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/apic.h) that expands to the call of the `irq_exit` function. The `irq_exit` checks deferred interrupts, current context and calls the `invoke_softirq` function:
--->

定期的に実行される遅延割り込みの有無をチェックし、このチェックを行ういくつかのポイントがあります。
<!--- [ja] TODO: ARM版も示す --->
この状況が発生する主なポイントは
[arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irq.c) で定義されている`do_IRQ`関数の呼び出しで、Linuxカーネルにおける実際の割り込み処理の主な実現性を提供します。
この関数が割込み処理を完了した時、
[arch/x86/include/asm/apic.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/apic.h)から`exiting_irq()`関数を呼びます。これは`irq_exit()`関数の呼び出しに展開されます。
`irq_exit()`は、遅延割り込みと現在のコンテキストをチェックし、`invoke_softirq()`関数を呼び出します：

```C
if (!in_interrupt() && local_softirq_pending())
    invoke_softirq();
```

<!---
   that executes the `__do_softirq` too. So what do we have in summary. Each `softirq` goes through the following stages: Registration of a `softirq` with the `open_softirq` function. Activation of a `softirq` by marking it as deferred with the `raise_softirq` function. After this, all marked `softirqs` will be r in the next time the Linux kernel schedules a round of executions of deferrable functions. And execution of the deferred functions that have the same type.
--->
`irq_exit()`は、`__do_softirq()`も実行します。まとめとして何を得ますか？？
それぞれの`softirq`は、以下のステージをとおります：
1. `open_softirq`による`softirq`の登録
1. `raise_softirq`関数により遅延（deferred）とマーキングされた`softirq`の活性化
1. このあと、全てのマークされた`softirqs`は、Linuxカーネルが遅延機能の実行のラウンドをスケジューリングした次のタイミングで実行されます。
1. そして、同じ種類を持つ遅延関数が実行されます。

<!---
As I already wrote, the `softirqs` are statically allocated and it is a problem for a kernel module that can be loaded. The second concept that built on top of `softirq` -- the `tasklets` solves this problem.
--->
既に書いたように、`softirqs`は静的に確保され、ロード可能なカーネルモジュールにとって問題となります。
`softirq`の上に構築された第２のコンセプトである`タスクレット（tasklets）`はこの問題を解決します。

<!---
   softirqは、割込みハンドラからも呼ばれるし、スレッドからも呼ばれる。
   処理が重い時は抜けていくので、hard-irqを保持しし続けることはないし、スレッドが走っていた場合にはhard-irqからの関数実行はしない。
```C
void irq_exit(void)
{
	account_system_vtime(current);
	trace_hardirq_exit();
	sub_preempt_count(IRQ_EXIT_OFFSET);
	if (!in_interrupt() && local_softirq_pending())
		invoke_softirq();

	rcu_irq_exit();
#ifdef CONFIG_NO_HZ
	/* Make sure that timer wheel updates are propagated */
	if (idle_cpu(smp_processor_id()) && !in_interrupt() && !need_resched())
		tick_nohz_stop_sched_tick(0);
#endif
	preempt_enable_no_resched();
}
```
--->

Tasklets
--------------------------------------------------------------------------------

<!---
If you read the source code of the Linux kernel that is related to the `softirq`, you notice that it is used very rarely. The preferable way to implement deferrable functions are `tasklets`. As I already wrote above the `tasklets` are built on top of the `softirq` concept and generally on top of two `softirqs`:
--->
もしあなたが`softirq`に関連するLinuxカーネルのソースコードを読んだ場合は、`softirq`はまれにしか使われていないことがわかります。遅延可能な機能を実現する好ましい方法は、`タスクレット`です。
私がすでに書いたように、 `タスクレット`は `softirq`コンセプトの上に構築され、一般に２つの`softirqs`の上に構築されます：
<!---
   [ja] softirq, softirqsで使い分けているみたい。仕組み、と、それぞれの要素（処理種別）とみたほうがいいな。
--->

* `TASKLET_SOFTIRQ`;
* `HI_SOFTIRQ`.

<!---
In short words, `tasklets` are `softirqs` that can be allocated and initialized at runtime and unlike `softirqs`, tasklets that have the same type cannot be run on multiple processors at a time. Ok, now we know a little bit about the `softirqs`, of course previous text does not cover all aspects about this, but now we can directly look on the code and to know more about the `softirqs` step by step on practice and to know about `tasklets`. Let's return back to the implementation of the `softirq_init` function that we talked about in the beginning of this part. This function is defined in the [kernel/softirq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c) source code file, let's look on its implementation:
--->

簡単に言うと、 `タスクレット`は実行時に割り当てられ、初期化される `softirqs`であり、`softirqs`とは異なり、同じ種類のタスクレットは一度に複数のプロセッサ上で実行することができません。
さて、私たちは `softirqs`について少しは知っていますが、これまでの文章はこれに関するすべての側面をカバーしているわけではありません。しかし、コードを直接見て、実際の手順で`softirqs`ステップについてもっと知ることや、`タスクレット`について知ることができます。
この部分の初めに説明した `softirq_init`関数の実装に戻りましょう。
この関数は、[kernel/softirq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c) ソースコードファイルに定義されています。その実装を見ていきましょう：

```C
void __init softirq_init(void)
{
        int cpu;

        for_each_possible_cpu(cpu) {
                per_cpu(tasklet_vec, cpu).tail =
                        &per_cpu(tasklet_vec, cpu).head;
                per_cpu(tasklet_hi_vec, cpu).tail =
                        &per_cpu(tasklet_hi_vec, cpu).head;
        }

        open_softirq(TASKLET_SOFTIRQ, tasklet_action);
        open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```

<!---
	We can see definition of the integer `cpu` variable at the beginning of the `softirq_init` function. Next we will use it as parameter for the `for_each_possible_cpu` macro that goes through the all possible processors in the system. If the `possible processor` is the new terminology for you, you can read more about it the [CPU masks](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html) chapter. In short words, `possible cpus` is the set of processors that can be plugged in anytime during the life of that system boot. All `possible processors` stored in the `cpu_possible_bits` bitmap, you can find its definition in the [kernel/cpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cpu.c):
--->

`softirq_init()`関数の先頭に整数`cpu`変数の定義があります。 次に、`cpu`変数をシステム内のすべての可能なプロセッサを通過する `for_each_possible_cpu`マクロのパラメータとして使用します。もし、`possible processor`という単語が初めてのものであれば、それについては
[CPU masks](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html) の章でもっと読むことができます。
簡単に言うと、`possible cpus`とは、システム起動の寿命中、どんなときにでもプラグインできるプロセッサのセットです。
すべての`possible cpus`は、`cpu_possible_bits`ビットマップに保存されています。
その定義は、[kernel/cpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cpu.c) に見つけることができます：

```C
static DECLARE_BITMAP(cpu_possible_bits, CONFIG_NR_CPUS) __read_mostly;
...
...
...
const struct cpumask *const cpu_possible_mask = to_cpumask(cpu_possible_bits);
```

<!---
Ok, we defined the integer `cpu` variable and go through the all possible processors with the `for_each_possible_cpu` macro and makes initialization of the two following [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) variables:
--->
さて、私たちは整数の `cpu`変数を定義し、`for_each_possible_cpu`マクロですべての可能なプロセッサーを調べ、以下の２つの
[per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)変数の初期化を行いました。

* `tasklet_vec`;
* `tasklet_hi_vec`;

<!---
These two `per-cpu` variables defined in the same source [code](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c) file as the `softirq_init` function and represent two `tasklet_head` structures:
--->
これらの２つの`per-cpu`変数は、`softirq_init()`関数と[同じソースコードファイル](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c)で定義されており、２つの`tasklet_head`構造体で表されます。

```C
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);
```

<!---
Where `tasklet_head` structure represents a list of `Tasklets` and contains two fields, head and tail:
--->
`tasklet_head`構造体は、`Tasklets`のリストを表しており、２つのフィールド headとtailとから成ります：

```C
struct tasklet_head {
        struct tasklet_struct *head;
        struct tasklet_struct **tail;
};
```

<!---
The `tasklet_struct` structure is defined in the [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h) and represents the `Tasklet`. Previously we did not see this word in this book. Let's try to understand what the `tasklet` is. Actually, the tasklet is one of mechanisms to handle deferred interrupt. Let's look on the implementation of the `tasklet_struct` structure:
--->
`tasklet_struct`構造体は
[include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h)で定義されており、`Tasklet`を表します。
以前に、この本でこの単語を見たことがありません。`tasklet_struct`構造体の実装を見てみましょう：

```C
struct tasklet_struct
{
        struct tasklet_struct *next;
        unsigned long state;
        atomic_t count;
        void (*func)(unsigned long);
        unsigned long data;
};
```

<!---
As we can see this structure contains five fields, they are:

* Next tasklet in the scheduling queue;
* State of the tasklet;
* Represent current state of the tasklet, active or not;
* Main callback of the tasklet;
* Parameter of the callback.
--->
この構造体は５つのフィールドを含んでいることがわかります。

* next: スケジューリングキュー内の次のタスクレットへのポインタ
* state: タスクレットの状態
* count: タスクレットの現在の状態（アクティブかそうでないか）を表す ???
* func: 主なタスクレットのコールバック関数へのポインタ
* data: コールバックのパラメータ

<!---
In our case, we set only for initialize only two arrays of tasklets in the `softirq_init` function: the `tasklet_vec` and the `tasklet_hi_vec`. Tasklets and high-priority tasklets are stored in the `tasklet_vec` and `tasklet_hi_vec` arrays, respectively. So, we have initialized these arrays and now we can see two calls of the `open_softirq` function that is defined in the [kernel/softirq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c) source code file:
--->
ここでは、 `softirq_init`関数でタスクレットの配列を、`tasklet_vec`と`tasklet_hi_vec`の２つだけ初期化するように設定しています。
タスクレットと高優先度なタスクレットは、それぞれ`tasklet_vec`配列と`tasklet_hi_vec`配列とに保存されます。
これらの配列の初期化を完了し、 `softirq_init()`関数の最後に
[kernel/softirq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c)で定義されている`open_softirq()`関数の呼び出しを２つ見ることができます。

```C
open_softirq(TASKLET_SOFTIRQ, tasklet_action);
open_softirq(HI_SOFTIRQ, tasklet_hi_action);
```
<!---
at the end of the `softirq_init` function. The main purpose of the `open_softirq` function is the initialization of `softirq`. Let's look on the implementation of the `open_softirq` function.
--->
`open_softirq`関数の主な目的は、`softirq`の初期化です。`open_softirq`関数の実装を見ていきましょう：

<!--- [ja] issue: 原文壊れてる？ --->
<!---
, in our case they are: `tasklet_action` and the `tasklet_hi_action` or the `softirq` function associated with the `HI_SOFTIRQ` softirq is named `tasklet_hi_action` and `softirq` function associated with the `TASKLET_SOFTIRQ` is named `tasklet_action`. The Linux kernel provides API for the manipulating of `tasklets`. First of all it is the `tasklet_init` function that takes `tasklet_struct`, function and parameter for it and initializes the given `tasklet_struct` with the given data:
--->
ここでは、`HI_SOFTIRQ` softirqに対応する`softirq`関数の名前は `tasklet_hi_action`で、
`TASKLET_SOFTIRQ`に対応する` softirq`関数の名前は `tasklet_action`です。
Linuxカーネルは、`tasklets`の操作のために、APIを提供します。まず最初に、`tasklet_init()`関数は与えられたデータ（`tasklet_struct`、関数、関数のためのパラメータ）で、与えられた`tasklet_struct`を初期化します。

```C
void tasklet_init(struct tasklet_struct *t,
                  void (*func)(unsigned long), unsigned long data)
{
    t->next = NULL;
    t->state = 0;
    atomic_set(&t->count, 0);
    t->func = func;
    t->data = data;
}
```

<!---
There are additional methods to initialize a tasklet statically with the two following macros:
--->
以下の２つのマクロが、タスクレットを静的に初期化するための追加の方法として存在します。

```C
DECLARE_TASKLET(name, func, data);
DECLARE_TASKLET_DISABLED(name, func, data);
```

<!---
The Linux kernel provides three following functions to mark a tasklet as ready to run:
--->
Linuxカーネルは、実行する準備ができた、と、タスクレットにマークするために、以下の３つの関数を提供します。

```C
void tasklet_schedule(struct tasklet_struct *t);
void tasklet_hi_schedule(struct tasklet_struct *t);
void tasklet_hi_schedule_first(struct tasklet_struct *t);
```

<!---
The first function schedules a tasklet with the normal priority, the second with the high priority and the third out of turn. Implementation of the all of these three functions is similar, so we will consider only the first -- `tasklet_schedule`. Let's look on its implementation:
--->
１つめの関数は、通常の優先度を有するタスクレットをスケジューリングし、
２つめの関数は、優先度を高くし、
３つめの関数は、ターンアラウンドをスケジューリングします。
これらの３つの関数の実装はすべて似ているので、最初の `tasklet_schedule`だけを検討します。
その実装を見てみましょう：

```C
static inline void tasklet_schedule(struct tasklet_struct *t)
{
    if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
        __tasklet_schedule(t);
}

void __tasklet_schedule(struct tasklet_struct *t)
{
        unsigned long flags;

        local_irq_save(flags);
        t->next = NULL;
        *__this_cpu_read(tasklet_vec.tail) = t;
        __this_cpu_write(tasklet_vec.tail, &(t->next));
        raise_softirq_irqoff(TASKLET_SOFTIRQ);
        local_irq_restore(flags);
}
```

<!---
As we can see it checks and sets the state of the given tasklet to the `TASKLET_STATE_SCHED` and executes the `__tasklet_schedule` with the given tasklet. The `__tasklet_schedule` looks very similar to the `raise_softirq` function that we saw above. It saves the `interrupt flag` and disables interrupts at the beginning. After this, it updates `tasklet_vec` with the new tasklet and calls the `raise_softirq_irqoff` function that we saw above. When the Linux kernel scheduler will decide to run deferred functions, the `tasklet_action` function will be called for deferred functions which are associated with the `TASKLET_SOFTIRQ` and `tasklet_hi_action` for deferred functions which are associated with the `HI_SOFTIRQ`. These functions are very similar and there is only one difference between them -- `tasklet_action` uses `tasklet_vec` and `tasklet_hi_action` uses `tasklet_hi_vec`.
--->
与えられたタスクレットの状態をチェックし、`TASKLET_STATE_SCHED`にしています。そして、与えられたタスクレットで、`__tasklet_schedule()`関数を実行していることがわかります。
`__tasklet_schedule()`関数は、前に見た`raise_softirq()`関数ととても良く似ているように見えます。
このあと、新しいタスクレットで`tasklet_vec`を更新し、前に見た`raise_softirq_irqoff()`関数を呼びます。
Linuxカーネルスケジューラが遅延関数の実行を決定する時、`TASKLET_SOFTIRQ`に関連する遅延関数には`tasklet_action()`関数が呼ばれ、`HI_SOFTIRQ`に関連する遅延関数のためには`tasklet_hi_action`が呼ばれます。
これらの関数はとても似ていて、唯一の違いは`tasklet_action`が`tasklet_vec`を使い、
`tasklet_hi_action` が `tasklet_hi_vec`を使うことです。

<!---
Let's look on the implementation of the `tasklet_action` function:
--->
`tasklet_action`仮数の実装を見てみましょう：

```C
static void tasklet_action(struct softirq_action *a)
{
    local_irq_disable();
    list = __this_cpu_read(tasklet_vec.head);
    __this_cpu_write(tasklet_vec.head, NULL);
    __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
    local_irq_enable();

    while (list) {
		if (tasklet_trylock(t)) {
	        t->func(t->data);
            tasklet_unlock(t);
	    }
		...
		...
		...
    }
}
```

<!---
In the beginning of the `tasklet_action` function, we disable interrupts for the local processor with the help of the `local_irq_disable` macro (you can read about this macro in the second [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-2.html) of this chapter). In the next step, we take a head of the list that contains tasklets with normal priority and set this per-cpu list to `NULL` because all tasklets must be executed in a generally way. After this we enable interrupts for the local processor and go through the list of tasklets in the loop. In every iteration of the loop we call the `tasklet_trylock` function for the given tasklet that updates state of the given tasklet on `TASKLET_STATE_RUN`:
--->
`tasklet_action`関数の先頭で、`local_irq_disable`マクロの助けでローカルプロセッサの割込みを禁止します
（このマクロについては、[この章の２つめ](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-2.html) できます）。
次のステップで、通常優先度（normal priority）のタスクレットを含むリストの先頭を取得し、
すべてのタスクレットが一般的な方法で実行されなければならないので、このper-cpuリストを`NULL`にセットします。
このあと、ローカルプロセッサの割込みを有効にし、ループ内でタスクレットのリストを走り抜けます。
このループの全ての繰り返しに於いて、与えられたタスクレットを`TASKLET_STATE_RUN`状態とするために`tasklet_trylock`関数を呼んでいます。

```C
static inline int tasklet_trylock(struct tasklet_struct *t)
{
    return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}
```

<!---
If this operation was successful we execute tasklet's action (it was set in the `tasklet_init`) and call the `tasklet_unlock` function that clears tasklet's `TASKLET_STATE_RUN` state.
--->
もしこの操作が成功したなら、タスクレットのaction（`tasklet_init`でセットされたもの）を実行し、タスクレットの`TASKLET_STATE_RUN`状態をクリアする`tasklet_unlock()`関数を呼びます。

<!---
In general, that's all about `tasklets` concept. Of course this does not cover full `tasklets`, but I think that it is a good point from where you can continue to learn this concept.
--->
一般的には、以上がタスクレットの概念です。
もちろん、これは完全な `タスクレット`をカバーするものではありませんが、あなたがこのコンセプトをどこから学び続けることができるかは良い点だと思います。

<!---
   The `tasklets` are [widely](http://lxr.free-electrons.com/ident?i=tasklet_init) used concept in the Linux kernel, but as I wrote in the beginning of this part there is third mechanism for deferred functions -- `workqueue`. In the next paragraph we will see what it is.
--->
`タスクレット`はLinuxカーネルで
[広く使われている](http://lxr.free-electrons.com/ident?i=tasklet_init)概念ですが、このパートの冒頭で書いたように、遅延関数のための第３のメカニズム - `ワークキュー（workqueue）`があります。
次の段落では、それが何であるかを見ていきます。

Workqueues
--------------------------------------------------------------------------------

<!---
The `workqueue` is another concept for handling deferred functions. It is similar to `tasklets` with some differences. Workqueue functions run in the context of a kernel process, but `tasklet` functions run in the software interrupt context. This means that `workqueue` functions must not be atomic as `tasklet` functions. Tasklets always run on the processor from which they were originally submitted. Workqueues work in the same way, but only by default. The `workqueue` concept represented by the:
[ja] TODO: ここから上で "〜関数"、は、"〜機能"ってしたほうが良さそう。
--->

`ワークキュー（workqueue）`は、遅延関数を処理するための、その他のコンセプトです。
幾らかの違いがありますが、`tasklets`と似ています。
ワークキュー機能はカーネルプロセスのコンテキストで走行しますが、`tasklet`機能はソフトウェア割り込みコンテキストで走行します。これが意味するところは、`workqueue`機能は、`tasklet`機能のようにアトミックである必要ないということです。
タスクレットは常にサブミットされたプロセッサ上で走ります。
タスクレットは、最初に申請されたプロセッサ上で常に実行されます。
ワークキューは同様に動きますが、デフォルトの場合のみです。
`workqueue`コンセプトは、Linuxカーネルのソースコードファイルの、
[kernel/workqueue.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/workqueue.c)で定義された構造体で表されます：


```C
struct worker_pool {
    spinlock_t              lock;
    int                     cpu;
    int                     node;
    int                     id;
    unsigned int            flags;

    struct list_head        worklist;
    int                     nr_workers;
...
...
...
```

<!---
structure that is defined in the [kernel/workqueue.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/workqueue.c) source code file in the Linux kernel. I will not write the source code of this structure here, because it has quite a lot of fields, but we will consider some of those fields.
--->
この構造体はとても多くのフィールドを持つので、ソースコードはここに書いていません。
しかし、それらのフィールドのいくつかは検討します。

<!---
In its most basic form, the work queue subsystem is an interface for creating kernel threads to handle work that is queued from elsewhere. All of these kernel threads are called -- `worker threads`. The work queue are maintained by the `work_struct` that defined in the [include/linux/workqueue.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/workqueue.h). Let's look on this structure:
--->

最も基本的な形式では、ワークキューサブシステムは、他の場所からキューに入れられた作業を処理するためのカーネルスレッドを作成するためのインタフェースです。 
これらのカーネルスレッドはすべて `worker threads`と呼ばれます。
ワークキューは、
[include/linux/workqueue.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/workqueue.h)で定義された`work_struct`によって維持されます。
この構造体を見てみましょう：

```C
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func;
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};
```

<!---
Here are two things that we are interested: `func` -- the function that will be scheduled by the `workqueue` and the `data` - parameter of this function. The Linux kernel provides special per-cpu threads that are called `kworker`:
--->
面白いことが2つあります。
`func`（`workqueue`によってスケジュールされる関数）と
`data`（この関数のパラメータ）です。
Linuxカーネルは、`kworker`と酔われる特別なper-cpuスレッドを提供します。

```
systemd-cgls -k | grep kworker
├─    5 [kworker/0:0H]
├─   15 [kworker/1:0H]
├─   20 [kworker/2:0H]
├─   25 [kworker/3:0H]
├─   30 [kworker/4:0H]
...
...
...
```

<!---
This process can be used to schedule the deferred functions of the workqueues (as `ksoftirqd` for `softirqs`). Besides this we can create new separate worker thread for a `workqueue`. The Linux kernel provides following macros for the creation of workqueue:
--->
このプロセス（`softirqs`のための`ksoftirqd`）は、ワークキューの遅延関数をスケジュールするために使うことができます。
これ以外にも、`workqueue`のための新しいワーカースレッドを作ることができます。
Linuxカーネルはワークキューを作るために、静的な生成には以下のマクロを提供します。

```C
#define DECLARE_WORK(n, f) \
    struct work_struct n = __WORK_INITIALIZER(n, f)
```

<!---
for static creation. It takes two parameters: name of the workqueue and the workqueue function. For creation of workqueue in runtime, we can use the:
--->
これは2つのパラメータをとります。ワークキューの名前と、ワークキューの関数です。
実行時にワークキューを作るためには、以下のマクロを使います：

```C
#define INIT_WORK(_work, _func)       \
    __INIT_WORK((_work), (_func), 0)

#define __INIT_WORK(_work, _func, _onstack)                     \
    do {                                                        \
            __init_work((_work), _onstack);                     \
            (_work)->data = (atomic_long_t) WORK_DATA_INIT();   \
            INIT_LIST_HEAD(&(_work)->entry);                    \
             (_work)->func = (_func);                           \
    } while (0)
```

<!---
macro that takes `work_struct` structure that has to be created and the function to be scheduled in this workqueue. After a `work` was created with the one of these macros, we need to put it to the `workqueue`. We can do it with the help of the `queue_work` or the `queue_delayed_work` functions:
--->
このマクロは、生成された`work_struct`構造体と、このワークキューでスケジュールされる関数とをとります。
これらのマクロの一つで`work`を作った後、`workqueue`につなげる必要があります。そのためには、`queue_work()`関数の助けを借りるか、`queue_delayed_work`関数の助けを借ります。

```C
static inline bool queue_work(struct workqueue_struct *wq,
                              struct work_struct *work)
{
    return queue_work_on(WORK_CPU_UNBOUND, wq, work);
}
```

<!---
The `queue_work` function just calls the `queue_work_on` function that queues work on specific processor. Note that in our case we pass the `WORK_CPU_UNBOUND` to the `queue_work_on` function. It is a part of the `enum` that is defined in the [include/linux/workqueue.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/workqueue.h) and represents workqueue which are not bound to any specific processor. The `queue_work_on` function tests and set the `WORK_STRUCT_PENDING_BIT` bit of the given `work` and executes the `__queue_work` function with the `workqueue` for the given processor and given `work`:
--->
`queue_work`関数は、すぐに 指示されたプロセッサの `queue_work_on`関数を呼びます。
この場合、`queue_work_on`へ`WORK_CPU_UNBOUND`を渡します。これは、
[include/linux/workqueue.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/workqueue.h)で定義された`enum`で、任意の特定のプロセッサにバインドされていないワークキューを表します。
`queue_work_on()`関数は、与えられた`work`の`WORK_STRUCT_PENDING_BIT`ビットのテストとセットを行い、与えられた`work`と与えられた`workqueue`で、`__queue_work()`関数を実行します。


```C
bool queue_work_on(int cpu, struct workqueue_struct *wq,
           struct work_struct *work)
{
    bool ret = false;
    ...
    if (!test_and_set_bit(WORK_STRUCT_PENDING_BIT, work_data_bits(work))) {
        __queue_work(cpu, wq, work);
        ret = true;
    }
    ...
    return ret;
}
```

<!---
The `__queue_work` function gets the `work pool`. Yes, the `work pool` not `workqueue`. Actually, all `works` are not placed in the `workqueue`, but to the `work pool` that is represented by the `worker_pool` structure in the Linux kernel. As you can see above, the `workqueue_struct` structure has the `pwqs` field which is list of `worker_pools`. When we create a `workqueue`, it stands out for each processor the `pool_workqueue`. Each `pool_workqueue` associated with `worker_pool`, which is allocated on the same processor and corresponds to the type of priority queue. Through them `workqueue` interacts with `worker_pool`. So in the `__queue_work` function we set the cpu to the current processor with the `raw_smp_processor_id` (you can find information about this macro in the fourth [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html) of the Linux kernel initialization process chapter), getting the `pool_workqueue` for the given `workqueue_struct` and insert the given `work` to the given `workqueue`:
--->
`__queue_work`関数は`work pool`を引数にとります。そう、`workqueue`ではなく、`work pool`です。
実際に、すべての`works`は`workqueue`にはありませんが、Linuxカーネルの`worker_pool`構造体で表される`work pool`にあります。
前述のとおり、`workqueue_struct`構造体は、`worker_pools`のリストである`pwqs`フィールドを持ちます。
`workqueue`を作成すると、各プロセッサに`pool_workqueue`が（立ちます｜めだちます）。

```C
saatatic void __queue_work(int cpu, struct workqueue_struct *wq,
                         struct work_struct *work)
{
...
...
...
if (req_cpu == WORK_CPU_UNBOUND)
    cpu = raw_smp_processor_id();

if (!(wq->flags & WQ_UNBOUND))
    pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);
else
    pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
...
...
...
insert_work(pwq, work, worklist, work_flags);
```

<!---
As we can create `works` and `workqueue`, we need to know when they are executed. As I already wrote, all `works` are executed by the kernel thread. When this kernel thread is scheduled, it starts to execute `works` from the given `workqueue`. Each worker thread executes a loop inside the `worker_thread` function. This thread makes many different things and part of these things are similar to what we saw before in this part. As it starts executing, it removes all `work_struct` or `works` from its `workqueue`.
--->
`works` と `ワークキュー（workqueue）` とを作れるようになったので、いつそれぞれが実行されるかを知る必要があります。
既に書いたように、すべての `works` は、カーネルスレッドで実行されます。
このカーネルスレッドがスケジュールされる時、与えられた `ワークキュー（workqueue）` から取得した`works`の実行を開始します。
それぞれのワーカスレッドが、 `worker_thread()`関数の中にあるループを実行します。
このスレッドは、たくさんの異なるものを作ったり、この章の前半で見たように、これらの一部は似ています。
実行を開始すると、`work_strcut`または`works`は、その`ワークキュー（workqueue）`から取り除かれます。

That's all.

Conclusion
--------------------------------------------------------------------------------

<!---
It is the end of the ninth part of the [Interrupts and Interrupt Handling](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) chapter and we continued to dive into external hardware interrupts in this part. In the previous part we saw initialization of the `IRQs` and main `irq_desc` structure. In this part we saw three concepts: the `softirq`, `tasklet` and `workqueue` that are used for the deferred functions.
--->
[Interrupts and Interrupt Handlingの章](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)の９部の終わりです。
引き続き、外部ハードウェア割り込みの部（external hardware interrupts）へと進みましょう。
前の部では、`IRQs`と、主要な`irq_desc`構造体の初期化を見ました。
この部では、３つのコンセプトをみました。遅延機能に使うための`softirq`、`tasklet`、`workqueue`です。

<!---
The next part will be last part of the `Interrupts and Interrupt Handling` chapter and we will look on the real hardware driver and will try to learn how it works with the interrupts subsystem.
--->
次の部では、`割り込みと割り込み処理`の章の最後となります。
本当のハードウェアドライバを見ていき、割り込みサブシステムとどのように動いているのかを知りましょう。

<!---
If you have any questions or suggestions, write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**
--->
質問や提案があれば、[twitter](https://twitter.com/0xAX) へコメントやpingを書いてください。

**英語は第一言語ではないことに気をつけてください。ご不便をおかけして申し訳ありません。
もし誤りを見つけられたら、[linux-insides](https://github.com/0xAX/linux-insides)へPRしてください。**

Links
--------------------------------------------------------------------------------

* [initcall](http://www.compsoc.man.ac.uk/~moz/kernelnewbies/documents/initcall/index.html)
* [IF](https://en.wikipedia.org/wiki/Interrupt_flag)
* [eflags](https://en.wikipedia.org/wiki/FLAGS_register)
* [CPU masks](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html)
* [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [Workqueue](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/workqueue.txt)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-8.html)
