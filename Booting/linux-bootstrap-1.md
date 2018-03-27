Kernel booting process. Part 1.
================================================================================

From the bootloader to the kernel
--------------------------------------------------------------------------------

If you have been reading my previous [blog posts](https://0xax.github.io/categories/assembler/), then you can see that, for some time now, I have been starting to get involved with low-level programming. I have written some posts about assembly programming for `x86_64` Linux and, at the same time, I have also started to dive into the Linux kernel source code.

�⤷���ʤ�����β���[�֥������](https://0xax.github.io/categories/assembler/)���ɤ���Τʤ顢����ޤǤδ֡���ϥ���٥�ץ���ߥ󥰤˴�Ϳ���Ƥ��ޤäƤ��뤳�Ȥ򤢤ʤ������򤹤뤳�Ȥ�����롣��Ϥ���ޤǤδ�`x86_64` Linux�Υ�����֥�ץ����ˤĤ��Ƥδ��Ĥ�����ƤˤĤ��ƽ񤤤Ƥ��Ƥ��뤷��Linuxx�����ͥ륽���������ɤ���Ƭ���Ϥ�Ƥ⤤�롣

I have a great interest in understanding how low-level things work, how programs run on my computer, how they are located in memory, how the kernel manages processes and memory, how the network stack works at a low level, and many many other things. So, I have decided to write yet another series of posts about the Linux kernel for the **x86_64** architecture.

��ϥ���٥��ư���Υ���ԥ塼������ǤΥץ�����ư������Υ������֡������ͥ�Υץ����ȥ���δ���������٥�ǤΥͥåȥ�������å���ư��ʤɿ����ʼ�ˡ�ˤĤ������Ѷ�̣����äƤ��롣
�����ơ����**x86_64**�������ƥ�����˴ؤ���Linux�����ͥ�˴ؤ���¾�ΰ�Ϣ����Ƥ򺣤ΤȤ���񤯤��Ȥ�迴���Ƥ��롣

Note that I'm not a professional kernel hacker and I don't write code for the kernel at work. It's just a hobby. I just like low-level stuff, and it is interesting for me to see how these things work. So if you notice anything confusing, or if you have any questions/remarks, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com) or just create an [issue](https://github.com/0xAX/linux-insides/issues/new). I appreciate it.

��ջ��ࡣ��ϥץ�Υ����ͥ�ϥå����ǤϤʤ����Ż��Ȥ��ƥ����ͥ�Υ����ɤ�񤤤ƤϤ��ʤ�������ϼ�̣������ϥ���٥�ؤ��뤳�Ȥ��������������Ƥ���餬�ɤΤ褦��ư���Τ����򤹤���˶�̣����äƤ��롣�����Ƥ⤷���ʤ������餫���𤷤Ƥ��뤳�Ȥ˵��Ť������������ո�������ʤ�Twitter [0xAX](https://twitter.com/0xAX)�˥ĥ����ȡ�[�᡼��](anotherworldofworld@gmail.com)�ˤƤ����󡢤ޤ���GitHub��[issue](https://github.com/0xAX/linux-insides/issues/new)��������Ƥ������������꤬�����פ��ޤ���

All posts will also be accessible at [github repo](https://github.com/0xAX/linux-insides) and, if you find something wrong with my English or the post content, feel free to send a pull request.

���Ƥ���Ƥ�[github repo](https://github.com/0xAX/linux-insides)��Ǳ������뤳�Ȥ����衢�⤷��αѸ����Ƥ������ƤˤĤ��Ƥδְ㤤�򸫤Ĥ����Τʤ餪���ڤ˥ץ�ꥯ�����Ȥ����äƤ���������

*Note that this isn't official documentation, just learning and sharing knowledge.*

*��ջ��ࡣ����ϸ����Υɥ�����ȤǤϤʤ����μ��γؽ��䶦ͭ�Ǥ��롣

**Required knowledge**

**�׵ᤵ����μ�**

* Understanding C code

* C����˴ؤ�������

* Understanding assembly code (AT&T syntax)

* ������֥饳���ɤ˴ؤ������� (AT&T���󥿥å���)

Anyway, if you are just starting to learn such tools, I will try to explain some parts during this and the following posts. Alright, this is the end of the simple introduction, and now we can start to dive into the Linux kernel and low-level stuff.

�Ȥˤ������⤷���ʤ������ʤȤ��Ƴؽ����Ϥ᤿�Τʤ顢����伡����ƤΤ����Ĥ��ξϤ��������ޤ�������ס�����ϴ�ñ��Ƴ�����ν����Ǥ��������ƺ��䤿����Linux�����ͥ�����٥�˴ؤ��뤳�Ȥ���Ƭ���뤳�Ȥ�Ϥ�뤳�Ȥ�����ޤ���

I've started to write this book at the time of the `3.18` Linux kernel, and many things might change from that time. If there are changes, I will update the posts accordingly.

���`3.18` Linux�����ͥ���ˤ����ܤ�񤭻Ϥ�Ƥ��ޤ��������Ƥ��λ����餿������Τ��Ȥ��ѹ��ˤʤä����⤷��ޤ��󡣤⤷�ѹ�������Τʤ餽��˱�������Ƥ򹹿����ޤ���

The Magical Power Button, What happens next?
--------------------------------------------------------------------------------

Although this is a series of posts about the Linux kernel, we will not be starting directly from the kernel code - at least not, in this paragraph. As soon as you press the magical power button on your laptop or desktop computer, it starts working. The motherboard sends a signal to the [power supply](https://en.wikipedia.org/wiki/Power_supply) device. After receiving the signal, the power supply provides the proper amount of electricity to the computer. Once the motherboard receives the [power good signal](https://en.wikipedia.org/wiki/Power_good_signal), it tries to start the CPU. The CPU resets all leftover data in its registers and sets up predefined values for each of them.

���줬Linux�����ͥ�˴ؤ����Ϣ����ƤǤ���ˤ�ؤ�餺�����ʤ��Ȥ⤳����Ǥϲ桹�ϥ����ͥ륳���ɤ���ľ�ܳ��Ϥ��뤳�Ȥ�̵���Ǥ��礦�����ʤ��ΥΡ���PC��ǥ����ȥå�PC���Ի׵Ĥ��Ÿ������å��򲡤��䤤�ʤ䡢����ԥ塼������ư��Ϥ�롣�ޥ����ܡ��ɤ�[�Ÿ�](https://en.wikipedia.org/wiki/Power_supply)�ǥХ����˿�������롣�������������塢�Ÿ���Ŭ�ڤ��̤����Ϥ򶡵뤹�롣���٥ޥ����ܡ��ɤ�[�ѥ���åɿ���](https://en.wikipedia.org/wiki/Power_good_signal)��������ȡ�CPU��ư�����褦��ߤ롣CPU�����ƤΥ쥸�����˻ĤäƤ���ǡ�����ꥻ�åȤ����ꥻ�åȤˤ�ꤽ����ͤϽ�������ͤˤʤ롣


The [80386](https://en.wikipedia.org/wiki/Intel_80386) CPU and later define the following predefined data in CPU registers after the computer resets:

[80386](https://en.wikipedia.org/wiki/Intel_80386) CPU�䤽��ʹߤ����ʤϥ���ԥ塼�������ꥻ�åȤ��줿�塢CPU�Υ쥸�����ϼ��Υץ�ǥե����󤵤줿�ǡ����ˤʤ롣

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

The processor starts working in [real mode](https://en.wikipedia.org/wiki/Real_mode). Let's back up a little and try to understand [memory segmentation](https://en.wikipedia.org/wiki/Memory_segmentation) in this mode. Real mode is supported on all x86-compatible processors, from the [8086](https://en.wikipedia.org/wiki/Intel_8086) CPU all the way to the modern Intel 64-bit CPUs. The `8086` processor has a 20-bit address bus, which means that it could work with a `0-0xFFFFF` or `1 megabyte` address space. But it only has `16-bit` registers, which have a maximum address of `2^16 - 1` or `0xffff` (64 kilobytes).
�ץ��å�����[�ꥢ��⡼��](https://en.wikipedia.org/wiki/Real_mode)��ư��Ϥ�ޤ��������ä��ᤵ���Ƥ��������������Ƥ��Υ⡼�ɤǤ�[���ꥻ�����ơ������](https://en.wikipedia.org/wiki/Memory_segmentation) �ˤĤ������򤷤Ƥ����ޤ��礦���ꥢ��⡼�ɤ�[8086](https://en.wikipedia.org/wiki/Intel_8086) CPU���鸽���Intel 64-bit CPU�ޤǤ����Ƥ�x86�ߴ��ץ��å����ǥ��ݡ��Ȥ���Ƥ��ޤ���`8086` �ץ��å�����
`0-0xFFFFF` �ޤ��� `1�ᥬ�Х���` �Υ��ɥ쥹���֤�Ϣ�Ȥ���ư��뤳�Ȥ����������̣����20bit�Υ��ɥ쥹�Х�����äƤ��ޤ���������`16-bit`�Υ쥸�����������äƤ��ʤ����ᡢ���祢�ɥ쥹��`2^16 - 1` �ޤ��� `0xffff` (64����Х���)�ˤʤ�ޤ���

[Memory segmentation](http://en.wikipedia.org/wiki/Memory_segmentation) is used to make use of all the address space available. All memory is divided into small, fixed-size segments of `65536` bytes (64 KB). Since we cannot address memory above `64 KB` with 16-bit registers, an alternate method was devised.

[���ꥻ�����ơ������](http://en.wikipedia.org/wiki/Memory_segmentation) �Ϥ��٤Ƥ�ͭ���ʥ��ɥ쥹���֤�Ȥ�����˻Ȥ��롣���ƤΥ���Ͼ����ʡ�`65536` �Х��� (64 KB)�θ��ꥵ�����Υ������Ȥ�ʬ�䤵��Ƥ��롣�桹��16-bit�Υ쥸������`64 KB`��Ķ��������򥢥ɥ�å��󥰤��뤳�Ȥ�����ʤ��Τǡ����ؼ�ˡ�Ȥ��ƹͤ��Ф��줿��

An address consists of two parts: a segment selector, which has a base address, and an offset from this base address. In real mode, the associated base address of a segment selector is `Segment Selector * 16`. Thus, to get a physical address in memory, we need to multiply the segment selector part by `16` and add the offset to it:

���ɥ쥹�ϡ��١������ɥ쥹����ĥ������ȥ��쥯�����Ȥ��Υ١������ɥ쥹����Υ��ե��åȡ�����2�Ĥ��鹽������롣�ꥢ��⡼�ɤǤϡ��������ȥ��쥯�����˴�Ϣ�����١������ɥ쥹��`Segment Selector * 16`�Ȥʤ롣���Τ褦�ˡ������ʪ�����ɥ쥹�����뤿��ˡ��桹�ϥ������ȥ��쥯����`16`��ݤ���碌��������Ф��ƥ��ե��åȤ�û�����ɬ�פ����롣

```
PhysicalAddress = Segment Selector * 16 + Offset
```

For example, if `CS:IP` is `0x2000:0x0010`, then the corresponding physical address will be:

�㤨�С��⤷`CS:IP` �� `0x2000:0x0010` �ʤ顢��������ʪ�����ɥ쥹�ϼ��Τ褦�ˤʡ�

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

But, if we take the largest segment selector and offset, `0xffff:0xffff`, then the resulting address will be:

���������⤷�Ǥ��礭���������Ȥ쥻�������ȥ��ե��åȡ�`0xffff:0xffff`�Ǥ��ä���祢�ɥ쥹�ϼ��Τ褦�ˤʤ롣

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

which is `65520` bytes past the first megabyte. Since only one megabyte is accessible in real mode, `0x10ffef` becomes `0x00ffef` with the [A20 line](https://en.wikipedia.org/wiki/A20_line) disabled.

`65520`�Х��Ȥ���Ƭ�Υ��ɥ쥹��ʬ�˸���롣�ꥢ��⡼�ɤǤ�1�ᥬ�Х��Ȥ���������������ʤ����ᡢ`0x10ffef`��[A20 line](https://en.wikipedia.org/wiki/A20_line)���ǥ������֥�Ǥ��뤳�Ȥ���`0x00ffef`�ˤʤ롣


Ok, now we know a little bit about real mode and memory addressing in this mode. Let's get back to discussing register values after reset.

OK�������Υ⡼�ɤǤΥꥢ��⡼�ɤȥ��ꥢ�ɥ�å��󥰤ˤĤ��Ƽ㴳�ΤäƤ��롣�����ꥻ�åȸ�Υ쥸�����ͤˤĤ��Ƥε����������

The `CS` register consists of two parts: the visible segment selector, and the hidden base address. While the base address is normally formed by multiplying the segment selector value by 16, during a hardware reset the segment selector in the CS register is loaded with `0xf000` and the base address is loaded with `0xffff0000`; the processor uses this special base address until `CS` is changed.

`CS`�쥸�����ϥӥ��֥륻�����ȥ��쥯���Ⱦ�̤Υ١������ɥ쥹��2�Ĥ����Ǥ��鹽������롣�١������ɥ쥹���̾糧�����ȥ��쥯����16�ܤȤ��Ƥ����Ʊ�����ϡ��ɥ������ꥻ�å����CS�쥸�����Υ������ȥ��쥯����`0xf000`���١������ɥ쥹��`0xffff0000`�Ȥʤ롣�ץ��å�����`CS`���ѹ������ޤ����̤ʥ١������ɥ쥹�Ȥ��ƻ��Ѥ��롣

The starting address is formed by adding the base address to the value in the EIP register:

���ϥ��ɥ쥹�ϥ١������ɥ쥹��EIP�쥸�������ͤ�û�������ΤǷ�������롣

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

We get `0xfffffff0`, which is 16 bytes below 4GB. This point is called the [Reset vector](http://en.wikipedia.org/wiki/Reset_vector). This is the memory location at which the CPU expects to find the first instruction to execute after reset. It contains a [jump](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) instruction that usually points to the BIOS entry point. For example, if we look in the [coreboot](http://www.coreboot.org/) source code (`src/cpu/x86/16bit/reset16.inc`), we will see:

�桹��4GB�μ�����16�Х��ȤǤ���`0xfffffff0`�����롣���Υݥ���Ȥ�[�ꥻ�åȥ٥�����](http://en.wikipedia.org/wiki/Reset_vector)�ȸƤФ�롣�����CPU���ꥻ�åȸ�¹Ԥ���ǽ��̿��򸫤Ĥ��뤳�Ȥ���Ԥ����������֤Ǥ��롣�̾�BIOS�Υ���ȥ꡼�ݥ���Ȥ򼨤���[������](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`)̿��ǹ�������롣�㤨��,[coreboot](http://www.coreboot.org/) �Υ����������� (`src/cpu/x86/16bit/reset16.inc`)�򸫤�Ȱʲ��Τ褦�ˤʤäƤ��롣

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

������`0xe9` �ȥ������襢�ɥ쥹����2��������� `_start - ( . + 2)` �Ǥ���`jmp` ̿�� [opcode](http://ref.x86asm.net/coder32.html#xE9)�򸫤뤳�Ȥ�����롣

We can also see that the `reset` section is `16` bytes and that is compiled to start from `0xfffffff0` address (`src/cpu/x86/16bit/reset16.lds`):

�桹��`reset` ��������� `16` �Х��ȤǤ���`0xfffffff0` ���ɥ쥹 (`src/cpu/x86/16bit/reset16.lds`)���鳫�Ϥ����褦�˥���ѥ��뤷�Ƥ��뤳�Ȥ��ǧ�Ǥ��롣

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

�� BIOS�����Ϥ���ޤ���������ȥϡ��ɥ������γ�ǧ�θ�ǡ�BIOS�ϥ֡��Ȳ�ǽ�ʥǥХ�����ȯ������ɬ�פ����롣�֡��Ƚ��BIOS�Υ���ե����졼������ΰ�˳�Ǽ���졢BIOS�������˵��ܤ���Ƥ���ǥХ������鵯ư���ߤ�褦�����椵��롣�ϡ��ɥǥ��������鵯ư���ߤ����BIOS�ϥ֡��ȥ���������õ����[MBR�ѡ��ƥ������](https://en.wikipedia.org/wiki/Master_boot_record)��ʬ�䤵�줿�ϡ��ɥǥ�������ˡ��֡��ȥ��������ϥ�����ñ�̤�`512`�Х��ȤǤ���ǽ�Υ�����������Ƭ`446`�Х��Ȥ˵�Ͽ����Ƥ��롣���κǽ�Υ��������κǸ��2�Х��Ȥ�BIOS���֡��Ȳ�ǽ�ǥХ����Ǥ��뤳�Ȥ�򼨤������`0x55` �� `0xaa`����Ͽ����Ƥ��롣

For example:

��:

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

�ʲ��Υ��ޥ�ɤǥӥ�ɤȼ¹�

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

This will instruct [QEMU](http://qemu.org) to use the `boot` binary that we just built as a disk image. Since the binary generated by the assembly code above fulfills the requirements of the boot sector (the origin is set to `0x7c00` and we end with the magic sequence), QEMU will treat the binary as the master boot record (MBR) of a disk image.

�����[QEMU]�����������ǥ��������᡼���Ȥ��ƥӥ�ɤ���`boot`�Х��ʥ꡼��Ȥ����Ȥ�ؼ����롣�Х��ʥ꡼�ϥ֡��ȥ�������(`0x7c00`)���׵�����������塢������֥饳���ɤ����������줿��ΤʤΤǡ�QEMU�ϥХ��ʥ꡼��ǥ��������᡼���Υޥ������֡��ȥ쥳����(MBR)�Ȥ��ư����롣

You will see:

���Τ褦�ˤʤ롣

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

In this example, we can see that the code will be executed in `16-bit` real mode and will start at `0x7c00` in memory. After starting, it calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt, which just prints the `!` symbol; it fills the remaining `510` bytes with zeros and finishes with the two magic bytes `0xaa` and `0x55`.

������Ǥϡ������ɤ�`16-bit`�ꥢ��⡼�ɤǼ¹Ԥ��졢����Υ��ɥ쥹`0x7c00`���鳫�Ϥ����Τ򸫤뤳�Ȥ�����롣���ϸ塢`!`����ܥ��ɽ������[0x10](http://www.ctyme.com/intr/rb-0106.htm)�����ߤ��ƤФ�롣�Ĥ��`510`�Х��Ȥϥ�����ᤵ��Ƥ���2�Х��ȤΥޥ��å������ɤǤ���`0xaa`��`0x55`�ǽ���롣

You can see a binary dump of this using the `objdump` utility:

`objdump`���Ѥ��뤳�ȤǤ��ΥХ��ʥ꡼����פ򸫤뤳�Ȥ�����롣

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

A real-world boot sector has code for continuing the boot process and a partition table instead of a bunch of 0's and an exclamation mark :) From this point onwards, the BIOS hands over control to the bootloader.

���������Υ֡��ȥ��������ϥ֡��ȥץ�����³���뤿��Υ����ɤ�0��«�ȥ���������᡼�����ޡ���������Υѡ��ƥ������ơ��֥�Ǥ��롣 :)�����������顢BIOS�ϥ֡��ȥ������˰����Ϥ���


**NOTE**: As explained above, the CPU is in real mode; in real mode, calculating the physical address in memory is done as follows:

**���** �嵭�������Ǥ�CPU�ϥꥢ��⡼�ɤǤ��롣�ꥢ��⡼�ɤǤϡ�����ˤ�����ʪ�����ɥ쥹�η׻��ϰʲ��Τ褦�ˤʤ�:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

just as explained above. We have only 16-bit general purpose registers; the maximum value of a 16-bit register is `0xffff`, so if we take the largest values, the result will be:

���礦�ɾ嵭�Τ褦����������롣16bit�����ѥ쥸�����Τߤ��롣: 16bit�쥸�����κ����ͤ�`0xffff`�Ǥ��ꡢ�����Ǥ⤷�����ͤǤ��ä��ʤ��̤ϰʲ��Τ褦�ˤʤ롣

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

where `0x10ffef` is equal to `1MB + 64KB - 16b`. An [8086](https://en.wikipedia.org/wiki/Intel_8086) processor (which was the first processor with real mode), in contrast, has a 20-bit address line. Since `2^20 = 1048576` is 1MB, this means that the actual available memory is 1MB.

`0x10ffef`��`1MB + 64KB - 16b`�������Ǥ��롣�ꥢ��⡼�ɤ���ä��ǽ�Υץ��å��Ǥ���[8086](https://en.wikipedia.org/wiki/Intel_8086)�ץ��å����о�Ū��20bit�Υ��ɥ쥹�饤�󤬤��롣`2^20 = 1048576`��1MB�ʤΤǡ�����ϼºݤ�ͭ�����꤬1MB�Ǥ��뤳�Ȥ��̣���Ƥ��롣

In general, real mode's memory map is as follows:

���̤ˡ��ꥢ��⡼�ɤΥ���ޥåפϼ��Τ褦�ˤʤ롣:

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

������ƤΤϤ���ǡ����CPU�ˤ��ǽ��̿��μ¹Ԥ�`0xFFFFF` (1MB)���Ϥ뤫���礭��`0xFFFFFFF0`���ɥ쥹�����֤����ȵ��ܤ�����CPU�ϥꥢ��⡼�ɤǤɤΤ褦�ˤ��Υ��ɥ쥹�˥�����������Τ���������[coreboot](http://www.coreboot.org/Developer_Manual/Memory_map)�Υɥ�����Ȥˤ���:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

At the start of execution, the BIOS is not in RAM, but in ROM.

�¹Գ��ϤǤ�BIOS��RAM�ǤϤʤ�ROM�Ǥ��롣

Bootloader
--------------------------------------------------------------------------------

There are a number of bootloaders that can boot Linux, such as [GRUB 2](https://www.gnu.org/software/grub/) and [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). The Linux kernel has a [Boot protocol](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt) which specifies the requirements for a bootloader to implement Linux support. This example will describe GRUB 2.

[GRUB 2](https://www.gnu.org/software/grub/)��[syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project)�Τ褦��Linux��֡��Ȥ��뤳�Ȥ������֡��ȥ������������Ĥ����롣

Linux kernel�ˤ�Linux�򥵥ݡ��Ȥ���褦�˼������줿bootloader���׵᤹��褦���������줿[Boot protocol](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt)��¸�ߤ��롣������Ȥ���GRUB2�ˤĤ��Ƶ��ܤ��褦��

Continuing from before, now that the `BIOS` has chosen a boot device and transferred control to the boot sector code, execution starts from [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). This code is very simple, due to the limited amount of space available, and contains a pointer which is used to jump to the location of GRUB 2's core image. The core image begins with [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), which is usually stored immediately after the first sector in the unused space before the first partition. The above code loads the rest of the core image, which contains GRUB 2's kernel and drivers for handling filesystems, into memory. After loading the rest of the core image, it executes the [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) function.

���������³���ơ���`BIOS`�֡��ȥǥХ��������򤷡��֡��ɥ����������ɤ������ž������[boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD)����¹Ԥ��롣���Υ����ɤϸ��ꤵ�줿ͭ���ʶ��֤Τ���ȤƤ�ñ��ǡ�GRUB 2�Υ����ɥ��᡼���Υ��ɥ쥹�˥����פ��뤿��Υݥ��󥿡���ޤ�Ǥ��롣�����ɥ��᡼�����̾���쥻�����θ������ѡ��ƥ����������ˤ���̤���Ѷ��֤��ܤ��ƽ񤭹��ޤ줿[diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD)�ǻϤޤ�ޤ����嵭�����ɤ�GRUB2�����ͥ��ե����륷���ƥ���갷������Υɥ饤�С���ޤ���������᡼���λĤ�Ȥ��ƥ��꡼���ɤ߹��ޤ�ޤ����Ĥ�Υ������᡼������ɤ�����ǡ�[grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c)��ǽ���¹Ԥ���ޤ���

The `grub_main` function initializes the console, gets the base address for modules, sets the root device, loads/parses the grub configuration file, loads modules, etc. At the end of execution, the `grub_main` function moves grub to normal mode. The `grub_normal_execute` function (from the `grub-core/normal/main.c` source code file) completes the final preparations and shows a menu to select an operating system. When we select one of the grub menu entries, the `grub_menu_execute_entry` function runs, executing the grub `boot` command and booting the selected operating system.

`grub_main`��ǽ�ϥ��󥽡�������������⥸�塼��Υ١������ɥ쥹����������롼�ȥǥХ����򥻥åȤ���grub����ե����졼�����ե�������ɤ߹��ߡ����Ϥ����⥸�塼����ɤ߹���ʤɤ�Ԥ��ޤ��������μ¹Ը塢`grub_main`��ǽ��grub�ΥΡ��ޥ�⡼�ɤ˰ܹԤ��ޤ���`grub_normal_execute`��ǽ��(�����������ɥե�����`grub-core/normal/main.c`����)�������λ�����ڥ졼�ƥ��󥰥����ƥ�����򤹤��˥塼��ɽ�����ޤ���grub��˥塼�Υ���ȥ꡼�ΰ�Ĥ����򤷤�����`grub_menu_execute_entry`��ǽ���¹Ԥ��졢grub��`boot`���ޥ�ɤ��¹Ԥ������򤵤줿���ڥ졼�ƥ��󥰥����ƥब�¹Ԥ���ޤ���

As we can read in the kernel boot protocol, the bootloader must read and fill some fields of the kernel setup header, which starts at the `0x01f1` offset from the kernel setup code. You may look at the boot [linker script](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L16) to confirm the value of this offset. The kernel header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S) starts from:

�����ͥ�֡��ȥץ�ȥ�����ɤ߹��ޤ��뤳�Ȥ������Τǡ������ͥ륻�åȥ��åץ����ɤ���fset��`0x01f1`�ǻϤޤ�֡��ȥ��������ɤ߹��ߡ������ͥ륻�åȥ��åץإå����Τ����Ĥ��Υե�����ɤ����ʤ���Фʤ�ޤ��󡣤��Υ��ե��å��ͤ��ǧ���뤿��˥֡���[��󥫡�������ץ�](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L16)�򸫤뤫�⤷��ޤ��󡣥����ͥ�إå���[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S)�ϼ��Τ褦�˳��Ϥ���ޤ���

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

�֡��ȥ������Ϥ���ȥإå����λĤ�� ([������](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L354)�Τ褦�ʡ�Linux�֡��ȥץ�ȥ����`write`�����פǻϤޤ��ΤȤ��ƥޡ������줿��ΤΤ�)���ޥ�ɥ饤�󤫤������ä��ͤޤ��ϥ֡�����˷׻����줿�ͤΤɤ��餫�����ޤ���(�桹�Ͼܺ�������Ķ���ʤ����������ͥ륻�åȥ��åץե��å��������ƤΥե�����ɤˤĤ����������ʤ��Ǥ��礦���������桹�ϤɤΤ褦�˥����ͥ뤬������ɤ��Ȥ����ˤĤ��Ƶ����������ˤ��٤��Ǥ������ʤ���[�֡��ȥץ�ȥ���](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L156)������ƤΥե�����ɤ������򸫤Ĥ��뤳�Ȥ�����ޤ���

As we can see in the kernel boot protocol, the memory will be mapped as follows after loading the kernel:

�桹�ϥ����ͥ�֡��ȥץ�ȥ���Ǹ�������Ǥ���褦�ˡ�����ϥ����ͥ뤬���ɤ��줿�弡�Τ褦�˥ޥåԥ󥰤���ޤ���

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

�����ơ��֡��ȥ������������ͥ�򥳥�ȥ��뤹�뤳�Ȥ�ܹԤ������Ϥ��ޤ���

```
X + sizeof(KernelBootSector) + 1
```

where `X` is the address of the kernel boot sector being loaded. In my case, `X` is `0x10000`, as we can see in a memory dump:

`X`�ϥ����ͥ�֡��ȥ������������ɤ���륢�ɥ쥹���Ǥ�����ξ��Ǥϡ�`X`��`0x10000`�ǡ��������׾�Ǹ��뤳�Ȥ�����ޤ���

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

The bootloader has now loaded the Linux kernel into memory, filled the header fields, and then jumped to the corresponding memory address. We can now move directly to the kernel setup code.

�֡��ȥ������Ϻ������Linux�����ͥ����ɤ����إå����ե�����ɤ���ᡢ�����Ƴ����Υ��ꥢ�ɥ쥹�˥����פ��ޤ����桹�Ϻ������ͥ륻�åȥ��åץ����ɤ�ľ�ܰ�ư���뤳�Ȥ�����ޤ���

The Beginning of the Kernel Setup Stage
�����ͥ륻�åȥ��åץ��ơ����ΤϤ���
--------------------------------------------------------------------------------

Finally, we are in the kernel! Technically, the kernel hasn't run yet; first, the kernel setup part must configure stuff such as the decompressor and some memory management related things, to name a few. After all these things are done, the kernel setup part will decompress the actual kernel and jump to it. Execution of the setup part starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S) at [_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292). It is a little strange at first sight, as there are several instructions before it.

�Ǹ�ˡ��桹�ϥ����ͥ����ˤ��ޤ�������Ū�ˡ������ͥ��̤���¹Ԥ���Ƥ��ޤ��󡣺ǽ顢�����ͥ륻�åȥ��åץѡ��Ȥ�Ÿ����̾����������Ϣ���뤤���Ĥ��Υ�������Ȥ��Ƥ�Τ������ʤ���Фʤ�ޤ��󡣤����Τ��Ȥ����ƽ���ä��塢�����ͥ륻�åȥ��åץѡ��ȤϼºݤΥ����ͥ��Ÿ�����������˥����פ��ޤ������åȥ��åץѡ��Ȥμ¹Ԥ�[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S)��[_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292)����Ϥޤ�ޤ������������ˤ����Ĥ���̿�᤬����Τǽ鸫�ǤϾ�����̯�Ǥ���

A long time ago, the Linux kernel used to have its own bootloader. Now, however, if you run, for example,

�Ϥ뤫�Ρ�Linux�����ͥ�ϥ֡��ȥ��������ȤȤ��ƻȤ��Ƥ��ޤ����������������ʤ��顢���ʤ����¹Ԥ���ʤ顢���Τ褦����ˤʤ�ޤ���

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

then you will see:

�����Ƥ��ʤ��ϲ����򸫤�Ǥ��礦��

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

Actually, the file `header.S` starts with the magic number [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (see image above), the error message that displays and, following that, the [PE](https://en.wikipedia.org/wiki/Portable_Executable) header:

�ºݡ�`header.S`�ե�����ϥޥ��å��ʥ�С�[MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable)(���᡼���򻲾Ȥ��Ƥ�������)�ǻϤޤꡢ���顼��å�������ɽ�����졢[PE](https://en.wikipedia.org/wiki/Portable_Executable)�إå����ϰʲ��Τ��Ȥˤʤ�ޤ���

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

�����[UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)�򥵥ݡ��Ȥ��Ƥ��륪�ڥ졼�ƥ��󥰡������ƥ����ɤ���ɬ�פ�����ޤ����桹��ľ���ˤ���������ư�����򸫤ʤ���������Ϥǥ��С�����Ǥ��礦��

The actual kernel setup entry point is:

�ºݤΥ����ͥ륻�åȥ��åץ���ȥ꡼�ݥ���Ȥϰʲ��Ǥ���

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (grub2 and others) knows about this point (at an offset of `0x200` from `MZ`) and makes a jump directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:

�֡��ȥ����� (grub2�ʤ�) �Ϥ��Υݥ����(`MZ`����`0x200`�Υ��ե��å�)���ΤäƤ��ꡢ���顼��å�������ɽ������`header.S` ��`.bstext`��������󤫤�Ϥޤ�ˤ⤫����餺�ºݤ�����ľ�ܥ����פ���褦�ˤ��ޤ���

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

The kernel setup entry point is:

�����ͥ륻�åȥ��åץ���ȥ꡼�ݥ���Ȥϰʲ��Ǥ���

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

�桹��`start_of_setup-1f`�ݥ���Ȥ˥����פ���`jmp`̿�ᥪ�ڥ�����(`0xeb`)�򸫤뤳�Ȥ�����ޤ���`Nf`ɽ��ˡ�ǡ��㤨��`2f`�ϥ������٥�`2:`�򻲾Ȥ��ޤ���; �桹�η�Ǥϡ������פ�ľ���¸�ߤ����٥�`1`�ǡ����åȥ��å�[�إå���](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/boot.txt#L156)�λĤ��ޤ�Ǥ��ޤ������åȥ��åץإå���ľ��桹��`start_of_setup` ��٥�ǻϤޤ�`.entrytext` ���������򸫤ޤ���

This is the first code that actually runs (aside from the previous jump instructions, of course). After the kernel setup part receives control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the kernel real mode, i.e., after the first 512 bytes. This can be seen in both the Linux kernel boot protocol and the grub2 source code:

����ϼºݼ¹Ԥ����ǽ�Υ����ɤǤ���(���������Υ�����̿����̤Ȥ���)�����ͥ륻�åȥ��åץѡ��Ȥϥ֡��ȥ��������饳��ȥ������������塢�ǽ��`jmp`̿��ϥ����ͥ�ꥢ��⡼�ɤκǽ餫��`0x200`���ե��åȤΰ��֤ˤ��ꡢ�㤨�С��ǽ��512�Х��Ȥθ�������Linux�����ͥ�֡��ȥץ�ȥ����grub2�����������ɤ�ξ���Ǹ��뤳�Ȥ��Ǥ��ޤ���


```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

This means that segment registers will have the following values after kernel setup starts:

����ϥ������ȥ쥸�����������ͥ륻�åȥ��åפ����Ϥ��줿��μ����ͤ���Ĥ��Ȥ��̣���ޤ���

```
gs = fs = es = ds = ss = 0x10000
cs = 0x10200
```

In my case, the kernel is loaded at `0x10000` address.

��ξ�硢�����ͥ��`0x10000`���ɥ쥹�����֤���ޤ���

After the jump to `start_of_setup`, the kernel needs to do the following:

`start_of_setup`�˥����פ����塢�����ͥ�ϼ��Τ��Ȥ�Ԥ�ɬ�פ�����ޤ���

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c)

* �������ȥ쥸�������ͤ����������Ȥ��ǧ����
* �⤷ɬ�פʤ顢�����������å������ꤹ��
* [bss](https://en.wikipedia.org/wiki/.bss)�����ꤹ��
* [main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c)��C�����ɤ˥����פ���

Let's look at the implementation.

���������򸫤Ƥߤޤ��礦��

Aligning the Segment Registers 
�������ȥ쥸�����ΰ��ֹ�碌
--------------------------------------------------------------------------------

First of all, the kernel ensures that the `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

�ޤ��Ϥ���ˡ������ͥ��`ds`��`es`�������ȥ쥸������Ʊ�����ɥ쥹�򼨤��ͤˤ��ޤ������ˡ�`cld`̿���Ȥäƥǥ��쥯�����ե饰�򥯥ꥢ���ޤ���

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, `grub2` loads kernel setup code at address `0x10000` by default and `cs` at `0x10200` because execution doesn't start from the start of file, but from the jump here:

���˽񤤤��褦�ˡ�`grub2`�ϥǥե���ȤȤ��ƥ��ɥ쥹`0x10000`��`cs`��`0x10200`�Ȥ��ƥ����ͥ륻�åȥ��åץ����ɤ��ɤ߹��ߤޤ����ʤ��ʤ�¹Ԥϥե�����κǽ餫�鳫�Ϥ���������[4d 5a](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L46)����`512`�Х��ȤΥ��ե��åȤΥ�����̿�ᤫ��Ϥޤ�ޤ���

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

which is at a `512` byte offset from [4d 5a](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L46). We also need to align `cs` from `0x10200` to `0x10000`, as well as all other segment registers. After that, we set up the stack:

�桹�Ϲ���¾�Υ������ȥ쥸������Ʊ�ͤ�`0x10200` ���� `0x10000`��`cs`�򥢥饤�󤹤�ɬ�פ�����ޤ���
���θ塢�桹��[6](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L494)��٥�Υ��ɥ쥹�˽���`ds`���ͤ򥹥��å����Ѥॹ���å������ꤷ��`lretw`̿���¹Ԥ��ޤ���

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack, followed by the address of the [6](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L494) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterward, `ds` and `cs` will have the same values.

`lretw`̿�᤬�ƤФ줿������٥�`6`�Υ��ɥ쥹��[���󥹥ȥ饯�����ݥ���](https://en.wikipedia.org/wiki/Program_counter)�쥸�����˥��ɤ��졢`ds`���ͤȰ���`cs`�˥��ɤ���롣���θ塢`ds`��`cs`��Ʊ���ͤˤʤ�Ǥ��礦��

Stack Setup
�����å�������
--------------------------------------------------------------------------------

Almost all of the setup code is in preparation for the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L569) is checking the `ss` register value and making a correct stack if `ss` is wrong:

���åȥ��åץ����ɤΤۤȤ�����Ƥϥꥢ��⡼�ɤ�C����Ķ������˽�������Ƥ��롣����[���ƥå�](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L569)�Ǥ�`ss`�쥸�������ͤ������å����졢�⤷`ss`���ְ�äƤ����������������å����ѹ�����롣

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

�����3�Ĥΰۤʤ륷�ʥꥪ��Ƴ�����Ȥ�����ޤ���

* `ss` has a valid value `0x1000` (as do all the other segment registers beside `cs`)
* `ss` is invalid and the `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and the `CAN_USE_HEAP` flag is not set (see below)

* `ss`��ͭ������`0x1000`�Ǥ���(`cs`��¾�����ƤΥ������ȥ쥸��������٤�)
* `ss`��̵����`CAN_USE_HEAP`�ե饰�����åȤ���Ƥ��� (��������)
* `ss`��̵����`CAN_USE_HEAP`�ե饰�����åȤ���Ƥ��ʤ� (��������)

Let's look at all three of these scenarios in turn:

�������֤ˤ����3�ĤΥ��ʥꥪ�򸫤Ƥ����ޤ��礦��

* `ss` has a correct address (`0x1000`). In this case, we go to label [2](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L584):

* `ss`�����������ɥ쥹(`0x1000`)�Ǥ������ξ��ϡ��桹�ϥ�٥�[2](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L584)�˹Ԥ��ޤ���

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we set the alignment of `dx` (which contains the value of `sp` as given by the bootloader) to `4` bytes and a check for whether or not it is zero. If it is zero, we put `0xfffc` (4 byte aligned address before the maximum segment size of 64 KB) in `dx`. If it is not zero, we continue to use the value of `sp` given by the bootloader (0xf7f4 in my case). After this, we put the value of `ax` into `ss`, which stores the correct segment address of `0x1000` and sets up a correct `sp`. We now have a correct stack:

�����ǲ桹��(�֡��ȥ�������Ϳ����줿`sp`���ͤ�ޤ�Ǥ���)`dx`�Υ��饤���Ȥ�`4`�Х��ȤȤ������ꤷ�����줬�����ɤ��������å����ޤ����⤷���줬����ʤ顢�桹��`dx`��`0xfffc`(64KB�κ��祻�����ȥ�����������4�Х��ȥ��饤�󤵤줿���ɥ쥹)�Ȥ��ޤ����⤷���줬����Ǥʤ��ä��Τʤ顢�֡��ȥ�������Ϳ����`sp`����(��ξ���0xf7f4)��Ȥ�³���ޤ������θ塢�桹��`0x1000`���������������ȥ��ɥ쥹�����ȥ����줿`ss`�����`ax`���ͤ����ꤷ��������`sp`�����ꤷ�ޤ����桹�Ϻ������������å��ˤʤ�ޤ���

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* In the second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L52) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L321) is a bitmask header which is defined as:

* ��2�Υ��ʥꥪ��(`ss` != `ds`)�Ǥ����ǽ顢�桹��(���åȥ��åץ����ɤν����Υ��ɥ쥹�Ǥ���)[_end](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L52)���ͤ�`dx`�����ꤷ���桹���ҡ��פ�Ȥ����Ȥ�����뤫�ɤ�����ǧ����`testb`̿���Ȥ�`loadflags`�إå����ե�����ɤ��ǧ���ޤ���[loadflags](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L321)�ϼ��Τ褦��������줿�ӥåȥޥ����إå����Ǥ���

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and, as we can read in the boot protocol:

�����ơ��֡��ȥץ��ȥ�����ɤळ�Ȥ�����ޤ���

```
Field name: loadflags
�ե������̾: ���ɥե饰

  This field is a bitmask.

  ���Υե�����ɤϥӥåȥޥ����Ǥ���

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.

  Bit 7 (write): CAN_USE_HEAP
    heap_end_ptr��ͭ���Ȥʤä����Ȥ򼨤��ͤȤ��Ƥ��ΥӥåȤ�1�Ȥ������ꤷ�ޤ���

```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (minimum stack size, `1024` bytes) to it. After this, if `dx` is not carried (it will not be carried, `dx = _end + 1024`), jump to label `2` (as in the previous case) and make a correct stack.

�⤷`CAN_USE_HEAP`�ӥåȤ����ꤵ�줿�ʤ顢�桹��`heap_end_ptr`��(`_end`��ؤ�����)`dx`�����졢(�Ǿ������å�������`1024`�Х���)`STACK_SIZE`�򤽤�˲û����ޤ����⤷`dx`������夬��ʤ��ä��ʤ�(`dx = _end + 1024`�Ϸ���夬��ʤ��Ǥ��礦)��٥�`2`(����ξ��Ȥ���)�˥����פ������������å��ˤ��ޤ���

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

* `CAN_USE_HEAP`�����ꤵ��Ƥ��ʤ������桹��`_end`����`_end + STACK_SIZE`�ޤǤ�Ǿ������å��Ȥ��ƻȤ������Ǥ���

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
BSS����
--------------------------------------------------------------------------------

The last two steps that need to happen before we can jump to the main C code are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking the "magic" signature. First, signature checking:

�Ǹ��2�ĤΥ��ƥåפǤϲ桹��C�����ɤ�main�˥����פ�������[BSS](https://en.wikipedia.org/wiki/.bss)���ꥢ�Ȥ������ꤷ��"�ޥ��å�"��̾������å�����ɬ�פ�����ޤ����ǽ�ν�̾������å����ޤ���

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the [setup_sig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L39) with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is reported.

����ϥޥ��å��ʥ�С�`0x5a5aaa55`��[setup_sig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L39)��ñ�����ӤǤ����⤷����餬�����Ǥʤ��ʤ���̿Ū�ʥ��顼�Ȥ�����𤵤�롣

If the magic number matches, knowing we have a set of correct segment registers and a stack, we only need to set up the BSS section before jumping into the C code.

�⤷�ޥ��å��ʥ�С������פ����顢�桹���������������ȥ쥸�����ȥ����å������ꤵ�줿���Ȥ��Τꡢ�桹��C�����ɤ˥����פ�������BSS�������������ꤹ�뤳�Ȥ�����ɬ�פǤ���

The BSS section is used to store statically allocated, uninitialized data. Linux carefully ensures this area of memory is first zeroed using the following code:

BSS������������Ū�˳�����Ƥ�졢���������Ƥ��ʤ��ǡ����Ȥ��ƻȤ��ޤ���Linux�ϼ��Υ����ɤ��Ѥ��ƺǽ�򥼥�Ȥ�������Τ��Υ��ꥢ����տ������ݤ��ޤ���

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First, the [__bss_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L47) address is moved into `di`. Next, the `_end + 3` address (+3 - aligns to 4 bytes) is moved into `cx`. The `eax` register is cleared (using a `xor` instruction), and the bss section size (`cx`-`di`) is calculated and put into `cx`. Then, `cx` is divided by four (the size of a 'word'), and the `stosl` instruction is used repeatedly, storing the value of `eax` (zero) into the address pointed to by `di`, automatically increasing `di` by four, repeating until `cx` reaches zero). The net effect of this code is that zeros are written through all words in memory from `__bss_start` to `_end`:

�ǽ顢[__bss_start](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/setup.ld#L47)���ɥ쥹��`di`�˰�ư����ޤ�������`_end + 3`���ɥ쥹(+3 - 4�Х��Ȥ˥��饤�󤹤�)��`cx`�˰�ư���ޤ��������ơ�`cx`��4('word'�Υ�����)�ǽ������졢`stosl`̿��Ϸ����֤��Ȥ�졢`eax` (����)���ͤ�`di`�Ǽ�����Ƥ��륢�ɥ쥹�˥��ȥ�������ưŪ��`di`��4��û�����`cx`������˻��ޤǷ����֤���ޤ������Υ����ɤ���̣�αƶ���`__bss_start`����`_end`�ޤ����ƤΥ�ɤ�𤷤ƥ���˥����񤫤�ޤ���


![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
�ᥤ��˥�����
--------------------------------------------------------------------------------

That's all - we have the stack and BSS, so we can jump to the `main()` C function:

���줬���Ƥ����桹�������å���BSS����Ĥ��ȡ������Ʋ桹��C�����`main()`�ؿ��˥����׽���ޤ���

```assembly
    calll main
```

The `main()` function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c). You can read about what this does in the next part.

`main()`�ؿ���[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/main.c)�ˤ��롣���ʤ��ϼ��ξϤǤ��줬���򤹤�Τ��ɤळ�Ȥ�����ޤ���

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about Linux kernel insides. If you have questions or suggestions, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com), or just create an [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part, we will see the first C code that executes in the Linux kernel setup, the implementation of memory routines such as `memset`, `memcpy`, `earlyprintk`, early console implementation and initialization, and much more.

�����Linux kernel insides�ˤĤ��Ƥκǽ�Υѡ��Ȥν����Ǥ����⤷��������������Τʤ�[0xAX](https://twitter.com/0xAX)��Ϣ���뤫��[email](anotherworldofworld@gmail.com)�Ǥ�����ĺ���뤫��[issue](https://github.com/0xAX/linux-internals/issues/new)��������Ƥ������������Υѡ��ȤǤϡ��桹��Linux�����ͥ륻�åȥ��åפǼ¹Ԥ����ǽ��C�����ɡ�`memset`, `memcpy`, `earlyprintk`�Τ褦�ʥ��꡼�롼����μ���������Υ��󥽡�������Ƚ�����ʤɤʤɤ��äѤ����򸫤�Ǥ��礦��

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

** �Ѹ�ϻ��������ǤϤ���ޤ��󡣤ޤ������ؤ򤪤������ޤ����⤷�ְ㤤�򸫤Ĥ����Τʤ�ɤ���[linux-insides](https://github.com/0xAX/linux-internals)�˥ץ�ꥯ���ꤲ�Ƥ��������� **

Links
���
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://en.wikipedia.org/wiki/Intel_8086)
  * [80386](http://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](http://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [CoreBoot developer manual](http://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](http://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](http://en.wikipedia.org/wiki/Power_good_signal)
