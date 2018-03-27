Kernel booting process. Part 1.
================================================================================

From the bootloader to the kernel
--------------------------------------------------------------------------------

If you have been reading my previous [blog posts](https://0xax.github.io/categories/assembler/), then you can see that, for some time now, I have been starting to get involved with low-level programming. I have written some posts about assembly programming for `x86_64` Linux and, at the same time, I have also started to dive into the Linux kernel source code.

もしあなたが私の過去の[ブログの投稿](https://0xax.github.io/categories/assembler/)を読んだのなら、これまでの間、私はローレベルプログラミングに関与してしまっていることをあなたは理解することが出来る。私はこれまでの間`x86_64` Linuxのアセンブラプログラムについての幾つかの投稿について書いてきているし、Linuxxカーネルソースコードに没頭し始めてもいる。

I have a great interest in understanding how low-level things work, how programs run on my computer, how they are located in memory, how the kernel manages processes and memory, how the network stack works at a low level, and many many other things. So, I have decided to write yet another series of posts about the Linux kernel for the **x86_64** architecture.

私はローレベルの動作、私のコンピューター上でのプログラムの動作、それらのメモリ配置、カーネルのプロセスとメモリの管理、ローレベルでのネットワークスタックの動作、など色々な手法について大変興味を持っている。
そして、私は**x86_64**アーキテクチャに関するLinuxカーネルに関する他の一連の投稿を今のところ書くことを決心している。

Note that I'm not a professional kernel hacker and I don't write code for the kernel at work. It's just a hobby. I just like low-level stuff, and it is interesting for me to see how these things work. So if you notice anything confusing, or if you have any questions/remarks, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com) or just create an [issue](https://github.com/0xAX/linux-insides/issues/new). I appreciate it.

注意事項。私はプロのカーネルハッカーではないし仕事としてカーネルのコードを書いてはいない。これは趣味だ。私はローレベル関することが好きだ。そしてそれらがどのように動作するのか理解する事に興味を持っている。そしてもしあなたが何らか混乱していることに気づいたか、質問や意見があるならTwitter [0xAX](https://twitter.com/0xAX)にツイート、[メール](anotherworldofworld@gmail.com)にてご一報、またはGitHubの[issue](https://github.com/0xAX/linux-insides/issues/new)を作成してください。ありがたく思います。

All posts will also be accessible at [github repo](https://github.com/0xAX/linux-insides) and, if you find something wrong with my English or the post content, feel free to send a pull request.

全ての投稿は[github repo](https://github.com/0xAX/linux-insides)もで閲覧することが出来、もし私の英語や投稿した内容についての間違いを見つけたのならお気軽にプルリクエストを送ってください。

*Note that this isn't official documentation, just learning and sharing knowledge.*

*注意事項。これは公式のドキュメントではなく、知識の学習や共有である。

**Required knowledge**

**要求される知識**

* Understanding C code

* C言語に関する理解

* Understanding assembly code (AT&T syntax)

* アセンブラコードに関する理解 (AT&Tシンタックス)

Anyway, if you are just starting to learn such tools, I will try to explain some parts during this and the following posts. Alright, this is the end of the simple introduction, and now we can start to dive into the Linux kernel and low-level stuff.

とにかく、もしあなたが手段として学習し始めたのなら、これや次の投稿のいくつかの章で説明します。大丈夫、これは簡単な導入部の終わりです。そして今私たちはLinuxカーネルやローレベルに関することに没頭することを始めることが出来ます。

I've started to write this book at the time of the `3.18` Linux kernel, and many things might change from that time. If there are changes, I will update the posts accordingly.

私は`3.18` Linuxカーネル時にこの本を書き始めています。そしてその時からたくさんのことが変更になったかもしれません。もし変更があるのならそれに応じて投稿を更新します。

The Magical Power Button, What happens next?
--------------------------------------------------------------------------------

Although this is a series of posts about the Linux kernel, we will not be starting directly from the kernel code - at least not, in this paragraph. As soon as you press the magical power button on your laptop or desktop computer, it starts working. The motherboard sends a signal to the [power supply](https://en.wikipedia.org/wiki/Power_supply) device. After receiving the signal, the power supply provides the proper amount of electricity to the computer. Once the motherboard receives the [power good signal](https://en.wikipedia.org/wiki/Power_good_signal), it tries to start the CPU. The CPU resets all leftover data in its registers and sets up predefined values for each of them.

これがLinuxカーネルに関する一連の投稿であるにも関わらず、少なくともこの節では我々はカーネルコードから直接開始することは無いでしょう。あなたのノートPCやデスクトップPCの不思議な電源スイッチを押すやいなや、コンピューターは動作し始める。マザーボードは[電源](https://en.wikipedia.org/wiki/Power_supply)デバイスに信号を送る。信号を受信した後、電源は適切な量の電力を供給する。一度マザーボードが[パワーグッド信号](https://en.wikipedia.org/wiki/Power_good_signal)を受け取ると、CPUを動作させるよう試みる。CPUは全てのレジスタに残っているデータをリセットし、リセットによりそれら値は初期設定値になる。


The [80386](https://en.wikipedia.org/wiki/Intel_80386) CPU and later define the following predefined data in CPU registers after the computer resets:

[80386](https://en.wikipedia.org/wiki/Intel_80386) CPUやそれ以降の製品はコンピューターがリセットされた後、CPUのレジスタは次のプリデファインされたデータになる。

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

The processor starts working in [real mode](https://en.wikipedia.org/wiki/Real_mode). Let's back up a little and try to understand [memory segmentation](https://en.wikipedia.org/wiki/Memory_segmentation) in this mode. Real mode is supported on all x86-compatible processors, from the [8086](https://en.wikipedia.org/wiki/Intel_8086) CPU all the way to the modern Intel 64-bit CPUs. The `8086` processor has a 20-bit address bus, which means that it could work with a `0-0xFFFFF` or `1 megabyte` address space. But it only has `16-bit` registers, which have a maximum address of `2^16 - 1` or `0xffff` (64 kilobytes).
プロセッサーは[リアルモード](https://en.wikipedia.org/wiki/Real_mode)で動作し始めます。少し話を戻させてください。そしてこのモードでの[メモリセグメンテーション](https://en.wikipedia.org/wiki/Memory_segmentation) について理解していきましょう。リアルモードは[8086](https://en.wikipedia.org/wiki/Intel_8086) CPUから現代のIntel 64-bit CPUまでの全てのx86互換プロセッサーでサポートされています。`8086` プロセッサーは
`0-0xFFFFF` または `1メガバイト` のアドレス空間と連携して動作することが出来る事を意味する20bitのアドレスバスを持っています。しかし`16-bit`のレジスタしか持っていないため、最大アドレスは`2^16 - 1` または `0xffff` (64キロバイト)になります。

[Memory segmentation](http://en.wikipedia.org/wiki/Memory_segmentation) is used to make use of all the address space available. All memory is divided into small, fixed-size segments of `65536` bytes (64 KB). Since we cannot address memory above `64 KB` with 16-bit registers, an alternate method was devised.

[メモリセグメンテーション](http://en.wikipedia.org/wiki/Memory_segmentation) はすべての有効なアドレス空間を使うために使われる。全てのメモリは小さな、`65536` バイト (64 KB)の固定サイズのセグメントに分割されている。我々は16-bitのレジスタで`64 KB`を超えたメモリをアドレッシングすることが出来ないので、代替手法として考え出された。

An address consists of two parts: a segment selector, which has a base address, and an offset from this base address. In real mode, the associated base address of a segment selector is `Segment Selector * 16`. Thus, to get a physical address in memory, we need to multiply the segment selector part by `16` and add the offset to it:

アドレスは、ベースアドレスを持つセグメントセレクターとこのベースアドレスからのオフセット、この2つから構成される。リアルモードでは、セグメントセレクターに関連したベースアドレスは`Segment Selector * 16`となる。このように、メモリの物理アドレスを得るために、我々はセグメントセレクタに`16`を掛け合わせ、それに対してオフセットを加算する必要がある。

```
PhysicalAddress = Segment Selector * 16 + Offset
```

For example, if `CS:IP` is `0x2000:0x0010`, then the corresponding physical address will be:

例えば、もし`CS:IP` が `0x2000:0x0010` なら、該当する物理アドレスは次のようにな。

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

But, if we take the largest segment selector and offset, `0xffff:0xffff`, then the resulting address will be:

しかし、もし最も大きいセグメントれセクターとオフセット、`0xffff:0xffff`であった場合アドレスは次のようになる。

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

which is `65520` bytes past the first megabyte. Since only one megabyte is accessible in real mode, `0x10ffef` becomes `0x00ffef` with the [A20 line](https://en.wikipedia.org/wiki/A20_line) disabled.

`65520`バイトは先頭のアドレス部分に現れる。リアルモードでは1メガバイトしかアクセス出来ないため、`0x10ffef`は[A20 line](https://en.wikipedia.org/wiki/A20_line)がディセーブルであることから`0x00ffef`になる。


Ok, now we know a little bit about real mode and memory addressing in this mode. Let's get back to discussing register values after reset.

OK。今このモードでのリアルモードとメモリアドレッシングについて若干知っている。さあリセット後のレジスタ値についての議論に戻ろう。

The `CS` register consists of two parts: the visible segment selector, and the hidden base address. While the base address is normally formed by multiplying the segment selector value by 16, during a hardware reset the segment selector in the CS register is loaded with `0xf000` and the base address is loaded with `0xffff0000`; the processor uses this special base address until `CS` is changed.

`CS`レジスタはビジブルセグメントセレクタと上位のベースアドレスの2つの要素から構成される。ベースアドレスが通常セグメントセレクタの16倍としていると同時、ハードウェアリセット中にCSレジスタのセグメントセレクタは`0xf000`、ベースアドレスは`0xffff0000`となる。プロセッサーは`CS`が変更されるまで特別なベースアドレスとして使用する。

The starting address is formed by adding the base address to the value in the EIP register:

開始アドレスはベースアドレスにEIPレジスタの値を加算したもので形成される。

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

We get `0xfffffff0`, which is 16 bytes below 4GB. This point is called the [Reset vector](http://en.wikipedia.org/wiki/Reset_vector). This is the memory location at which the CPU expects to find the first instruction to execute after reset. It contains a [jump](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) instruction that usually points to the BIOS entry point. For example, if we look in the [coreboot](http://www.coreboot.org/) source code (`src/cpu/x86/16bit/reset16.inc`), we will see:

我々は4GBの手前の16バイトである`0xfffffff0`を得る。このポイントは[リセットベクター](http://en.wikipedia.org/wiki/Reset_vector)と呼ばれる。これはCPUがリセット後実行する最初の命令を見つけることを期待したメモリ配置である。通常BIOSのエントリーポイントを示した[ジャンプ](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`)命令で構成される。例えば,[coreboot](http://www.coreboot.org/) のソースコード (`src/cpu/x86/16bit/reset16.inc`)を見ると以下のようになっている。

```assembly
    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...
```

Here we can see the `jmp` instruction [opcode](http://ref.x86asm.net/coder32.html#xE9), which is `0xe9`, and its destination address at `_start - ( . + 2)`.

ここで`0xe9` とジャンプ先アドレスから2を引いた値 `_start - ( . + 2)` である`jmp` 命令 [opcode](http://ref.x86asm.net/coder32.html#xE9)を見ることが出来る。

We can also see that the `reset` section is `16` bytes and that is compiled to start from `0xfffffff0` address (`src/cpu/x86/16bit/reset16.lds`):

我々は`reset` セクションが `16` バイトでかつ`0xfffffff0` アドレス (`src/cpu/x86/16bit/reset16.lds`)から開始されるようにコンパイルしてあることを確認できる。

```
SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}
```

Now the BIOS starts; after initializing and checking the hardware, the BIOS needs to find a bootable device. A boot order is stored in the BIOS configuration, controlling which devices the BIOS attempts to boot from. When attempting to boot from a hard drive, the BIOS tries to find a boot sector. On hard drives partitioned with an [MBR partition layout](https://en.wikipedia.org/wiki/Master_boot_record), the boot sector is stored in the first `446` bytes of the first sector, where each sector is `512` bytes. The final two bytes of the first sector are `0x55` and `0xaa`, which designates to the BIOS that this device is bootable.

今 BIOSが開始されます。初期化とハードウェアの確認の後で、BIOSはブート可能なデバイスを発見する必要がある。ブート順はBIOSのコンフィグレーション領域に格納され、BIOSがそこに記載されているデバイスから起動を試みるように制御される。ハードディスクから起動を試みる時、BIOSはブートセクターを探す。[MBRパーティション](https://en.wikipedia.org/wiki/Master_boot_record)で分割されたハードディスク上に、ブートセクターはセクタ単位が`512`バイトである最初のセクターの先頭`446`バイトに記録されている。この最初のセクターの最後の2バイトにBIOSがブート可能デバイスであることをを示すために`0x55` と `0xaa`が記録されている。

For example:

例:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Build and run this with:

以下のコマンドでビルドと実行

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

This will instruct [QEMU](http://qemu.org) to use the `boot` binary that we just built as a disk image. Since the binary generated by the assembly code above fulfills the requirements of the boot sector (the origin is set to `0x7c00` and we end with the magic sequence), QEMU will treat the binary as the master boot record (MBR) of a disk image.

これは[QEMU]が今しがたディスクイメージとしてビルドした`boot`バイナリーを使うことを指示する。バイナリーはブートセクター(`0x7c00`)の要求を満たした上、アセンブラコードから生成されたものなので、QEMUはバイナリーをディスクイメージのマスターブートレコード(MBR)として扱われる。

You will see:

次のようになる。

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

In this example, we can see that the code will be executed in `16-bit` real mode and will start at `0x7c00` in memory. After starting, it calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt, which just prints the `!` symbol; it fills the remaining `510` bytes with zeros and finishes with the two magic bytes `0xaa` and `0x55`.

この例では、コードが`16-bit`リアルモードで実行され、メモリのアドレス`0x7c00`から開始されるのを見ることが出来る。開始後、`!`シンボルを表示する[0x10](http://www.ctyme.com/intr/rb-0106.htm)割り込みが呼ばれる。残りの`510`バイトはゼロ埋めされており2バイトのマジックコードである`0xaa`と`0x55`で終わる。

You can see a binary dump of this using the `objdump` utility:

`objdump`を用いることでこのバイナリーダンプを見ることが出来る。

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

A real-world boot sector has code for continuing the boot process and a partition table instead of a bunch of 0's and an exclamation mark :) From this point onwards, the BIOS hands over control to the bootloader.

現実世界のブートセクターはブートプロセスを続けるためのコードで0の束とエクスクラメーションマークの代わりのパーティションテーブルである。 :)この前方から、BIOSはブートローダーに引き渡す。


**NOTE**: As explained above, the CPU is in real mode; in real mode, calculating the physical address in memory is done as follows:

**注意** 上記の説明ではCPUはリアルモードである。リアルモードでは、メモリにおける物理アドレスの計算は以下のようになる:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

just as explained above. We have only 16-bit general purpose registers; the maximum value of a 16-bit register is `0xffff`, so if we take the largest values, the result will be:

ちょうど上記のように説明される。16bitの汎用レジスタのみある。: 16bitレジスタの最大値は`0xffff`であり、そこでもし最大値であったなら結果は以下のようになる。

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

where `0x10ffef` is equal to `1MB + 64KB - 16b`. An [8086](https://en.wikipedia.org/wiki/Intel_8086) processor (which was the first processor with real mode), in contrast, has a 20-bit address line. Since `2^20 = 1048576` is 1MB, this means that the actual available memory is 1MB.

`0x10ffef`は`1MB + 64KB - 16b`と等価である。リアルモードを持った最初のプロセッサである[8086](https://en.wikipedia.org/wiki/Intel_8086)プロセッサは対象的に20bitのアドレスラインがある。`2^20 = 1048576`は1MBなので、これは実際の有効メモリが1MBであることを意味している。

In general, real mode's memory map is as follows:

一般に、リアルモードのメモリマップは次のようになる。:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

In the beginning of this post, I wrote that the first instruction executed by the CPU is located at address `0xFFFFFFF0`, which is much larger than `0xFFFFF` (1MB). How can the CPU access this address in real mode? The answer is in the [coreboot](http://www.coreboot.org/Developer_Manual/Memory_map) documentation:

この投稿のはじめで、私はCPUによる最初の命令の実行は`0xFFFFF` (1MB)よりはるかに大きい`0xFFFFFFF0`アドレスに配置されると記載した。CPUはリアルモードでどのようにこのアドレスにアクセスするのか？答えは[coreboot](http://www.coreboot.org/Developer_Manual/Memory_map)のドキュメントにある:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

At the start of execution, the BIOS is not in RAM, but in ROM.

実行開始ではBIOSはRAMではなくROMである。

Bootloader
--------------------------------------------------------------------------------

There are a number of bootloaders that can boot Linux, such as [GRUB 2](https://www.gnu.org/software/grub/) and [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). The Linux kernel has a [Boot protocol](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt) which specifies the requirements for a bootloader to implement Linux support. This example will describe GRUB 2.

[GRUB 2](https://www.gnu.org/software/grub/)や[syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project)のようなLinuxをブートすることが出来るブートローダーがいくつかある。

Linux kernelにはLinuxをサポートするように実装されたbootloaderを要求するように明示された[Boot protocol](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt)が存在する。この例としてGRUB2について記載しよう。

Continuing from before, now that the `BIOS` has chosen a boot device and transferred control to the boot sector code, execution starts from [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). This code is very simple, due to the limited amount of space available, and contains a pointer which is used to jump to the location of GRUB 2's core image. The core image begins with [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), which is usually stored immediately after the first sector in the unused space before the first partition. The above code loads the rest of the core image, which contains GRUB 2's kernel and drivers for handling filesystems, into memory. After loading the rest of the core image, it executes the [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) function.

以前から継続して、今`BIOS`ブートデバイスを選択し、ブードセクタコードの制御を転送し、[boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD)から実行する。このコードは限定された有効な空間のためとても単純で、GRUB 2のコードイメージのアドレスにジャンプするためのポインターを含んでいる。コードイメージは通常第一セクタの後ろの第一パーティションの前にある未使用空間に接して書き込まれた[diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD)で始まります。上記コードがGRUB2カーネルやファイルシステムを取り扱うためのドライバーを含んだコアイメージの残りとしてメモリーに読み込まれます。残りのコアイメージをロードした後で、[grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c)機能が実行されます。

The `grub_main` function initializes the console, gets the base address for modules, sets the root device, loads/parses the grub configuration file, loads modules, etc. At the end of execution, the `grub_main` function moves grub to normal mode. The `grub_normal_execute` function (from the `grub-core/normal/main.c` source code file) completes the final preparations and shows a menu to select an operating system. When we select one of the grub menu entries, the `grub_menu_execute_entry` function runs, executing the grub `boot` command and booting the selected operating system.

`grub_main`機能はコンソールを初期化し、モジュールのベースアドレスを取得し、ルートデバイスをセットし、grubコンフィグレーションファイルを読み込み・解析し、モジュールを読み込むなどを行います。これらの実行後、`grub_main`機能はgrubのノーマルモードに移行します。`grub_normal_execute`機能は(ソースコードファイル`grub-core/normal/main.c`から)準備をを完了しオペレーティングシステムを選択するメニューを表示します。grubメニューのエントリーの一つを選択した時、`grub_menu_execute_entry`機能が実行され、grubの`boot`コマンドが実行され選択されたオペレーティングシステムが実行されます。

As we can read in the kernel boot protocol, the bootloader must read and fill some fields of the kernel setup header, which starts at the `0x01f1` offset from the kernel setup code. You may look at the boot [linker script](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L16) to confirm the value of this offset. The kernel header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S) starts from:

カーネルブートプロトコルを読み込ませることが出来るので、カーネルセットアップコードからfsetの`0x01f1`で始まるブートローダーは読み込み、カーネルセットアップヘッダーのいくつかのフィールドを埋めなければなりません。このオフセット値を確認するためにブート[リンカースクリプト](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L16)を見るかもしれません。カーネルヘッダー[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S)は次のように開始されます。

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

The bootloader must fill this and the rest of the headers (which are only marked as being type `write` in the Linux boot protocol, such as in [this example](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L354)) with values which it has either received from the command line or calculated during boot. (We will not go over full descriptions and explanations for all fields of the kernel setup header now, but we shall do so when we discuss how the kernel uses them; you can find a description of all fields in the [boot protocol](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L156).)

ブートローダーはこれとヘッダーの残りを ([この例](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L354)のような、Linuxブートプロトコルの`write`タイプで始まるものとしてマークされたもののみ)コマンドラインから受け取った値またはブート中に計算された値のどちらかで埋めます。(我々は詳細説明を超えないし今カーネルセットアップフェっダーの全てのフィールドについて説明しないでしょう。しかし我々はどのようにカーネルがそれらをどう使うかについて議論した時にすべきです。あなたは[ブートプロトコル](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L156)上で全てのフィールドの説明を見つけることが出来ます。

As we can see in the kernel boot protocol, the memory will be mapped as follows after loading the kernel:

我々はカーネルブートプロトコルで見る事ができるように、メモリはカーネルがロードされた後次のようにマッピングされます。

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

So, when the bootloader transfers control to the kernel, it starts at:

そして、ブートローダーがカーネルをコントロールすることを移行し、開始します。

```
X + sizeof(KernelBootSector) + 1
```

where `X` is the address of the kernel boot sector being loaded. In my case, `X` is `0x10000`, as we can see in a memory dump:

`X`はカーネルブートセクターがロードされるアドレス場所です。私の場合では、`X`は`0x10000`で、メモリダンプ上で見ることが出来ます。

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

The bootloader has now loaded the Linux kernel into memory, filled the header fields, and then jumped to the corresponding memory address. We can now move directly to the kernel setup code.

ブートローダーは今メモリにLinuxカーネルをロードし、ヘッダーフィールドを埋め、そして該当のメモリアドレスにジャンプします。我々は今カーネルセットアップコードに直接移動することが出来ます。

The Beginning of the Kernel Setup Stage
カーネルセットアップステージのはじめ
--------------------------------------------------------------------------------

Finally, we are in the kernel! Technically, the kernel hasn't run yet; first, the kernel setup part must configure stuff such as the decompressor and some memory management related things, to name a few. After all these things are done, the kernel setup part will decompress the actual kernel and jump to it. Execution of the setup part starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S) at [_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292). It is a little strange at first sight, as there are several instructions before it.

最後に、我々はカーネルの中にいます。技術的に、カーネルは未だ実行されていません。最初、カーネルセットアップパートが展開と名前が少し関連するいくつかのメモリ管理としてものを構成しなければなりません。これらのことが全て終わった後、カーネルセットアップパートは実際のカーネルを展開し、そこにジャンプします。セットアップパートの実行は[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S)の[_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292)から始まります。それより前にいくつかの命令があるので初見では少し奇妙です。

A long time ago, the Linux kernel used to have its own bootloader. Now, however, if you run, for example,

はるか昔、Linuxカーネルはブートローダー自身として使われていました。今、しかしながら、あなたが実行するなら、このような例になります。

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

then you will see:

そしてあなたは下記を見るでしょう。

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

Actually, the file `header.S` starts with the magic number [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (see image above), the error message that displays and, following that, the [PE](https://en.wikipedia.org/wiki/Portable_Executable) header:

実際、`header.S`ファイルはマジックナンバー[MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable)(イメージを参照してください)で始まり、エラーメッセージが表示され、[PE](https://en.wikipedia.org/wiki/Portable_Executable)ヘッダーは以下のことになります。

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

It needs this to load an operating system with [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) support. We won't be looking into its inner workings right now and will cover it in upcoming chapters.

これは[UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)をサポートしているオペレーティング・システムをロードする必要があります。我々は直ぐにその内部の動作の中を見なく、きたる章でカバーするでしょう。

The actual kernel setup entry point is:

実際のカーネルセットアップエントリーポイントは以下です。

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (grub2 and others) knows about this point (at an offset of `0x200` from `MZ`) and makes a jump directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:

ブートローダー (grub2など) はこのポイント(`MZ`から`0x200`のオフセット)を知っており、エラーメッセージを表示する`header.S` は`.bstext`セクションから始まるにもかかわらず実際そこに直接ジャンプするようにします。

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

The kernel setup entry point is:

カーネルセットアップエントリーポイントは以下です。

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

Here we can see a `jmp` instruction opcode (`0xeb`) that jumps to the `start_of_setup-1f` point. In `Nf` notation, `2f`, for example, refers to the local label `2:`; in our case, it is the label `1` that is present right after the jump, and it contains the rest of the setup [header](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L156). Right after the setup header, we see the `.entrytext` section, which starts at the `start_of_setup` label.

我々は`start_of_setup-1f`ポイントにジャンプする`jmp`命令オペコード(`0xeb`)を見ることが出来ます。`Nf`表記法で、例えば`2f`はローカルラベル`2:`を参照します。; 我々の件では、ジャンプの直後に存在するラベル`1`で、セットアップ[ヘッダー](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L156)の残りを含んでいます。セットアップヘッダー直後我々は`start_of_setup` ラベルで始まる`.entrytext` セクションを見ます。

This is the first code that actually runs (aside from the previous jump instructions, of course). After the kernel setup part receives control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the kernel real mode, i.e., after the first 512 bytes. This can be seen in both the Linux kernel boot protocol and the grub2 source code:

これは実際実行される最初のコードです。(もちろん、前のジャンプ命令は別として)カーネルセットアップパートはブートローダーからコントロールを受信した後、最初の`jmp`命令はカーネルリアルモードの最初から`0x200`オフセットの位置にあり、例えば、最初の512バイトの後ろ。これはLinuxカーネルブートプロトコルとgrub2ソースコードの両方で見ることができます。


```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

This means that segment registers will have the following values after kernel setup starts:

これはセグメントレジスタがカーネルセットアップが開始された後の次の値を持つことを意味します。

```
gs = fs = es = ds = ss = 0x10000
cs = 0x10200
```

In my case, the kernel is loaded at `0x10000` address.

私の場合、カーネルは`0x10000`アドレスに配置されます。

After the jump to `start_of_setup`, the kernel needs to do the following:

`start_of_setup`にジャンプした後、カーネルは次のことを行う必要があります。

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c)

* セグメントレジスター値が等しいことを確認する
* もし必要なら、正しいスタックを設定する
* [bss](https://en.wikipedia.org/wiki/.bss)を設定する
* [main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c)のCコードにジャンプする

Let's look at the implementation.

さあ実装を見てみましょう。

Aligning the Segment Registers 
セグメントレジスタの位置合わせ
--------------------------------------------------------------------------------

First of all, the kernel ensures that the `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

まずはじめに、カーネルは`ds`と`es`セグメントレジスタは同じアドレスを示す様にします。次に、`cld`命令を使ってディレクションフラグをクリアします。

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, `grub2` loads kernel setup code at address `0x10000` by default and `cs` at `0x10200` because execution doesn't start from the start of file, but from the jump here:

前に書いたように、`grub2`はデフォルトとしてアドレス`0x10000`、`cs`を`0x10200`としてカーネルセットアップコードを読み込みます。なぜなら実行はファイルの最初から開始せず、この[4d 5a](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L46)から`512`バイトのオフセットのジャンプ命令から始まります。

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

which is at a `512` byte offset from [4d 5a](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L46). We also need to align `cs` from `0x10200` to `0x10000`, as well as all other segment registers. After that, we set up the stack:

我々は更に他のセグメントレジスタと同様に`0x10200` から `0x10000`に`cs`をアラインする必要があります。
その後、我々は[6](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L494)ラベルのアドレスに従い`ds`の値をスタックに積むスタックを設定し、`lretw`命令を実行します。

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack, followed by the address of the [6](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L494) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterward, `ds` and `cs` will have the same values.

`lretw`命令が呼ばれた時、ラベル`6`のアドレスが[インストラクションポインタ](https://en.wikipedia.org/wiki/Program_counter)レジスタにロードされ、`ds`の値と一緒に`cs`にロードされる。その後、`ds`と`cs`は同じ値になるでしょう。

Stack Setup
スタックの設定
--------------------------------------------------------------------------------

Almost all of the setup code is in preparation for the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L569) is checking the `ss` register value and making a correct stack if `ss` is wrong:

セットアップコードのほとんど全てはリアルモードでC言語環境向けに準備されている。次の[ステップ](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L569)では`ss`レジスタの値がチェックされ、もし`ss`が間違っていたら正しいスタックに変更される。

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

これは3つの異なるシナリオを導くことが出来ます。

* `ss` has a valid value `0x1000` (as do all the other segment registers beside `cs`)
* `ss` is invalid and the `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and the `CAN_USE_HEAP` flag is not set (see below)

* `ss`は有効な値`0x1000`です。(`cs`と他の全てのセグメントレジスタと比べて)
* `ss`は無効で`CAN_USE_HEAP`フラグがセットされている (下記参照)
* `ss`は無効で`CAN_USE_HEAP`フラグがセットされていない (下記参照)

Let's look at all three of these scenarios in turn:

さあ順番にこれら3つのシナリオを見ていきましょう。

* `ss` has a correct address (`0x1000`). In this case, we go to label [2](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L584):

* `ss`は正しいアドレス(`0x1000`)です。この場合は、我々はラベル[2](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L584)に行きます。

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we set the alignment of `dx` (which contains the value of `sp` as given by the bootloader) to `4` bytes and a check for whether or not it is zero. If it is zero, we put `0xfffc` (4 byte aligned address before the maximum segment size of 64 KB) in `dx`. If it is not zero, we continue to use the value of `sp` given by the bootloader (0xf7f4 in my case). After this, we put the value of `ax` into `ss`, which stores the correct segment address of `0x1000` and sets up a correct `sp`. We now have a correct stack:

ここで我々は(ブートローダーで与えられた`sp`の値を含んでいる)`dx`のアライメントを`4`バイトとして設定し、それがゼロかどうかチェックします。もしそれがゼロなら、我々は`dx`を`0xfffc`(64KBの最大セグメントサイズの前の4バイトアラインされたアドレス)とします。もしそれがゼロでなかったのなら、ブートローダーが与える`sp`の値(私の場合は0xf7f4)を使い続けます。この後、我々は`0x1000`の正しいセグメントアドレスがストアされた`ss`の中に`ax`の値を設定し、正しい`sp`を設定します。我々は今正しいスタックになります。

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* In the second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L52) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L321) is a bitmask header which is defined as:

* 第2のシナリオは(`ss` != `ds`)です。最初、我々は(セットアップコードの終わりのアドレスである)[_end](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L52)の値を`dx`に設定し、我々がヒープを使うことが出来るかどうか確認する`testb`命令を使い`loadflags`ヘッダーフィールドを確認します。[loadflags](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L321)は次のように定義されたビットマスクヘッダーです。

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and, as we can read in the boot protocol:

そして、ブートプロコトル内で読むことが出来ます。

```
Field name: loadflags
フィールド名: ロードフラグ

  This field is a bitmask.

  このフィールドはビットマスクです。

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.

  Bit 7 (write): CAN_USE_HEAP
    heap_end_ptrが有効となったことを示す値としてこのビットを1として設定します。

```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (minimum stack size, `1024` bytes) to it. After this, if `dx` is not carried (it will not be carried, `dx = _end + 1024`), jump to label `2` (as in the previous case) and make a correct stack.

もし`CAN_USE_HEAP`ビットが設定されたなら、我々は`heap_end_ptr`を(`_end`を指し示す)`dx`に入れ、(最小スタックサイズ`1024`バイト)`STACK_SIZE`をそれに加算します。もし`dx`が繰り上がらなかったなら(`dx = _end + 1024`は繰り上がらないでしょう)ラベル`2`(前回の場合として)にジャンプし正しいスタックにします。

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

* `CAN_USE_HEAP`が設定されていない時、我々は`_end`から`_end + STACK_SIZE`までを最小スタックとして使うだけです。

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
BSS設定
--------------------------------------------------------------------------------

The last two steps that need to happen before we can jump to the main C code are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking the "magic" signature. First, signature checking:

最後の2つのステップでは我々がCコードのmainにジャンプする前に[BSS](https://en.wikipedia.org/wiki/.bss)エリアとして設定し、"マジック"署名をチェックする必要があります。最初の署名をチェックします。

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the [setup_sig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L39) with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is reported.

これはマジックナンバー`0x5a5aaa55`と[setup_sig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L39)の単純な比較です。もしこれらが等価でないなら致命的なエラーとして報告される。

If the magic number matches, knowing we have a set of correct segment registers and a stack, we only need to set up the BSS section before jumping into the C code.

もしマジックナンバーが一致したら、我々は正しいセグメントレジスタとスタックが設定されたことを知り、我々はCコードにジャンプする前にBSSセクションを設定することだけが必要です。

The BSS section is used to store statically allocated, uninitialized data. Linux carefully ensures this area of memory is first zeroed using the following code:

BSSセクションは静的に割り当てられ、初期化されていないデータとして使われます。Linuxは次のコードを用いて最初をゼロとしたメモリのこのエリアを注意深く確保します。

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First, the [__bss_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L47) address is moved into `di`. Next, the `_end + 3` address (+3 - aligns to 4 bytes) is moved into `cx`. The `eax` register is cleared (using a `xor` instruction), and the bss section size (`cx`-`di`) is calculated and put into `cx`. Then, `cx` is divided by four (the size of a 'word'), and the `stosl` instruction is used repeatedly, storing the value of `eax` (zero) into the address pointed to by `di`, automatically increasing `di` by four, repeating until `cx` reaches zero). The net effect of this code is that zeros are written through all words in memory from `__bss_start` to `_end`:

最初、[__bss_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L47)アドレスは`di`に移動されます。次、`_end + 3`アドレス(+3 - 4バイトにアラインする)は`cx`に移動します。そして、`cx`は4('word'のサイズ)で除算され、`stosl`命令は繰り返し使われ、`eax` (ゼロ)の値を`di`で示されているアドレスにストアし、自動的に`di`に4を加算し、`cx`がゼロに至るまで繰り返されます。このコードの正味の影響は`__bss_start`から`_end`まで全てのワードを介してメモリにゼロが書かれます。


![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
メインにジャンプ
--------------------------------------------------------------------------------

That's all - we have the stack and BSS, so we can jump to the `main()` C function:

それが全てだ。我々がスタックとBSSを持つこと。そして我々はC言語の`main()`関数にジャンプ出来ます。

```assembly
    calll main
```

The `main()` function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c). You can read about what this does in the next part.

`main()`関数は[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c)にある。あなたは次の章でこれが何をするのか読むことが出来ます。

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about Linux kernel insides. If you have questions or suggestions, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com), or just create an [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part, we will see the first C code that executes in the Linux kernel setup, the implementation of memory routines such as `memset`, `memcpy`, `earlyprintk`, early console implementation and initialization, and much more.

これはLinux kernel insidesについての最初のパートの終わりです。もし疑問や助言があるのなら[0xAX](https://twitter.com/0xAX)で連絡するか、[email](anotherworldofworld@gmail.com)でご一報頂けるか、[issue](https://github.com/0xAX/linux-internals/issues/new)を作成してください。次のパートでは、我々はLinuxカーネルセットアップで実行される最初のCコード、`memset`, `memcpy`, `earlyprintk`のようなメモリールーチンの実装、初期のコンソール実装と初期化などなどいっぱい、を見るでしょう。

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

** 英語は私の第一言語ではありません。またご不便をおかけします。もし間違いを見つけたのならどうか[linux-insides](https://github.com/0xAX/linux-internals)にプルリクを投げてください。 **

Links
リンク
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel速 Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://en.wikipedia.org/wiki/Intel_8086)
  * [80386](http://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](http://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [CoreBoot developer manual](http://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](http://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](http://en.wikipedia.org/wiki/Power_good_signal)
