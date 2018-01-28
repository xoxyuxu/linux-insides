Interrupts and Interrupt Handling. Part 1.
================================================================================

Introduction
--------------------------------------------------------------------------------

<!---
This is the first part of the new chapter of the [linux insides](http://0xax.gitbooks.io/linux-insides/content/) book. We have come a long way in the previous [chapter](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) of this book. We started from the earliest [steps](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) of kernel initialization and finished with the [launch](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-10.html) of the first `init` process. Yes, we saw several initialization steps which are related to the various kernel subsystems. But we did not dig deep into the details of these subsystems. With this chapter, we will try to understand how the various kernel subsystems work and how they are implemented. As you can already understand from the chapter's title, the first subsystem will be [interrupts](http://en.wikipedia.org/wiki/Interrupt).
--->
これは、[linux insides](http://0xax.gitbooks.io/linux-insides/content/) bookの新しい章の第一部です。
この本の[前の章](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)まで、長い道のりをやってきました。カーネル初期化の[最初のステップ](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)から始まり、最初の `init`プロセスの[開始](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-10.html)までを終わりました。そう、様々なカーネルサブシステムに関するいくつかの初期化ステップを見ました。
しかし、これらのサブシステムの詳細について深く掘り下げていませんでした。
この章では、様々なカーネルサブシステムがどのように動いているのか、どのように実装されているのかを理解しようと試みます。
この章では、タイトルから判るように、最初のサブシステムは [割り込み](http://en.wikipedia.org/wiki/Interrupt)です。


What is an Interrupt?
--------------------------------------------------------------------------------

<!---
We have already heard of the word `interrupt` in several parts of this book. We even saw a couple of examples of interrupt handlers. In the current chapter we will start from the theory i.e.,

* What are `interrupts` ?
* What are `interrupt handlers`?

We will then continue to dig deeper into the details of `interrupts` and how the Linux kernel handles them.
--->
すでに、この本のいくつかのパートで `割り込み` という語は聞いていますね。
いくつか、割り込みハンドラの例も見ています。この章では、以下の理論から始めます：

* `割り込み` とは何か
* `割り込みハンドラ` とは何か

続いて、`割り込み` の詳細と、Linuxカーネルが割り込みをどのように処理するのか、詳しく調べていきます。

<!---
The first question that arises in our mind when we come across word `interrupt` is `What is an interrupt?` An interrupt is an `event` raised by software or hardware when it needs the CPU's attention. For example, we press a button on the keyboard and what do we expect next? What should the operating system and computer do after this? To simplify matters, assume that each peripheral device has an interrupt line to the CPU. A device can use it to signal an interrupt to the CPU. However, interrupts are not signaled directly to the CPU. In the old machines there was a [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) which is a chip responsible for sequentially processing multiple interrupt requests from multiple devices. In the new machines there is an [Advanced Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) commonly known as - `APIC`. An `APIC` consists of two separate devices:
--->
私達が `割り込み` という語に出会った時に心に浮かぶ最初の疑問は、 `割り込みとはなんだろう？`でしょう。
割り込みは、CPUの注目が必要と生る時に、ソフトウェアやハードウェアによって起こされる `イベント`です。
例えば、キーボードのボタンを押下し、次に何をきたいしますか？
オペレーティングシステムとコンピュータが、このあとに何をすべきですか？
問題を簡略化するために、各周辺デバイスにCPUに対する割り込みラインがあるとします。
デバイスは、それを使用してCPUへの割り込みを通知することができます。
しかし、割り込みは CPUに直接通知しません。
古いマシンには、複数デバイスからの複数の割り込み要求をシーケンシャルに処理するための責任者となるチップの [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) がありました。
新しいマシンには、 `APIC` と知られている、 [Advanced Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) があります。
`APIC` は、以下の２つの別れたデバイスから構成されます：

* `Local APIC`
* `I/O APIC`

<!---
The first - `Local APIC` is located on each CPU core. The local APIC is responsible for handling the CPU-specific interrupt configuration. The local APIC is usually used to manage interrupts from the APIC-timer, thermal sensor and any other such locally connected I/O devices.
--->
最初の `Local APIC`は、各CPUコアに配置されます。
local APICは、CPU固有の割り込み設定処理責任者です。local APICは、APIC-timer、温度センサ、その他のローカルに接続されたI/Oデバイスからの割り込み管理に使われます。

<!---
The second - `I/O APIC` provides multi-processor interrupt management. It is used to distribute external interrupts among the CPU cores. More about the local and I/O APICs will be covered later in this chapter. As you can understand, interrupts can occur at any time. When an interrupt occurs, the  operating system must handle it immediately. But what does it mean `to handle an interrupt`? When an interrupt occurs, the  operating system must ensure the following steps:
--->
ふたつ目の `I/O APIC` は、マルチプロセッサ割り込み管理を提供します。
CPUコア間に外部割り込みを分配するために使用されます。
local APIとI/O APICの詳細については、この章の後方でカバーします。
御承知のとおり、割り込みはいつでも推すことができます。
割込みが起きた時、オペレーティングシステムは直ちに処理しなければなりません。
しかし、 `割込みを処理すること` とは何を意味するのでしょうか。
割込みが起きた時、オペレーティングシステムは確実に以下のステップを実行しなければなりません：

<!---
* The kernel must pause execution of the current process; (preempt current task);
* The kernel must search for the handler of the interrupt and transfer control (execute interrupt handler);
* After the interrupt handler completes execution, the interrupted process can resume execution.
--->
* カーネルは、カレントプロセスの実行を停止する（現行作業を中断する）
* カーネルは割込みのハンドラを探し、制御を移します（割込みハンドラを実行）
* 割込みハンドラが実行を完了したあと、割りこまれたプロセスは実行を復旧することができます

<!---
Of course there are numerous intricacies involved in this procedure of handling interrupts. But the above 3 steps form the basic skeleton of the procedure.
--->
もちろん、割り込み処理のこの手順には、数多くの複雑さがあります。しかし、上記３ステップは手順の基本骨子からきています。

<!---
Addresses of each of the interrupt handlers are maintained in a special location referred to as the - `Interrupt Descriptor Table` or `IDT`. The processor uses a unique number for recognizing the type of interruption or exception. This number is called - `vector number`. A vector number is an index in the `IDT`. There is limited amount of the vector numbers and it can be from `0` to `255`. You can note the following range-check upon the vector number within the Linux kernel source-code:
--->
それぞれの割込みハンドラのアドレスは、`Interrupt Descriptor Table` または `IDT`と云われる特別な場所を参照して維持されます。プロセッサは割込みや例外の種類を理解するために、ユニークな番号を使います。
この番号は `ヴェクタ番号` と呼ばれます。ヴェクタ番号は、`IDT`内のインデックスです。
ヴェクタ番号の量には限界があり、`0`から`255`までが範囲です。
Linuxカーネルソースコード内に、ヴェクタ番号に対して以下の範囲チェックがあることに注意してください：

```C
BUG_ON((unsigned)n > 0xFF);
```

<!---
You can find this check within the Linux kernel source code related to interrupt setup (eg. The `set_intr_gate`, `void set_system_intr_gate` in [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h)). The first 32 vector numbers from `0` to `31` are reserved by the processor and used for the processing of architecture-defined exceptions and interrupts. You can find the table with the description of these vector numbers in the second part of the Linux kernel initialization process - [Early interrupt and exception handling](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html). Vector numbers from `32` to `255` are designated as user-defined interrupts and are not reserved by the processor. These interrupts are generally assigned to external I/O devices to enable those devices to send interrupts to the processor.
--->
このチェックは、割込みセットアップに関するLinuxカーネルソースコード内
（例えば、カーネルソースの[arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h) `set_intr_gate` や `void set_system_intr_gate`）で見つけることができます。
最初の３２のヴェクタ番号（`0`から`31`）は、プロセッサに予約されており、アーキテクチャ定義の例外と割込み処理に使われます。
これらのヴェクタ番号を記述した表は、Linuxカーネル初期化手順の第二部[Early interrupt and exception handling](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html) で見ることができます。
`32`から`255`までのヴェクタ番号は、ユーザ定義割込みとして設計されており、プロセッサに予約されていません。
これらの割込みは、一般的に外部I/Oデバイスがプロセッサに割込みを送るために、外部I/Oデバイスに接続されています。

<!---
Now let's talk about the types of interrupts. Broadly speaking, we can split interrupts into 2 major classes:

* External or hardware generated interrupts
* Software-generated interrupts
--->
割込み種別について話しましょう。
大まかに言えば、割り込みを2つの主要なクラスに分割できます。

* 外部またはハードウェアが生成した割込み
* ソフトウェアが生成した割込み

<!---
The first - external interrupts are received through the `Local APIC` or pins on the processor which are connected to the `Local APIC`. The second - software-generated interrupts are caused by an exceptional condition in the processor itself (sometimes using special architecture-specific instructions). A common example for an exceptional condition is `division by zero`. Another example is exiting a program with the `syscall` instruction.
--->
最初の外部割込みは、`Local APIC`を介して、または、`Local APIC`に接続されたプロセッサ上の端子を介して受信します。
ふたつめ、ソフトウェアが生成した割込みは、プロセッサ自身が例外状態を起こします（特別なアーキテクチャ定義の命令を用います）。例外状態の一般的な例は、`ゼロ除算`です。他の例は、`syscall`命令でプログラムを終了することです。

<!---
As mentioned earlier, an interrupt can occur at any time for a reason which the code and CPU have no control over. On the other hand, exceptions are `synchronous` with program execution and can be classified into 3 categories:
--->
先に述べたように、割り込みは、コードとCPUが制御できない理由でいつでも発生する可能性があります。
言い換えると、例外は、プログラムの実行で `同期(synchronous)`であり、３つのカテゴリに分類することができます：

* `Faults`
* `Traps`
* `Aborts`

<!---
A `fault` is an exception reported before the execution of a "faulty" instruction (which can then be corrected). If corrected, it allows the interrupted program to be resume.
--->
`fault`は、（修正可能な）"不正な"命令の実行前にレポートされる例外です。もし修正されれば、割りこまれたプログラムの復旧が許容されます。

<!---
Next a `trap` is an exception which is reported immediately following the execution of the `trap` instruction. Traps also allow the interrupted program to be continued just as a `fault` does.
--->
次は`trap`で、`trap`命令の実行に続いて直ちに通知される例外です。Trapも、ちょうど`fault`と同じように、割り込んだプログラムを継続させることが許容されます。

<!---
Finally an `abort` is an exception that does not always report the exact instruction which caused the exception and does not allow the interrupted program to be resumed.
--->
最後に、`abort`は、常に例外を発生させるわけではない命令を通知する例外で、割りこまれたプログラムを復旧することは許されません。

<!---
Also we already know from the previous [part](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html) that interrupts can be classified as `maskable` and `non-maskable`. Maskable interrupts are interrupts which can be blocked with the two following instructions for `x86_64` - `sti` and `cli`. We can find them in the Linux kernel source code:
--->
[前のパート](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html) で既に知っているように、割込みは `マスクできるもの` と `マスクできないもの` とに分類できます。
マスクできる割込みは、`x86_64`の場合、以下の２つの命令（`sti`と`cli`）でブロックすることができる割込みです。
Linuxカーネルソースコードで見つけることができます：

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

and

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

<!---
These two instructions modify the `IF` flag bit within the interrupt register. The `sti` instruction sets the `IF` flag and the `cli` instruction clears this flag. Non-maskable interrupts are always reported. Usually any failure in the hardware is mapped to such non-maskable interrupts.
--->
これら２つの命令は、割込みレジスタの`IF`フラグを変更します。
`sti`命令は `IF`フラグをセットし、`cli`命令は、このフラグをクリアします。
マスクできない割込みは、常に通知されます。一般的に、ハードウェア不良がこのようなマスク不可割込みに割り当てられます。

<!---
If multiple exceptions or interrupts occur at the same time, the processor handles them in order of their predefined priorities. We can determine the priorities from the highest to the lowest in the following table:
--->
もし複数の例外や割込みが同時に起こったなら、プロセッサは定義済みの優先順位にしたがって処理します。
次の表にある、最上位から最下位の優先順位を決定することができます：

```
+----------------------------------------------------------------+
|              |                                                 |
|   Priority   | Description                                     |
|              |                                                 |
+--------------+-------------------------------------------------+
|              | Hardware Reset and Machine Checks               |
|     1        | - RESET                                         |
|              | - Machine Check                                 |
+--------------+-------------------------------------------------+
|              | Trap on Task Switch                             |
|     2        | - T flag in TSS is set                          |
|              |                                                 |
+--------------+-------------------------------------------------+
|              | External Hardware Interventions                 |
|              | - FLUSH                                         |
|     3        | - STOPCLK                                       |
|              | - SMI                                           |
|              | - INIT                                          |
+--------------+-------------------------------------------------+
|              | Traps on the Previous Instruction               |
|     4        | - Breakpoints                                   |
|              | - Debug Trap Exceptions                         |
+--------------+-------------------------------------------------+
|     5        | Nonmaskable Interrupts                          |
+--------------+-------------------------------------------------+
|     6        | Maskable Hardware Interrupts                    |
+--------------+-------------------------------------------------+
|     7        | Code Breakpoint Fault                           |
+--------------+-------------------------------------------------+
|     8        | Faults from Fetching Next Instruction           |
|              | Code-Segment Limit Violation                    |
|              | Code Page Fault                                 |
+--------------+-------------------------------------------------+
|              | Faults from Decoding the Next Instruction       |
|              | Instruction length > 15 bytes                   |
|     9        | Invalid Opcode                                  |
|              | Coprocessor Not Available                       |
|              |                                                 |
+--------------+-------------------------------------------------+
|     10       | Faults on Executing an Instruction              |
|              | Overflow                                        |
|              | Bound error                                     |
|              | Invalid TSS                                     |
|              | Segment Not Present                             |
|              | Stack fault                                     |
|              | General Protection                              |
|              | Data Page Fault                                 |
|              | Alignment Check                                 |
|              | x87 FPU Floating-point exception                |
|              | SIMD floating-point exception                   |
|              | Virtualization exception                        |
+--------------+-------------------------------------------------+
```

<!---
Now that we know a little about the various types of interrupts and exceptions, it is time to move on to a more practical part. We start with the description of the `Interrupt Descriptor Table`. As mentioned earlier, the `IDT` stores entry points of the interrupts and exceptions handlers. The `IDT` is similar in structure to the `Global Descriptor Table` which we saw in the second part of the [Kernel booting process](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html). But of course it has some differences. Instead of `descriptors`, the `IDT` entries are called `gates`. It can contain one of the following gates:
--->
現状、割込みと例外のさまざまな種類について少しはわかりました。より実践的な部分に移るときです。
`Interrupt Descriptor Table` の記述を始めます。最初に示したように、`IDT`は割込みハンドラと例外ハンドラのエントリポイントを維持します。`IDT`は、[Kernel booting process](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)の第二部で見た`Global Descriptor Table`と似た構造です。
しかし、もちろん差異があります。`descriptros`の代わりに、`IDT`エントリは `gates`と呼ばれます。`x86`アーキテクチャでは、以下のゲートのうち１つを含むことができます。

* Interrupt gates
* Task gates
* Trap gates.

<!---
in the `x86` architecture. Only [long mode](http://en.wikipedia.org/wiki/Long_mode) interrupt gates and trap gates can be referenced in the `x86_64`. Like the `Global Descriptor Table`, the `Interrupt Descriptor table` is an array of 8-byte gates on `x86` and an array of 16-byte gates on `x86_64`. We can remember from the second part of the [Kernel booting process](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html), that `Global Descriptor Table` must contain `NULL` descriptor as its first element. Unlike the `Global Descriptor Table`, the `Interrupt Descriptor Table` may contain a gate; it is not mandatory. For example, you may remember that we have loaded the Interrupt Descriptor table with the `NULL` gates only in the earlier [part](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html) while transitioning into [protected mode](http://en.wikipedia.org/wiki/Protected_mode):
--->
[long mode](http://en.wikipedia.org/wiki/Long_mode)割込みゲートと、trapゲートだけは、`x86_64`で参照することができます。
`Global Descriptor Table`のように、`Interrupt Descriptor table`は `x86`では８バイトゲートの配列で、`x86_64`では16バイトゲートの配列です。
[Kernel booting process](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)の第二部を思い出してください。`Global Descriptor Table`は、最初の要素に`NULL`デスクリプタを含まなければなりませんでした。
`Global Descriptor Table`とは違って、`Interrupt Descriptor Table`はゲートを含んでもかまいません（必須ではありません）。
例えば、[前の方](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html) で、[プロテクトモード](http://en.wikipedia.org/wiki/Protected_mode) へ遷移させる際に、`NULL` ゲートのみで IDTをロードしました。

```C
/*
 * Set up the IDT
 */
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

<!---
from the [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c). The `Interrupt Descriptor table` can be located anywhere in the linear address space and the base address of it must be aligned on an 8-byte boundary on `x86` or 16-byte boundary on `x86_64`. The base address of the `IDT` is stored in the special register - `IDTR`. There are two instructions on `x86`-compatible processors to modify the `IDTR` register:
--->
[arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c)から引用。
`Interrupt Descriptor table` は、リニアアドレス空間の何処にでも配置することができ、ベースアドレスは８バイト境界（ `x86` のとき）または１６バイト境界（ `x86_64` のとき）である必要があります。
`IDT` のベースアドレスは、特別なレジスタ（ `IDTR` ）に保存されます。
`IDTR` レジスタを変更するために、 `x86` 互換プロセッサの２つの命令があります：

* `LIDT`
* `SIDT`

<!---
The first instruction `LIDT` is used to load the base-address of the `IDT` i.e., the specified operand into the `IDTR`. The second instruction `SIDT` is used to read and store the contents of the `IDTR` into the specified operand. The `IDTR` register is 48-bits on the `x86` and contains the following information:
--->
最初の命令 `LIDT` は、`IDTI` のベースアドレスをロードするために使います。
すなわち、与えられたオペランドを`IDTR`に転送します。
ふたつ目の命令 `SIDT` は、 `IDTR` のコンテンツを読出し、与えられたオペランドへ 保存します。
`IDTR` レジスタは `x86` では、４８ビットで、以下の情報を含みます：

```
+-----------------------------------+----------------------+
|                                   |                      |
|     Base address of the IDT       |   Limit of the IDT   |
|                                   |                      |
+-----------------------------------+----------------------+
47                                16 15                    0
```

<!---
Looking at the implementation of `setup_idt`, we have prepared a `null_idt` and loaded it to the `IDTR` register with the `lidt` instruction. Note that `null_idt` has `gdt_ptr` type which is defined as:
--->
`setup_idr`の実装を見ましょう。`null_idt`を準備し、それを `lidt`命令で `IDTR`レジスタへロードします。
`null_idt`が、以下のように定義された `gdt_ptr`型を持つことに注意してください。

```C
struct gdt_ptr {
        u16 len;
        u32 ptr;
} __attribute__((packed));
```

<!---
Here we can see the definition of the structure with the two fields of 2-bytes and 4-bytes each (a total of 48-bits) as we can see in the diagram. Now let's look at the `IDT` entries structure. The `IDT` entries structure is an array of the 16-byte entries which are called gates in the `x86_64`. They have the following structure:
--->
ダイアグラム図を見て判るように、２バイトと４バイトの２つのフィールド（合計で４８ビット）で構造体が定義されているのが見られます。
`IDT`エントリ構造について見ていきましょう。
`IDT`エントリ構造は、`x86_64`ではゲートと呼ばれる１６バイトのエントリの配列です。
以下の構造を持ちます：

```
127                                                                             96
+-------------------------------------------------------------------------------+
|                                                                               |
|                                Reserved                                       |
|                                                                               |
+--------------------------------------------------------------------------------
95                                                                              64
+-------------------------------------------------------------------------------+
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
+-------------------------------------------------------------------------------+
63                               48 47      46  44   42    39             34    32
+-------------------------------------------------------------------------------+
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 -------------------------------------------------------------------------------+
31                                   16 15                                      0
+-------------------------------------------------------------------------------+
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
+-------------------------------------------------------------------------------+
```

<!---
To form an index into the IDT, the processor scales the exception or interrupt vector by sixteen. The processor handles the occurrence of exceptions and interrupts just like it handles calls of a procedure when it sees the `call` instruction. A processor uses a unique number or `vector number` of the interrupt or the exception as the index to find the necessary `Interrupt Descriptor Table` entry. Now let's take a closer look at an `IDT` entry.
--->
インデックスからIDRへ変換するために、プロセッサは例外や割込みヴェクタを１６だけスケーリングします。
プロセッサは`call`命令を見る時、プロシージャの呼び出しを処理するのとどうように、例外と割込みの発生を処理します。
プロセッサは、必要な`Interrupt Descriptor Table`エントリを見つけるためのインデックスとして、ユニークな番号／割込みや例外の`ヴェクタ番号`を使います。
`IDT`エントリに焦点を当てましょう。

<!---
As we can see, `IDT` entry on the diagram consists of the following fields:
--->
ダイアグラム上の`IDT`エントリは、以下のフィールドから構成されています：

<!---
* `0-15` bits  - offset from the segment selector which is used by the processor as the base address of the entry point of the interrupt handler;
* `16-31` bits - base address of the segment select which contains the entry point of the interrupt handler;
* `IST` - a new special mechanism in the `x86_64`, will see it later;
* `DPL` - Descriptor Privilege Level;
* `P` - Segment Present flag;
* `48-63` bits - second part of the handler base address;
* `64-95` bits - third part of the base address of the handler;
* `96-127` bits - and the last bits are reserved by the CPU.
--->
* `0-15` bits  - 割込みハンドラのエントリポイントのベースアドレスとして、プロセッサに使われるセグメントセレクタからのオフセット
* `16-31` bits - 割込みハンドラのエントリポイントを含むセグメントセレクタのベースアドレス
* `IST` - `x86_64`の新しい特別なメカニズムです。後ほど出てきます。
* `DPL` - 特権レベルのデスクリプタ
* `P` - セグメント存在フラグ
* `48-63` bits - ２つめの部分のハンドラのベースアドレス
* `64-95` bits - ３つめの部分のハンドラのベースアドレス
* `96-127` bits - 最後のビットはCPUに予約されています。

<!---
And the last `Type` field describes the type of the `IDT` entry. There are three different kinds of handlers for interrupts:
--->
`Type`フィールドは、`IDT`エントリの種別を記述しています。割込みのために、三つのハンドラ種別があります。

* Interrupt gate
* Trap gate
* Task gate

<!---
The `IST` or `Interrupt Stack Table` is a new mechanism in the `x86_64`. It is used as an alternative to the legacy stack-switch mechanism. Previously the `x86` architecture provided a mechanism to automatically switch stack frames in response to an interrupt. The `IST` is a modified version of the `x86` Stack switching mode. This mechanism unconditionally switches stacks when it is enabled and can be enabled for any interrupt in the `IDT` entry related with the certain interrupt (we will soon see it). From this we can understand that `IST` is not necessary for all interrupts. Some interrupts can continue to use the legacy stack switching mode. The `IST` mechanism provides up to seven `IST` pointers in the [Task State Segment](http://en.wikipedia.org/wiki/Task_state_segment) or `TSS` which is the special structure which contains information about a process. The `TSS` is used for stack switching during the execution of an interrupt or exception handler in the Linux kernel. Each pointer is referenced by an interrupt gate from the `IDT`.
--->
`IST` すなわち `Interrupt Stack Table` は、`x86_64`の新しいメカニズムです。
レガシの stack-switchメカニズムの代替として使われます。
以前の`x86`アーキテクチャは、割込みに応答してスタックフレームを自動的に切り替えるためのメカニズムを提供していました。`IST`は、`x86`のスタック切り替えモードの修正版です。
このメカニズムは、スタックが有効になった時に無条件にスタックを切り替え、特定の割込みに関連する`IDT`エントリ内の任意の割込みに対して有効にすることができます（すぐに参照される）。
ここから、`IST`は全ての割込みに対して必要ではないことが理解できます。
奥の割込みは、レガシのスタック切り替えモードを使い続けることができます。
`IST`メカニズムは、プロセスに関する情報を含む、特別な構造である [Task State Segment](http://en.wikipedia.org/wiki/Task_state_segment)あるいは`TSS`の中にある `IST`ポインタを最大７つまで提供します。
`TSS`は、Linuxカーネルの割込みハンドラや例外ハンドラの実行の間、スタック切り替えのために使われます。
各ポインタは、`IDT`から、割込みゲートによって参照されます。

<!---
The `Interrupt Descriptor Table` represented by the array of the `gate_desc` structures:
--->
`Interrupt Descriptor Table` は、`gate_desc`構造体の配列で表されます。

```C
extern gate_desc idt_table[];
```

where `gate_desc` is:

```C
#ifdef CONFIG_X86_64
...
...
...
typedef struct gate_struct64 gate_desc;
...
...
...
#endif
```

and `gate_struct64` defined as:

```C
struct gate_struct64 {
        u16 offset_low;
        u16 segment;
        unsigned ist : 3, zero0 : 5, type : 5, dpl : 2, p : 1;
        u16 offset_middle;
        u32 offset_high;
        u32 zero1;
} __attribute__((packed));
```

<!---
Each active thread has a large stack in the Linux kernel for the `x86_64` architecture. The stack size is defined as `THREAD_SIZE` and is equal to:
--->
`x86_64`アーキテクチャのLinuxカーネルのアクティブなスレッドは、おおきなスタックを持ちます。
スタックサイズは `THREAD_SIZE`で定義され、その値は以下のとおりです：

```C
#define PAGE_SHIFT      12
#define PAGE_SIZE       (_AC(1,UL) << PAGE_SHIFT)
...
...
...
#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

<!---
The `PAGE_SIZE` is `4096`-bytes and the `THREAD_SIZE_ORDER` depends on the `KASAN_STACK_ORDER`. As we can see, the `KASAN_STACK` depends on the `CONFIG_KASAN` kernel configuration parameter and is defined as:
--->
`PAGE_SIZE`は、`4096`バイトで、`THREAD_SIZE_ORDER`は、`KASAN_STACK_ORDER`に依存します。
見てのとおり、`KASAN_STACK`は カーネルコンフィギュレーションパラメタの`CONFIG_KASAN`に依存しており、以下のように定義されます：

```C
#ifdef CONFIG_KASAN
    #define KASAN_STACK_ORDER 1
#else
    #define KASAN_STACK_ORDER 0
#endif
```

<!---
`KASan` is a runtime memory [debugger](http://lwn.net/Articles/618180/). Thus, the `THREAD_SIZE` will be `16384` bytes if `CONFIG_KASAN` is disabled or `32768` if this kernel configuration option is enabled. These stacks contain useful data as long as a thread is alive or in a zombie state. While the thread is in user-space, the kernel stack is empty except for the `thread_info` structure (details about this structure are available in the fourth [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html) of the Linux kernel initialization process) at the bottom of the stack. The active or zombie threads aren't the only threads with their own stack. There also exist specialized stacks that are associated with each available CPU. These stacks are active when the kernel is executing on that CPU. When the user-space is executing on the CPU, these stacks do not contain any useful information. Each CPU has a few special per-cpu stacks as well. The first is the `interrupt stack` used for the external hardware interrupts. Its size is determined as follows:
--->
`KASan(Kernel address sanitizer)`は、[実行時メモリデバッガ](http://lwn.net/Articles/618180/)です。
したがって、`CONFIG_KASAN` が無効の時には `THREAD_SIZE`は `16384`となり、有効の時には`32768` となります。
このスタックは、スレッドが生きている間、または、ゾンビ状態にあるとき、有益なデータを含みます。
スレッドがユーザ空間にあるとき、スタックボトムにある `thread_info`構造体を除いてカーネルスタックは空っぽです。
`thread_info`構造体の詳細については、[Linuxカーネル初期化手順の第四部](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)にあります。
アクティブまたはゾンビスレッドは、自身のスタックを持つ唯一のスレッドではありません。
<!--- [JA]？意味を展開して書きましょう --->
それぞれの有効なCPUに関連する特別なスタックも存在します。
これらのスタックは、カーネルがそのCPUで実行するときに有効です。
ユーザ空間がそのCPUで実行するとき、これらのスタックは有益な情報を含みません。
それぞれのCPUが、少しだけ特別なper-cpuなスタックを持ちます。
最初は、外部ハードウェア割込みで使われる`interrupt stack`です。
そのサイズは以下のように、`16384`バイトと決定されます：

```C
#define IRQ_STACK_ORDER (2 + KASAN_STACK_ORDER)
#define IRQ_STACK_SIZE (PAGE_SIZE << IRQ_STACK_ORDER)
```

<!---
or `16384` bytes. The per-cpu interrupt stack represented by the `irq_stack_union` union in the Linux kernel for `x86_64`:
--->
pe-cpu割込みスタックは、`x86_64`のLinuxカーネルでは、`irq_stack_union`共用体で表されます。


```C
union irq_stack_union {
	char irq_stack[IRQ_STACK_SIZE];

    struct {
		char gs_base[40];
		unsigned long stack_canary;
	};
};
```

<!---
The first `irq_stack` field is a 16 kilobytes array. Also you can see that `irq_stack_union` contains a structure with the two fields:
--->
最初の `irq_stack`フィールドは、１６キロバイトの配列です。
`irq_stack_union`は２つの構造体フィールドを含んでいることが見てとれます：

* `gs_base` - The `gs` register always points to the bottom of the `irqstack` union. On the `x86_64`, the `gs` register is shared by per-cpu area and stack canary (more about `per-cpu` variables you can read in the special [part](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)).  All per-cpu symbols are zero based and the `gs` points to the base of the per-cpu area. You already know that [segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation) is abolished in the long mode, but we can set the base address for the two segment registers - `fs` and `gs` with the [Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register) and these registers can be still be used as address registers. If you remember the first [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) of the Linux kernel initialization process, you can remember that we have set the `gs` register:

* `gs_base` - 常に`irqstack`共用体のボトムを指し示している`gs`レジスタ。`x86_64`では、`gs`レジスタは per-cpu領域と、スタックのかなりあとを共有しています（`per-cpu`変数の詳細は、[special part](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)で読めます）。
すべてのper-cpuシンボルは、ゼロベースで、`gs`はper-cpu領域のベースを指し示しています。
既に[segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation)は、ロングモードでは廃止されていることを知っていますが、[Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register)で、２つのセグメントレジスタ（`fs`と`gs`）にベースアドレスをセットすることができます。
これらのレジスタは、未だアドレスレジスタとして使えます。
Linuxカーネル初期化手順の[最初のパート](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)を覚えているなら、`gs`レジスタをセットしたことも覚えているでしょう：

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

<!---
where `initial_gs` points to the `irq_stack_union`:
--->
ここで `initial_gs`は、 `irq_stack_union` を指しています：

```assembly
GLOBAL(initial_gs)
.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

<!---
* `stack_canary` - [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) for the interrupt stack is a `stack protector`
to verify that the stack hasn't been overwritten. Note that `gs_base` is a 40 bytes array. `GCC` requires that stack canary will be on the fixed offset from the base of the `gs` and its value must be `40` for the `x86_64` and `20` for the `x86`.
--->
* `stack_canary` - 割込みスタックのための [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)は、スタックが上書きされないことを検証するための`stack protector`です。
`gs_base` が４０バイトの配列であることに注意してください。
`GCC`は、スタックカナリアが `gs`のベースから固定オフセットにあることと、その値が`x86_64`の場合に`40`、`x86`の場合に`20`であることを要求します。

<!---
The `irq_stack_union` is the first datum in the `percpu` area, we can see it in the `System.map`:
--->
`irq_stack_union` は、`per-cpu`領域の最初のデータで、`System.map`内で見ることができます：

```
0000000000000000 D __per_cpu_start
0000000000000000 D irq_stack_union
0000000000004000 d exception_stacks
0000000000009000 D gdt_page
...
...
...
```

<!---
We can see its definition in the code:
--->
コードでは、その定義は以下のように見ることができます：

```C
DECLARE_PER_CPU_FIRST(union irq_stack_union, irq_stack_union) __visible;
```

<!---
Now, it's time to look at the initialization of the `irq_stack_union`. Besides the `irq_stack_union` definition, we can see the definition of the following per-cpu variables in the [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h):
--->
さて、`irq_stack_union`の初期化について見て行く時です。
`irq_stack_union`定義の他に、[arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h)内には、以下のper-cpu変数が定義されていることが見えます：

```C
DECLARE_PER_CPU(char *, irq_stack_ptr);
DECLARE_PER_CPU(unsigned int, irq_count);
```

<!---
The first is the `irq_stack_ptr`. From the variable's name, it is obvious that this is a pointer to the top of the stack. The second - `irq_count` is used to check if a CPU is already on an interrupt stack or not. Initialization of the `irq_stack_ptr` is located in the `setup_per_cpu_areas` function in [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup_percpu.c):
--->
最初は、`irq_stack_ptr`です。
変数名から、これがスタックの先頭へのポインタであることは明らかです。
２つめ、`irq_count`は、CPUが既に割込みスタックにあるかどうかを調べるために使われます。
`irq_stack_ptr`の初期化は、[arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup_percpu.c)にある `setup_per_cpu_areas` 関数に配置されます：

```C
void __init setup_per_cpu_areas(void)
{
...
...
#ifdef CONFIG_X86_64
for_each_possible_cpu(cpu) {
    ...
    ...
    ...
    per_cpu(irq_stack_ptr, cpu) =
            per_cpu(irq_stack_union.irq_stack, cpu) +
            IRQ_STACK_SIZE - 64;
    ...
    ...
    ...
#endif
...
...
}
```

<!---
Here we go over all the CPUs one-by-one and setup `irq_stack_ptr`. This turns out to be equal to the top of the interrupt stack minus `64`. Why `64`?TODO  [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c) source code file is following:
--->
ここではすべてのCPUを1つずつ調べ、 `irq_stack_ptr`を設定します。
これは、割り込みスタックの先頭から`64`を引いたものと等しくなることが分かります。

<!--- [JA][TODO] feedback "TODO" --->
Why `64`? TODO
-> answer. remove it from v4.9-rc1

```
commit 4950d6d48a0c43cc61d0bbb76fb10e0214b79c66
Author: Josh Poimboeuf <jpoimboe@redhat.com>
Date:   Thu Aug 18 10:59:08 2016 -0500

    x86/dumpstack: Remove 64-byte gap at end of irq stack
    
    There has been a 64-byte gap at the end of the irq stack for at least 12
    years.  It predates git history, and I can't find any good reason for
    it.  Remove it.  What's the worst that could happen?
    
    Signed-off-by: Josh Poimboeuf <jpoimboe@redhat.com>
...
    Link: http://lkml.kernel.org/r/14f9281c5475cc44af95945ea7546bff2e3836db.1471535549.git.jpoimboe@redhat.com
    Signed-off-by: Ingo Molnar <mingo@kernel.org>
```

ソースコードファイルは [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c) で、以下のとおりです：


```C
void load_percpu_segment(int cpu)
{
        ...
        ...
        ...
        loadsegment(gs, 0);
        wrmsrl(MSR_GS_BASE, (unsigned long)per_cpu(irq_stack_union.gs_base, cpu));
}
```

and as we already know the `gs` register points to the bottom of the interrupt stack.

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr

	GLOBAL(initial_gs)
	.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

<!---
Here we can see the `wrmsr` instruction which loads the data from `edx:eax` into the [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register) pointed by the `ecx` register. In our case the model specific register is `MSR_GS_BASE` which contains the base address of the memory segment pointed by the `gs` register. `edx:eax` points to the address of the `initial_gs` which is the base address of our `irq_stack_union`.
--->
ここで、`edx:eax`のデータを`ecx`レジスタが指し示す[Model specific register](http://en.wikipedia.org/wiki/Model-specific_register) へロードする`wrmr`命令が見えます。
この場合、Model Specificレジスタは、`gs`レジスタに指し示されているメモリセグメントのベースアドレスを含む`MSR_GS_BASE`です。`edx:eax`は、`irq_stack_union`のベースアドレスである `initial_gs`のアドレスを指し示します。

<!---
We already know that `x86_64` has a feature called `Interrupt Stack Table` or `IST` and this feature provides the ability to switch to a new stack for events non-maskable interrupt, double fault etc. There can be up to seven `IST` entries per-cpu. Some of them are:
--->
既に私達は、`x86_64`は`Interrupt Stack Table` ないし `IST`と呼ばれる機能を有することを知っています。
そして、この機能は、マスカブルではない割り込み、ダブルフォールトなどのイベントのために新しいスタックに切り替える機能を提供します。
CPU毎に最大７つの`IST`エントリがあります。いくつか例を示します：

* `DOUBLEFAULT_STACK`
* `NMI_STACK`
* `DEBUG_STACK`
* `MCE_STACK`

or

```C
#define DOUBLEFAULT_STACK 1
#define NMI_STACK 2
#define DEBUG_STACK 3
#define MCE_STACK 4
```

<!---
All interrupt-gate descriptors which switch to a new stack with the `IST` are initialized with the `set_intr_gate_ist` function. For example:
--->
`IST`で新しいスタックに切り替える全ての割込みゲートデスクリプタは、`set_intr_gate_ist`関数で初期化されます。例えば：

```C
set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
...
...
...
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

<!---
where `&nmi` and `&double_fault` are addresses of the entries to the given interrupt handlers:
--->
ここで、`&nmi` と `&double_fault` は、与えられた割込みハンドラへのエントリのアドレスです。

```C
asmlinkage void nmi(void);
asmlinkage void double_fault(void);
```

<!---
defined in the [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/entry_64.S)
--->
[arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/entry_64.S)で定義されています。

```assembly
idtentry double_fault do_double_fault has_error_code=1 paranoid=2
...
...
...
ENTRY(nmi)
...
...
...
END(nmi)
```

<!---
When an interrupt or an exception occurs, the new `ss` selector is forced to `NULL` and the `ss` selector’s `rpl` field is set to the new `cpl`. The old `ss`, `rsp`, register flags, `cs`, `rip` are pushed onto the new stack. In 64-bit mode, the size of interrupt stack-frame pushes is fixed at 8-bytes, so we will get the following stack:
--->
割込みまたは例外が発生した時、新しい`ss`セレクタは強制的に`NULL`になり、`ss`セレクタの`rpl`フィールドは、新しい`cpl`になります。
古い`ss`、`rsp`、レジスタフラグ、`cs`、`rip`は、新しいスタックにプッシュされます。
64bitモードでは、割込みスタックフレームへプッシュする大きさは固定で８バイトなので、スタックは以下のようになります：

```
+---------------+
|               |
|      SS       | 40
|      RSP      | 32
|     RFLAGS    | 24
|      CS       | 16
|      RIP      | 8
|   Error code  | 0
|               |
+---------------+
```

<!---
If the `IST` field in the interrupt gate is not `0`, we read the `IST` pointer into `rsp`. If the interrupt vector number has an error code associated with it, we then push the error code onto the stack. If the interrupt vector number has no error code, we go ahead and push the dummy error code on to the stack. We need to do this to ensure stack consistency. Next, we load the segment-selector field from the gate descriptor into the CS register and must verify that the target code-segment is a 64-bit mode code segment by the checking bit `21` i.e. the `L` bit in the `Global Descriptor Table`. Finally we load the offset field from the gate descriptor into `rip` which will be the entry-point of the interrupt handler. After this the interrupt handler begins to execute and when the interrupt handler finishes its execution, it must return control to the interrupted process with the `iret` instruction. The `iret` instruction unconditionally pops the stack pointer (`ss:rsp`) to restore the stack of the interrupted process and does not depend on the `cpl` change.
--->
割込みゲート内の `IST` フィールドが `0` でなければ、`IST`ポインタを`rsp`へ読み込みます。
割込みヴェクタ番号が、それに付随するエラーコードを有しているなら、何処かへ行って、スタックにダミーのエラーコードをプッシュします。
スタックの整合性を確保するためには、これを行う必要があります。
つぎに、ゲートデスクリプタからセグメントセレクタフィールドをCSレジスタへ読み込みます。
そして、ターゲットのコードセグメントが、ビット`21`（`GDT`の`L`ビット）を確認して64bitモードのコードセグメントであることを検証しなければなりません。
最後に、ゲートデスクリプタから 割込みハンドラのエントリポイントであるoffsetフィールドを`rip`に読み込みます。
このあと、割込みハンドラが実行を開始し、割込みハンドラがその実行を終了する時には、`iret`命令で割り込み処理の制御から返らなければなりません。
`iret`命令は、割込み処理のスタックを復旧するために無条件にスタックポインタ（`ss:rsp`）をポップします。`cpl`の変化に依存しません。

<!---
That's all.
--->
それで全部です。


まとめ<!--- Conclusion --->
--------------------------------------------------------------------------------

<!---
It is the end of the first part of `Interrupts and Interrupt Handling` in the Linux kernel. We covered some theory and the first steps of initialization of stuffs related to interrupts and exceptions. In the next part we will continue to dive into the more practical aspects of interrupts and interrupt handling.
--->
これは、Linuxカーネルの `Interrupts and Interrupt Handling`の最初のパートの終わりです。 
私たちは、いくつかの理論と、割り込みと例外に関連するスタッフの初期化の最初のステップについて説明しました。
次のパートでは、割り込みと割り込み処理のより実用的な側面について引き続き検討します。

<!---
If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).
--->
ご質問やご意見がありましたら、私にコメントを書いたり、[twitter]（https://twitter.com/0xAX）でpingしてください。

<!---
**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-insides).**
--->
**英語は私の母国語ではありません。ご迷惑をおかけして申し訳ありません。 間違いが見つかった場合は、[linux-insides]（https://github.com/0xAX/linux-insides）にPRを送ってください。**


Links
--------------------------------------------------------------------------------

* [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)
* [Advanced Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [long mode](http://en.wikipedia.org/wiki/Long_mode)
* [kernel stacks](https://www.kernel.org/doc/Documentation/x86/kernel-stacks)
* [Task State Segment](http://en.wikipedia.org/wiki/Task_state_segment)
* [segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation)
* [Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register)
* [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)
* [Previous chapter](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)
