Interrupts and Interrupt Handling. Part 3.
================================================================================

割り込み処理（Exception Handling）
--------------------------------------------------------------------------------

<!---
This is the third part of the [chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) about an interrupts and an exceptions handling in the Linux kernel and in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) we stopped at the `setup_arch` function from the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blame/master/arch/x86/kernel/setup.c) source code file.
---->
[Linuxカーネルにおける割り込み処理と例外処理に関する章](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) の３番目のパートです。
[前のパート](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) では、[arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blame/master/arch/x86/kernel/setup.c) ソースコードファイルの `setup_arch` 関数で止まりました。

<!---
We already know that this function executes initialization of architecture-specific stuff. In our case the `setup_arch` function does [x86_64](https://en.wikipedia.org/wiki/X86-64) architecture related initializations. The `setup_arch` is big function, and in the previous part we stopped on the setting of the two exceptions handlers for the two following exceptions:
--->
この関数はアーキテクチャ依存のものの初期化を行うことを既に知っています。
ここでは、`setup_arch` 関数は [x86_64](https://en.wikipedia.org/wiki/X86-64) アーキテクチャに関する初期化を行います。
`setup_arch` は大きな関数で、前のパートでは、以下の２つの例外処理の設定で止まっていました。

<!---
* `#DB` - debug exception, transfers control from the interrupted process to the debug handler;
* `#BP` - breakpoint exception, caused by the `int 3` instruction.
--->
* `#DB` - デバッグ例外。割りこまれたプロセスからデバッグハンドラへ制御を移す
* `#BP` - ブレイクポイント例外。 `int 3` 命令によって起こされます

<!---
These exceptions allow the `x86_64` architecture to have early exception processing for the purpose of debugging via the [kgdb](https://en.wikipedia.org/wiki/KGDB).
--->
これらの例外は `x86_64` アーキテクチャに、 [kgdb](https://en.wikipedia.org/wiki/KGDB) を介してデバッグする目的のために、早期割り込み処理を持たせることを許します。
<!---
As you can remember we set these exceptions handlers in the `early_trap_init` function:
--->
[arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) の `early_trap_init` 関数内で、これらの例外ハンドラを設定したことを思い出せます：

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```

<!---
from the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c). We already saw implementation of the `set_intr_gate_ist` and `set_system_intr_gate_ist` functions in the previous part and now we will look on the implementation of these two exceptions handlers.
--->
既に前のパートで `set_intr_gate_ist` 関数と `set_system_intr_gate_ist` 関数の実装を見ています。


デバッグ例外とブレイクポイント例外（Debug and Breakpoint exceptions）
--------------------------------------------------------------------------------

<!---
Ok, we setup exception handlers in the `early_trap_init` function for the `#DB` and `#BP` exceptions and now time is to consider their implementations. But before we will do this, first of all let's look on details of these exceptions.
--->
さて、 `early_trap_init` 関数内で `#DB` 例外と `#BP` 例外のための例外ハンドラを設定しました。
例外ハンドラの実装について考える時です。
しかし、これを行う前に、最初に、これらの例外の詳細について見ていきましょう。

<!---
The first exceptions - `#DB` or `debug` exception occurs when a debug event occurs. For example - attempt to change the contents of a [debug register](http://en.wikipedia.org/wiki/X86_debug_register). Debug registers are special registers that were presented in `x86` processors starting from the [Intel 80386](http://en.wikipedia.org/wiki/Intel_80386) processor and as you can understand from name of this CPU extension, main purpose of these registers is debugging.
--->
最初の例外、デバッグイベントが起きた時に生じる `#DB` 例外 あるいは `debug` 例外について。
例えば、 [debug register](http://en.wikipedia.org/wiki/X86_debug_register) の内容を変更しようとするとき。
デバッグレジスタは、 [Intel 80386](http://en.wikipedia.org/wiki/Intel_80386) プロセッサから始まる `x86` プロセッサに存在する特別なレジスタで、このCPU拡張の名前から理解できるように、このレジスタの主な目的はデバッギングです。

<!---
These registers allow to set breakpoints on the code and read or write data to trace it. Debug registers may be accessed only in the privileged mode and an attempt to read or write the debug registers when executing at any other privilege level causes a [general protection fault](https://en.wikipedia.org/wiki/General_protection_fault) exception. That's why we have used `set_intr_gate_ist` for the `#DB` exception, but not the `set_system_intr_gate_ist`.
--->
このレジスタは、コード上にブレイクポイントをセットすることや、データの読み込み、書き出しのトレースすることを許します。
デバッグレジスタは特権モードでのみアクセスでき、他の特権レベルでデバッグレジスタを読み書きしようとした時には [general protection fault](https://en.wikipedia.org/wiki/General_protection_fault) 例外を起こします。
`#DB` 例外のために、`set_system_intr_gate_ist` を使わず、`set_intr_gate_ist` を使う理由です。

<!---
The verctor number of the `#DB` exceptions is `1` (we pass it as `X86_TRAP_DB`) and as we may read in specification, this exception has no error code:
--->
`#DB` 例外のヴェクタ番号は `1` （ `X86_TRAP_DB` で渡しています）で、仕様を読むと、この例外はエラーコードを持ちません：

```
+-----------------------------------------------------+
|Vector|Mnemonic|Description         |Type |Error Code|
+-----------------------------------------------------+
|1     | #DB    |Reserved            |F/T  |NO        |
+-----------------------------------------------------+
```

<!---
The second exception is `#BP` or `breakpoint` exception occurs when processor executes the [int 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3) instruction. Unlike the `DB` exception, the `#BP` exception may occur in userspace. We can add it anywhere in our code, for example let's look on the simple program:
--->
２つめの例外は、プロセッサが [int 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3) 命令を実行した時に発生する、 `#BP` 例外、あるいは `breakpoint` 例外です。
`DB` 例外とは異なり、 `#BP` 例外はユーザ空間で発生します。
コードの何処にでも追加することができます。例えば単純なプログラムを見ましょう：

```C
// breakpoint.c
#include <stdio.h>

int main() {
    int i;
    while (i < 6){
	    printf("i equal to: %d\n", i);
	    __asm__("int3");
		++i;
    }
}
```

<!---
If we will compile and run this program, we will see following output:
--->
コンパイル・実行すると、以下の出力を見るでしょう：

```
$ gcc breakpoint.c -o breakpoint
i equal to: 0
Trace/breakpoint trap
```

<!---
But if will run it with gdb, we will see our breakpoint and can continue execution of our program:
--->
しかし、gdbで実行すると、ブレイクポイントが見えて、プログラムの実行を継続することができます：

```
$ gdb breakpoint
...
...
...
(gdb) run
Starting program: /home/alex/breakpoints 
i equal to: 0

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
(gdb) c
Continuing.
i equal to: 1

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
(gdb) c
Continuing.
i equal to: 2

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
...
...
...
```

<!---
From this moment we know a little about these two exceptions and we can move on to consideration of their handlers.
--->
この瞬間から、２つの例外について少しわかり、これらのハンドラの考察へと移ることができます。


割り込みハンドラの前の準備（Preparation before an exception handler）
--------------------------------------------------------------------------------

<!--- [JA] typo s/In or case/In our case,/ --->
<!---
As you may note before, the `set_intr_gate_ist` and `set_system_intr_gate_ist` functions takes an addresses of exceptions handlers in theirs second parameter. In our case, our two exception handlers will be:
--->
以前に注意したように、 `set_intr_gate_ist` 関数と `set_system_intr_gate_ist` 関数とは、２番目の引数で割り込みハンドラのアドレスを取ります。
ここでは、２つの例外ハンドラは以下となります：

* `debug`;
* `int3`.

<!---
You will not find these functions in the C code. all of that could be found in the kernel's `*.c/*.h` files only definition of these functions which are located in the [arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/traps.h) kernel header file:
--->
Cコード内では、これらの関数を見つけることができません。
カーネルの `*.c/*.h` ファイル内では、 [arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/traps.h) カーネルヘッダファイル内に、これらの関数の定義のみを見つけることができます：

```C
asmlinkage void debug(void);
```

and

```C
asmlinkage void int3(void);
```

<!---
You may note `asmlinkage` directive in definitions of these functions. The directive is the special specificator of the [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection). Actually for a `C` functions which are called from assembly, we need in explicit declaration of the function calling convention. In our case, if function made with `asmlinkage` descriptor, then `gcc` will compile the function to retrieve parameters from stack.
--->
これらの関数の定義で、`asmlinkage` ディレクティブに気づきます。
このディレクティブは、 [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection) の特別な指定子です。
実際に、アセンブラから呼ばれる`C` 関数のために、関数呼び出し規則の明示的な宣言が必要です。
ここでは、関数が `asmlinkage` デスクリプタで生成され、 `gcc` はスタックからパラメタを参照するように関数をコンパイルします。
<!---
So, both handlers are defined in the [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) assembly source code file with the `idtentry` macro:
--->
いずれのハンドラも、[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) アセンブラソースコードファイル内で、 `idtentry` マクロを使って定義されています。

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

and

```assembly
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

<!---
Each exception handler may be consists from two parts. The first part is generic part and it is the same for all exception handlers. An exception handler should to save  [general purpose registers](https://en.wikipedia.org/wiki/Processor_register) on the stack, switch to kernel stack if an exception came from userspace and transfer control to the second part of an exception handler. The second part of an exception handler does certain work depends on certain exception. For example page fault exception handler should find virtual page for given address, invalid opcode exception handler should send `SIGILL` [signal](https://en.wikipedia.org/wiki/Unix_signal) and etc.
--->
それぞれの例外ハンドラは、２つの部分から構成されます。
１つめの部分は、一般的な部分で、全ての例外ハンドラで同じです。
例外ハンドラは、スタックに [general purpose registers](https://en.wikipedia.org/wiki/Processor_register) を保存する必要があります。
また、例外がユーザ空間から来た場合はカーネルスタックへ切り替える必要があります。
そして、例外ハンドラの２つめの部分に制御を移します。
例外ハンドラの２つめの部分は、特定の例外に依存した特定の作業を行います。

例えば、ページフォールト例外ハンドラは、与えられたアドレスの仮想ページ（virtual page）を見つける必要があります。
未定義命令（invalid opcode）例外ハンドラは、 `SIGILL`  [signal](https://en.wikipedia.org/wiki/Unix_signal) を送る必要があります。

<!---
As we just saw, an exception handler starts from definition of the `idtentry` macro from the [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/entry_64.S) assembly source code file, so let's look at implementation of this macro. As we may see, the `idtentry` macro takes five arguments:
--->
ちょうど見てきたように、例外ハンドラは [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) アセンブラソースコードファイル の `idtentry` マクロの定義から始まります。
このマクロの実装について見ていきましょう。
見てのとおり `idtentry` マクロは５つの引数を持ちます：

<!---
* `sym` - defines global symbol with the `.globl name` which will be an an entry of exception handler;
* `do_sym` - symbol name which represents a secondary entry of an exception handler;
* `has_error_code` - information about existence of an error code of exception.
---->
* `sym` - 例外ハンドラのエントリとなるグローバルシンボルを `.globl name` で定義する名前。
* `do_sym` - 例外ハンドラの２つめのエントリを表すシンボル名
* `has_error_code` - 例外のエラーコードの存在に関する情報

<!---
The last two parameters are optional:

* `paranoid` - shows us how we need to check current mode (will see explanation in details later);
* `shift_ist` - shows us is an exception running at `Interrupt Stack Table`.
--->
最後の２つのパラメタはオプションです：

* `paranoid` - 現在のモードをチェックする必要があるかどうかを示す（詳細な説明は後ほど）
* `shift_ist` - 例外が  `Interrupt Stack Table` で走行しているかを示す

<!---
Definition of the `.idtentry` macro looks:
--->
`.idtentry` マクロの定義は、以下のように見られます：

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
...
...
...
END(\sym)
.endm
```

<!---
Before we will consider internals of the `idtentry` macro, we should to know state of stack when an exception occurs. As we may read in the [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html), the state of stack when an exception occurs is following:
--->
`idtentry` マクロの内部を検討する前に、例外が発生した時のスタックの状態を知っておく必要があります。
[Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) を読むと、例外が発生した時のスタックの状態は以下のとおりです：

```
    +------------+
+40 | %SS        |
+32 | %RSP       |
+24 | %RFLAGS    |
+16 | %CS        |
 +8 | %RIP       |
  0 | ERROR CODE | <-- %RSP
    +------------+
```

<!---
Now we may start to consider implementation of the `idtmacro`. Both `#DB` and `BP` exception handlers are defined as:
--->
さて、 `idtmacro` の実装を検討し始めましょう。
`#DB` 例外ハンドラ と `BP` 例外ハンドラは、以下のように定義されます：

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

<!---
If we will look at these definitions, we may know that compiler will generate two routines with `debug` and `int3` names and both of these exception handlers will call `do_debug` and `do_int3` secondary handlers after some preparation. The third parameter defines existence of error code and as we may see both our exception do not have them. As we may see on the diagram above, processor pushes error code on stack if an exception provides it. In our case, the `debug` and `int3` exception do not have error codes. This may bring some difficulties because stack will look differently for exceptions which provides error code and for exceptions which not. That's why implementation of the `idtentry` macro starts from putting a fake error code to the stack if an exception does not provide it:
--->
これらの定義を見た時、コンパイラが `debug` と `int3` という名前のサブルーチンを作ることを知っています。
どちらの例外ハンドラも、準備の後に２つ目のハンドラ `do_debug` と `do_int3` とを呼び出します。
３つめのパラメタはエラーコードの存在を定義します。どちらの例外もエラーコードを持たないことは前記のとおりです。
上記のダイアグラムで見たように、プロセッサはエラーコードがある場合はスタックに積みます。
ここでは、 `debug` 例外と `int3` 例外とはエラーコードを持ちません。
エラーコードをもつ例外と、もたない例外とでは、スタック上で異なるように見えてしまうので、複雑さをもたらします。
それは `idtentry` マクロの実装が、例外がエラーコードを持たない場合に、偽のエラーコードを積むことから始める理由です。

```assembly
.ifeq \has_error_code
    pushq	$-1
.endif
```

<!---
But it is not only fake error-code. Moreover the `-1` also represents invalid system call number, so that the system call restart logic will not be triggered.
--->
しかし、これは偽のエラーコードだけではありません。
さらにいうと、 `-1` は無効なシステムコール番号も表しますので、システムコール再起動ロジックは起動されません。

<!---
The last two parameters of the `idtentry` macro `shift_ist` and `paranoid` allow to know do an exception handler runned at stack from `Interrupt Stack Table` or not. You already may know that each kernel thread in the system has own stack. In addition to these stacks, there are some specialized stacks associated with each processor in the system. One of these stacks is - exception stack. The [x86_64](https://en.wikipedia.org/wiki/X86-64) architecture provides special feature which is called - `Interrupt Stack Table`. This feature allows to switch to a new stack for designated events such as an atomic exceptions like `double fault` and etc. So the `shift_ist` parameter allows us to know do we need to switch on `IST` stack for an exception handler or not.
--->
`idtentry` マクロの最後の２つのパラメタである `shift_ist` と `paranoid` とは、
`Interrupt Stack Table` のスタックで例外ハンドラが走行するのかを知るために許可されます。

システム内の各カーネルスレッドは固有のスタックを持っていることを既に知っています。
これらのスタックに加えて、システム内の個々のプロセッサに特別なスタックもあります。
この特別なスタックの１つが、例外スタックです。
[x86_64](https://en.wikipedia.org/wiki/X86-64) アーキテクチャは、 `Interrupt Stack Table` と呼ばれる特別な機能を提供します。
この機能は、専用のイベント（例えば アトミックな例外、 `double fault` など ）のために新しいスタックへと切り替えることを許可します。
`shift_ist` パラメタは、例外ハンドラのために、 `IST` スタックへ切り替える必要があるかどうかを知ることを許可します。

<!---
The second parameter - `paranoid` defines the method which helps us to know did we come from userspace or not to an exception handler. The easiest way to determine this is to via `CPL` or `Current Privilege Level` in `CS` segment register. If it is equal to `3`, we came from userspace, if zero we came from kernel space:
--->
３つめのパラメタ `paranoid` は、例外ハンドラへユーザ空間から来たかどうかを知るためのメソッドを定義します。
これを決定するための最も簡単な方法は、 `CS` セグメントレジスタにある `CPL` （ `Current Privilege Level` ）を介することです。
これが `3` であれば、ユーザ空間からきており、ゼロであればカーネル空間からきています：

```
testl $3,CS(%rsp)
jnz userspace
...
...
...
// we are from the kernel space
```

<!---
But unfortunately this method does not give a 100% guarantee. As described in the kernel documentation:
--->
しかし、この方法は 100%保証してくれるものではありません。
カーネルドキュメントの記述によると：

<!---
> if we are in an NMI/MCE/DEBUG/whatever super-atomic entry context,
> which might have triggered right after a normal entry wrote CS to the
> stack but before we executed SWAPGS, then the only safe way to check
> for GS is the slower method: the RDMSR.
--->
NMI、MCE、DEBUG、あるいはその他の super-atomicエントリコンテキストに居る場合、
＊＊＊＊＊＊＊＊＊＊

<!---
In other words for example `NMI` could happen inside the critical section of a [swapgs](http://www.felixcloutier.com/x86/SWAPGS.html) instruction. In this way we should check value of the `MSR_GS_BASE` [model specific register](https://en.wikipedia.org/wiki/Model-specific_register) which stores pointer to the start of per-cpu area. So to check did we come from userspace or not, we should to check value of the `MSR_GS_BASE` model specific register and if it is negative we came from kernel space, in other way we came from userspace:
--->
言い換えると、例えば `NMI` は [swapgs](http://www.felixcloutier.com/x86/SWAPGS.html) 命令のクリティカルセクション内で起こすことが可能です。
このとき、per-cpu領域の先頭ポインタを保持している [model specific register](https://en.wikipedia.org/wiki/Model-specific_register) の `MSR_GS_BASE` の値をチェックする必要があります。
ユーザ空間からきたかどうかをチェックするためには、 `MSR_GS_BASE` model specific registerの値をチェックする必要があり、その値が負であるならカーネル空間から、そうでなければユーザ空間からきています：

```assembly
movl $MSR_GS_BASE,%ecx
rdmsr
testl %edx,%edx
js 1f
```

<!--- [JA] typo s/unsigned 4 bytes/signed 4 bytes/ --->
<!---
In first two lines of code we read value of the `MSR_GS_BASE` model specific register into `edx:eax` pair. We can't set negative value to the `gs` from userspace. But from other side we know that direct mapping of the physical memory starts from the `0xffff880000000000` virtual address. In this way, `MSR_GS_BASE` will contain an address from `0xffff880000000000` to `0xffffc7ffffffffff`. After the `rdmsr` instruction will be executed, the smallest possible value in the `%edx` register will be - `0xffff8800` which is `-30720` in unsigned 4 bytes. That's why kernel space `gs` which points to start of `per-cpu` area will contain negative value.
--->
コードの最初の２行では、`MSR_GS_BASE` model specific registerの値を `edx:eax` ペアへ入れています。
ユーザ空間からは、 `gs` へ負の値を設定することができません。
しかし、カーネル空間では、物理メモリのダイレクトマッピングが仮想アドレス `0xffff880000000000`から始まっていることを知っています。
この方法では、 `MSR_GS_BASE` は、アドレスの `0xffff880000000000` から `0xffffc7ffffffffff` を含んでいます。
`rdmsr` 命令を実行した後、`%edx` レジスタ内の値が取りうる最小値は `0xffff8800` すなわち、 符号付き4byteで見ると `-30720`  です。
カーネル空間が、`per-cpu` 領域の始まりをポイントしている `gs` が負の値を含む理由です。

<!---
After we pushed fake error code on the stack, we should allocate space for general purpose registers with:
--->
スタックに偽のエラーコードを積んだあと、汎用レジスタのために以下のマクロで空間を確保しなければなりません：

```assembly
ALLOC_PT_GPREGS_ON_STACK
```

<!---
macro which is defined in the [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) header file. This macro just allocates 15*8 bytes space on the stack to preserve general purpose registers:
--->
このマクロは [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) ヘッダファイル内で定義されています。
このマクロは、汎用レジスタを保存するために、ちょうど１２０バイト（15 * 8）の空間をスタック上に確保します：

```assembly
.macro ALLOC_PT_GPREGS_ON_STACK addskip=0
    addq	$-(15*8+\addskip), %rsp
.endm
```

<!---
So the stack will look like this after execution of the `ALLOC_PT_GPREGS_ON_STACK`:
--->
`ALLOC_PT_GPREGS_ON_STACK` の実行後、スタックは以下のようになります：

```
     +------------+
+160 | %SS        |
+152 | %RSP       |
+144 | %RFLAGS    |
+136 | %CS        |
+128 | %RIP       |
+120 | ERROR CODE |
     |------------|
+112 |            |
+104 |            |
 +96 |            |
 +88 |            |
 +80 |            |
 +72 |            |
 +64 |            |
 +56 |            |
 +48 |            |
 +40 |            |
 +32 |            |
 +24 |            |
 +16 |            |
  +8 |            |
  +0 |            | <- %RSP
     +------------+
```

<!---
After we allocated space for general purpose registers, we do some checks to understand did an exception come from userspace or not and if yes, we should move back to an interrupted process stack or stay on exception stack:
--->
汎用レジスタのための空間を確保したあと、割り込みがユーザ空間から来たかどうかを判断するため、いくつかチェックします。
もしユーザ空間からきていたなら、割り込みプロセススタックへ移る必要があり、カーネル空間からであれば例外スタックにとどまります：

```assembly
.if \paranoid
    .if \paranoid == 1
	    testb	$3, CS(%rsp)
	    jnz	1f
	.endif
	call	paranoid_entry
.else
	call	error_entry
.endif
```

<!---
Let's consider all of these there cases in course.
-->
これらすべてのケースをコースで検討しましょう。


ユーザ空間で発生した例外（An exception occured in userspace）
--------------------------------------------------------------------------------

<!---
In the first let's consider a case when an exception has `paranoid=1` like our `debug` and `int3` exceptions. In this case we check selector from `CS` segment register and jump at `1f` label if we came from userspace or the `paranoid_entry` will be called in other way.
--->
最初に、`debug` 例外と `int3` 例外のように、例外が `paranoid=1` のときのケースを考えてみましょう。
このケースでは、 `CS` セグメントレジスタの選択をチェックし、ユーザ空間から来ていた場合は `1f` へジャンプ、そうでなければ  `paranoid_entry` を呼び出します。

<!---
Let's consider first case when we came from userspace to an exception handler. As described above we should jump at `1` label. The `1` label starts from the call of the
--->
最初のケース、ユーザ空間から例外ハンドラに来た時について検討しましょう。
上記のように、`1` ラベルへジャンプします。
`1` ラベルは、以前にスタック上に確保された領域に汎用レジスタを保存する error_entryサブルーチンの呼び出しから始まります：

```assembly
call	error_entry
```
<!---
routine which saves all general purpose registers in the previously allocated area on the stack:
--->
以下のマクロでレジスタの保存を行います：

```assembly
SAVE_C_REGS 8
SAVE_EXTRA_REGS 8
```

<!---
These both macros are defined in the  [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) header file and just move values of general purpose registers to a certain place at the stack, for example:
--->
どちらのマクロも [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) ヘッダファイルで定義されており、汎用レジスタの値をスタック上の特定の場所へコピーします。
たとえば：

```assembly
.macro SAVE_EXTRA_REGS offset=0
	movq %r15, 0*8+\offset(%rsp)
	movq %r14, 1*8+\offset(%rsp)
	movq %r13, 2*8+\offset(%rsp)
	movq %r12, 3*8+\offset(%rsp)
	movq %rbp, 4*8+\offset(%rsp)
	movq %rbx, 5*8+\offset(%rsp)
.endm
```

<!---
After execution of `SAVE_C_REGS` and `SAVE_EXTRA_REGS` the stack will look:
--->
`SAVE_C_REGS` と `SAVE_EXTRA_REGS` の実行後、スタックは以下のように見えます：

```
     +------------+
+160 | %SS        |
+152 | %RSP       |
+144 | %RFLAGS    |
+136 | %CS        |
+128 | %RIP       |
+120 | ERROR CODE |
     |------------|
+112 | %RDI       |
+104 | %RSI       |
 +96 | %RDX       |
 +88 | %RCX       |
 +80 | %RAX       |
 +72 | %R8        |
 +64 | %R9        |
 +56 | %R10       |
 +48 | %R11       |
 +40 | %RBX       |
 +32 | %RBP       |
 +24 | %R12       |
 +16 | %R13       |
  +8 | %R14       |
  +0 | %R15       | <- %RSP
     +------------+
```

<!---
After the kernel saved general purpose registers at the stack, we should check that we came from userspace space again with:
--->
カーネルが汎用レジスタをスタックに保存したあと、ユーザ空間から来たかどうかを再び確認します：

```assembly
testb	$3, CS+8(%rsp)
jz	.Lerror_kernelspace
```
<!---
because we may have potentially fault if as described in documentation truncated `%RIP` was reported. Anyway, in both cases the [SWAPGS](http://www.felixcloutier.com/x86/SWAPGS.html) instruction will be executed and values from `MSR_KERNEL_GS_BASE` and `MSR_GS_BASE` will be swapped. From this moment the `%gs` register will point to the base address of kernel structures. So, the `SWAPGS` instruction is called and it was main point of the `error_entry` routing.
--->
何故ならば、truncated `%RIP`として報告されたように、faultする可能性を持つためです。
どちらの場合も、 [SWAPGS](http://www.felixcloutier.com/x86/SWAPGS.html) 命令が実行され、`MSR_KERNEL_GS_BASE` と `MSR_GS_BASE` の値が入れ替わります。
この瞬間から、 `%gs` レジスタはカーネル構造体のベースアドレスを指します。
`SWAPGS` 命令が呼ばれ、それが `error_entry` ルーチンのメインポイントです。

<!---
Now we can back to the `idtentry` macro. We may see following assembler code after the call of `error_entry`:
--->
さて、 `idtentry` マクロに返ることができます。
`error_entry` の呼び出しのあと、以下のアセンブラコードが見えるでしょう：

```assembly
movq	%rsp, %rdi
call	sync_regs
```

<!---
Here we put base address of stack pointer `%rdi` register which will be first argument (according to [x86_64 ABI](https://www.uclibc.org/docs/psABI-x86_64.pdf)) of the `sync_regs` function and call this function which is defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) source code file:
--->
スタックポインタのベースアドレスを、`%rdi` レジスタへ格納します。
`sync_regs` 関数の第一引数です（[x86_64 ABI](https://www.uclibc.org/docs/psABI-x86_64.pdf) に基づく）。
そして、[arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) ソースコードファイルに定義された `sync_regs` 関数を呼びます。

```C
asmlinkage __visible notrace struct pt_regs *sync_regs(struct pt_regs *eregs)
{
	struct pt_regs *regs = task_pt_regs(current);
	*regs = *eregs;
	return regs;
}
```

<!--- [JA] typo s/task_ptr_regs/task_pt_regs/ --->
<!---
This function takes the result of the `task_ptr_regs` macro which is defined in the [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h) header file, stores it in the stack pointer and return it. The `task_ptr_regs` macro expands to the address of `thread.sp0` which represents pointer to the normal kernel stack:
--->
この関数は、[arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h)ヘッダファイルで定義された `task_pt_regs` マクロの結果を使ってスタックポインタに格納し、それを返します。
`task_pt_regs` マクロは、標準のカーネルスタックへのポインタを表す  `thread.sp0` のアドレスへ展開されます：

```C
#define task_pt_regs(tsk)       ((struct pt_regs *)(tsk)->thread.sp0 - 1)
```

<!---
As we came from userspace, this means that exception handler will run in real process context. After we got stack pointer from the `sync_regs` we switch stack:
--->
ユーザ空間から来たので、これは、例外ハンドラは本当のプロセスコンテキストで走ることを意味します。
`sync_regs` からスタックポインタを取得したのでスタックを切り替えます。

```assembly
movq	%rax, %rsp
```

<!---
The last two steps before an exception handler will call secondary handler are:
--->
例外ハンドラが２番目のハンドラを前の最後の２つのステップは：

<!---
1. Passing pointer to `pt_regs` structure which contains preserved general purpose registers to the `%rdi` register:
--->
1. 予約された汎用レジスタを含む `pt_regs` 構造体へのポインタを `%rdi` へ渡す
   （２番めの例外ハンドラの第１引数として渡すため）。

```assembly
movq	%rsp, %rdi
```

<!---
as it will be passed as first parameter of secondary exception handler.
--->

<!---
2. Pass error code to the `%rsi` register as it will be second argument of an exception handler and set it to `-1` on the stack for the same purpose as we did it before - to prevent restart of a system call:
--->
2. エラーコードを、２番めの例外ハンドラの第２引数として、 `%rsi` レジスタへ渡し、
以前にしたのと同じ目的で（システムコールの再起動を防止するため）
スタック上のエラーコードに `-1` をセットします：

```
.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```

<!---
Additionally you may see that we zeroed the `%esi` register above in a case if an exception does not provide error code. 
--->
加えて、上記で例外がエラーコードを提供しない場合には、
`%esi` レジスタをゼロにしていることが見られます。

<!---
In the end we just call secondary exception handler:
--->
最後に、２番めの例外ハンドラを呼びます：

```assembly
call	\do_sym
```

<!---
which:
--->
ここで：

```C
dotraplinkage void do_debug(struct pt_regs *regs, long error_code);
```

<!---
will be for `debug` exception and:
--->
は、`debug` 例外のために呼ばれます。

```C
dotraplinkage void notrace do_int3(struct pt_regs *regs, long error_code);
```

<!---
will be for `int 3` exception. In this part we will not see implementations of secondary handlers, because of they are very specific, but will see some of them in one of next parts.
--->
は、 `int 3` 例外のために呼ばれます。
このパートでは、２番めのハンドラの実装については見ません。
２番めのハンドラはとても固有のものだからです。しかし、次のパートでいくつか見られるでしょう。

<!---
We just considered first case when an exception occurred in userspace. Let's consider last two.
--->
ユーザ空間で例外が起きた時の１つめの場合について検討しました。
残り２つを検討しましょう。


カーネル空間での例外（paranoid>0） （An exception with paranoid > 0 occurred in kernelspace）
--------------------------------------------------------------------------------

<!---
In this case an exception was occurred in kernelspace and `idtentry` macro is defined with `paranoid=1` for this exception. This value of `paranoid` means that we should use slower way that we saw in the beginning of this part to check do we really came from kernelspace or not. The `paranoid_entry` routing allows us to know this:
-->
この場合、例外がカーネル空間で発生し、この例外のために `idtentry` マクロが `paranoid=1` で定義されます。
`paranoid` の値は、このパートの最初で見たように、本当にカーネル空間から来たのかどうかを確認するために、遅い方法を使うことを意味します。
`paranoid_entry` ルーチンは、以下のようにわかります：

```assembly
ENTRY(paranoid_entry)
	cld
	SAVE_C_REGS 8
	SAVE_EXTRA_REGS 8
	movl	$1, %ebx
	movl	$MSR_GS_BASE, %ecx
	rdmsr
	testl	%edx, %edx
	js	1f
	SWAPGS
	xorl	%ebx, %ebx
1:	ret
END(paranoid_entry)
```

<!---
As you may see, this function represents the same that we covered before. We use second (slow) method to get information about previous state of an interrupted task. As we checked this and executed `SWAPGS` in a case if we came from userspace, we should to do the same that we did before: We need to put pointer to a structure which holds general purpose registers to the `%rdi` (which will be first parameter of a secondary handler) and put error code if an exception provides it to the `%rsi` (which will be second parameter of a secondary handler):
--->
見てのとおり、この関数は以前にカバーしたのと同じです。
割りこまれたタスクの前の状態に関する情報を得るために、２つめの（遅い）方法を使います。
これを確認するので、 ユーザ空間から来た場合には `SWAPGS` を実行するので、前にやったのと同様にします：
汎用レジスタを保持する構造体へのポインタを `%rdi` へ設定し（２番めのハンドラの第１引数）、例外がエラーコードを提供するならば `%rsi` へ設定します（２番めのハンドラの第２引数）。


```assembly
movq	%rsp, %rdi

.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```

<!--- [JA] typo s/fram/frame/ --->
<!---
The last step before a secondary handler of an exception will be called is cleanup of new `IST` stack frame:
--->
例外の２番めのハンドラが呼ばれる前の最後のステップは、 `IST` スタックフレームのクリーンアップです：

```assembly
.if \shift_ist != -1
	subq	$EXCEPTION_STKSZ, CPU_TSS_IST(\shift_ist)
.endif
```

<!---
You may remember that we passed the `shift_ist` as argument of the `idtentry` macro. Here we check its value and if its not equal to `-1`, we get pointer to a stack from `Interrupt Stack Table` by `shift_ist` index and setup it.
--->
`idtentry` マクロの引数として、 `shift_ist` を渡していたことを思い出してください。
ここでその値をチェックし、 `-1` と異なる時、 `shift_ist` をインデックスとした `Interrupt Stack Table` からスタックへのポインタを取得し、設定します。

<!---
In the end of this second way we just call secondary exception handler as we did it before:
--->
この２つめの方法の最後に、前にしたように２番めの例外ハンドラを呼びます。

```assembly
call	\do_sym
```

<!---
The last method is similar to previous both, but an exception occured with `paranoid=0` and we may use fast method determination of where we are from.
--->
最後の方法は、前のどちらとも似ていますが、例外は `paranoid=0` で発生します。
そして、何処から来たのかを決定するために高速な方法を使います。


例外ハンドラからの脱出 （Exit from an exception handler）
--------------------------------------------------------------------------------

<!---
After secondary handler will finish its works, we will return to the `idtentry` macro and the next step will be jump to the `error_exit`:
--->
２番めのハンドラがその作業を終えたあと、 `idtentry` マクロへ返ってきます。
つぎのステップは、 `error_exit` ルーチン へのジャンプです：

```assembly
jmp	error_exit
```

<!---
routine. The `error_exit` function defined in the same [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) assembly source code file and the main goal of this function is to know where we are from (from userspace or kernelspace) and execute `SWPAGS` depends on this. Restore registers to previous state and execute `iret` instruction to transfer control to an interrupted task.
--->
`error_exit` 関数は、[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) アセンブラソースコードファイルで定義されます。
この関数の主なゴールは、何処から来たのか（ユーザ空間化カーネル空間か）を知ることと、それに依存して `SWPAGS` を実行することです。
レジスタ群を前の状態に戻し、割りこまれたタスクへ制御を移すために `iret` 命令を実行します。

<!---
That's all.
--->
以上です。


まとめ（Conclusion）
--------------------------------------------------------------------------------

<!---
It is the end of the third part about interrupts and interrupt handling in the Linux kernel. We saw the initialization of the [Interrupt descriptor table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) in the previous part with the `#DB` and `#BP` gates and started to dive into preparation before control will be transferred to an exception handler and implementation of some interrupt handlers in this part. In the next part we will continue to dive into this theme and will go next by the `setup_arch` function and will try to understand interrupts handling related stuff.
--->
Linuxカーネルの割り込みと割り込み処理に関する３つめのパートの終わりです。
前のパートでは、 `#DB` ゲート `#BP` ゲート の[Interrupt descriptor table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) の初期化を見ました。
そして、このパートではいくつかの割り込みハンドラの実装と例外ハンドラへ制御を移す前の準備へ取り組みました。
つぎのパートでは、引き続きこれらのテーマに取り組み、 `setup_arch` 関数で次へ進みます。
割り込み処理に関することについて理解することを試みます。

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Debug registers](http://en.wikipedia.org/wiki/X86_debug_register)
* [Intel 80385](http://en.wikipedia.org/wiki/Intel_80386)
* [INT 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3)
* [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [TSS](http://en.wikipedia.org/wiki/Task_state_segment)
* [GNU assembly .error directive](https://sourceware.org/binutils/docs/as/Error.html#Error)
* [dwarf2](http://en.wikipedia.org/wiki/DWARF)
* [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html)
* [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [system call](http://en.wikipedia.org/wiki/System_call)
* [swapgs](http://www.felixcloutier.com/x86/SWAPGS.html)
* [SIGTRAP](https://en.wikipedia.org/wiki/Unix_signal#SIGTRAP)
* [Per-CPU variables](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [kgdb](https://en.wikipedia.org/wiki/KGDB)
* [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)
