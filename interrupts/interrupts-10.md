Interrupts and Interrupt Handling. Part 10.
================================================================================

Last part
-------------------------------------------------------------------------------

<!---
This is the tenth part of the [chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) about interrupts and interrupt handling in the Linux kernel and in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-9.html) we saw a little about deferred interrupts and related concepts like `softirq`, `tasklet` and `workqeue`. In this part we will continue to dive into this theme and now it's time to look at real hardware driver.
--->
Linuxカーネルの[割り込みと割り込み処理の章](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)の１０番目のパートです。
[前のパート](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-9.html)では、
遅延割り込みと、それに関係する`softirq`、`tasklet`、`workqueue`を見ました。
このパートでは、このテーマについて引き続き潜り、本当のハードウェアドライバについて見ていきます。

<!---
Let's consider serial driver of the [StrongARM** SA-110/21285 Evaluation Board](http://netwinder.osuosl.org/pub/netwinder/docs/intel/datashts/27813501.pdf) board for example and will look how this driver requests an [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) line, 
what happens when an interrupt is triggered and etc. The source code of this driver is placed in the [drivers/tty/serial/21285.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/serial/21285.c) source code file. Ok, we have source code, let's start.
--->


[StrongARM** SA-110/21285 Evaluation Board](http://netwinder.osuosl.org/pub/netwinder/docs/intel/datashts/27813501.pdf)のシリアルドライバを仮定し、このドライバがどのようにして[IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)ラインを要求するのか、
割込みがトリガした時に何が起きるのか、を見ます。


Initialization of a kernel module
--------------------------------------------------------------------------------

<!---
We will start to consider this driver as we usually did it with all new concepts that we saw in this book. We will start to consider it from the intialization. As you already may know, the Linux kernel provides two macros for initialization and finalization of a driver or a kernel module:
--->
このドライバでは、この本で見たすべての新しいコンセプトでこのドライバを考えていきます。
初期化から考え真面目ましょう。既に御存知のとおり、Linuxカーネルはドライバまたはカーネルモジュールの初期化と終了のために、２つのマクロを提供します


* `module_init`;
* `module_exit`.

<!---
And we can find usage of these macros in our driver source code:
--->
ドライバソースコードに、これらのマクロの使い方を見つけることができます：

```C
module_init(serial21285_init);
module_exit(serial21285_exit);
```

<!---
The most part of device drivers can be compiled as a loadable kernel [module](https://en.wikipedia.org/wiki/Loadable_kernel_module) or in another way they can be statically linked into the Linux kernel. In the first case initialization of a device driver will be produced via the `module_init` and `module_exit` macros that are defined in the [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h):
--->
デバイスドライバの大部分は、[ローダブルカーネルモジュール](https://en.wikipedia.org/wiki/Loadable_kernel_module)としてコンパイルできます。
その他の手段としては、Linuxカーネルに静的にリンクすることができます。
カーネルモジュールの場合、デバイスドライバは、`module_init` マクロと `module_exit` マクロで提供されます。
これらのマクロは[include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) で定義されています。

```C
#define module_init(initfn)                                     \
        static inline initcall_t __inittest(void)               \
        { return initfn; }                                      \
        int init_module(void) __attribute__((alias(#initfn)));

#define module_exit(exitfn)                                     \
        static inline exitcall_t __exittest(void)               \
        { return exitfn; }                                      \
        void cleanup_module(void) __attribute__((alias(#exitfn)));
```

<!---
and will be called by the [initcall](http://kernelnewbies.org/Documents/InitcallMechanism) functions:
--->
そして、以下の関数は[initcall機能](http://kernelnewbies.org/Documents/InitcallMechanism)によって呼ばれます。

* `early_initcall`
* `pure_initcall`
* `core_initcall`
* `postcore_initcall`
* `arch_initcall`
* `subsys_initcall`
* `fs_initcall`
* `rootfs_initcall`
* `device_initcall`
* `late_initcall`

<!---
that are called in the `do_initcalls` from the [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c). Otherwise, if a device driver is statically linked into the Linux kernel, implementation of these macros will be following:
--->
[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) にある
`do_initcalls`関数で呼ばれます。言い換えると、デバイスドライバをLinuxカーネルに静的にリンクするなら、これらのマクロは以下のように実装されます：


```C
#define module_init(x)  __initcall(x);
#define module_exit(x)  __exitcall(x);
```

<!---
In this way implementation of module loading placed in the [kernel/module.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/module.c) source code file and initialization occurs in the `do_init_module` function. We will not dive into details about loadable modules in this chapter, but will see it in the special chapter that will describe Linux kernel modules. Ok, the `module_init` macro takes one parameter - the `serial21285_init` in our case. As we can understand from function's name, this function does stuff related to the driver initialization. Let's look at it:
--->
この場合、モジュール読出しの実装は、ソースコードファイル[kernel/module.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/module.c)の中にあり、`do_init_module` 関数内で初期化されます。
この章では、ローダブルモジュールについて深く潜りませんが、Linuxカーネルモジュールについて記述した特別な章を見られるでしょう。
OK、`module_init` マクロ は１つのパラメータ（この場合、`serial21285_init`）を持ちます。
関数名から判るように、この関数はドライバの初期化に関するものです。見ていきましょう：


```C
static int __init serial21285_init(void)
{
	int ret;

	printk(KERN_INFO "Serial: 21285 driver\n");

	serial21285_setup_ports();

	ret = uart_register_driver(&serial21285_reg);
	if (ret == 0)
		uart_add_one_port(&serial21285_reg, &serial21285_port);

	return ret;
}
```

<!---
As we can see, first of all it prints information about the driver to the kernel buffer and the call of the `serial21285_setup_ports` function. This function setups the base [uart](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver/transmitter) clock of the `serial21285_port` device:
--->
見て判るように、最初にカーネルバッファへドライバに関する情報を表示し、`serial21285_setup_ports`関数を呼んでいます。
この関数は、`serial21285_port`デバイスの[uart](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver/transmitter)ベースクロックを設定します。


```C
unsigned int mem_fclk_21285 = 50000000;

static void serial21285_setup_ports(void)
{
	serial21285_port.uartclk = mem_fclk_21285 / 4;
}
```

<!---
Here the `serial21285` is the structure that describes `uart` driver:
--->
以下が、`serial21285`で、`uart`ドライバを表す構造体です。

```C
static struct uart_driver serial21285_reg = {
	.owner			= THIS_MODULE,
	.driver_name	= "ttyFB",
	.dev_name		= "ttyFB",
	.major			= SERIAL_21285_MAJOR,
	.minor			= SERIAL_21285_MINOR,
	.nr			    = 1,
	.cons			= SERIAL_21285_CONSOLE,
};
```

<!---
If the driver registered successfully we attach the driver-defined port `serial21285_port` structure with the `uart_add_one_port` function from the [drivers/tty/serial/serial_core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/serial/serial_core.c) source code file and return from the `serial21285_init` function:
--->
ドライバが正常に登録されると、
[drivers/tty/serial/serial_core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/serial/serial_core.c)にある `uart_add_one_port` 関数で、ドライバ定義のポートである`serial21285_port`構造体をアタッチし、`serial21285_init`関数から返ります：

```C
if (ret == 0)
	uart_add_one_port(&serial21285_reg, &serial21285_port);

return ret;
```

<!---
That's all. Our driver is initialized. When an `uart` port will be opened with the call of the `uart_open` function from the [drivers/tty/serial/serial_core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/serial/serial_core.c), it will call the `uart_startup` function to start up the serial port. This function will call the `startup` function that is part of the `uart_ops` structure. Each `uart` driver has the definition of this structure, in our case it is:
--->
以上が全てで、ドライバは初期化されました。
`uart`ポートが、[drivers/tty/serial/serial_core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/serial/serial_core.c)の`uart_open`関数を呼び出して開く時に、シリアルポートを開始するために `uart_startup` 関数を呼びます。
この関数は、`uart_ops`構造体の `startup` 関数（メソッド）を呼びます。
各`uart`ドライバは、この構造体の定義を持っていて、この場合は`serial21285`構造体です。


```C
static struct uart_ops serial21285_ops = {
	...
	.startup	= serial21285_startup,
	...
}
```

<!---
`serial21285` structure. As we can see the `.strartup` field references on the `serial21285_startup` function. Implementation of this function is very interesting for us, because it is related to the interrupts and interrupt handling.
--->
`serial21285_startup`関数を参照する `.startup`フィールドが見られます。
この関数の実装は、割込みと割込みハンドリングとに関連して、とても興味深いです。


Requesting irq line
--------------------------------------------------------------------------------

<!---
Let's look at the implementation of the `serial21285` function:
--->
`serial21285`関数の実装を見ていきましょう：

```C
static int serial21285_startup(struct uart_port *port)
{
	int ret;

	tx_enabled(port) = 1;
	rx_enabled(port) = 1;

	ret = request_irq(IRQ_CONRX, serial21285_rx_chars, 0,
			  serial21285_name, port);
	if (ret == 0) {
		ret = request_irq(IRQ_CONTX, serial21285_tx_chars, 0,
				  serial21285_name, port);
		if (ret)
			free_irq(IRQ_CONRX, port);
	}

	return ret;
}
```

<!---
First of all about `TX` and `RX`. A serial bus of a device consists of just two wires: one for sending data and another for receiving. As such, serial devices should have two serial pins: the receiver - `RX`, and the transmitter - `TX`. With the call of first two macros: `tx_enabled` and `rx_enabled`, we enable these wires. The following part of these function is the greatest interest for us. Note on `request_irq` functions. This function registers an interrupt handler and enables a given interrupt line. Let's look at the implementation of this function and get into the details. This function defined in the [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h) header file and looks as:
--->
最初に`TX`と`RX`について。
デバイスのシリアルバスは、ちょうど２つのワイヤで構成されます。１つはデータ送信、他方は受信です。
したがって、シリアルデバイスは２つのシリアルピンを有するべきです。受信器（`RX`）と送信器（`TX`）です。
ソースコードの最初の２つのマクロ、`tx_enabled`と`rx_enabled`は、これらのワイヤを有効にします。
この関数の続きの部分は、とても興味深いです。
`request_irq`関数に注意してください。この関数は割込みハンドラを登録し、与えられた割込みラインを有効にします。
この関数の実装を詳細にみていきましょう。
この関数は、[include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h)ヘッダファイルで定義されており、以下のように見えます：

```C
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
            const char *name, void *dev)
{
        return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```

<!---
As we can see, the `request_irq` function takes five parameters:

* `irq` - the interrupt number that being requested;
* `handler` - the pointer to the interrupt handler;
* `flags` - the bitmask options;
* `name` - the name of the owner of an interrupt;
* `dev` - the pointer used for shared interrupt lines;
--->
見て判るように、`request_irq`関数は５つのパラメータを取ります：

* `irq` - 要求する割込み番号
* `handler` - 割込みハンドラへのポインタ
* `flags` - ビットマスクのオプション
* `name` - 割込みのオーナーとなる名前
* `dev` - 割込みラインを共有するためのポインタ

<!---
Now let's look at the calls of the `request_irq` functions in our example. As we can see the first parameter is `IRQ_CONRX`. We know that it is number of the interrupt, but what is it `CONRX`? This macro defined in the [arch/arm/mach-footbridge/include/mach/irqs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/arm/mach-footbridge/include/mach/irqs.h) header file. We can find the full list of interrupts that the `21285` board can generate. Note that in the second call of the `request_irq` function we pass the `IRQ_CONTX` interrupt number. Both these interrupts will handle `RX` and `TX` event in our driver. Implementation of these macros is easy:
--->
さて、この例での`request_irq`関数の呼び出しについて見てみましょう。
最初のパラメータは `IRQ_CONRX`と見えます。
割込みの番号であることを知っていますが、`CONRX`とはなんでしょうか。
このマクロは、[arch/arm/mach-footbridge/include/mach/irqs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/arm/mach-footbridge/include/mach/irqs.h)ヘッダファイルで定義されます。
このファイルには、`21285`ボードが生成する割込みの全てのリストを見つけることができます。
`request_irq`関数の２つ目の呼び出しでは、割込み番号`IRQ_CONTX`を渡しています。
この割込みは、本ドライバではそれぞれ`RX`と`TX`イベントを処理します。このマクロの実装は単純です：

```C
#define IRQ_CONRX               _DC21285_IRQ(0)
#define IRQ_CONTX               _DC21285_IRQ(1)
...
...
...
#define _DC21285_IRQ(x)         (16 + (x))
```

<!---
The [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) IRQs on this board are from `0` to `15`, so, our interrupts will have first two numbers: `16` and `17`. Second parameters for two calls of the `request_irq` functions are `serial21285_rx_chars` and `serial21285_tx_chars`. These functions will be called when an `RX` or `TX` interrupt occurred. We will not dive in this part into details of these functions, because this chapter covers the interrupts and interrupts handling but not device and drivers. The next parameter - `flags` and as we can see, it is zero in both calls of the `request_irq` function. All acceptable flags are defined as `IRQF_*` macros in the [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h). Some of it:
--->
このボードの、[ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) IRQsは、０から１５なので、
割込み番号は最初の２つ、１６と１７になります。
２つの`request_irq`関数を呼ぶ２つ目のパラメータは、それぞれ`serial21285_rx_chars` と `serial21285_tx_chars` です。
この関数は、それぞれ`RX`割込みと`TX`割込みが起きた時に呼ばれます。
この章では、割込みと割り込み処理をカバーしますが、ドライバとデバイスとはカバーしないので、この関数の詳細には潜りません。
そのつぎのパラメータ `flags`は、どちらの`request_irq`関数の呼び出しでもゼロです。
全ての許容されるフラグは、[include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h)ファイルの`IRQF_*`マクロで定義されます。いくつか抜粋します：

<!---
* `IRQF_SHARED` - allows sharing the irq among several devices;
* `IRQF_PERCPU` - an interrupt is per cpu;
* `IRQF_NO_THREAD` - an interrupt cannot be threaded;
* `IRQF_NOBALANCING` - excludes this interrupt from irq balancing;
* `IRQF_IRQPOLL` - an interrupt is used for polling;
* and etc.
--->

* `IRQF_SHARED` - この割込みは、複数のデバイス間でirqを共有する
* `IRQF_PERCPU` - この割込みは、CPU毎とする
* `IRQF_NO_THREAD` - この割込みは、スレッド化できない
* `IRQF_NOBALANCING` - この割込みは、irq balancingから除外する
* `IRQF_IRQPOLL` - 割込みはポーリングのために使う
* and etc.

<!---
In our case we pass `0`, so it will be `IRQF_TRIGGER_NONE`. This flag means that it does not imply any kind of edge or level triggered interrupt behaviour. To the fourth parameter (`name`), we pass the `serial21285_name` that defined as:
--->
本ドライバでは `0`を渡していますので、`IRQF_TRIGGER_NONE`となります。
このフラグが意味するところは、エッヂトリガ/レベルトリガ割込み動作ではないこと暗示しないことを意味します。
<!--- [JA] よくわからない --->
４番目のパラメータ（`name`）は、以下のように定義された `serial21285_name` を渡します：

```C
static const char serial21285_name[] = "Footbridge UART";
```

<!--
and will be displayed in the output of the `/proc/interrupts`. And in the last parameter we pass the pointer to the our main `uart_port` structure. Now we know a little about `request_irq` function and its parameters, let's look at its implemenetation. As we can see above, the `request_irq` function just makes a call of the `request_threaded_irq` function inside. The `request_threaded_irq` function defined in the [kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/manage.c) source code file and allocates a given interrupt line. If we will look at this function, it starts from the definition of the `irqaction` and the `irq_desc`:
--->
また、このパラメタは`/proc/interrupts`の出力で表示されます。
最後のパラメタは、メインの`uart_port`構造体へのポインタを渡します。
ここまでで、`request_irq`関数とそのパラメータについてについて少しわかりました。実装について見ていきましょう。
前に見たように、`request_irq`関数は内部で`request_threaded_irq`関数を呼び出す作りになっています。
`request_threaded_irq`関数は、[kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/manage.c)で定義されており、与えられた割込みラインを確保します。
この関数を見るなら、`irqaction`と`irq_desc`の定義から始めましょう。

```C
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
                         irq_handler_t thread_fn, unsigned long irqflags,
                         const char *devname, void *dev_id)
{
        struct irqaction *action;
        struct irq_desc *desc;
        int retval;
		...
		...
		...
}
```

<!---
We arelady saw the `irqaction` and the `irq_desc` structures in this chapter. The first structure represents per interrupt action descriptor and contains pointers to the interrupt handler, name of the device, interrupt number, etc. The second structure represents a descriptor of an interrupt and contains pointer to the `irqaction`, interrupt flags, etc. Note that the `request_threaded_irq` function called by the `request_irq` with the additioanal parameter: `irq_handler_t thread_fn`. If this parameter is not `NULL`, the `irq` thread will be created and the given `irq` handler will be executed in this thread. In the next step we need to make following checks:
--->
この章で、すでに`irqaction`と`irq_desc`構造体を見ました。
最初の構造体は、割込み動作デスクリプタ毎を表しており、割込みハンドラへのポインタ、デバイスの名前、割込み番号などを含んでいます。
２つ目の構造体は、割込みのデスクリプタを表しており、`irqaction`へのポインタ、割込みフラグなどを含んでいます。
`request_threaded_irq`関数は、追加のパラメータ（`irq_handler_t thread_fn`）とともに `request_irq`関数に呼ばれます。
このパラメータが `NULL` でなければ、`irq` スレッドが作られ、与えられた `irq` ハンドラはこのスレッドで実行されます。
つぎのステップでは、以下のチェックを行う必要があります：

```C
if (((irqflags & IRQF_SHARED) && !dev_id) ||
            (!(irqflags & IRQF_SHARED) && (irqflags & IRQF_COND_SUSPEND)) ||
            ((irqflags & IRQF_NO_SUSPEND) && (irqflags & IRQF_COND_SUSPEND)))
               return -EINVAL;
```

<!---
First of all we check that real `dev_id` is passed for the shared interrupt and the `IRQF_COND_SUSPEND` only makes sense for shared interrupts. Otherwise we exit from this function with the `-EINVAL` error. After this we convert the given `irq` number to the `irq` descriptor with the help of the `irq_to_desc` function that defined in the [kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/irqdesc.c) source code file and exit from this function with the `-EINVAL` error if it was not successful:
--->
まずはじめに、共有割込みならば、実在する`dev_id`が渡されているか、
共有割込みの時のみ`IRQF_COND_SUSPEND`がセットされているか、を確認します。
よろしくなければ、`-EINVAL`エラーでこの関数を終了します。
このあと、[kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/irqdesc.c)で定義された`irq_to_desc` 関数によって、与えられた`irq`番号を`irq`デスクリプタに変換します。
もし変換に失敗したら、この関数を`-EINVAL`で終了します。

```C
desc = irq_to_desc(irq);
if (!desc)
    return -EINVAL;
```

<!---
The `irq_to_desc` function checks that given `irq` number is less than maximum number of IRQs and returns the irq descriptor where the `irq` number is offset from the `irq_desc` array:
--->
`irq_to_desc`関数は、与えられた`irq`番号が、IRQsの最大番号より小さいかどうかをチェックし、`irq`番号が`irq_desc`配列からのオフセットとなる irqデスクリプタを返します。

```C
struct irq_desc *irq_to_desc(unsigned int irq)
{
        return (irq < NR_IRQS) ? irq_desc + irq : NULL;
}
```

<!---
As we have converted `irq` number to the `irq` descriptor we make the check the status of the descriptor that an interrupt can be requested:
--->
`irq`番号から、`irq`デスクリプタへの変換が完了したので、デスクリプタのステータスをチェックして、割込みを要求されることができるかを確認します。
<!--- [JA] もう少しわかりやすく --->

```C
if (!irq_settings_can_request(desc) || WARN_ON(irq_settings_is_per_cpu_devid(desc)))
    return -EINVAL;
```

<!---
and exit with the `-EINVAL` otherways. After this we check the given interrupt handler. If it was not passed to the `request_irq` function, we check the `thread_fn`. If both handlers are `NULL`, we return with the `-EINVAL`. If an interrupt handler was not passed to the `request_irq` function, but the `thread_fn` is not null, we set handler to the `irq_default_primary_handler`:
--->
だめなら`-EINVAL`で終了します。
このあと、与えられた割込みハンドラを確認します。
`request_irq`関数に渡されていないかどうかは、`thread_fn`で調べます。どちらのハンドラも `NULL` であれば、`-EINVAL`で返ります。
割込みハンドラが`request_irq`関数に渡されていないけれども、`thread_fn`が `NULL`ではないとき、ハンドラを`irq_default_primary_handler`にセットします。

```C
if (!handler) {
    if (!thread_fn)
        return -EINVAL;
	handler = irq_default_primary_handler;
}
```

<!---
In the next step we allocate memory for our `irqaction` with the `kzalloc` function and return from the function if this operation was not successful:
--->
つぎのステップは、`kzalloc`関数で`irqaction`のためのメモリを確保します。
`kzalloc`に失敗したら、`-ENOMEM`で、この関数から返ります。

```C
action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
if (!action)
    return -ENOMEM;
```

<!---
More about `kzalloc` will be in the separate chapter about [memory management](http://0xax.gitbooks.io/linux-insides/content/mm/index.html) in the Linux kernel. As we allocated space for the `irqaction`, we start to initialize this structure with the values of interrupt handler, interrupt flags, device name, etc:
--->
`kzalloc`については、Linuxカーネルの[memory management](http://0xax.gitbooks.io/linux-insides/content/mm/index.html)に関する別の章で触れます。
`irqaction`のために容量を確保したので、割込みハンドラ、割込みフラグやデバイス名などの値で、この構造体を初期化します：

```C
action->handler = handler;
action->thread_fn = thread_fn;
action->flags = irqflags;
action->name = devname;
action->dev_id = dev_id;
```

<!---
In the end of the `request_threaded_irq` function we call the `__setup_irq` function from the [kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/manage.c) and registers a given `irqaction`. Release memory for the `irqaction` and return:
--->
`request_threaded_irq` 関数の最後に、[kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/manage.c)から `__setup_irq` 関数を呼び、与えられた `irqaction` を登録します。
`__setup_irq` がエラーで戻ってきた場合は、`irqaction`のためのメモリを開放し、そのエラーで戻ります。
<!--- [JA] 開放するのはエラーの時だけよね。他の記述とバランスが取れないぬ。 --->

```C
chip_bus_lock(desc);
retval = __setup_irq(irq, desc, action);
chip_bus_sync_unlock(desc);

if (retval)
	kfree(action);

return retval;
```

<!---
Note that the call of the `__setup_irq` function is placed between the `chip_bus_lock` and the `chip_bus_sync_unlock` functions. These functions lock/unlock access to slow busses (like [i2c](https://en.wikipedia.org/wiki/I%C2%B2C)) chips. Now let's look at the implementation of the `__setup_irq` function. In the beginning of the `__setup_irq` function we can see a couple of different checks. First of all we check that the given interrupt descriptor is not `NULL`, `irqchip` is not `NULL` and that given interrupt descriptor module owner is not `NULL`. After this we check if the interrupt is nested into another interrupt thread or not, and if it is nested we replace the `irq_default_primary_handler` with the `irq_nested_primary_handler`.
--->
`etup_irq` 関数の予備d祭が、`chip_bus_lock`関数と `chip_bus_sync_unlock` 関数の間にあることに注意してください。
これらの関数は、低速バス（たとえば[i2c](https://en.wikipedia.org/wiki/I%C2%B2C)）へのアクセスをロック・アンロックします。
`__setup_irq` 関数の実装について見ていきましょう。
`__setup_irq` 関数の最初で、異なるチェックを見ることができます。
最初に与えられた割込みデスクリプタ（第一引数のdesc）が`NULL`ではないこと、
`irqchip`が`NULL`ではないこと（desc->irq_data.chipが&no_irq_chipではないこと）、
割込みデスクリプタモジュールのオーナが`NULL`ではないこと（try_module_get(desc->owner)が0を返すこと）。
その後、割込みが他の割込みスレッドにネストしているかどうかをチェックし、もしネストしていれば
`irq_default_primary_handler` を `irq_nested_primary_handler` で上書きします。

<!---
In the next step we create an irq handler thread with the `kthread_create` function, if the given interrupt is not nested and the `thread_fn` is not `NULL`:
--->
次のステップで、与えられた割込みがネストしていなくて、`thread_fn` が `NULL`ではないなら、`kthread_create` でIRQハンドラスレッドを作成します。

```C
if (new->thread_fn && !nested) {
	struct task_struct *t;
	t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
	...
}
```

<!---
And fill the rest of the given interrupt descriptor fields in the end. So, our `16` and `17` interrupt request lines are registered and the `serial21285_rx_chars` and `serial21285_tx_chars` functions will be invoked when an interrupt controller will get event releated to these interrupts. Now let's look at what happens when an interrupt occurs. 
--->

そして、最後に割込みデスクリプタフィールドの残りを埋めます。
`16`と`17`の割込み要求ラインが登録され、これらの割込みに関するイベントを割込みコントローラが取得した時に、`serial21285_rx_chars` 関数と `serial21285_tx_chars` 関数が起こされます。


Prepare to handle an interrupt
--------------------------------------------------------------------------------

<!--- [JA] x86依存 --->
<!---
In the previous paragraph we saw the requesting of the irq line for the given interrupt descriptor and registration of the `irqaction` structure for the given interrupt. We already know that when an interrupt event occurs, an interrupt controller notifies the processor about this event and processor tries to find appropriate interrupt gate for this interrupt. If you have read the eighth [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-8.html) of this chapter, you may remember the `native_init_IRQ` function. This function makes initialization of the local [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller). The following part of this function is the most interesting part for us right now: 
--->
前節では、与えられた割込みデスクリプタのIRQ線の要求の仕方と、与えられた割込みのための`irqaction`構造体の登録について見ました。
既に割込みイベントが起きた時に何が起きるのかを知っています。割込みコントローラはこのイベントについてプロセッサに通知し、
プロセッサはこの割り込みのために適切な割り込みゲートを見つけようとします。
既に本章の[パート８](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-8.html)を読んでいれば、`native_init_IRQ` 関数を思い出すでしょう。この関数はローカルの[APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)の初期化を行います。
この関数の後続の部分は、今の我々にとっては大変興味深いです：

```C
for_each_clear_bit_from(i, used_vectors, first_system_vector) {
	set_intr_gate(i, irq_entries_start +
		8 * (i - FIRST_EXTERNAL_VECTOR));
}
```

<!---
Here we iterate over all the cleared bit of the `used_vectors` bitmap starting at `first_system_vector` that is:
--->
`used_vectors` ビットマップを、`first_system_vector`から初めて全てのクリアされたビットを繰り返します。

```C
int first_system_vector = FIRST_SYSTEM_VECTOR; // 0xef
```

<!---
and set interrupt gates with the `i` vector number and the `irq_entries_start + 8 * (i - FIRST_EXTERNAL_VECTOR)` start address. Only one thing is unclear here - the `irq_entries_start`. This symbol defined in the [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry_entry_64.S) assembly file and provides `irq` entries. Let's look at it:
--->
割込みゲートの、`i`ベクタ番号 と `irq_entries_start + 8 * (i - FIRST_EXTERNAL_VECTOR)` スタートアドレスが示すビットをセットします。

```assembly
	.align 8
ENTRY(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
	pushq	$(~vector+0x80)
    vector=vector+1
	jmp	common_interrupt
	.align	8
    .endr
END(irq_entries_start)
```

<!---
Here we can see the [GNU assembler](https://en.wikipedia.org/wiki/GNU_Assembler) `.rept` instruction which repeats the sequence of lines that are before `.endr` - `FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR` times. As we already know, the `FIRST_SYSTEM_VECTOR` is `0xef`, and the `FIRST_EXTERNAL_VECTOR` is equal to `0x20`. So, it will work:
--->
[GNU assembler](https://en.wikipedia.org/wiki/GNU_Assembler) の `.rept` 命令は、`.endr`の手前の行までを、 `FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR` 回くりかえします。既にご承知のとおり、 `FIRST_SYSTEM_VECTOR` は `0xef` で、`FIRST_EXTERNAL_VECTOR` は `0x20` です。
したがって、207回繰り返しとなります。

```python
>>> 0xef - 0x20
207
```

<!---
times. In the body of the `.rept` instruction we push entry stubs on the stack (note that we use negative numbers for the interrupt vector numbers, because positive numbers already reserved to identify [system calls](https://en.wikipedia.org/wiki/System_call)), increase the `vector` variable and jump on the `common_interrupt` label. In the `common_interrupt` we adjust vector number on the stack and execute `interrupt` number with the `do_IRQ` parameter:
--->
`.rept` 命令の内側では、スタックにスタブエントリを積み上げ（割込みベクタ番号の数値は、[system calls](https://en.wikipedia.org/wiki/System_call)を識別するために正の値は既に使われているので、負の値を用いることに注意してください）、
`vector`変数を+1し、`common_interrupt` ラベルへジャンプします。
`common_interrupt` では、スタック上のベクタ番号を調整し、 `do_IRQ` のパラメータとして `interrupt` 番号使って実行します。


```assembly
common_interrupt:
	addq	$-0x80, (%rsp)
	interrupt do_IRQ
```

<!---
The macro `interrupt` defined in the same source code file and saves [general purpose](https://en.wikipedia.org/wiki/Processor_register) registers on the stack, change the userspace `gs` on the kernel with the `SWAPGS` assembler instruction if need, increase [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) - `irq_count` variable that shows that we are in interrupt and call the `do_IRQ` function. This function defined in the [arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irq.c) source code file and handles our device interrupt. Let's look at this function. The `do_IRQ` function takes one parameter - `pt_regs` structure that stores values of the userspace registers:
--->
同じソースコード内に定義された`interrupt`マクロは、
[general purposeレジスタ](https://en.wikipedia.org/wiki/Processor_register)をスタックに保存し、
必要なら ユーザ空間の`gs`を `SWAPGS` アセンブラ命令でカーネルのものに変えます。
そして、割込みに要ることを示すための、[per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) `irq_count`変数をインクリメントし、 `do_IRQ` 関数を呼びます。
この関数は、 arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irq.c) に定義されており、デバイス割込みを処理します。
この関数を見ていきましょう。
`do_IRQ` 関数は、１つのパラメータを取ります。ユーザ空間のレジスタ値を保存した `pt_regs`構造体です。


```C
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
    struct pt_regs *old_regs = set_irq_regs(regs);
    unsigned vector = ~regs->orig_ax;
    unsigned irq;

	irq_enter();
    exit_idle();
	...
	...
	...
}
```

<!---
At the beginning of this function we can see call of the `set_irq_regs` function that returns saved `per-cpu` irq register pointer and the calls of the `irq_enter` and `exit_idle` functions. The first function `irq_enter` enters to an interrupt context with the updating `__preempt_count` variable and the second function - `exit_idle` checks that current process is `idle` with [pid](https://en.wikipedia.org/wiki/Process_identifier) - `0` and notify the `idle_notifier` with the `IDLE_END`.
--->
この関数の最初に、`per-cpu` irqレジスタを保存したポインタを返す `set_irq_regs` 関数を呼んでいるのが見えます。
そして、 `irq_enter` 関数と `exit_idle` 関数とを呼んでいます。
ひとつ目の関数、 `irq_enter()` は、 `__preempt_count` 変数を更新して、割り込みコンテキストに入ります。
ふたつ目の関数、 `exit_idle()` は、カレントプロセスが `idle`であるか（[pid](https://en.wikipedia.org/wiki/Process_identifier)が `0`）を確認し、0でなければ、`IDLE_END` で `idle_notifier` に通知します。

<!---
In the next step we read the `irq` for the current cpu and call the `handle_irq` function:
--->
つぎのステップでは、現在のCPUから `irq`を読出し、 `handle_irq` 関数を呼び出します：

```C
irq = __this_cpu_read(vector_irq[vector]);

if (!handle_irq(irq, regs)) {
	...
	...
	...
}
...
...
...
```

<!---
The `handle_irq` function defined in the [arch/x86/kernel/irq_64.c](https://github.com/torvalds/linux/blob/arch/x86/kernel/irq_64.c) source code file, checks the given interrupt descriptor and call the `generic_handle_irq_desc`:
--->
[arch/x86/kernel/irq_64.c](https://github.com/torvalds/linux/blob/arch/x86/kernel/irq_64.c) に定義された
`handle_irq` 関数は、与えられた割り込みデスクリプタを確認し、 `generic_handle_irq_desc` 関数を呼びます。

```C
desc = irq_to_desc(irq);
	if (unlikely(!desc))
		return false;
generic_handle_irq_desc(irq, desc);
```

<!---
Where the `generic_handle_irq_desc` calls the interrupt handler:
--->
`generic_handle_irq_desc` は、割り込みハンドラを呼び出す場所は：

```C
static inline void generic_handle_irq_desc(unsigned int irq, struct irq_desc *desc)
{
       desc->handle_irq(irq, desc);
}
```

<!---
But stop... What is it `handle_irq` and why do we call our interrupt handler from the interrupt descriptor when we know that `irqaction` points to the actual interrupt handler? Actually the `irq_desc->handle_irq` is a high-level API for the calling interrupt handler routine. It setups during initialization of the [device tree](https://en.wikipedia.org/wiki/Device_tree) and [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) initialization. The kernel selects correct function and call chain of the `irq->action(s)` there. In this way, the `serial21285_tx_chars` or the `serial21285_rx_chars` function will be executed after an interrupt will occur.
--->
しかし待ってください。`handle_irq`とはなんでしょうか。
なぜ、`irqaction` が実際の割り込みハンドラを指していることがわかっている割り込みデスクリプタから、割り込みハンドラを呼び出すのでしょうか。
実際に、`irq_desc->handle_irq` は割り込みハンドラルーチンのためのハイレベルなAPIです。
[device tree](https://en.wikipedia.org/wiki/Device_tree) の初期化中と [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) の初期化中にセットアップされます。
カーネルは正しい関数を選択し、そこで `irq->action(s)` のチェインを呼びます。
この方法だと、割込みが起きた後に`serial21285_tx_chars` 関数または `serial21285_rx_chars` 関数が実行されるでしょう。

<!---
In the end of the `do_IRQ` function we call the `irq_exit` function that will exit from the interrupt context, the `set_irq_regs` with the old userspace registers and return:
--->
`do_IRQ` 関数の最後に、 `irq_exit` 関数を呼びます。この関数は、割り込みコンテキストから終了します。
その後、割り込んだ時のユーザ空間レジスタ を引数にして、`set_irq_regs` 関数を呼び、関数を終了します。

```C
irq_exit();
set_irq_regs(old_regs);
return 1;
```

<!---
We already know that when an `IRQ` finishes its work, deferred interrupts will be executed if they exist.
--->
`IRQ` がその仕事を完了した時に、遅延割り込みが待たされていれば、実行されることを既に知っています。


Exit from interrupt
--------------------------------------------------------------------------------

<!---
Ok, the interrupt handler finished its execution and now we must return from the interrupt. When the work of the `do_IRQ` function will be finsihed, we will return back to the assembler code in the [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry_entry_64.S) to the `ret_from_intr` label. First of all we disable interrupts with the `DISABLE_INTERRUPTS` macro that expands to the `cli` instruction and decreases value of the `irq_count` [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) variable. Remember, this variable had value - `1`, when we were in interrupt context:
--->
OK, 割り込みハンドラが実行を完了し、割り込みから戻らなければなりません。
`do_IRQP関数の作業が完了した時、[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry_entry_64.S) のアセンブラコードの中にある `ret_from_intr` ラベル へ戻ります。
まず最初に、 `cli` 命令に展開される `DISABLE_INTERRUPTS` マクロで割り込みを禁止し、
`irq_count` [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) 変数の値を減らします。
この変数は、割り込みコンテキストに居る場合に `1`の値を持つことを思い出してください。

```assembly
DISABLE_INTERRUPTS(CLBR_NONE)
TRACE_IRQS_OFF
decl	PER_CPU_VAR(irq_count)
```

<!---
In the last step we check the previous context (user or kernel), restore it in a correct way and exit from an interrupt with the:
--->
最後のステップでは、以前のコンテキストがユーザかカーネルかを確認し、正しい道へ戻します。
以下の `INTERRUPT_RETURN` マクロで割り込みから抜けます。

```assembly
INTERRUPT_RETURN
```

<!---
where the `INTERRUPT_RETURN` macro is:
--->
`INTERRUPT_RETURN` マクロは以下

```C
#define INTERRUPT_RETURN	jmp native_iret
```

<!---
and
--->
で、 `native_iret` は、以下のようになっています。

```assembly
ENTRY(native_iret)

.global native_irq_return_iret
native_irq_return_iret:
	iretq
```

<!---
That's all.
--->
以上で終わりです。


Conclusion
--------------------------------------------------------------------------------

<!---
It is the end of the tenth part of the [Interrupts and Interrupt Handling](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) chapter and as you have read in the beginning of this part - it is the last part of this chapter. This chapter started from the explanation of the theory of interrupts and we have learned what is it interrupt and kinds of interrupts, then we saw exceptions and handling of this kind of interrupts, deferred interrupts and finally we looked on the hardware interrupts and the handling of theirs in this part. Of course, this part and even this chapter does not cover full aspects of interrupts and interrupt handling in the Linux kernel. It is not realistic to do this. At least for me. It was the big part, I don't know how about you, but it was really big for me. This theme is much bigger than this chapter and I am not sure that somewhere there is a book that covers it. We have missed many part and aspects of interrupts and interrupt handling, but I think it will be good point to dive in the kernel code related to the interrupts and interrupts handling.
--->
[Interrupts and Interrupt Handling](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)の章の１０番目のパートの最後です。
このパートの最初から読んできたなら... ここはこの章の最後のパートです。
この章では、割り込みの理論の説明から始め、割り込みの種類と割り込みの種類を学習しました。
そして、例外と割り込みのこの種類の処理、遅延割り込みと見てきました。
最後に、このパートでハードウェア割り込みと、その処理について見ました。
もちろん、このパートとこの章でさえ、Linuxカーネルでの割り込みと割り込み処理の完全な側面については触れていません。
これを行うことは現実的ではありません。 少なくとも私にとっては。
それは大きな部分だった、あなたにとってどうか分からないが、渡しにとってはそれは本当に大きなものだった。
このテーマはこの章より遥かに大きく、どこかにこのテーマをカバーしている本があるのか知りません。
私たちは割り込みと割り込み処理の多くの部分と側面を見逃していましたが、割り込みと割り込み処理に関連するカーネルコードを掘り下げるのがよいと思います。

<!---
If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).
--->
質問や提案があれば、[twitter](https://twitter.com/0xAX) で、私にコメントかpingをください。

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Serial driver documentation](https://www.kernel.org/doc/Documentation/serial/driver)
* [StrongARM** SA-110/21285 Evaluation Board](http://netwinder.osuosl.org/pub/netwinder/docs/intel/datashts/27813501.pdf)
* [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [module](https://en.wikipedia.org/wiki/Loadable_kernel_module)
* [initcall](http://kernelnewbies.org/Documents/InitcallMechanism)
* [uart](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver/transmitter) 
* [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) 
* [memory management](http://0xax.gitbooks.io/linux-insides/content/mm/index.html)
* [i2c](https://en.wikipedia.org/wiki/I%C2%B2C)
* [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [GNU assembler](https://en.wikipedia.org/wiki/GNU_Assembler)
* [Processor register](https://en.wikipedia.org/wiki/Processor_register)
* [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [pid](https://en.wikipedia.org/wiki/Process_identifier)
* [device tree](https://en.wikipedia.org/wiki/Device_tree)
* [system calls](https://en.wikipedia.org/wiki/System_call)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-9.html)
