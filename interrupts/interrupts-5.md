Interrupts and Interrupt Handling. Part 5.
================================================================================

例外ハンドラの実装（Implementation of exception handlers）
--------------------------------------------------------------------------------

<!---
This is the fifth part about an interrupts and exceptions handling in the Linux kernel and in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-4.html) we stopped on the setting of interrupt gates to the [Interrupt descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table). We did it in the `trap_init` function from the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) source code file. We saw only setting of these interrupt gates in the previous part and in the current part we will see implementation of the exception handlers for these gates. The preparation before an exception handler will be executed is in the [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) assembly file and occurs in the [idtentry](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S#L820) macro that defines exceptions entry points:
--->
これは、Linuxカーネルの割り込みハンドラと例外ハンドラに関する５番目のパートです。
[前のパート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-4.html)では、[Interrupt descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)への割り込みゲートの設定で止まっています。
それは [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)ソースコードファイルの `trap_init`関数で行いました。
前回では、これらの割り込みゲートの設定だけをみましたが、今回は、これらのゲートの例外ハンドラの実装を見ます。
例外ハンドラが実行される前の準備は、[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) アセンブリファイルで、例外エントリポイントを定義する [idtentry](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S#L820) マクロ内で発生します。

```assembly
idtentry divide_error			        do_divide_error			       has_error_code=0
idtentry overflow			            do_overflow			           has_error_code=0
idtentry invalid_op			            do_invalid_op			       has_error_code=0
idtentry bounds				            do_bounds			           has_error_code=0
idtentry device_not_available		    do_device_not_available		   has_error_code=0
idtentry coprocessor_segment_overrun	do_coprocessor_segment_overrun has_error_code=0
idtentry invalid_TSS			        do_invalid_TSS			       has_error_code=1
idtentry segment_not_present		    do_segment_not_present		   has_error_code=1
idtentry spurious_interrupt_bug		    do_spurious_interrupt_bug	   has_error_code=0
idtentry coprocessor_error		        do_coprocessor_error		   has_error_code=0
idtentry alignment_check		        do_alignment_check		       has_error_code=1
idtentry simd_coprocessor_error		    do_simd_coprocessor_error	   has_error_code=0
```

<!---
The `idtentry` macro does following preparation before an actual exception handler (`do_divide_error` for the `divide_error`, `do_overflow` for the `overflow` and etc.) will get control. In another words the `idtentry` macro allocates place for the registers ([pt_regs](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/ptrace.h#L43) structure) on the stack, pushes dummy error code for the stack consistency if an interrupt/exception has no error code, checks the segment selector in the `cs` segment register and switches depends on the previous state(userspace or kernelspace). After all of these preparations it makes a call of an actual interrupt/exception handler:
--->
 `idtentry`マクロは、実際の例外ハンドラ（ `divide_error`例外は `do_divide_error`、 `overflow`例外は `do_overflow`など）が制御を得る前に以下の準備を行います。

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
	...
	...
	...
	call	\do_sym
	...
	...
	...
END(\sym)
.endm
```

<!---
After an exception handler will finish its work, the `idtentry` macro restores stack and general purpose registers of an interrupted task and executes [iret](http://x86.renejeschke.de/html/file_module_x86_id_145.html) instruction:
--->
例外ハンドラがその作業を終えたあと、 `idtentry`マクロがスタックと、割りこまれたタスクの汎用レジスタとを元の状態に戻して、[iret](http://x86.renejeschke.de/html/file_module_x86_id_145.html) 命令を実行します：

```assembly
ENTRY(paranoid_exit)
	...
	...
	...
	RESTORE_EXTRA_REGS
	RESTORE_C_REGS
	REMOVE_PT_GPREGS_FROM_STACK 8
	INTERRUPT_RETURN
END(paranoid_exit)
```

<!---
where `INTERRUPT_RETURN` is:
--->
ここで、 `INTERRUPT_RETURN`は：

```assembly
#define INTERRUPT_RETURN	jmp native_iret
...
ENTRY(native_iret)
.global native_irq_return_iret
native_irq_return_iret:
iretq
```

<!---
More about the `idtentry` macro you can read in the third part of the [http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-3.html](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-3.html) chapter. Ok, now we saw the preparation before an exception handler will be executed and now time to look on the handlers. First of all let's look on the following handlers:
--->
`idtentry`マクロの詳細は、[割り込み処理と例外処理の第三回](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-3.html) で読めます。
さて、例外ハンドラが実行される前の準備をみました。今、ハンドラを見ます。
最初に以下のハンドラを見てみましょう：

* divide_error
* overflow
* invalid_op
* coprocessor_segment_overrun
* invalid_TSS
* segment_not_present
* stack_segment
* alignment_check

<!---
All these handlers defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) source code file with the `DO_ERROR` macro:
--->
これら全てのハンドラは、 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) ソースコードファイル内で、 `DO_ERROR`マクロを使って定義されます：

```C
DO_ERROR(X86_TRAP_DE,     SIGFPE,  "divide error",                divide_error)
DO_ERROR(X86_TRAP_OF,     SIGSEGV, "overflow",                    overflow)
DO_ERROR(X86_TRAP_UD,     SIGILL,  "invalid opcode",              invalid_op)
DO_ERROR(X86_TRAP_OLD_MF, SIGFPE,  "coprocessor segment overrun", coprocessor_segment_overrun)
DO_ERROR(X86_TRAP_TS,     SIGSEGV, "invalid TSS",                 invalid_TSS)
DO_ERROR(X86_TRAP_NP,     SIGBUS,  "segment not present",         segment_not_present)
DO_ERROR(X86_TRAP_SS,     SIGBUS,  "stack segment",               stack_segment)
DO_ERROR(X86_TRAP_AC,     SIGBUS,  "alignment check",             alignment_check)
``` 

<!---
As we can see the `DO_ERROR` macro takes 4 parameters:
--->
`DO_ERROR`マクロが４つのパラメータを取ることが見て取れます。順に：

<!---
* Vector number of an interrupt;
* Signal number which will be sent to the interrupted process;
* String which describes an exception;
* Exception handler entry point.
--->
* 割り込みのヴェクタ番号
* 割りこまれたプロセスに送られるシグナル番号
* 例外の種類をあらわす文字列
* 例外ハンドラのエントリポイント

<!---
This macro defined in the same source code file and expands to the function with the `do_handler` name:
--->
このマクロは同じソースコード内に定義され、 `do_handler`という名前の関数に展開されます：

```C
#define DO_ERROR(trapnr, signr, str, name)                              \
dotraplinkage void do_##name(struct pt_regs *regs, long error_code)     \
{                                                                       \
        do_error_trap(regs, error_code, str, trapnr, signr);            \
}
```

<!---
Note on the `##` tokens. This is special feature - [GCC macro Concatenation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html#Concatenation) which concatenates two given strings. For example, first `DO_ERROR` in our example will expands to the:
--->
`##` トークンに注意してください。
これは特別な機能で、２つの文字列を結合します（[GCC macro Concatenation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html#Concatenation)）。
例えば、 最初の  `DO_ERROR`は以下のように展開されます：

```C
dotraplinkage void do_divide_error(struct pt_regs *regs, long error_code)     \
{
	...
}
```

<!---
We can see that all functions which are generated by the `DO_ERROR` macro just make a call of the `do_error_trap` function from the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c). Let's look on implementation of the `do_error_trap` function.
--->
`DO_ERROR`マクロによって生成された全ての関数は、ちょうど [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) の `do_error_trap`関数の呼び出しを生成することがわかります。


Trap handlers
--------------------------------------------------------------------------------

<!---
The `do_error_trap` function starts and ends from the two following functions:
--->
`do_error_trap`関数は、[include/linux/context_tracking.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking.h) によると以下の２つの関数から開始、終了します

```C
enum ctx_state prev_state = exception_enter();
...
...
...
exception_exit(prev_state);
```

<!---
from the [include/linux/context_tracking.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking.h). The context tracking in the Linux kernel subsystem which provide kernel boundaries probes to keep track of the transitions between level contexts with two basic initial contexts: `user` or `kernel`. The `exception_enter` function checks that context tracking is enabled. After this if it is enabled, the `exception_enter` reads previous context and compares it with the `CONTEXT_KERNEL`. If the previous context is `user`, we call `context_tracking_exit` function from the [kernel/context_tracking.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/context_tracking.c) which inform the context tracking subsystem that a processor is exiting user mode and entering the kernel mode:
--->
Linuxカーネルサブシステムの、カーネル境界を提供するコンテキストトラッキングは、２つの基本的な初期コンテキスト（ `user` `kernel`）のレベルコンテキスト間の繊維の記録を保持するためにプローブします。
`exception_enter`関数は、コンテキストトラッキングが有効であることを確認します。
このあと、もし有効であれば `exception_enter`は以前のコンテキスト状態を読み、 `CONTEXT_KERNEL`と比較します。
もし以前のコンテキストが `user`であれば、プロセッサがユーザーモードにあった、カーネルモードへ入ろうとしていることをコンテキストトラッキングサブシステムへ通知する [kernel/context_tracking.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/context_tracking.c) の `context_tracking_exit`関数を呼び出します：

```C
if (!context_tracking_is_enabled())
	return 0;

prev_ctx = this_cpu_read(context_tracking.state);
if (prev_ctx != CONTEXT_KERNEL)
	context_tracking_exit(prev_ctx);

return prev_ctx;
```

<!---
If previous context is non `user`, we just return it. The `pre_ctx` has `enum ctx_state` type which defined in the [include/linux/context_tracking_state.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking_state.h) and looks as:
--->
以前のコンテキストが `user` でなければ、すぐに返ります。
`pre_ctx`は、 [include/linux/context_tracking_state.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking_state.h) で定義された `enum ctx_state`型で、以下のとおりです：

```C
enum ctx_state {
	CONTEXT_KERNEL = 0,
	CONTEXT_USER,
	CONTEXT_GUEST,
} state;
```

<!---
The second function is `exception_exit` defined in the same [include/linux/context_tracking.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking.h) file and checks that context tracking is enabled and call the `contert_tracking_enter` function if the previous context was `user`:
--->
２つめの関数は [include/linux/context_tracking.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking.h) で定義された  `exception_exit`で、コンテキストトラッキングが有効であるか確認し、以前のコンテキストが `user`であれば、 `contert_tracking_enter`関数を呼び出します：

```C
static inline void exception_exit(enum ctx_state prev_ctx)
{
	if (context_tracking_is_enabled()) {
		if (prev_ctx != CONTEXT_KERNEL)
			context_tracking_enter(prev_ctx);
	}
}
```

<!---
The `context_tracking_enter` function informs the context tracking subsystem that a processor is going to enter to the user mode from the kernel mode. We can see the following code between the `exception_enter` and `exception_exit`:
--->
`context_tracking_enter`関数は、プロセッサがカーネルモードからユーザモードへ切り替わることをコンテキストトラッキングサブシステムへ通知します。

```C
if (notify_die(DIE_TRAP, str, regs, error_code, trapnr, signr) !=
		NOTIFY_STOP) {
	conditional_sti(regs);
	do_trap(trapnr, signr, str, regs, error_code,
		fill_trap_info(regs, signr, trapnr, &info));
}
```

<!---
First of all it calls the `notify_die` function which defined in the [kernel/notifier.c](https://github.com/torvalds/linux/tree/master/kernel/notifier.c). To get notified for [kernel panic](https://en.wikipedia.org/wiki/Kernel_panic), [kernel oops](https://en.wikipedia.org/wiki/Linux_kernel_oops), [Non-Maskable Interrupt](https://en.wikipedia.org/wiki/Non-maskable_interrupt) or other events the caller needs to insert itself in the `notify_die` chain and the `notify_die` function does it. The Linux kernel has special mechanism that allows kernel to ask when something happens and this mechanism called `notifiers` or `notifier chains`. This mechanism used for example for the `USB` hotplug events (look on the [drivers/usb/core/notify.c](https://github.com/torvalds/linux/tree/master/drivers/usb/core/notify.c)), for the memory [hotplug](https://en.wikipedia.org/wiki/Hot_swapping) (look on the [include/linux/memory.h](https://github.com/torvalds/linux/tree/master/include/linux/memory.h), the `hotplug_memory_notifier` macro and etc...), system reboots and etc. A notifier chain is thus a simple, singly-linked list. When a Linux kernel subsystem wants to be notified of specific events, it fills out a special `notifier_block` structure and passes it to the `notifier_chain_register` function. An event can be sent with the call of the `notifier_call_chain` function. First of all the `notify_die` function fills `die_args` structure with the trap number, trap string, registers and other values:
--->
最初に、[kernel/notifier.c](https://github.com/torvalds/linux/tree/master/kernel/notifier.c) で定義された`notify_die`関数を呼び出します。
[kernel panic](https://en.wikipedia.org/wiki/Kernel_panic)や
[kernel oops](https://en.wikipedia.org/wiki/Linux_kernel_oops)や
[Non-Maskable Interrupt](https://en.wikipedia.org/wiki/Non-maskable_interrupt)や
他のイベントの通知を受けるために、呼び出し元は自身を`notify_die`チェインに挿入する必要があります。
`notify_die`関数がそれを行います。
Linuxカーネルは、なにか起こる時を問い合わせるための特別なメカニズムを持っています。
このメカニズムは`notifiers`や`notifier chains`と呼ばれます。
このメカニズムは、例えば`USB`ホットプラグイベント（[drivers/usb/core/notify.c](https://github.com/torvalds/linux/tree/master/drivers/usb/core/notify.c)を参照してください）や、
[メモリホットプラグ](https://en.wikipedia.org/wiki/Hot_swapping)（[include/linux/memory.h](https://github.com/torvalds/linux/tree/master/include/linux/memory.h)の`hotplug_memory_notifier`マクロなどを参照してください）、
システムリブートなどで使われます。
通知鎖（notify chain）は、したがってシンプルで、シングルリンクリストです。
Linuxカーネルサブシステムは、特定のイベントの通知をしたいときに、特別な`notifier_block`構造体を埋めて、`notifier_chain_register`関数を呼びます。
イベントは`notifier_call_chain`関数の呼び出しで送ることができます。
`notify_die`関数の最初では、`die_args`構造体を、トラップ番号、トラップ文字列、レジスタ、その他の値で埋めます：


```C
struct die_args args = {
       .regs   = regs,
       .str    = str,
       .err    = err,
       .trapnr = trap,
       .signr  = sig,
}
```

<!---
and returns the result of the `atomic_notifier_call_chain` function with the `die_chain`:
--->
そして、`die_chain`を引数に呼び出した`atomic_notifier_call_chain`関数の結果を返します：

```C
static ATOMIC_NOTIFIER_HEAD(die_chain);
return atomic_notifier_call_chain(&die_chain, val, &args);
```

<!---
which just expands to the `atomic_notifier_head` structure that contains lock and `notifier_block`:
--->
`die_chain`は、 lockと`notifier_block`を含む`atomic_notifier_head`構造体に展開されます：

```C
struct atomic_notifier_head {
        spinlock_t lock;
        struct notifier_block __rcu *head;
};
```

<!---
The `atomic_notifier_call_chain` function calls each function in a notifier chain in turn and returns the value of the last notifier function called. If the `notify_die` in the `do_error_trap` does not return `NOTIFY_STOP` we execute `conditional_sti` function from the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) that checks the value of the [interrupt flag](https://en.wikipedia.org/wiki/Interrupt_flag) and enables interrupt depends on it:
--->
`atomic_notifier_call_chain`関数は、順番にnotifier chain内の各関数を呼び出して、最後のnotifier関数呼び出しの値を返します。
`do_error_trap`の`notify_die`が`NOTIFY_STOP`を返さない時、[interrupt flag](https://en.wikipedia.org/wiki/Interrupt_flag)の値をチェックし、それに依存した割り込みを有効にする [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)にある`conditional_sti`関数を実行します：

```C
static inline void conditional_sti(struct pt_regs *regs)
{
        if (regs->flags & X86_EFLAGS_IF)
                local_irq_enable();
}
```

<!---
more about `local_irq_enable` macro you can read in the second [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-2.html) of this chapter. The next and last call in the `do_error_trap` is the `do_trap` function. First of all the `do_trap` function defined the `tsk` variable which has `task_struct` type and represents the current interrupted process. After the definition of the `tsk`, we can see the call of the `do_trap_no_signal` function:
--->
`local_irq_enable`マクロに関する詳細は、 この章の [第二回](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-2.html) を読むことができます。
次の`do_error_trap`内の最後の呼び出しは、`do_trap`関数です。
`do_trap`関数の最初に、`task_struct`型である`tsk`変数を定義します。これは現在の割りこまれたプロセスを表します。
`tsk`の定義のあと、`do_trap_no_signal`関数を呼び出していることがわかります：


```C
struct task_struct *tsk = current;

if (!do_trap_no_signal(tsk, trapnr, str, regs, error_code))
	return;
```

<!---
The `do_trap_no_signal` function makes two checks:

* Did we come from the [Virtual 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) mode;
* Did we come from the kernelspace.
--->
`do_trap_no_signal`関数は、以下の２つのチェックを行います：

* [Virtual 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) モードから来たのか
* カーネル空間から来たのか

```C
if (v8086_mode(regs)) {
	...
}

if (!user_mode(regs)) {
	...
}

return -1;
```

<!---
We will not consider first case because the [long mode](https://en.wikipedia.org/wiki/Long_mode) does not support the [Virtual 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) mode. In the second case we invoke `fixup_exception` function which will try to recover a fault and `die` if we can't:
--->
最初のケースは、 [long mode](https://en.wikipedia.org/wiki/Long_mode) が[Virtual 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) モードをサポートしませんので、検討しません。
ふたつ目のケースでは、faultから復旧しようと試みて、不可能なときは`die`関数を実行する、`fixup_exception`関数を呼び出します：

```C
if (!fixup_exception(regs)) {
	tsk->thread.error_code = error_code;
	tsk->thread.trap_nr = trapnr;
	die(str, regs, error_code);
}
```

<!---
The `die` function defined in the [arch/x86/kernel/dumpstack.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/dumpstack.c) source code file, prints useful information about stack, registers, kernel modules and caused kernel [oops](https://en.wikipedia.org/wiki/Linux_kernel_oops). If we came from the userspace the `do_trap_no_signal` function will return `-1` and the execution of the `do_trap` function will continue. If we passed through the `do_trap_no_signal` function and did not exit from the `do_trap` after this, it means that previous context was - `user`.  Most exceptions caused by the processor are interpreted by Linux as error conditions, for example division by zero, invalid opcode and etc. When an exception occurs the Linux kernel sends a [signal](https://en.wikipedia.org/wiki/Unix_signal) to the interrupted process that caused the exception to notify it of an incorrect condition. So, in the `do_trap` function we need to send a signal with the given number (`SIGFPE` for the divide error, `SIGILL` for the overflow exception and etc...). First of all we save error code and vector number in the current interrupts process with the filling `thread.error_code` and `thread_trap_nr`:
--->
`die`関数は、[arch/x86/kernel/dumpstack.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/dumpstack.c) ソースコードファイルで定義されており、以下の有益な情報を表示します：
- スタック
- レジスタ
- カーネルモジュール
- dieの要因となった kernel [oops](https://en.wikipedia.org/wiki/Linux_kernel_oops)

もしユーザ空間から来た場合、`do_trap_no_signal`関数は `-1`を返し、`do_trap`関数の実行を継続します。
`do_trap_no_signal`関数の実行をパスし、かつ、このあとで`do_trap`から返ってこない場合、以前のコンテキストは`user`であることを意味します。
プロセッサによって引き起こされるほとんどの例外は、Linuxによってエラー状態として解釈されます。
例えば、ゼロ除算、無効命令、などです。
例外が発生した時、Linuxカーネルは不正な状態を通知する原因となった割りこまれたプロセスへ [signal](https://en.wikipedia.org/wiki/Unix_signal) を送ります。

したがって、`do_trap`関数では、与えられた番号（`SIGFPE`は除算エラー、`SIGILL`はオーバーフロー例外、など）をシグナルで送る必要があります。
最初に、現在の割りこんだプロセス内の全てのエラーコードとヴェクタ番号とを`thread.error_code`と`thread_trap_nr`に保存します：

```C
tsk->thread.error_code = error_code;
tsk->thread.trap_nr = trapnr;
```

<!---
After this we make a check do we need to print information about unhandled signals for the interrupted process. We check that `show_unhandled_signals` variable is set, that `unhandled_signal` function from the [kernel/signal.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/signal.c) will return unhandled signal(s) and [printk](https://en.wikipedia.org/wiki/Printk) rate limit:
--->
このあと、割りこまれたプロセスに、処理できなかったシグナルに関する情報を表示する必要があるか、次の順にチェックします：
- `show_unhandled_signals`変数がセットされているか、
- [kernel/signal.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/signal.c) の `unhandled_signal`関数は処理できないシグナルを返しますので、それが非ゼロか、
- [printk](https://en.wikipedia.org/wiki/Printk) rate limitにひっかかっていないか、

```C
#ifdef CONFIG_X86_64
	if (show_unhandled_signals && unhandled_signal(tsk, signr) &&
	    printk_ratelimit()) {
		pr_info("%s[%d] trap %s ip:%lx sp:%lx error:%lx",
			tsk->comm, tsk->pid, str,
			regs->ip, regs->sp, error_code);
		print_vma_addr(" in ", regs->ip);
		pr_cont("\n");
	}
#endif
```

<!---
And send a given signal to interrupted process:
--->
そして、割り込んだプロセスへ与えられたシグナルを送ります：

```C
force_sig_info(signr, info ?: SEND_SIG_PRIV, tsk);
```

<!---
This is the end of the `do_trap`. We just saw generic implementation for eight different exceptions which are defined with the `DO_ERROR` macro. Now let's look on another exception handlers.
--->
これで`do_trap`は終了です。
`DO_ERROR`マクロで定義された ８つの異なる例外のための、共通の実装をみてきました。
他の例外ハンドラについて見ていきましょう。


Double fault
--------------------------------------------------------------------------------

<!---
The next exception is `#DF` or `Double fault`. This exception occurs when the processor detected a second exception while calling an exception handler for a prior exception. We set the trap gate for this exception in the previous part:
--->
つぎの例外は`#DF`（`double fault`）です。
この例外は、先に発生した例外のために例外ハンドラを呼んでいる時に、プロセッサが２つめの例外を検出した時に起きます。
前のパートでこの例外のためのトラップゲートを設定しました：

```C
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

<!---
Note that this exception runs on the `DOUBLEFAULT_STACK` [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks) which has index - `1`:
--->
この例外は、index - `1` を持つ`DOUBLEFAULT_STACK` [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks) 上で走行します：

```C
#define DOUBLEFAULT_STACK 1
```

<!---
The `double_fault` is handler for this exception and defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c). The `double_fault` handler starts from the definition of two variables: string that describes exception and interrupted process, as other exception handlers:
--->
`double_fault`は、この例外のためのハンドラで、 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) で定義されます。
`double_fault`ハンドラは、２つの変数の定義から始まります：他の例外ハンドラと同じように、例外を記述する文字列と、割りこまれたプロセス

```C
static const char str[] = "double fault";
struct task_struct *tsk = current;
```

<!---
The handler of the double fault exception split on two parts. The first part is the check which checks that a fault is a `non-IST` fault on the `espfix64` stack. Actually the `iret` instruction restores only the bottom `16` bits when returning to a `16` bit segment. The `espfix` feature solves this problem. So if the `non-IST` fault on the espfix64 stack we modify the stack to make it look like `General Protection Fault`:
--->
double fault例外のハンドラは２つに分かれています。
１つめは、`espfix64`スタック上でおきた`non-IST` faultであるかどうかをチェックします。
`iret`命令は `16`ビットセグメントへ返る時には、下位 `16`ビットのみ復旧します。
`espfix`機能は、この問題を解決します。
したがって、`non-IST` faultが`espfix64`スタック上で起きたなら、`General Protection Fault`と見えるようにスタックを修正します：

```C
struct pt_regs *normal_regs = task_pt_regs(current);

memmove(&normal_regs->ip, (void *)regs->sp, 5*8);
ormal_regs->orig_ax = 0;
regs->ip = (unsigned long)general_protection;
regs->sp = (unsigned long)&normal_regs->orig_ax;
return;
```

<!---
In the second case we do almost the same that we did in the previous exception handlers. The first is the call of the `ist_enter` function that discards previous context, `user` in our case:
--->
２つめは、前の例外ハンドラで行ったこととほとんど同じです。
最初は、前のコンテキスト（ここでは`user`）を破棄する`ist_enter`関数の呼び出しです。

```C
ist_enter(regs);
```

<!---
And after this we fill the interrupted process with the vector number of the `Double fault` exception and error code as we did it in the previous handlers:
--->
このあと、前のハンドラで行ったのと同じように、`Double fault`例外のヴェクタ番号と、エラーコードとで、割り込んだプロセスを埋めます：

```C
tsk->thread.error_code = error_code;
tsk->thread.trap_nr = X86_TRAP_DF;
```

<!---
Next we print useful information about the double fault ([PID](https://en.wikipedia.org/wiki/Process_identifier) number, registers content):
--->
次に、double faultに関する有益な情報、[PID](https://en.wikipedia.org/wiki/Process_identifier)番号、レジスタコンテンツ、を表示します：

```C
#ifdef CONFIG_DOUBLEFAULT
	df_debug(regs, error_code);
#endif
```

<!---
And die:
--->
そして、`die`関数を呼びます：

```
	for (;;)
		die(str, regs, error_code);
```

<!---
That's all.
--->
以上で全てです。


Device not available exception handler
--------------------------------------------------------------------------------

<!---
The next exception is the `#NM` or `Device not available`. The `Device not available` exception can occur depending on these things:
--->
つぎの例外は、`#NM`（`Device not available`）です。
`Device not available`例外は、以下のものに依存して生じます：

<!---
* The processor executed an [x87 FPU](https://en.wikipedia.org/wiki/X87) floating-point instruction while the EM flag in [control register](https://en.wikipedia.org/wiki/Control_register) `cr0` was set;
* The processor executed a `wait` or `fwait` instruction while the `MP` and `TS` flags of register `cr0` were set;
* The processor executed an [x87 FPU](https://en.wikipedia.org/wiki/X87), [MMX](https://en.wikipedia.org/wiki/MMX_%28instruction_set%29) or [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) instruction while the `TS` flag in control register `cr0` was set and the `EM` flag is clear.
--->
* プロセッサが、[control register](https://en.wikipedia.org/wiki/Control_register)`cr0`の EM フラグがセットされている間に、[x87 FPU](https://en.wikipedia.org/wiki/X87) 浮動小数点演算命令を実行した時
* プロセッサが、`cr0`レジスタの`MP`フラグと`TS`フラグとがセットされている間に、`wait`や`fwait`命令を実行したとき
* プロセッサが、`cr0`の`TS`フラグがセットされていて、かつ、`EM`フラグがクリアされているときに、[x87 FPU](https://en.wikipedia.org/wiki/X87), [MMX](https://en.wikipedia.org/wiki/MMX_%28instruction_set%29)命令や [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) 命令を実行したとき

<!---
The handler of the `Device not available` exception is the `do_device_not_available` function and it defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) source code file too. It starts and ends from the getting of the previous context, as other traps which we saw in the beginning of this part:
--->
`Device not available`例外のハンドラは、`do_device_not_available`関数で、 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) ソースコードファイルに定義されています。
これは、前のコンテキストの取得から開始して終了します。
これは、この部分の初めに見た他のトラップと同様です：

```C
enum ctx_state prev_state;
prev_state = exception_enter();
...
...
...
exception_exit(prev_state);
```

<!---
In the next step we check that `FPU` is not eager:
--->
つぎのステップでは、`FPU`が熱心ではないことをチェックします：
<!--- [JA] busyかどうかを見ているのでしょうか？ --->

```C
BUG_ON(use_eager_fpu());
```
<!---
When we switch into a task or interrupt we may avoid loading the `FPU` state. If a task will use it, we catch `Device not Available exception` exception. If we loading the `FPU` state during task switching, the `FPU` is eager. In the next step we check `cr0` control register on the `EM` flag which can show us is `x87` floating point unit present (flag clear) or not (flag set):
--->
タスクまたは割り込みへ切り替わるとき、`FPU`状態を読み込むことを除外します。
タスクが`FPU`を使うなら、`Device not Available exception`例外をキャッチします。
タスクスイッチの間に`FPU`状態を読み込むなら、`FPU`は熱心です。
次のステップは`cr0`制御レジスタの`EM`フラグをチェックします。
`EM`フラグは、`x87`浮動小数点ユニットが存在する（フラグクリア）か、しない（フラグセット）かを表します（EM=FPU EMulationの意味。1ならエミュレート有効となりますので、ユニットは存在しない）。

```C
#ifdef CONFIG_MATH_EMULATION
	if (read_cr0() & X86_CR0_EM) {
		struct math_emu_info info = { };

		conditional_sti(regs);

		info.regs = regs;
		math_emulate(&info);
		exception_exit(prev_state);
		return;
	}
#endif
```

<!---
If the `x87` floating point unit not presented, we enable interrupts with the `conditional_sti`, fill the `math_emu_info` (defined in the [arch/x86/include/asm/math_emu.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/math_emu.h)) structure with the registers of an interrupt task and call `math_emulate` function from the [arch/x86/math-emu/fpu_entry.c](https://github.com/torvalds/linux/tree/master/arch/x86/math-emu/fpu_entry.c). As you can understand from function's name, it emulates `X87 FPU` unit (more about the `x87` we will know in the special chapter). In other way, if `X86_CR0_EM` flag is clear which means that `x87 FPU` unit is presented, we call the `fpu__restore` function from the [arch/x86/kernel/fpu/core.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/fpu/core.c) which copies the `FPU` registers from the `fpustate` to the live hardware registers. After this `FPU` instructions can be used:
--->
`x87`浮動小数点ユニットが存在しなければ、`conditional_sti`関数で割り込みを有効にし、割り込んだタスクのレジスタで`math_emu_info`（  [arch/x86/include/asm/math_emu.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/math
_emu.h) で定義）を埋めて、  [arch/x86/math-emu/fpu_entry.c](https://github.com/torvalds/linux/tree/master/arch/x86/math-emu/fpu_entr
y.c)の`math_emulate`関数を呼び出します。
関数名から判るように、`X87 FPU`ユニット（`x87`については別の特別な章で知ることができるでしょう）をエミュレートします。
言い換えると、`x87 FPU`が存在することを意味する`X86_CR0_EM`フラグがクリアされている場合は、`fpustate`から 生きているハードウェアレジスタへ`FPU`レジスタをコピーする[arch/x86/kernel/fpu/core.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/fpu/core.c) 
の`fpu__restore`関数を呼び出します。
このあと、`FPU`命令を使うことができます。

```C
fpu__restore(&current->thread.fpu);
```


General protection fault exception handler
--------------------------------------------------------------------------------

<!---
The next exception is the `#GP` or `General protection fault`. This exception occurs when the processor detected one of a class of protection violations called `general-protection violations`. It can be:
-->
次の例外は、`#GP`（`General protection fault`）です。
この例外は、プロセッサが`general-protection violations`と呼ばれる保護違反クラスのひとつを検出した時に発生します。
例えば以下のとき：

<!---
* Exceeding the segment limit when accessing the `cs`, `ds`, `es`, `fs` or `gs` segments;
* Loading the `ss`, `ds`, `es`, `fs` or `gs` register with a segment selector for a system segment.;
* Violating any of the privilege rules;
* and other...
--->
* `cs`, `ds`, `es`, `fs`, `gs` セグメントへアクセスする時に、セグメント限界を超える場合
* システムセグメントのためのセグメントセレクタとして `ss`, `ds`, `es`, `fs`, `gs` レジスタを読み出す場合
* 特権ルールに違反する場合

訳注：まだ意味を理解せずに書いているので修正すべきです。

<!---
The exception handler for this exception is the `do_general_protection` from the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c). The `do_general_protection` function starts and ends as other exception handlers from the getting of the previous context:
--->
この例外の例外ハンドラは、 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) の`do_general_protection`です。
`do_general_protection`関数は、他の例外ハンドラと同じように、前のコンテキストを取得することから開始し、終了します：

```C
prev_state = exception_enter();
...
exception_exit(prev_state);
```

<!---
After this we enable interrupts if they were disabled and check that we came from the [Virtual 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) mode:
--->
このあと、割り込みが無効にされていた場合には割り込みを有効にし、[Virtual 
8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) モードから来たかどうかをチェックします：

```C
conditional_sti(regs);

if (v8086_mode(regs)) {
	local_irq_enable();
	handle_vm86_fault((struct kernel_vm86_regs *) regs, error_code);
	goto exit;
}
```

<!---
As long mode does not support this mode, we will not consider exception handling for this case. In the next step check that previous mode was kernel mode and try to fix the trap. If we can't fix the current general protection fault exception we fill the interrupted process with the vector number and error code of the exception and add it to the `notify_die` chain:
--->
ロングモードがこのモードをサポートしていませんので、このケースの例外処理については検討しません。
次のステップは、前のモードがカーネルモードかどうかをチェックし、trapを修正することを試みます。
現在の一般保護違反例外を修正できなければ、ヴェクタ番号と例外のエラーコードで割り込んだプロセスを埋めます。そして、`notify_die`チェインに追加します。

```C
if (!user_mode(regs)) {
	if (fixup_exception(regs))
		goto exit;

	tsk->thread.error_code = error_code;
	tsk->thread.trap_nr = X86_TRAP_GP;
	if (notify_die(DIE_GPF, "general protection fault", regs, error_code,
		       X86_TRAP_GP, SIGSEGV) != NOTIFY_STOP)
		die("general protection fault", regs, error_code);
	goto exit;
}
```

<!---
If we can fix exception we go to the `exit` label which exits from exception state:
--->
例外を修正することができれば、例外状態から抜ける`exit`ラベルへジャンプします。

```C
exit:
	exception_exit(prev_state);
```

<!---
If we came from user mode we send `SIGSEGV` signal to the interrupted process from user mode as we did it in the `do_trap` function:
--->
ユーザモードから来ていたならば、`do_trap`関数内で行ったようにユーザモードから割り込んだプロセスに`SIGSEGV`シグナルを送ります。

```C
if (show_unhandled_signals && unhandled_signal(tsk, SIGSEGV) &&
		printk_ratelimit()) {
	pr_info("%s[%d] general protection ip:%lx sp:%lx error:%lx",
		tsk->comm, task_pid_nr(tsk),
		regs->ip, regs->sp, error_code);
	print_vma_addr(" in ", regs->ip);
	pr_cont("\n");
}

force_sig_info(SIGSEGV, SEND_SIG_PRIV, tsk);
```

<!---
That's all.
--->
以上です。


まとめ（Conclusion）
--------------------------------------------------------------------------------

<!---
It is the end of the fifth part of the [Interrupts and Interrupt Handling](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) chapter and we saw implementation of some interrupt handlers in this part. In the next part we will continue to dive into interrupt and exception handlers and will see handler for the [Non-Maskable Interrupts](https://en.wikipedia.org/wiki/Non-maskable_interrupt), handling of the math [coprocessor](https://en.wikipedia.org/wiki/Coprocessor) and [SIMD](https://en.wikipedia.org/wiki/SIMD) coprocessor exceptions and many many more.
--->
[Interrupts and Interrupt 
Handling](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)の章の第五回の終わりです。
今回は、いくつかの割り込みハンドラの実装を見てきました。
次の回では、割り込みハンドラと例外ハンドラへ潜り続けます。そして、[Non-Maskable 
Interrupts](https://en.wikipedia.org/wiki/Non-maskable_interrupt)のハンドラ、[SIMD](https://en.wikipedia.org/wiki/SIMD)コプロセッサ例外、などなどを見ていきます。

<!---
If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).
--->
何か質問や提案があれば、コメントを書いたり、[twitter](https://twitter.com/0xAX)でpingしてください。

<!---
**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**
--->
**英語は私の第一言語ではないことに注意してください。ご不便をおかけして申し訳ございません。もし何か誤りを見つけたなら、 [linux-insides](https://github.com/0xAX/linux-insides)へPRを送ってください。**


Links
--------------------------------------------------------------------------------

* [Interrupt descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [iret instruction](http://x86.renejeschke.de/html/file_module_x86_id_145.html)
* [GCC macro Concatenation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html#Concatenation)
* [kernel panic](https://en.wikipedia.org/wiki/Kernel_panic)
* [kernel oops](https://en.wikipedia.org/wiki/Linux_kernel_oops)
* [Non-Maskable Interrupt](https://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [hotplug](https://en.wikipedia.org/wiki/Hot_swapping)
* [interrupt flag](https://en.wikipedia.org/wiki/Interrupt_flag)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [signal](https://en.wikipedia.org/wiki/Unix_signal)
* [printk](https://en.wikipedia.org/wiki/Printk)
* [coprocessor](https://en.wikipedia.org/wiki/Coprocessor)
* [SIMD](https://en.wikipedia.org/wiki/SIMD)
* [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [x87 FPU](https://en.wikipedia.org/wiki/X87)
* [control register](https://en.wikipedia.org/wiki/Control_register)
* [MMX](https://en.wikipedia.org/wiki/MMX_%28instruction_set%29)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-4.html)
