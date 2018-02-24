# Kernel Boot Process

This chapter describes the linux kernel boot process. Here you will see a
series of posts which describes the full cycle of the kernel loading process:

この章ではカーネルブーチプロセスについて記載する。
ここでカーネルローディングプロセスのフルサイクルについての一連の投稿を見れる。

* [From the bootloader to kernel](linux-bootstrap-1.md) - describes all stages from turning on the computer to running the first instruction of the kernel.

* [ブートローダーからカーネル](linux-bootstrap-1.md) - コンピューターの電源オンからカーネルの最初の命令が実行されるまでの全てのステージについての説明

* [First steps in the kernel setup code](linux-bootstrap-2.md) - describes first steps in the kernel setup code. You will see heap initialization, query of different parameters like EDD, IST and etc...

* [カーネルのセットアップコード内の最初のステップ](linux-bootstrap-2.md) - カーネルのセットアップコード内の最初のステップについての説明。ヒープ領域の初期化、EDD、ISTなどのような異なるパラメータの問い合わせ

* [Video mode initialization and transition to protected mode](linux-bootstrap-3.md) - describes video mode initialization in the kernel setup code and transition to protected mode.

* [ビデオモードの初期化とプロテクトモードへの移行](linux-bootstrap-3.md) - カーネルのセットアップコード内でのビデオモードの初期化とプロテクトモードへの移行についての説明。

* [Transition to 64-bit mode](linux-bootstrap-4.md) - describes preparation for transition into 64-bit mode and details of transition.

* [64ビットモードへの移行](linux-bootstrap-4.md) - 64ビットモードへの移行の準備と移行の詳細についての説明

* [Kernel Decompression](linux-bootstrap-5.md) - describes preparation before kernel decompression and details of direct decompression.

* [圧縮されたカーネルの展開](linux-bootstrap-5.md) - 圧縮されたカーネルの展開の準備とダイレクト展開の詳細についての説明

* [Kernel random address randomization](linux-bootstrap-6.md) - describes randomization of the Linux kernel load address.

* [カーネルのアドレスのランダム化](linux-bootstrap-6.md) - Linuxカーネルの読み込みアドレスのランダム化についての説明
