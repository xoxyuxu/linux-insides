Interrupts and Interrupt Handling. Part 2.
================================================================================

Start to dive into interrupt and exceptions handling in the Linux kernel
--------------------------------------------------------------------------------

<!---
We saw some theory about interrupts and exception handling in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) and as I already wrote in that part, we will start to dive into interrupts and exceptions in the Linux kernel source code in this part. As you already can note, the previous part mostly described theoretical aspects and in this part we will start to dive directly into the Linux kernel source code. We will start to do it as we did it in other chapters, from the very early places. We will not see the Linux kernel source code from the earliest [code lines](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292) as we saw it for example in the [Linux kernel booting process](http://0xax.gitbooks.io/linux-insides/content/Booting/index.html) chapter, but we will start from the earliest code which is related to the interrupts and exceptions. In this part we will try to go through the all interrupts and exceptions related stuff which we can find in the Linux kernel source code.
--->
[前のパート]（http：///0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html）で、割り込みと例外処理に関する理論を見てきましたが、すでにそこで書いたように、このパートではLinuxカーネルソースコードの中で割り込みと例外を知り始めるでしょう。
既に注意しておいたように、前のパートは主に理論的な側面を説明しており、このパートではLinuxカーネルのソースコードに直接触れていきます。
私たちは他の章で、非常に早い時期からそれをやっているように、ここでもやり始めます。
我々が見たように、[最初のコード行](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292) のLinuxカーネルソースコードは表示されません。[Linuxカーネルブートプロセス](http：///xax.gitbooks.io/linux-insides/content/Booting/index.html)の章の例を参照してください。
ただし、割り込みと例外に関連する最も初期のコードから開始します。
このパートでは、Linuxカーネルのソースコードにあるすべての割り込みと例外に関連するものを調べようとします。

<!---
If you've read the previous parts, you can remember that the earliest place in the Linux kernel `x86_64` architecture-specific source code which is related to the interrupt is located in the [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c) source code file and represents the first setup of the [Interrupt Descriptor Table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table). It occurs right before the transition into the [protected mode](http://en.wikipedia.org/wiki/Protected_mode) in the `go_to_protected_mode` function by the call of the `setup_idt`:
--->
もし前のパートを読んでいたなら、割込みに関連するLinuxカーネルの`x86_64`アーキテクチャ依存部のソースコードの最も早い場所は、[arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c)に位置しており、[Interrupt Descriptor Table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)の最初のセットアップを表していることを思い出してください。
そのセットアップは、setup_idt`関数の呼び出しによって、`go_to_protected_mode`関数内で[protected mode](http://en.wikipedia.org/wiki/Protected_mode)へ遷移する前に行われます。

```C
void go_to_protected_mode(void)
{
	...
	setup_idt();
	...
}
```

<!---
The `setup_idt` function is defined in the same source code file as the `go_to_protected_mode` function and just loads the address of the `NULL` interrupts descriptor table:
--->
`setup_idt`関数は、`go_to_protected_mode`関数と同じソースコードファイルに定義されており、ちょうど`NULL`割込みデスクリプタテーブルのアドレスをロードします。

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

<!---
where `gdt_ptr` represents a special 48-bit `GDTR` register which must contain the base address of the `Global Descriptor Table`:
--->
ここで `gdt_ptr` は、`Global Descriptor Table`のベースアドレスに含まれる必要のある、特別な48-bitの`GDTR`レジスタを表しています。

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

<!---
Of course in our case the `gdt_ptr` does not represent the `GDTR` register, but `IDTR` since we set `Interrupt Descriptor Table`. You will not find an `idt_ptr` structure, because if it had been in the Linux kernel source code, it would have been the same as `gdt_ptr` but with different name. So, as you can understand there is no sense to have two similar structures which differ only by name. You can note here, that we do not fill the `Interrupt Descriptor Table` with entries, because it is too early to handle any interrupts or exceptions at this point. That's why we just fill the `IDT` with `NULL`.
--->
当然のことながら、 `gdt_ptr`は `GDTR`レジスタではなく、`Interrupt Descriptor Table`を設定しているので`IDTR`レジスタを表しています。
Linuxカーネルソースコード内に、`idt_ptr`構造体を見つけることができないでしょう。
`idt_ptr`構造体は、`gdt_ptr`構造体と名前以外は同じなのです。
したがって、名前だけが異なる２つの同じような構造体を持つことは無駄だと理解できるでしょう。
`Interrupt Descriptor Table`をエントリで埋めていないことに注意してください。
この場所では、どのような割込みや例外も処理するには早すぎるのです。
`IDT`を`NULL`で埋める理由は、このためです。

<!---
After the setup of the [Interrupt descriptor table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table), [Global Descriptor Table](http://en.wikipedia.org/wiki/GDT) and other stuff we jump into [protected mode](http://en.wikipedia.org/wiki/Protected_mode) in the - [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pmjump.S). You can read more about it in the [part](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html) which describes the transition to protected mode.
--->
[Interrupt descriptor table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table), [Global Descriptor Table](http://en.wikipedia.org/wiki/GDT)と、ほかのもののセットアップのあと、[arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pmjump.S)で[プロテクトモード](https://ja.wikipedia.org/wiki/%E3%83%97%E3%83%AD%E3%83%86%E3%82%AF%E3%83%88%E3%83%A2%E3%83%BC%E3%83%89)に遷移します。
これについての詳細は、[プロテクトモードへの遷移について記述したパート](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html)で読むことができます。

<!---
We already know from the earliest parts that entry to protected mode is located in the `boot_params.hdr.code32_start` and you can see that we pass the entry of the protected mode and `boot_params` to the `protected_mode_jump` in the end of the [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c):
--->
既に、前のほうのパートから、`boot_params.hdr.code32_start`に位置するプロテクトモードへ遷移することを知っています。
[arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c)の最後に、プロテクトモードのエントリと`boot_params`とを`protected_mode_jump`へ渡すを見ることができます：

```C
protected_mode_jump(boot_params.hdr.code32_start,
			    (u32)&boot_params + (ds() << 4));
```

<!---
The `protected_mode_jump` is defined in the [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pmjump.S) and gets these two parameters in the `ax` and `dx` registers using one of the [8086](http://en.wikipedia.org/wiki/Intel_8086) calling  [conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions):
--->
`protected_mode_jump`は、[arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pmjump.S)で定義されます。
 [8086](http://en.wikipedia.org/wiki/Intel_8086)の[calling conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)の１つを用いて、`ax`レジスタと`dx`レジスタに２つのパラメータを取ります。

```assembly
GLOBAL(protected_mode_jump)
	...
	...
	...
	.byte	0x66, 0xea		# ljmpl opcode
2:	.long	in_pm32			# offset
	.word	__BOOT_CS		# segment
...
...
...
ENDPROC(protected_mode_jump)
```

<!---
where `in_pm32` contains a jump to the 32-bit entry point:
--->
ここで、`in_mp32`は32bitエントリポイントへのジャンプを含みます。

```assembly
GLOBAL(in_pm32)
	...
	...
	jmpl	*%eax // %eax contains address of the `startup_32`
	...
	...
ENDPROC(in_pm32)
```

<!---
As you can remember the 32-bit entry point is in the [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S) assembly file, although it contains `_64` in its name. We can see the two similar files in the `arch/x86/boot/compressed` directory:
--->
32bitエントリポイントは、アセンブラファイル [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)にあることを思い出せるので、その名前に`_64`が含まれています。
`arch/x86/boot/compressed`ディレクトリには、２つの似たようなファイルが見られます：

* `arch/x86/boot/compressed/head_32.S`.
* `arch/x86/boot/compressed/head_64.S`;

<!---
But the 32-bit mode entry point is the second file in our case. The first file is not even compiled for `x86_64`. Let's look at the [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/Makefile):
--->
しかし、この場合、32bitモードエントリポイントは、２つめのファイルに有ります。
最初のファイルは `x86_64`のためにコンパイルされません。
[arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/Makefile)を見てみましょう：

```
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
...
...
```

<!---
We can see here that `head_*` depends on the `$(BITS)` variable which depends on the architecture. You can find it in the [arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile):
--->
アーキテクチャ依存の`$(BITS)`変数に基づいた`head_*`を見ることができます。
`BITS`は、[arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)に見つけることができます：

```
ifeq ($(CONFIG_X86_32),y)
...
	BITS := 32
else
	BITS := 64
	...
endif
```

<!---
Now as we jumped on the `startup_32` from the [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S) we will not find anything related to the interrupt handling here. The `startup_32` contains code that makes preparations before the transition into [long mode](http://en.wikipedia.org/wiki/Long_mode) and directly jumps in to it. The `long mode` entry is located in `startup_64` and it makes preparations before the [kernel decompression](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html) that occurs in the `decompress_kernel` from the [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/misc.c). After the kernel is decompressed, we jump on the `startup_64` from the [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S). In the `startup_64` we start to build identity-mapped pages. After we have built identity-mapped pages, checked the [NX](http://en.wikipedia.org/wiki/NX_bit) bit, setup the `Extended Feature Enable Register` (see in links), and updated the early `Global Descriptor Table` with the `lgdt` instruction, we need to setup `gs` register with the following code:
--->
今、[arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)から`startup_32`に飛んできたのに、
ここまで割り込み処理に関するものを見つけられていません。
`startup_32`は、[long mode](http://en.wikipedia.org/wiki/Long_mode)へ遷移するする前の準備をするコードで、直接そこへジャンプします。
`long mode`エントリは、`startup_64`に配置され、  [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/misc.c)の `decompress_kernel` で行われる[kernel decompression](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html)よりも前に準備されます。
<!--- [JA] TODO 機能理解 --->
`startup_64`では、identity-mapped pagesを作り始めます。
identity-mapped pagesを作り終えたら、[NX](http://en.wikipedia.org/wiki/NX_bit)ビットを確認し、`Extended Feature Enable Register`レジスタをセットアップします。そして、`lgdt`関数で早期の`Global Descriptor Table`を更新します。
以下のコードで、`gs`レジスタをセットアップする必要があります：

```assembly
movl	$MSR_GS_BASE,%ecx
movl	initial_gs(%rip),%eax
movl	initial_gs+4(%rip),%edx
wrmsr
```

<!---
We already saw this code in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html). First of all pay attention on the last `wrmsr` instruction. This instruction writes data from the `edx:eax` registers to the [model specific register](http://en.wikipedia.org/wiki/Model-specific_register) specified by the `ecx` register. We can see that `ecx` contains `$MSR_GS_BASE` which is declared in the [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/msr-index.h) and looks like:
--->
このコードは、[前のパート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html)で見ています。
最初に、最後の`wrmsr`命令に注意を払います。
この命令は`edx:eax`レジスタから、`ecx`レジスタが与える[model specific register](http://en.wikipedia.org/wiki/Model-specific_register)へデータを書き込みます。
`ecx`は、 [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/msr-index.h)で宣言された `$MSR_GS_BASE` （以下）を含みます。

```C
#define MSR_GS_BASE             0xc0000101
```

<!---
From this we can understand that `MSR_GS_BASE` defines the number of the `model specific register`. Since registers `cs`, `ds`, `es`, and `ss` are not used in the 64-bit mode, their fields are ignored. But we can access memory over `fs` and `gs` registers. The model specific register provides a `back door` to the hidden parts of these segment registers and allows to use 64-bit base address for segment register addressed by the `fs` and `gs`. So the `MSR_GS_BASE` is the hidden part and this part is mapped on the `GS.base` field. Let's look on the `initial_gs`:
--->
ここから、`MSR_GS_BASE`が`model specific register`の番号を定義していることがわかります。
`cs`、`ds`、`es`、`ss`レジスタは、64ビットモードで使用されていないので、これらのフィールドは無効にされます。
しかし、`fs`レジスタと`gs`レジスタとを介してメモリをアクセスすることができます。
model specific registerが、これらのセグメントレジスタの隠れた部分への`back door`を提供し、`fs`と`gs`によってアドレッシングされたセグメントレジスタのための64ビットベースアドレスを使うことを許します。
`MSR_GS_BASE`は隠れた部分なので、この部分は`GS.base`フィールドにマップされます。
`initial_gs`について見てみましょう：

```assembly
GLOBAL(initial_gs)
	.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

<!----
We pass `irq_stack_union` symbol to the `INIT_PER_CPU_VAR` macro which just concatenates the `init_per_cpu__` prefix with the given symbol. In our case we will get the `init_per_cpu__irq_stack_union` symbol. Let's look at the [linker](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vmlinux.lds.S) script. There we can see following definition:
--->
与えられたシンボルに `init_per_cpu__` 接頭語を追加する `INIT_PER_CPU_VAR`マクロに、 `irq_stack_union` シンボルを渡しています。
私達の場合、 `init_per_cpu__irq_stack_union` シンボルを得ます。
[linker](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vmlinux.lds.S) scriptを見てみましょう：

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(irq_stack_union);
```

<!---
It tells us that the address of the `init_per_cpu__irq_stack_union` will be `irq_stack_union + __per_cpu_load`. Now we need to understand where `init_per_cpu__irq_stack_union` and `__per_cpu_load` are what they mean. The first `irq_stack_union` is defined in the [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h) with the `DECLARE_INIT_PER_CPU` macro which expands to call the `init_per_cpu_var` macro:
--->
`init_per_cpu__irq_stack_union` のアドレスは、`irq_stack_union + __per_cpu_load` となることを告げています。
ここで、`init_per_cpu__irq_stack_union` と `__per_cpu_load` とが何処にあるのか、難の意味があるのかを知る必要があります。
最初の `irq_stack_union` は、[arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h) で、`init_per_cpu_var` を呼び出すように展開する `DECLARE_INIT_PER_CPU` マクロで 定義されています。

```C
DECLARE_INIT_PER_CPU(irq_stack_union);

#define DECLARE_INIT_PER_CPU(var) \
       extern typeof(per_cpu_var(var)) init_per_cpu_var(var)

#define init_per_cpu_var(var)  init_per_cpu__##var
```

<!---
If we expand all macros we will get the same `init_per_cpu__irq_stack_union` as we got after expanding the `INIT_PER_CPU` macro, but you can note that it is not just a symbol, but a variable. Let's look at the `typeof(per_cpu_var(var))` expression. Our `var` is `irq_stack_union` and the `per_cpu_var` macro is defined in the [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/percpu.h):
--->
全てのマクロを展開すると、後ほど `INIT_PER_CPU` マクロを 展開して得られるように `init_per_cpu__irq_stack_union` と同じものを得ます。
しかし、シンボルではなく、変数であることに注意できます。
`typeof(per_cpu_var(var))` 式を見ましょう。
私達の場合、`var` は、 `irq_stack_union` で、`per_cpu_var` マクロは [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/percpu.h) で定義されています：

```C
#define PER_CPU_VAR(var)        %__percpu_seg:var
```

<!---
where:
--->
ここで：

```C
#ifdef CONFIG_X86_64
    #define __percpu_seg gs
endif
```

<!---
So, we are accessing `gs:irq_stack_union` and getting its type which is `irq_union`. Ok, we defined the first variable and know its address, now let's look at the second `__per_cpu_load` symbol. There are a couple of `per-cpu` variables which are located after this symbol. The `__per_cpu_load` is defined in the [include/asm-generic/sections.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic-sections.h):
--->
したがって、`gs:irq_stack_union`をアクセスし、その型は `irq_union` です。
さて、１つめの変数を定義し、そのアドレスを知りました。２つめの `__per_cpu_load` シンボルを見ていきましょう。
このシンボルの後に位置する、いくつかの `per-cpu` 変数があります。
`__per_cpu_load` は、 [include/asm-generic/sections.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic-sections.h)で定義されています：

```C
extern char __per_cpu_load[], __per_cpu_start[], __per_cpu_end[];
```

<!---
and presented base address of the `per-cpu` variables from the data area. So, we know the address of the `irq_stack_union`, `__per_cpu_load` and we know that `init_per_cpu__irq_stack_union` must be placed right after `__per_cpu_load`. And we can see it in the [System.map](http://en.wikipedia.org/wiki/System.map):
--->
そして、`per-cpu` 変数のベースアドレスは、データ領域から存在します。
`irq_stack_union` のアドレスを知り、`init_per_cpu__irq_stack_union` は、`__per_cpu_load` の直後に置く必要があることを知ります。
それは、 [System.map](http://en.wikipedia.org/wiki/System.map) で見ることができます：

```
...
...
...
ffffffff819ed000 D __init_begin
ffffffff819ed000 D __per_cpu_load
ffffffff819ed000 A init_per_cpu__irq_stack_union
...
...
...
```

<!---
Now we know about `initial_gs`, so let's look at the code:
--->
`initial_gs`についてわかりましたので、コードで見てみましょう：

```assembly
movl	$MSR_GS_BASE,%ecx
movl	initial_gs(%rip),%eax
movl	initial_gs+4(%rip),%edx
wrmsr
```

<!---
Here we specified a model specific register with `MSR_GS_BASE`, put the 64-bit address of the `initial_gs` to the `edx:eax` pair and execute the `wrmsr` instruction for filling the `gs` register with the base address of the `init_per_cpu__irq_stack_union` which will be at the bottom of the interrupt stack. After this we will jump to the C code on the `x86_64_start_kernel` from the [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head64.c). In the `x86_64_start_kernel` function we do the last preparations before we jump into the generic and architecture-independent kernel code and one of these preparations is filling the early `Interrupt Descriptor Table` with the interrupts handlers entries or `early_idt_handlers`. You can remember it, if you have read the part about the [Early interrupt and exception handling](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html) and can remember following code:
--->
`MSR_GS_BASE` で model specific register を与え、`initial_gs` の64bitアドレスを、`edx:eax`ペアレジスタに代入し、
割込みスタックのそこに位置することになる `init_per_cpu__irq_stack_union` のベースアドレスで `gs` レジスタを埋めるために `wrmsr` 命令を実行します。
このあと、Cコードの[arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head64.c) にある、`x86_64_start_kernel` へジャンプします。
`x86_64_start_kernel`関数では、汎用的でアーキテクチャに依存しないカーネルコードに飛び込む前に最後の準備を行い、初期の `Interrupt Descriptor Table` を割り込みハンドラエントリまたは `early_idt_handlers`で埋めています。
[Early interrupt and exception handling](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html) に関するパートを読んでいるなら、以下のコードを思い出すことができます：

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handlers[i]);

load_idt((const struct desc_ptr *)&idt_descr);
```

<!---
but I wrote `Early interrupt and exception handling` part when Linux kernel version was - `3.18`. For this day actual version of the Linux kernel is `4.1.0-rc6+` and ` Andy Lutomirski` sent the [patch](https://lkml.org/lkml/2015/6/2/106) and soon it will be in the mainline kernel that changes behaviour for the `early_idt_handlers`. **NOTE** While I wrote this part the [patch](https://github.com/torvalds/linux/commit/425be5679fd292a3c36cb1fe423086708a99f11a) already turned in the Linux kernel source code. Let's look on it. Now the same part looks like:
--->
しかし、`Early interrupt and exception handling` のパートを書いたのは、Linuxカーネルバージョンが `3.18`の時でした。
昨今のLinuxカーネルバージョンは `4.1.0-rc6+` で、`Andy Lutomirski` が[patch](https://lkml.org/lkml/2015/6/2/106) を送り、
直ぐにメインラインカーネルへ取り込まれ、`early_idt_handlers` の挙動を変えました。
**注意** 私は [patch](https://github.com/torvalds/linux/commit/425be5679fd292a3c36cb1fe423086708a99f11a) が既に適用されたもので書いています。
それについて見ていきましょう。同じ部位は以下のように見えます。

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handler_array[i]);

load_idt((const struct desc_ptr *)&idt_descr);
```

<!---
AS you can see it has only one difference in the name of the array of the interrupts handlers entry points. Now it is `early_idt_handler_arry`:
--->
見て判るように、たった一つの差異があり、割込みハンドラのエントリポイントの配列名が異なるだけです。
`early_idt_handler_arry` は：

```C
extern const char early_idt_handler_array[NUM_EXCEPTION_VECTORS][EARLY_IDT_HANDLER_SIZE];
```

where `NUM_EXCEPTION_VECTORS` and `EARLY_IDT_HANDLER_SIZE` are defined as:

```C
#define NUM_EXCEPTION_VECTORS 32
#define EARLY_IDT_HANDLER_SIZE 9
```

<!---
So, the `early_idt_handler_array` is an array of the interrupts handlers entry points and contains one entry point on every nine bytes. You can remember that previous `early_idt_handlers` was defined in the [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S). The `early_idt_handler_array` is defined in the same source code file too:  
--->
`early_idt_handler_array` は割込みハンドラのエントリポイントの配列なので、９バイト毎に１つのエントリポイントを含んでいます。
以前の `early_idt_handlers` は、[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S) 内に定義されていたことを思い出せます。
`early_idt_handler_array` は、同じソースコードファイルで定義されています。

```assembly
ENTRY(early_idt_handler_array)
...
...
...
ENDPROC(early_idt_handler_common)
```

<!---
It fills `early_idt_handler_arry` with the `.rept NUM_EXCEPTION_VECTORS` and contains entry of the `early_make_pgtable` interrupt handler (more about its implementation you can read in the part about [Early interrupt and exception handling](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html)). For now we come to the end of the `x86_64` architecture-specific code and the next part is the generic kernel code. Of course you already can know that we will return to the architecture-specific code in the `setup_arch` function and other places, but this is the end of the `x86_64` early code.
--->
`.rept NUM_EXCEPTION_VECTORS` で、`early_idt_handler_arry` を埋めており、
`early_make_pgtable` 割込みハンドラのエントリを含んでいます（その実装についての詳細は、[Early interrupt and exception handling](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html)を読むことができます）。
ここまでで、`x86_64` アーキテクチャ依存コードの終わりまで来ました。次のパートは、ジェネリックなカーネルコードです。
もちろん、すでに `setup_arch` 関数でアーキテクチャ依存コードへ返ってくることを知っていますし、他でもそうです。しかし、`x86_64` の早期コードの終了です。


<!--- Setting stack canary for the interrupt stack --->
割込みスタックのスタックカナリアのセッティング
-------------------------------------------------------------------------------

<!---
The next stop after the [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S) is the biggest `start_kernel` function from the [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c). If you've read the previous [chapter](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) about the Linux kernel initialization process, you must remember it. This function does all initialization stuff before kernel will launch first `init` process with the [pid](https://en.wikipedia.org/wiki/Process_identifier) - `1`. The first thing that is related to the interrupts and exceptions handling is the call of the `boot_init_stack_canary` function.
--->
[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S) のあとの次のストップは、[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) にある 最も大きな `start_kernel` 関数です。
すでに Linuxカーネル初期化手順に関する [前の章](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) を読んでいるなら、思い出してください。
この関数は、カーネルが最初の [pid](https://en.wikipedia.org/wiki/Process_identifier) が１である `init` プロセスをローンチする前に、全ての初期化処理を行います。
最初の、割込みに関することと、割り込み処理に関するものは、 `boot_init_stack_canary` 関数の呼び出しです。

<!---
This function sets the [canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) value to protect interrupt stack overflow. We already saw a little some details about implementation of the `boot_init_stack_canary` in the previous part and now let's take a closer look on it. You can find implementation of this function in the [arch/x86/include/asm/stackprotector.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/stackprotector.h) and its depends on the `CONFIG_CC_STACKPROTECTOR` kernel configuration option. If this option is not set this function will not do anything:
--->
この関数は、割込みスタックのオーバーフローを保護するために [canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) の値をセットします。
すでに、前のパートで `boot_init_stack_canary` の実装について詳細を少し見ました。
今はそれを詳しく見てみましょう。
この関数の実装は、 [arch/x86/include/asm/stackprotector.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/stackprotector.h) で見ることができ、 `CONFIG_CC_STACKPROTECTOR` カーネルコンフィギュレーションオプションに依存しています。
もしこのオプションがセットされていなければ、この関数は何もしません：

```C
#ifdef CONFIG_CC_STACKPROTECTOR
...
...
...
#else
static inline void boot_init_stack_canary(void)
{
}
#endif
```

<!---
If the `CONFIG_CC_STACKPROTECTOR` kernel configuration option is set, the `boot_init_stack_canary` function starts from the check stat `irq_stack_union` that represents [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) interrupt stack has offset equal to forty bytes from the `stack_canary` value:
--->
`CONFIG_CC_STACKPROTECTOR` カーネルコンフィギュレーションオプションがセットされていたら、 `boot_init_stack_canary` 関数は
CPU毎の（[per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) ）割込みスタックが、
`stack_canary` からオフセット値が 40バイトであると表される `irq_stack_union` のチェックから開始します。

```C
#ifdef CONFIG_X86_64
        BUILD_BUG_ON(offsetof(union irq_stack_union, stack_canary) != 40);
#endif
```

<!---
As we can read in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) the `irq_stack_union` represented by the following union:
--->
[前のパート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) で読むことができるように、 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h) で定義された `irq_stack_union` は以下の共用体で表されます：

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
which defined in the [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h). We know that [union](http://en.wikipedia.org/wiki/Union_type) in the [C](http://en.wikipedia.org/wiki/C_%28programming_language%29) programming language is a data structure which stores only one field in a memory. We can see here that structure has first field - `gs_base` which is 40 bytes size and represents bottom of the `irq_stack`. So, after this our check with the `BUILD_BUG_ON` macro should end successfully. (you can read the first part about Linux kernel initialization [process](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) if you're interesting about the `BUILD_BUG_ON` macro).
--->
[C](http://en.wikipedia.org/wiki/C_%28programming_language%29) 言語での [共用体(union)](http://en.wikipedia.org/wiki/Union_type) は、メモリ上に1つのフィールドだけを保存できるデータ構造であることを知っています。
構造体は40バイトのサイズと `irq_stack` の底を表す最初のフィールド（ `gs_base` ）をもっていることが見えます。
そして、この後、 `BUILD_BUG_ON` マクロでチェックは正常に終了するでしょう
（`BUILD_BUG_ON` マクロについて興味があるなら、[Linux kernel initialization processの最初のパート](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) で読むことができます）。

<!---
After this we calculate new `canary` value based on the random number and [Time Stamp Counter](http://en.wikipedia.org/wiki/Time_Stamp_Counter):
--->
このあと、乱数と [Time Stamp Counter](http://en.wikipedia.org/wiki/Time_Stamp_Counter) とを元に、新しい `canary` 値を計算します。

```C
get_random_bytes(&canary, sizeof(canary));
tsc = __native_read_tsc();
canary += tsc + (tsc << 32UL);
```

<!---
and write `canary` value to the `irq_stack_union` with the `this_cpu_write` macro:
--->
そして、 `this_cpu_write` マクロで `canary` 値を `irq_stack_union` に書きます。

```C
this_cpu_write(irq_stack_union.stack_canary, canary);
```

<!---
more about `this_cpu_*` operation you can read in the [Linux kernel documentation](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt).
--->
`this_cpu_*` マクロの動作の詳細については、[Linux kernel documentation](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt) で読むことができます。


Disabling/Enabling local interrupts
--------------------------------------------------------------------------------

<!---
The next step in the [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) which is related to the interrupts and interrupts handling after we have set the `canary` value to the interrupt stack - is the call of the `local_irq_disable` macro.
--->
[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) での、 `canary` 値を割り込みスタックに書き込んだ後の、割り込みと割り込み処理に関連する次のステップは、 `local_irq_disable` マクロの呼び出しです。

<!---
This macro defined in the [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqflags.h) header file and as you can understand, we can disable interrupts for the CPU with the call of this macro. Let's look on its implementation. First of all note that it depends on the `CONFIG_TRACE_IRQFLAGS_SUPPORT` kernel configuration option:
--->
このマクロは、 [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqflags.h) ヘッダファイルで定義されており、あなたの理解通りです。
このマクロを呼び出すことで、そのCPUの割り込みを無効にすることができます。
その実装について見ていきましょう。
まず最初に、`CONFIG_TRACE_IRQFLAGS_SUPPORT` カーネルコンフィギュレーションオプションに依存することに注意してください。

```C
#ifdef CONFIG_TRACE_IRQFLAGS_SUPPORT
...
#define local_irq_disable() \
         do { raw_local_irq_disable(); trace_hardirqs_off(); } while (0)
...
#else
...
#define local_irq_disable()     do { raw_local_irq_disable(); } while (0)
...
#endif
```

<!---
They are both similar and as you can see have only one difference: the `local_irq_disable` macro contains call of the `trace_hardirqs_off` when `CONFIG_TRACE_IRQFLAGS_SUPPORT` is enabled. There is special feature in the [lockdep](http://lwn.net/Articles/321663/) subsystem - `irq-flags tracing` for tracing `hardirq` and `softirq` state. In our case `lockdep` subsystem can give us interesting information about hard/soft irqs on/off events which are occurs in the system. The `trace_hardirqs_off` function defined in the [kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/locking/lockdep.c):
--->
どちらも似ており、見てのとおりたった1つの差異があることが見えます：
`local_irq_disable` マクロは、 `CONFIG_TRACE_IRQFLAGS_SUPPORT` が有効になっている時に、 `trace_hardirqs_off` の呼び出しを含んでいます。
[lockdep](http://lwn.net/Articles/321663/) サブシステムに特別な機能（ `hardirq` 状態 と `softirq` 状態 を追跡するための `irq-flags tracing` ）があります。
この場合、 `lockdep` サブシステムは、システムで生じるhard/soft割り込みのon/offイベントについて面白い情報を与えてくれることができます。
`trace_hardirqs_off` 関数は、 [kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/locking/lockdep.c) で定義され、 `trace_hardirqs_off_caller` 関数を呼び出します。

```C
void trace_hardirqs_off(void)
{
         trace_hardirqs_off_caller(CALLER_ADDR0);
}
EXPORT_SYMBOL(trace_hardirqs_off);
```

<!---
and just calls `trace_hardirqs_off_caller` function. The `trace_hardirqs_off_caller` checks the `hardirqs_enabled` field of the current process and increases the `redundant_hardirqs_off` if call of the `local_irq_disable` was redundant or the `hardirqs_off_events` if it was not. These two fields and other `lockdep` statistic related fields are defined in the [kernel/locking/lockdep_insides.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/locking/lockdep_insides.h) and located in the `lockdep_stats` structure:
--->
`trace_hardirqs_off_caller` は、カレントプロセスの `hardirqs_enabled` フィールドを確認し、 `local_irq_disable` の呼び出しが冗長であった場合に`redundant_hardirqs_off` をインクリメントしたり、冗長でなければ `hardirqs_off_events` をインクリメントします。
これら二つのフィールドと、ほかの関連する `lockdep` 統計は、 [kernel/locking/lockdep_insides.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/locking/lockdep_insides.h) で定義され、 `lockdep_stats` 構造体にあります：

```C
struct lockdep_stats {
...
...
...
int     softirqs_off_events;
int     redundant_softirqs_off;
...
...
...
}
```

<!---
If you will set `CONFIG_DEBUG_LOCKDEP` kernel configuration option, the `lockdep_stats_debug_show` function will write all tracing information to the `/proc/lockdep`:
--->
`CONFIG_DEBUG_LOCKDEP` カーネルコンフィギュレーションオプションをセットするなら、 `lockdep_stats_debug_show` 関数はすべてのトレース情報を `/proc/lockdep` に書き出します：

```C
static void lockdep_stats_debug_show(struct seq_file *m)
{
#ifdef CONFIG_DEBUG_LOCKDEP
	unsigned long long hi1 = debug_atomic_read(hardirqs_on_events),
	                         hi2 = debug_atomic_read(hardirqs_off_events),
							 hr1 = debug_atomic_read(redundant_hardirqs_on),
    ...
	...
	...
    seq_printf(m, " hardirq on events:             %11llu\n", hi1);
    seq_printf(m, " hardirq off events:            %11llu\n", hi2);
    seq_printf(m, " redundant hardirq ons:         %11llu\n", hr1);
#endif
}
```

<!---
and you can see its result with the:
--->
その結果は以下のように見られます：

```
$ sudo cat /proc/lockdep
 hardirq on events:             12838248974
 hardirq off events:            12838248979
 redundant hardirq ons:               67792
 redundant hardirq offs:         3836339146
 softirq on events:                38002159
 softirq off events:               38002187
 redundant softirq ons:                   0
 redundant softirq offs:                  0
```

<!---
Ok, now we know a little about tracing, but more info will be in the separate part about `lockdep` and `tracing`. You can see that the both `local_disable_irq` macros have the same part - `raw_local_irq_disable`. This macro defined in the [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irqflags.h) and expands to the call of the:
--->
さて、今、トレースについて少し知りましが、より詳しい情報は、 `lockdep` と `tracing` に関する別のパートにあるでしょう。
両方の `local_disable_irq` マクロは、 `raw_local_irq_disable` と同じ部分を持っています。
このマクロは[arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irqflags.h) で定義され、以下の呼び出しに展開されます：

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

<!---
And you already must remember that `cli` instruction clears the [IF](http://en.wikipedia.org/wiki/Interrupt_flag) flag which determines ability of a processor to handle an interrupt or an exception. Besides the `local_irq_disable`, as you already can know there is an inverse macro - `local_irq_enable`. This macro has the same tracing mechanism and very similar on the `local_irq_enable`, but as you can understand from its name, it enables interrupts with the `sti` instruction:
--->
`cli` 命令は、割り込みや例外を処理するためのプロセッサの働きを決定するための [IF](http://en.wikipedia.org/wiki/Interrupt_flag) フラグをクリアすることを思い出す必要があります。
`local_irq_disable` 意外に、反対のマクロ（ `local_irq_enable` ）があることを知ることができます。
このマクロは、同じトレースメカニズムを持ち、 `local_irq_enable` ととても似ています。
しかし、その名前から理解することができるように、 `sti` 命令で割り込みを有効にします。

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

<!---
Now we know how `local_irq_disable` and `local_irq_enable` work. It was the first call of the `local_irq_disable` macro, but we will meet these macros many times in the Linux kernel source code. But for now we are in the `start_kernel` function from the [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) and we just disabled `local` interrupts. Why local and why we did it? Previously kernel provided a method to disable interrupts on all processors and it was called `cli`. This function was [removed](https://lwn.net/Articles/291956/) and now we have `local_irq_{enabled,disable}` to disable or enable interrupts on the current processor. After we've disabled the interrupts with the `local_irq_disable` macro, we set the:
--->
ここで、 `local_irq_disable` と `local_irq_enable` がどのように働くかを知りました。
`local_irq_disable` マクロの呼び出しが最初ですが、これらのマクロは Linuxカーネルソースコードでたくさん出会います。
しかし今は、 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) からの `start_kernel` 関数にいて、 `local` 割り込みを無効にしました。
なぜローカルなのか、なぜそうしたのか。
以前に、カーネルはすべてのプロセッサの割り込みを無効にする方法を提供しており、 `cli` と呼んでいました。
この関数は、[削除され](https://lwn.net/Articles/291956/) 、現在のプロセッサ上の割り込みを有効または無効にするために、 `local_irq_{enabled,disable}` を使います。
`local_irq_disable` マクロで割り込みを無効にした後、以下の代入を行います：

```C
early_boot_irqs_disabled = true;
```

<!---
The `early_boot_irqs_disabled` variable defined in the [include/linux/kernel.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/kernel.h):
--->
`early_boot_irqs_disabled` 変数は、[include/linux/kernel.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/kernel.h) で定義されており、異なる場所で使われています。

```C
extern bool early_boot_irqs_disabled;
```

<!---
and used in the different places. For example it used in the `smp_call_function_many` function from the [kernel/smp.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/smp.c) for the checking possible deadlock when interrupts are disabled:
--->
例えば、 割り込みが無効なときに、デッドロックが起こりうるかをチェックするために [kernel/smp.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/smp.c) の `smp_call_function_many` 関数で使われています。

```C
WARN_ON_ONCE(cpu_online(this_cpu) && irqs_disabled()
                     && !oops_in_progress && !early_boot_irqs_disabled);
```

Early trap initialization during kernel initialization
--------------------------------------------------------------------------------

<!---
The next functions after the `local_disable_irq` are `boot_cpu_init` and `page_address_init`, but they are not related to the interrupts and exceptions (more about this functions you can read in the chapter about Linux kernel [initialization process](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)). The next is the `setup_arch` function. As you can remember this function located in the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel.setup.c) source code file and makes initialization of many different architecture-dependent [stuff](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html). The first interrupts related function which we can see in the `setup_arch` is the - `early_trap_init` function. This function defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) and fills `Interrupt Descriptor Table` with the couple of entries:
--->
`local_disable_irq` のあとの、次の関数は、`boot_cpu_init` と `page_address_init` ですが、割り込みと例外に関係がありません（これらの関数の詳細については、Linux kernel [initialization process](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) に関する章で読むことができます）。
次は `setup_arch` 関数です。
この関数は、[arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel.setup.c) に配置されていたことを覚えてるように、たくさんの[アーキテクチャ依存のもの](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html) を初期化します。
`setup_arch` で見ることのできる、最初の割り込みに関する関数は、`early_trap_init` 関数です。
この関数は[arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) で定義され、いくつかのエントリで `Interrupt Descriptor Table` を埋めます。

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
#ifdef CONFIG_X86_32
        set_intr_gate(X86_TRAP_PF, page_fault);
#endif
        load_idt(&idt_descr);
}
```

<!---
Here we can see calls of three different functions:
--->
ここまでで3つの異なる関数呼び出しをみることができました。

* `set_intr_gate_ist`
* `set_system_intr_gate_ist`
* `set_intr_gate`

<!---
All of these functions defined in the [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h) and do the similar thing but not the same. The first `set_intr_gate_ist` function inserts new an interrupt gate in the `IDT`. Let's look on its implementation:
--->
これらの関数すべては、[arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h) に定義され、似たようなことをしますが同じではありません。
最初の `set_intr_gate_ist` 関数は、 `IDT` に新しい割り込みゲートを挿入します。
その実装を見てみましょう：

```C
static inline void set_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS);
}
```

<!---
First of all we can see the check that `n` which is [vector number](http://en.wikipedia.org/wiki/Interrupt_vector_table) of the interrupt is not greater than `0xff` or 255. We need to check it because we remember from the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) that vector number of an interrupt must be between `0` and `255`. In the next step we can see the call of the `_set_gate` function that sets a given interrupt gate to the `IDT` table:
--->
すべての最初に、割り込みの [ベクタ番号](http://en.wikipedia.org/wiki/Interrupt_vector_table) である `n` が、 `0xff` (255) より大きくないことを検証することが見られます。
前の [パート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) で、ベクタ番号は `0` から `255` の間である必要があることを忘れていないので、チェックする必要があります。
この次のステップでは、`IDT` テーブルに割り込みゲートをセットする `set_gate` 関数の呼び出しが見られます：

```C
static inline void _set_gate(int gate, unsigned type, void *addr,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate_desc s;

        pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
        write_idt_entry(idt_table, gate, &s);
        write_trace_idt_entry(gate, &s);
}
```

<!---
Here we start from the `pack_gate` function which takes clean `IDT` entry represented by the `gate_desc` structure and fills it with the base address and limit, [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks), [Privilege level](http://en.wikipedia.org/wiki/Privilege_level), type of an interrupt which can be one of the following values:
--->
`gate_desc` 構造体で表される `IDT` エントリをきれいにするための `pack_gate` 関数から始まり、ベースアドレス、限界値、[Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)、[Privilege level](http://en.wikipedia.org/wiki/Privilege_level)、以下の値の１つである割り込みの型、で埋めます。

* `GATE_INTERRUPT`
* `GATE_TRAP`
* `GATE_CALL`
* `GATE_TASK`

<!---
and set the present bit for the given `IDT` entry:
--->
そして、 与えられた `IDT` エントリのために、presentビットをセットします。

```C
static inline void pack_gate(gate_desc *gate, unsigned type, unsigned long func,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate->offset_low        = PTR_LOW(func);
        gate->segment           = __KERNEL_CS;
        gate->ist               = ist;
        gate->p                 = 1;
        gate->dpl               = dpl;
        gate->zero0             = 0;
        gate->zero1             = 0;
        gate->type              = type;
        gate->offset_middle     = PTR_MIDDLE(func);
        gate->offset_high       = PTR_HIGH(func);
}
```

<!---
After this we write just filled interrupt gate to the `IDT` with the `write_idt_entry` macro which expands to the `native_write_idt_entry` and just copy the interrupt gate to the `idt_table` table by the given index:
--->
このあと、
ちょうど埋められた割り込みゲートを、 `native_write_idt_entry` に展開される `write_idt_entry` マクロで `IDT` へ書き込みます。
そして与えられたインデックスで割り込みゲートを `idt_table` テーブルへコピーします。

```C
#define write_idt_entry(dt, entry, g)           native_write_idt_entry(dt, entry, g)

static inline void native_write_idt_entry(gate_desc *idt, int entry, const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

<!---
where `idt_table` is just array of `gate_desc`:
-->
ここで `idt_table` は、 `gate_desc` の配列です。

```C
extern gate_desc idt_table[];
```

<!---
That's all. The second `set_system_intr_gate_ist` function has only one difference from the `set_intr_gate_ist`:
--->
以上がすべてです。
２つめの `set_system_intr_gate_ist` 関数は、 `set_intr_gate_ist` から１つだけ差異を持ちます：

```C
static inline void set_system_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0x3, ist, __KERNEL_CS);
}
```

<!---
Do you see it? Look on the fourth parameter of the `_set_gate`. It is `0x3`. In the `set_intr_gate` it was `0x0`. We know that this parameter represent `DPL` or privilege level. We also know that `0` is the highest privilege level and `3` is the lowest.Now we know how `set_system_intr_gate_ist`, `set_intr_gate_ist`, `set_intr_gate` are work and we can return to the `early_trap_init` function. Let's look on it again:
--->
見えますか？
`_set_gate` の４つめのパラメータを見ましょう。
それは `0x3` です。 `set_intr_gate` 内では、それは `0x0` です。
このパラメータは、 `DPL` すなわち 特権レベル を表していることを知っています。
また、`0` が最高の特権レベルで、`3` が最低であることも知っています。
ここで、 `set_system_intr_gate_ist` と `set_intr_gate_ist` と `set_intr_gate` とが、どのように働き、 `early_trap_init` 関数へ戻ることができます。
再度見てみましょう：


```C
set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
```

<!---
We set two `IDT` entries for the `#DB` interrupt and `int3`. These functions takes the same set of parameters:
--->
`#DB` 割り込みと `int3` 割り込みのために、２つの `IDT` エントリをセットします。
これらの関数は、同じパラメータセットをとります：

<!---
* vector number of an interrupt;
* address of an interrupt handler;
* interrupt stack table index.
--->
* 割り込みのヴェクタ番号
* 割り込みハンドラのアドレス
* 割り込みスタックテーブルインデックス

<!---
That's all. More about interrupts and handlers you will know in the next parts.
--->
以上すべてです。割り込みとハンドラの詳細について、次のパートで知ることができます。

Conclusion
--------------------------------------------------------------------------------

<!---
It is the end of the second part about interrupts and interrupt handling in the Linux kernel. We saw the some theory in the previous part and started to dive into interrupts and exceptions handling in the current part. We have started from the earliest parts in the Linux kernel source code which are related to the interrupts. In the next part we will continue to dive into this interesting theme and will know more about interrupt handling process.
--->
Linuxカーネルの割り込みと割り込みハンドラに関する２つめのパートの最後です。
前のパートで理論を見て、このパートでは割り込みハンドラと例外ハンドラにとびこみました。
割り込みに関するLinuxカーネルソースコードの最初の部分から始めました。
次のパートでは、面白いテーマへ潜り続け、割り込みハンドラ手順についてもっと知ることができるでしょう。

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [List of x86 calling conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)
* [8086](http://en.wikipedia.org/wiki/Intel_8086)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [Extended Feature Enable Register](http://en.wikipedia.org/wiki/Control_register#Additional_Control_registers_in_x86-64_series)
* [Model-specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Process identifier](https://en.wikipedia.org/wiki/Process_identifier)
* [lockdep](http://lwn.net/Articles/321663/)
* [irqflags tracing](https://www.kernel.org/doc/Documentation/irqflags-tracing.txt)
* [IF](http://en.wikipedia.org/wiki/Interrupt_flag)
* [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)
* [Union type](http://en.wikipedia.org/wiki/Union_type)
* [this_cpu_* operations](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt)
* [vector number](http://en.wikipedia.org/wiki/Interrupt_vector_table)
* [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [Privilege level](http://en.wikipedia.org/wiki/Privilege_level)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html)
