Interrupts and Interrupt Handling. Part 8.
================================================================================

<!---
Non-early initialization of the IRQs
--->

早期ではないIRQの初期化
--------------------------------------------------------------------------------

<!----
This is the eighth part of the Interrupts and Interrupt Handling in the Linux kernel [chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) and in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-7.html) we started to dive into the external hardware [interrupts](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29). We looked on the implementation of the `early_irq_init` function from the [kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/irqdesc.c) source code file and saw the initialization of the `irq_desc` structure in this function. Remind that `irq_desc` structure (defined in the [include/linux/irqdesc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqdesc.h#L46) is the foundation of interrupt management code in the Linux kernel and represents an interrupt descriptor. In this part we will continue to dive into the initialization stuff which is related to the external hardware interrupts.
--->
ここは、[Linuxカーネルの章](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) である、割り込みと割り込み処理の８番目のパートです。
[前のパート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-7.html) では、外部ハードウェア [割り込み](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) へ潜りました。
[kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/irqdesc.c) ソースコードファイルにある、`early_irq_init` 関数へ潜りはじめ、その関数の `irq_desc` 構造体の初期化について見ました。
`irq_desc` 構造体（ [include/linux/irqdesc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqdesc.h#L46) で定義されています）は、Linuxカーネルの割り込み管理コードの基盤で、割り込みデスクリプ他として表されます。
このパートでは、外部ハードウェア割り込みに関することの初期化処理に潜っていきます。

<!---
Right after the call of the `early_irq_init` function in the [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) we can see the call of the `init_IRQ` function. This function is architecture-specific and defined in the [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c). The `init_IRQ` function makes initialization of the `vector_irq` [percpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) variable that defined in the same [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) source code file:
--->
[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) で、 `early_irq_init` 関数の呼び出しのあとすぐに、 `init_IRQ` 関数の呼び出しをみることができます。
この関数は、アーキテクチャ依存で、 [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) に定義されています。
`init_IRQ` 関数は、同じ [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) に定義された `vector_irq` [percpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) 変数の初期化を行います。

```C
...
DEFINE_PER_CPU(vector_irq_t, vector_irq) = {
         [0 ... NR_VECTORS - 1] = -1,
};
...
```

<!---
and represents `percpu` array of the interrupt vector numbers. The `vector_irq_t` defined in the [arch/x86/include/asm/hw_irq.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/hw_irq.h) and expands to the:
--->
そして、 `CPU毎(percpu)` の割り込みベクタ番号の配列を表します。
`vector_irq_t` は [arch/x86/include/asm/hw_irq.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/hw_irq.h) で定義され、以下のように展開します：

```C
typedef int vector_irq_t[NR_VECTORS];
```

<!---
where `NR_VECTORS` is count of the vector number and as you can remember from the first [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) of this chapter it is `256` for the [x86_64](https://en.wikipedia.org/wiki/X86-64):
--->
ここで `NR_VECTORS` は、ヴェクタ番号の数で、この章の最初の [パート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) から思い出せるように、その値は [x86_64](https://en.wikipedia.org/wiki/X86-64) では `256` です。

```C
#define NR_VECTORS                       256
```

<!---
So, in the start of the `init_IRQ` function we fill the `vector_irq` [percpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) array with the vector number of the `legacy` interrupts:
--->
したがって、 `init_IRQ` 関数の最初では、 [CPU毎(percpu)](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) 配列を `legacy` 割り込みのヴェクタ番号で埋めます：

```C
void __init init_IRQ(void)
{
	int i;

	for (i = 0; i < nr_legacy_irqs(); i++)
		per_cpu(vector_irq, 0)[IRQ0_VECTOR + i] = i;
...
...
...
}
```

<!---
This `vector_irq` will be used during the first steps of an external hardware interrupt handling in the `do_IRQ` function from the [arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irq.c):
--->
この `vector_irq` は、 [arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irq.c) の `do_IRQ` 関数内で外部ハードウェア割り込み処理の最初のステップの間使われます：

```C
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
	...
	...
	...
	irq = __this_cpu_read(vector_irq[vector]);

	if (!handle_irq(irq, regs)) {
		...
		...
		...
	}

	exiting_irq();
	...
	...
	return 1;
}
```

<!---
Why is `legacy` here? Actually all interrupts are handled by the modern [IO-APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#I.2FO_APICs) controller. But these interrupts (from `0x30` to `0x3f`) by legacy interrupt-controllers like [Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Programmable_Interrupt_Controller). If these interrupts are handled by the `I/O APIC` then this vector space will be freed and re-used. Let's look on this code closer. First of all the `nr_legacy_irqs` defined in the [arch/x86/include/asm/i8259.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/i8259.h) and just returns the `nr_legacy_irqs` field from the `legacy_pic` structure:
--->
なぜ ここで `legacy` なのでしょうか。
実際に、すべての割り込みは、モダンな [IO-APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#I.2FO_APICs) コントローラ によって処理されます。
しかし、これらの割り込み（ `0x30` から `0x3f` ）は、[Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) のようなレガシー割り込みコントローラによって処理されます。
これらの割り込みが `IO APIC`で処理されるなら、このヴェクタ空間は解放され、再利用されます。
このコードを注意深く見てみましょう。
最初に、 [arch/x86/include/asm/i8259.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/i8259.h) に定義された `nr_legacy_irqs` 関数 が `legacy_pic` 構造体の `nr_legacy_irqs` フィールドを返します：

```C
static inline int nr_legacy_irqs(void)
{
        return legacy_pic->nr_legacy_irqs;
}
```

<!---
This structure defined in the same header file and represents non-modern programmable interrupts controller:
--->
この構造体は、同じヘッダファイルに定義され、モダンではないプログラマブル割り込みコントローラを表します：

```C
struct legacy_pic {
        int nr_legacy_irqs;
        struct irq_chip *chip;
        void (*mask)(unsigned int irq);
        void (*unmask)(unsigned int irq);
        void (*mask_all)(void);
        void (*restore_mask)(void);
        void (*init)(int auto_eoi);
        int (*irq_pending)(unsigned int irq);
        void (*make_irq)(unsigned int irq);
};
```

<!---
Actual default maximum number of the legacy interrupts represented by the `NR_IRQ_LEGACY` macro from the [arch/x86/include/asm/irq_vectors.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irq_vectors.h):
--->
実際に、
[arch/x86/include/asm/irq_vectors.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irq_vectors.h) の `NR_IRQ_LEGACY` マクロによって表されるでフォルトのレガシ割り込みの最大値は：

```C
#define NR_IRQS_LEGACY                    16
```

<!--- [ja] 誤字 vecto_irq - vector_irq --->
<!---
In the loop we are accessing the `vector_irq` per-cpu array with the `per_cpu` macro by the `IRQ0_VECTOR + i` index and write the legacy vector number there. The `IRQ0_VECTOR` macro defined in the [arch/x86/include/asm/irq_vectors.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irq_vectors.h) header file and expands to the `0x30`:
--->
ループ中、 `IRQ0_VECTOR + i` をインデックスとして、`per_cpu` マクロでCPU毎の `vecto_irq` 配列をアクセスし、レガシヴェクタ番号を書き込んでいます。
`IRQ0_VECTOR` マクロは、[arch/x86/include/asm/irq_vectors.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irq_vectors.h) ヘッダファイルで定義され、展開すると `0x30` になります：

```C
#define FIRST_EXTERNAL_VECTOR           0x20

#define IRQ0_VECTOR                     ((FIRST_EXTERNAL_VECTOR + 16) & ~15)
```

<!---
Why is `0x30` here? You can remember from the first [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) of this chapter that first 32 vector numbers from `0` to `31` are reserved by the processor and used for the processing of architecture-defined exceptions and interrupts. Vector numbers from `0x30` to `0x3f` are reserved for the [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture). So, it means that we fill the `vector_irq` from the `IRQ0_VECTOR` which is equal to the `32` to the `IRQ0_VECTOR + 16` (before the `0x30`).
--->
なぜ `0x30` なのでしょうか。
この章の最初の[パート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) を思い出してください。
最初の `0` から　`31` の３２個のベクタ番号は、プロセッサによって予約され、アーキテクチャ定義の例外処理と割り込み処理とに使われます。
ヴェクタ番号 `0x30` から `0x3f` は、[ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) のために予約されます。
したがって、 `vector_irq` の `IRQ0_VECTOR` （値は `32` に等しい ） から、 `IRQ0_VECTOR + 16` （ 前述の `0x30` ）を埋めます。

<!---
In the end of the `init_IRQ` function we can see the call of the following function:
--->
`init_IRQ` 関数の最後では、 [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c) にある、以下の関数の呼び出しを見ることができます：

```C
x86_init.irqs.intr_init();
```

<!---
from the [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c) source code file. If you have read [chapter](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) about the Linux kernel initialization process, you can remember the `x86_init` structure. This structure contains a couple of files which are points to the function related to the platform setup (`x86_64` in our case), for example `resources` - related with the memory resources, `mpparse` - related with the parsing of the [MultiProcessor Configuration Table](https://en.wikipedia.org/wiki/MultiProcessor_Specification) table and etc.). As we can see the `x86_init` also contains the `irqs` field which contains three following fields:
--->
すでにLinuxカーネル初期化手順に関する[章](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) を読んでいたなら、 `x86_init` 構造体を思い出してください。
この構造体はプラットフォーム（ここでは `x86_64` ）セットアップに関する関数をポイントする、いくつかのファイルを含んでいます。
例えば、 `resources` は、メモリリソースに関するもの、 `mpparse` は[MultiProcessor Configuration Table](https://en.wikipedia.org/wiki/MultiProcessor_Specification) テーブルの解析に関するもの、など。
`x86_init` 構造体が、以下の三つのフィールドを含んだ `irqs` フィールドも含んでいることを見ることができます：

```C
struct x86_init_ops x86_init __initdata 
{
	...
	...
	...
    .irqs = {
                .pre_vector_init        = init_ISA_irqs,
                .intr_init              = native_init_IRQ,
                .trap_init              = x86_init_noop,
	},
	...
	...
	...
}
```

<!---
Now, we are interesting in the `native_init_IRQ`. As we can note, the name of the `native_init_IRQ` function contains the `native_` prefix which means that this function is architecture-specific. It defined in the [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) and executes general initialization of the [Local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#Integrated_local_APICs) and initialization of the [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) irqs. Let's look on the implementation of the `native_init_IRQ` function and will try to understand what occurs there. The `native_init_IRQ` function starts from the execution of the following function:
--->
さて、 `native_init_IRQ` に興味がわいてきましたね。
`native_init_IRQ` 関数の名前が、 `native_` プリフィックスを含んでいることに注意しましょう。
[arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) で定義され、 [Local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#Integrated_local_APICs) の一般的な初期化を実行し、[ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) 割り込みの初期化を実行します。
`native_init_IRQ` 関数の実装を見ていきましょう。そして、そこで何がおきているのかを理解してみましょう。
`native_init_IRQ` 関数は、以下の関数の実行からスタートします：

```C
x86_init.irqs.pre_vector_init();
```

<!---
As we can see above, the `pre_vector_init` points to the `init_ISA_irqs` function that defined in the same [source code](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) file and as we can understand from the function's name, it makes initialization of the `ISA` related interrupts. The `init_ISA_irqs` function starts from the definition of the `chip` variable which has a `irq_chip` type:
--->
上を見て判るように、 `pre_vector_init` は 同じ [ソースコード](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) で定義された `init_ISA_irqs` 関数を指し示しています。
関数名から理解できるように、割り込みに関する `ISA` の初期化処理を行います。
`init_ISA_irqs` 関数は `irq_chip` 型を持つ `chip` 変数の定義から始まります：


```C
void __init init_ISA_irqs(void)
{
	struct irq_chip *chip = legacy_pic->chip;
	...
	...
	...
```

<!---
The `irq_chip` structure defined in the [include/linux/irq.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irq.h) header file and represents hardware interrupt chip descriptor. It contains:
--->
`irq_chip` 構造体は、 [include/linux/irq.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irq.h) ヘッダファイルで定義され、ハードウェア割り込みチップのデスクリプタを表しています。
それは以下のフィールドを含みます：

<!---
* `name` - name of a device. Used in the `/proc/interrupts`:
--->
* `name` - デバイスの名前。`/proc/interrupts` で使われます：

```C
$ cat /proc/interrupts
           CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7       
  0:         16          0          0          0          0          0          0          0   IO-APIC   2-edge      timer
  1:          2          0          0          0          0          0          0          0   IO-APIC   1-edge      i8042
  8:          1          0          0          0          0          0          0          0   IO-APIC   8-edge      rtc0
```

<!---
look on the last column;
--->
最後の列を見てください。

* `(*irq_mask)(struct irq_data *data)`  - mask an interrupt source;
* `(*irq_ack)(struct irq_data *data)` - start of a new interrupt;
* `(*irq_startup)(struct irq_data *data)` - start up the interrupt;
* `(*irq_shutdown)(struct irq_data *data)` - shutdown the interrupt
* and etc.

<!--- [JA] 誤記 --->
<!---
fields. Note that the `irq_data` structure represents set of the per irq chip data passed down to chip functions. It contains `mask` - precomputed bitmask for accessing the chip registers, `irq` - interrupt number, `hwirq` - hardware interrupt number, local to the interrupt domain, `chip` - low level interrupt hardware access and etc.
--->
`irq_data` 構造体は、割り込みチップ関数へ渡す 割り込みチップデータ毎の集合を表すことに注意してください。
この構造体は、以下のフィールドを含みます：

* `mask` - チップレジスタにアクセスするための計算済みのビットマスク
* `irq` - 割り込み番号
* `hwirq` - ハードウェア割り込み番号。ローカルから割り込みドメイン
* `chip` - 低レベル割り込みハードウェアアクセス
* others... 

<!---
After this depends on the `CONFIG_X86_64` and `CONFIG_X86_LOCAL_APIC` kernel configuration option call the `init_bsp_APIC` function from the [arch/x86/kernel/apic/apic.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/apic/apic.c):
--->
このあと、
`CONFIG_X86_64` と `CONFIG_X86_LOCAL_APIC` カーネルコンフィギュレーションオプションに依存して、[arch/x86/kernel/apic/apic.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/apic/apic.c) から `init_bsp_APIC` 関数を呼び出します。

```C
#if defined(CONFIG_X86_64) || defined(CONFIG_X86_LOCAL_APIC)
	init_bsp_APIC();
#endif
```

<!---
This function makes initialization of the [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) of `bootstrap processor` (or processor which starts first). It starts from the check that we found [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) config (read more about it in the sixth [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-6.html) of the Linux kernel initialization process chapter) and the processor has `APIC`:
--->
この関数は、`bootstrap processor` （あるいは最初に起動したプロセッサ）の [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) の初期化処理を行います。
[SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) の設定（詳細については、Linuxカーネルの初期化手順の章の [６番目のパート](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-6.html)）が有効かつプロセッサが `APIC` を有するかを確認することから始まります：

```C
if (smp_found_config || !cpu_has_apic)
	return;
```

<!--- [JA] わかりにくいので関数冒頭をまるごとコピー --->
```C
p_APIC(void)
{
        unsigned int value;

        /*
         * Don't do the setup now if we have a SMP BIOS as the
         * through-I/O-APIC virtual wire mode might be active.
         */
        if (smp_found_config || !boot_cpu_has(X86_FEATURE_APIC))
                return;

        /*
         * Do not trust the local APIC being empty at bootup.
         */
        clear_local_APIC();

        /*
         * Enable APIC.
         */
        value = apic_read(APIC_SPIV);
        value &= ~APIC_VECTOR_MASK;
        value |= APIC_SPIV_APIC_ENABLED;

#ifdef CONFIG_X86_32
        /* This bit is reserved on P4/Xeon and should be cleared */
        if ((boot_cpu_data.x86_vendor == X86_VENDOR_INTEL) &&
            (boot_cpu_data.x86 == 15))
                value &= ~APIC_SPIV_FOCUS_DISABLED;
        else
#endif
                value |= APIC_SPIV_FOCUS_DISABLED;
        value |= SPURIOUS_APIC_VECTOR;
        apic_write(APIC_SPIV, value);
```

<!---
In other way we return from this function. In the next step we call the `clear_local_APIC` function from the same source code file that shutdowns the local `APIC` (more about it will be in the chapter about the `Advanced Programmable Interrupt Controller`) and enable `APIC` of the first processor by the setting `unsigned int value` to the `APIC_SPIV_APIC_ENABLED`: 
--->
ほかの方法ではこの関数から戻ります。
次のステップは、同じソースコードファイルから ローカル `APIC` を停止する `clear_local_APIC` 関数を呼び出し、 `unsigned int value` を `APIC_SPIV_APIC_ENABLED` に設定することで、最初のプロセッサの `APIC`を有効にします：

```C
value = apic_read(APIC_SPIV);
value &= ~APIC_VECTOR_MASK;
value |= APIC_SPIV_APIC_ENABLED;
```
<!---
and writing it with the help of the `apic_write` function:
--->
`apic_write` 関数の助けを借りて書き込みます：

```C
apic_write(APIC_SPIV, value);
```

<!---
After we have enabled `APIC` for the bootstrap processor, we return to the `init_ISA_irqs` function and in the next step we initialize legacy `Programmable Interrupt Controller` and set the legacy chip and handler for the each legacy irq:
--->
ブートストラッププロセッサのために `APIC` を有効にしたあと、 `init_ISA_irqs` 関数に戻ります。次のステップでは、レガシの `Programmable Interrupt Controller` を初期化し、それぞれのレガシ割り込みのためにレガシチップとハンドラを設定します。

```C
legacy_pic->init(0);
 
for (i = 0; i < nr_legacy_irqs(); i++)
    irq_set_chip_and_handler(i, chip, handle_level_irq);
```

<!---
Where can we find `init` function? The `legacy_pic` defined in the [arch/x86/kernel/i8259.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/i8259.c) and it is:
--->
`init` 関数はどこで見つけられるのでしょうか。
`legacy_pic` は[arch/x86/kernel/i8259.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/i8259.c) に定義されており、それは以下です：

```C
struct legacy_pic *legacy_pic = &default_legacy_pic;
```

<!---
Where the `default_legacy_pic` is:
--->
ここで、 `default_legacy_pic` は以下のとおり：

```C
struct legacy_pic default_legacy_pic = {
	...
	...
	...
	.init = init_8259A,
	...
	...
	...
}
```

<!---
The `init_8259A` function defined in the same source code file and executes initialization of the [Intel 8259](https://en.wikipedia.org/wiki/Intel_8259) ``Programmable Interrupt Controller` (more about it will be in the separate chapter about `Programmable Interrupt Controllers` and `APIC`).
--->
`init_8259A` 関数は同じソースコードファイルに定義されており、 [Intel 8259](https://en.wikipedia.org/wiki/Intel_8259) ``Programmable Interrupt Controller`` （PICに関する詳細は、別の `Programmable Interrupt Controllers` と `APIC` に関する章に書くつもりです） の初期化処理を実行します。

<!---
Now we can return to the `native_init_IRQ` function, after the `init_ISA_irqs` function finished its work. The next step is the call of the `apic_intr_init` function that allocates special interrupt gates which are used by the [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) architecture for the [Inter-processor interrupt](https://en.wikipedia.org/wiki/Inter-processor_interrupt). The `alloc_intr_gate` macro from the [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h) used for the interrupt descriptor allocation:
--->
`init_ISA_irqs` 関数がその仕事を完了した後、 `native_init_IRQ` 関数に戻ることができます。
次のステップは、 [Inter-processor interrupt](https://en.wikipedia.org/wiki/Inter-processor_interrupt) のために [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) アーキテクチャで使われる特別な割り込みゲートを確保する `apic_intr_init` 関数の呼び出しです。

```C
#define alloc_intr_gate(n, addr)                        \
do {                                                    \
        alloc_system_vector(n);                         \
        set_intr_gate(n, addr);                         \
} while (0)
```

<!---
As we can see, first of all it expands to the call of the `alloc_system_vector` function that checks the given vector number in the `used_vectors` bitmap (read previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-7.html) about it) and if it is not set in the `used_vectors` bitmap we set it. After this we test that the `first_system_vector` is greater than given interrupt vector number and if it is greater we assign it:
--->
見て判るように、最初に与えられたヴェクタ番号が `used_vectors` ビットマップ（前の [パート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-7.html) を読んでください）にあるのかの検査を行い、 `used_vectors` でセットされていなければセットする `alloc_system_vector` 関数の呼び出しに展開されます。その後、 `first_system_vector` が与えられた割り込みヴェクタ番号より大きければ、その値を代入します：

```C
tatic inline void alloc_system_vector(int vector)
{
	if (!test_bit(vector, used_vectors)) {
		set_bit(vector, used_vectors);
		if (first_system_vector > vector)
			first_system_vector = vector;
	} else {
		BUG();
	}
}
```

<!---
We already saw the `set_bit` macro, now let's look on the `test_bit` and the `first_system_vector`. The first `test_bit` macro defined in the [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/bitops.h) and looks like this:
--->>
`set_bit` マクロはすでに見ました。今は `test_bit` と、`first_system_vector` とを見ましょう。
最初の `test_bit` マクロは、[arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/bitops.h) で定義されており、以下のとおりです：

```C
#define test_bit(nr, addr)                      \
        (__builtin_constant_p((nr))             \
         ? constant_test_bit((nr), (addr))      \
         : variable_test_bit((nr), (addr)))
```

<!---
We can see the [ternary operator](https://en.wikipedia.org/wiki/Ternary_operation) here make a test with the [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) built-in function `__builtin_constant_p` tests that given vector number (`nr`) is known at compile time. If you're feeling misunderstanding of the `__builtin_constant_p`, we can make simple test:
--->
[三項演算子(ternary operator)](https://en.wikipedia.org/wiki/Ternary_operation) を見ることができます。
[gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) ビルトイン関数である `__builtin_constant_p` は、与えられたヴェクタ番号（ `nr` ）がコンパイル時に判っているかを判定する条件式にあります。
`__builtin_constant_p` の誤った理解を感じるなら、簡単なテストを行うことができます：

```C
#include <stdio.h>

#define PREDEFINED_VAL 1

int main() {
	int i = 5;
	printf("__builtin_constant_p(i) is %d\n", __builtin_constant_p(i));
	printf("__builtin_constant_p(PREDEFINED_VAL) is %d\n", __builtin_constant_p(PREDEFINED_VAL));
	printf("__builtin_constant_p(100) is %d\n", __builtin_constant_p(100));

	return 0;
}
```

<!---
and look on the result:
--->
そして、その結果は以下のとおりです：

```
$ gcc test.c -o test
$ ./test
__builtin_constant_p(i) is 0
__builtin_constant_p(PREDEFINED_VAL) is 1
__builtin_constant_p(100) is 1
```

<!---
Now I think it must be clear for you. Let's get back to the `test_bit` macro. If the `__builtin_constant_p` will return non-zero, we call `constant_test_bit` function:
--->
今、明確にならなければならないと想います。
`test_bit` マクロに戻りましょう。
`__builtin_constant_p` が非ゼロを返したら、`constant_test_bit` 関数を呼び出します：

```C
static inline int constant_test_bit(int nr, const void *addr)
{
	const u32 *p = (const u32 *)addr;

	return ((1UL << (nr & 31)) & (p[nr >> 5])) != 0;
}
```

<!---
and the `variable_test_bit` in other way:
--->
そして、そうではない場合は `variable_test_bit` 関数を呼び出します：
```C
static inline int variable_test_bit(int nr, const void *addr)
{
        u8 v;
        const u32 *p = (const u32 *)addr;

        asm("btl %2,%1; setc %0" : "=qm" (v) : "m" (*p), "Ir" (nr));
        return v;
}
```

<!---
What's the difference between two these functions and why do we need in two different functions for the same purpose? As you already can guess main purpose is optimization. If we will write simple example with these functions:
--->
これらの２つの関数の間の違いはなんでしょうか。同じ目的のために二つの異なる関数を必要とする理由はなんでしょうか。
すでに推測できるように、主な目的は最適化です。
これらの関数で単純な例を書くならば：

```C
#define CONST 25

int main() {
	int nr = 24;
	variable_test_bit(nr, (int*)0x10000000);
	constant_test_bit(CONST, (int*)0x10000000)
	return 0;
}
```

<!---
and will look on the assembly output of our example we will see following assembly code:
--->
そして、例のアセンブリ出力を見ていきましょう。
`constant_test_bit` は、以下のアセンブリコードが見られるでしょう：

```assembly
pushq	%rbp
movq	%rsp, %rbp

movl	$268435456, %esi
movl	$25, %edi
call	constant_test_bit
```
<!---
for the `constant_test_bit`, and:
--->
`variable_test_bit` は、以下のコードが見られるでしょう：

```assembly
pushq	%rbp
movq	%rsp, %rbp

subq	$16, %rsp
movl	$24, -4(%rbp)
movl	-4(%rbp), %eax
movl	$268435456, %esi
movl	%eax, %edi
call	variable_test_bit
```

<!---
for the `variable_test_bit`. These two code listings starts with the same part, first of all we save base of the current stack frame in the `%rbp` register. But after this code for both examples is different. In the first example we put `$268435456` (here the `$268435456` is our second parameter - `0x10000000`) to the `esi` and `$25` (our first parameter) to the `edi` register and call `constant_test_bit`. We put function parameters to the `esi` and `edi` registers because as we are learning Linux kernel for the `x86_64` architecture we use `System V AMD64 ABI` [calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions). All is pretty simple. When we are using predefined constant, the compiler can just substitute its value. Now let's look on the second part. As you can see here, the compiler can not substitute value from the `nr` variable. In this case compiler must calculate its offset on the program's [stack frame](https://en.wikipedia.org/wiki/Call_stack). We subtract `16` from the `rsp` register to allocate stack for the local variables data and put the `$24` (value of the `nr` variable) to the `rbp` with offset `-4`. Our stack frame will be like this:
--->
これらの二つのコードは、同じ部分から開始しています。最初に `%rbp` レジスタに現在のスタックフレームのベースアドレスを保存します。
しかし、このコードの後はどちらの例も異なります。
最初の例では、 `esi` に `$268435456` （ `0x10000000` ）を代入し、 `edi` レジスタに `$25` を代入し、 `constant_test_bit` を呼び出します。
`System V AMD64 ABI` [calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions) を使う `x86_64` アーキテクチャのためのLinuxカーネルを学んでいるので、関数パラメータを `esi` レジスタ と `edi` レジスタ とに格納します。

```
         <- stack grows 

	          %[rbp]
                 |
+----------+ +---------+ +---------+ +--------+
|          | |         | | return  | |        |
|    nr    |-|         |-|         |-|  argc  |
|          | |         | | address | |        |
+----------+ +---------+ +---------+ +--------+
                 |
              %[rsp]
```

<!---
After this we put this value to the `eax`, so `eax` register now contains value of the `nr`. In the end we do the same that in the first example, we put the `$268435456` (the first parameter of the `variable_test_bit` function) and the value of the `eax` (value of `nr`) to the `edi` register (the second parameter of the `variable_test_bit function`). 
--->
このあと、この値を `eax` に格納します。この時点で `eax` レジスタは `nr` の値を含んでいます。
最後に、最初の例と同じようにします。 `$268435456` （`variable_test_bit` 関数の第１引数）を格納し、 `eax` の値（ `nr` の値）を `edi` レジスタ（`variable_test_bit` 関数の第２引数）に格納します。

<!---
The next step after the `apic_intr_init` function will finish its work is the setting interrupt gates from the `FIRST_EXTERNAL_VECTOR` or `0x20` to the `0x256`:
--->
`apic_intr_init` 関数がその仕事を終えた後の次のステップは、割り込みゲートの `FIRST_EXTERNAL_VECTOR` （ `0x20` ）から、 `0x256` までの設定です。

```C
i = FIRST_EXTERNAL_VECTOR;

#ifndef CONFIG_X86_LOCAL_APIC
#define first_system_vector NR_VECTORS
#endif

for_each_clear_bit_from(i, used_vectors, first_system_vector) {
	set_intr_gate(i, irq_entries_start + 8 * (i - FIRST_EXTERNAL_VECTOR));
}
```

<!---
But as we are using the `for_each_clear_bit_from` helper, we set only non-initialized interrupt gates. After this we use the same `for_each_clear_bit_from` helper to fill the non-filled interrupt gates in the interrupt table with the `spurious_interrupt`:
-->
しかし、 `for_each_clear_bit_from` マクロを使っているので、未初期化の割り込みゲートのみを設定します。
このあと、
未初期化の割り込みゲートを`spurious_interrupt` で埋めるために、同じ `for_each_clear_bit_from` マクロを使います。

```C
#ifdef CONFIG_X86_LOCAL_APIC
for_each_clear_bit_from(i, used_vectors, NR_VECTORS)
    set_intr_gate(i, spurious_interrupt);
#endif
```
<!---
Where the `spurious_interrupt` function represent interrupt handler for the `spurious` interrupt. Here the `used_vectors` is the `unsigned long` that contains already initialized interrupt gates. We already filled first `32` interrupt vectors in the `trap_init` function from the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c) source code file:
--->
ここで`spurious_interrupt` 関数は、 `spurious` 割り込みのための割り込みハンドラを表します。
`used_vectors` は、すでに初期化した割り込みゲートを含む `unsigned long` 型です。
最初の `32` の割り込みヴェクタは、 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c) ソースコードファイルの `trap_init` 関数内で、すでに埋めています。

```C
for (i = 0; i < FIRST_EXTERNAL_VECTOR; i++)
    set_bit(i, used_vectors);
```

<!---
You can remember how we did it in the sixth [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-6.html) of this chapter.
--->
この章の [６番目のパート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-6.html) でどのように行ったのかを思い出してください。

<!---
In the end of the `native_init_IRQ` function we can see the following check:
--->
`native_init_IRQ` 関数の最後で、以下のチェックをしていることが見られます：

```C
if (!acpi_ioapic && !of_ioapic && nr_legacy_irqs())
	setup_irq(2, &irq2);
```

<!---
First of all let's deal with the condition. The `acpi_ioapic` variable represents existence of [I/O APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#I.2FO_APICs). It defined in the [arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/acpi/boot.c). This variable set in the `acpi_set_irq_model_ioapic` function that called during the processing `Multiple APIC Description Table`. This occurs during initialization of the architecture-specific stuff in the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c) (more about it we will know in the other chapter about [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)). Note that the value of the `acpi_ioapic` variable depends on the `CONFIG_ACPI` and `CONFIG_X86_LOCAL_APIC` Linux kernel configuration options. If these options did not set, this variable will be just zero:
--->
まず、条件式に対処しましょう。
`acpi_ioapic` 変数は、 [I/O APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#I.2FO_APICs) の存在を表しています。
この変数は [arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/acpi/boot.c) に定義されています。
この変数は `Multiple APIC Description Table` の処理中に呼ばれる `acpi_set_irq_model_ioapic` 関数で設定されます。
これは [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c) 内で、アーキテクチャ依存のものの初期渦中に起こります（詳細については、[APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) に関するほかの章で知ることができます）。
`acpi_ioapic` 変数の値は、 `CONFIG_ACPI` と `CONFIG_X86_LOCAL_APIC` Linuxカーネルコンフィギュレーションオプションに依存します。
これらのオプションがセットされていなければ、この変数はちょうどゼロになります：

```C
#define acpi_ioapic 0
```

<!---
The second condition - `!of_ioapic && nr_legacy_irqs()` checks that we do not use [Open Firmware](https://en.wikipedia.org/wiki/Open_Firmware) `I/O APIC` and legacy interrupt controller. We already know about the `nr_legacy_irqs`. The second is `of_ioapic` variable defined in the [arch/x86/kernel/devicetree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/devicetree.c) and initialized in the `dtb_ioapic_setup` function that build information about `APICs` in the [devicetree](https://en.wikipedia.org/wiki/Device_tree). Note that `of_ioapic` variable depends on the `CONFIG_OF` Linux kernel configuration option. If this option is not set, the value of the `of_ioapic` will be zero too:
--->
２つめの条件、 `!of_ioapic && nr_legacy_irqs()` は、 [Open Firmware](https://en.wikipedia.org/wiki/Open_Firmware) `I/O APIC` を使っていないことと、レガシ割り込みコントローラを使っていないことをチェックしてます。
すでに `nr_legacy_irqs` については知っています。
`of_ioapic` 変数は、[arch/x86/kernel/devicetree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/devicetree.c) で定義されており、 [devicetree](https://en.wikipedia.org/wiki/Device_tree) 内の`APICs` に関する情報を作る `dtb_ioapic_setup` 関数内で初期化されます。
`of_ioapic` 変数は `CONFIG_OF` Linuxカーネルコンフィギュレーションオプションに依存することに注意してください。
もしこのオプションがセットされていなければ、 `of_ioapic` の値もゼロとなります：
```C
#ifdef CONFIG_OF
extern int of_ioapic;
...
...
...
#else
#define of_ioapic 0
...
...
...
#endif
```

<!---
If the condition will return non-zero value we call the:
--->
この条件式が非ゼロの値となるなら、以下の関数を呼びます：

```C
setup_irq(2, &irq2);
```

<!---
function. First of all about the `irq2`. The `irq2` is the `irqaction` structure that defined in the [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irqinit.c) source code file and represents `IRQ 2` line that is used to query devices connected cascade:
--->
最初に `irq2` について。 `irq2` は、[arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irqinit.c) ソースコードファイルで定義されている `irqaction` 構造体で、化スケート接続されたデバイスを照会するために用いる `IRQ 2` ラインを表しています。

```C
static struct irqaction irq2 = {
	.handler = no_action,
    .name = "cascade",
    .flags = IRQF_NO_THREAD,
};
```

<!---
Some time ago interrupt controller consisted of two chips and one was connected to second. The second chip that was connected to the first chip via this `IRQ 2` line. This chip serviced lines from `8` to `15` and after this lines of the first chip. So, for example [Intel 8259A](https://en.wikipedia.org/wiki/Intel_8259) has following lines:
--->
少し昔は割り込みコントローラは２つのチップで構成され、１つは他方のチップに接続されていました。 １つめのチップに接続された２つめのチップは、この `IRQ 2` ラインを介していました。
このチップは、１つめのチップのラインのあとの `8` から `15` のラインをサービスしていました。
したがって、例えば  [Intel 8259A](https://en.wikipedia.org/wiki/Intel_8259) は以下のラインを持っています：

* `IRQ 0`  - system time;
* `IRQ 1`  - keyboard;
* `IRQ 2`  - used for devices which are cascade connected;
* `IRQ 8`  - [RTC](https://en.wikipedia.org/wiki/Real-time_clock);
* `IRQ 9`  - reserved;
* `IRQ 10` - reserved;
* `IRQ 11` - reserved;
* `IRQ 12` - `ps/2` mouse;
* `IRQ 13` - coprocessor;
* `IRQ 14` - hard drive controller;
* `IRQ 1`  - reserved;
* `IRQ 3`  - `COM2` and `COM4`;
* `IRQ 4`  - `COM1` and `COM3`;
* `IRQ 5`  - `LPT2`;
* `IRQ 6`  - drive controller;
* `IRQ 7`  - `LPT1`.

<!---
The `setup_irq` function defined in the [kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/manage.c) and takes two parameters:
--->
[kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/manage.c) に定義された `setup_irq` 関数は、２つのパラメータがあります：

* vector number of an interrupt;
* `irqaction` structure related with an interrupt.

<!---
This function initializes interrupt descriptor from the given vector number at the beginning:
--->
この関数は、与えられたヴェクタ番号の最初から割り込みデスクリプタを初期化します：

```C
struct irq_desc *desc = irq_to_desc(irq);
```

<!---
And call the `__setup_irq` function that setups given interrupt: 
--->
そして、与えられた割り込みを設定する `__setup_irq` 関数を呼び出します：

```C
chip_bus_lock(desc);
retval = __setup_irq(irq, desc, act);
chip_bus_sync_unlock(desc);
return retval;
```

<!---
Note that the interrupt descriptor is locked during `__setup_irq` function will work. The `__setup_irq` function makes many different things: It creates a handler thread when a thread function is supplied and the interrupt does not nest into another interrupt thread, sets the flags of the chip, fills the `irqaction` structure and many many more.
--->
割り込みデスクリプタは、  `__setup_irq` 関数の実行中にロックされることに注意してください。
 `__setup_irq` 関数は多くの異なることを行います：
スレッド関数が与えられるときにはハンドラスレッドを作成し、
割り込みがほかの割り込みスレッドにネストしない
チップのフラグを設定し、
 `irqaction` 構造体を埋めて、、、ほかにも他くさい。
<!---
All of the above it creates `/prov/vector_number` directory and fills it, but if you are using modern computer all values will be zero there:
--->
上記のすべては `/prov/vector_number` ディレクトリを作り、それを埋めます。
しかし、モダンなコンピュータを使っているなら、そこのすべての値はゼロとなるでしょう：

```
$ cat /proc/irq/2/node
0

$cat /proc/irq/2/affinity_hint 
00

cat /proc/irq/2/spurious 
count 0
unhandled 0
last_unhandled 0 ms
```

<!---
because probably `APIC` handles interrupts on the our machine.
--->
なぜならば、おそらく `APIC` が割り込みを処理するからです。

That's all.

Conclusion
--------------------------------------------------------------------------------

It is the end of the eighth part of the [Interrupts and Interrupt Handling](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) chapter and we continued to dive into external hardware interrupts in this part. In the previous part we started to do it and saw early initialization of the `IRQs`. In this part we already saw non-early interrupts initialization in the `init_IRQ` function. We saw initialization of the `vector_irq` per-cpu array which is store vector numbers of the interrupts and will be used during interrupt handling and initialization of other stuff which is related to the external hardware interrupts.

In the next part we will continue to learn interrupts handling related stuff and will see initialization of the `softirqs`.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [percpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Intel 8259](https://en.wikipedia.org/wiki/Intel_8259)
* [Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)
* [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture)
* [MultiProcessor Configuration Table](https://en.wikipedia.org/wiki/MultiProcessor_Specification)
* [Local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#Integrated_local_APICs)
* [I/O APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#I.2FO_APICs)
* [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [Inter-processor interrupt](https://en.wikipedia.org/wiki/Inter-processor_interrupt)
* [ternary operator](https://en.wikipedia.org/wiki/Ternary_operation)
* [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions)
* [PDF. System V Application Binary Interface AMD64](http://x86-64.org/documentation/abi.pdf)
* [Call stack](https://en.wikipedia.org/wiki/Call_stack)
* [Open Firmware](https://en.wikipedia.org/wiki/Open_Firmware)
* [devicetree](https://en.wikipedia.org/wiki/Device_tree)
* [RTC](https://en.wikipedia.org/wiki/Real-time_clock)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-7.html)
