---
title: Linux源码解析--从开机加电到main函数
date: 2024-07-12 22:06:07
tags: [Linux,源码解析]
categories: Linux
---

# 摘要
本文所参考的源码为linux0.11，对源码进行解析。
# 总体介绍
说明一下整体的思路。首先启动bios，bios在内存中建立中断向量表和中断服务程序。然后bios会发出0x19中断，将软盘中的第一扇区加载到内存中。第一扇区对应的是bootsect.s程序，此时处于实模式状态下，该程序的作用是将软盘中的后续扇区加载到内存中来，也就是setup.s和system模块。bootsect.s先规划内存，然后在把自己从0x07C00的位置移动到0x90000后bootsect执行0x13中断，加载setup程序。setup加载进入内存后开始加载第三部分代码，即system模块，system模块由head.s汇编好的目标代码和操作系统用C语言编写好的内核程序编译成的目标代码链接而成。bootsect.s跳转到setup程序的内存地址，开始执行setup程序。setup程序的主要目的是为了进入保护模式。setup程序首先关中断，然后将system模块移动到内存的最开始位置0x00000,覆盖掉原先存放在此处的bios中断向量表和bios数据区。setup对idtr和gdtr进行初始化，即建立gdt和idt。接着打开A20，cpu进入32位寻址，寻址空间变成4GB。setup为了建立保护模式下的中断机制，接着对可编程中断控制器8259A进行重新编程。setup程序设置CR0寄存器的第0位为1，cpu的工作方式设置为保护模式，再通过jmpi 0,8 跳转至内核代码段，开始执行head.s。head程序创建内核分页机制，废除了原来setup创建的gdt和idt，创建了新的gdt,同时构造了中断描述符表，最后将main函数入口地址压栈，ret指令执行main函数。

# bios启动
下面详细地逐步分析。

首先bios程序被固定在主板的一块很小的ROM芯片中。加电开始部件都进入16位实模式，CPU在加电瞬间CS置为0xFFFF,IP的值置为0x0000,这样CS:IP指向bios的入口地址0xFFFF0。bios启动之后会做检测显卡内存等一系列工作，然后在内存中建立中断向量表和中断服务程序。

随后，计算机硬件体系结构与bios联手，让CPU收到int 0x19中断，中断向量把CPU指向0x0E6F2，这是对应的中断服务程序的入口地址，中断服务程序将软盘中的第一扇区加载到内存中。第一扇区中的程序是由bootsect.s中的汇编代码汇编而成，bootsect会将软盘中的后续代码载入内存。

随后开始执行bootsect.s （boot/bootsect.s） 该汇编类似于intel汇编
```nasm
SYSSIZE = 0x3000    !给出system模块的大小 0x3000字节=192KB

.globl begtext, begdata, begbss, endtext, enddata, endbss !定义6个全局标识符
.text                                        !文本段
begtext:                                     
.data                                        !数据段
begdata:                                     
.bss                                         !未初始化数据段
begbss:                                        
.text                                        !文本段

SETUPLEN = 4					! nr of setup-sectors setup程序的扇区数
BOOTSEG  = 0x07c0			! original address of boot-sector bootsect的起始地址
INITSEG  = 0x9000			! we move boot here - out of the way  bootsect移动的目标地址
SETUPSEG = 0x9020			! setup starts here                  setup程序的起始地址
SYSSEG   = 0x1000			! system loaded at 0x10000 (65536).   system模块的起始加载地址
ENDSEG   = SYSSEG + SYSSIZE		! where to stop loading           system模块的终止地址
```
然后指定根文件系统设备时第2个硬盘的第一个分区
```nasm
ROOT_DEV = 0x306
```
在规划好内存后，bootsect要把自己从0x07C00移动到0x90000

因为8068/8088规定段寄存器不能直接赋值，必须要由寄存器给他传值，因此想初始化数据段首地址必须要由ax中转一下。
```nasm
entry start                            !从start标号开始
start:
	mov	ax,#BOOTSEG                    !代码段寄存器指向BOOTSEG
	mov	ds,ax
	mov	ax,#INITSEG                    !附加段寄存器指向INITSEG
	mov	es,ax
	mov	cx,#256
	sub	si,si                          !源地址ds:si = 0x07C0:0x0000
	sub	di,di                          !目的地址es:di = 0x9000:0x0000
	rep
	movw                              !每次移动一个字，一共移动256个字，这是bootsect的全部大小
	jmpi	go,INITSEG              !因为循环后已经移过去了，所以之间跳转到INITSEG为基址go段处
go:	mov	ax,cs                         !段对齐
	mov	ds,ax
	mov	es,ax
! put stack at 0x9ff00.
	mov	ss,ax                          ! 讲栈基址寄存器设置为CS相同的位置，同时讲栈顶指针指向 
                                         偏移地址0xFF00处
	mov	sp,#0xFF00		! arbitrary value >>512
```
注意，压栈是由高地址到低地址，故栈顶指针选择在高地址处。即此时，在内存中开辟了一个栈，起始地址为0x9FF00,距离setup程序（0x90200 + 515B*4 ）有很大一部分栈空间。
# bootsect程序
接着bootsect执行0x13中断服务程序，加载setup程序到内存。
```nasm
load_setup:
	mov	dx,#0x0000		! drive 0, head 0   前4行都是传参，确定setup模块在软盘中的位置
	mov	cx,#0x0002		! sector 2, track 0
	mov	bx,#0x0200		! address = 512, in INITSEG
	mov	ax,#0x0200+SETUPLEN	! service 2, nr of sectors
	int	0x13			! read it
	jnc	ok_load_setup		! ok - continue   ！执行成功了就跳转到ok_load_setup
	mov	dx,#0x0000
	mov	ax,#0x0000		! reset the diskette
	int	0x13
	j	load_setup
```
 中断int 0x13将setup模块从磁盘的第2个扇区读到0x90200处，共4个扇区，如果成功，就执行ok_load_setup，在ok_load_setup通过jmpi 0,SETUPSEG跳转到setup程序，如果失败，则复位寄存器，重新执行int 0x13中断读取setup。

ok_load_setup中代码较长，故略过一些对理解启动操作系统无关的代码。

下面的代码会在屏幕中显示'Loading system ...'
```nasm
! Print some inane message

	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#24
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg1
	mov	ax,#0x1301		! write string, move cursor
	int	0x10
```
然后调用read_it 读system模块
```nasm
    mov	ax,#SYSSEG   
	mov	es,ax		! segment of 0x010000
	call	read_it   !读磁盘上的system模块
	call	kill_motor  !关闭驱动器马达
```
最后加载一些程序后，执行
```nasm
jmpi	0,SETUPSEG
```
跳转到setup程序继续执行，至此bootsect程序全部执行完毕。事实上，bootsect一共有256行代码，这行是在135行，后面的一百多行代码都是一些子程序，故在此也不说明了。
# setup.s执行
随后是setup.s的执行（boot/setup.s）汇编方式同bootsect，类似于intel的x86汇编

setup.s开头的代码时加载一些机器系统的数据到内存的0x90000~0x901FD处，此时将bootsect程序给覆盖掉了。

接着setup关中断，直到main函数中才会打开中断。
```nasm
	cli			! no interrupts allowed !
```
接着，setup将原来加载在0x10000处的system模块(内核程序)移动到0x00000处
```nasm
	mov	ax,#0x0000
	cld			! 'direction'=0, movs moves forward
do_move:
	mov	es,ax		! destination segment  目的地址：ds:di (0x0000:0x0)
	add	ax,#0x1000
	cmp	ax,#0x9000
	jz	end_move
	mov	ds,ax		! source segment 源地址：ds:si (0x1000:0x0)
	sub	di,di
	sub	si,si
	mov 	cx,#0x8000   !移动0x8000字
	rep
	movsw        !移动一个字
	jmp	do_move
```
移动之后，内核程序将覆盖原先存放在0x00000处的由bios建立ide中断向量表和bios数据区，这样操作系统将不具备响应中断的能力，这也是为什么要关中断的原因。在移动system模块的时候，移动0x8000字，即从0x10000到0x90000,这是因为system模块最大长度不会超过0x80000(512k),故bootsect会把自己移动到0x90000，并加载setup到其后面。

接着setup要初始化GDT和IDT。简单说，GDT(全局描述符表)是系统中存放段描述符的数组。其中存放着每一个人物的局部描述符表LDT和任务状态段TSS。IDT(中断描述符表)为保护模式下所有中断程序的入口地址，类似于实模式下的中断向量表。
```nasm
end_move:
	mov	ax,#SETUPSEG	! right, forgot this at first. didn't work :-)
	mov	ds,ax           ! 先让ds指向setup段
	lidt	idt_48		! load idt with 0,0   !加载中断描述符表寄存器
	lgdt	gdt_48		! load gdt with whatever appropriate !加载全局描述符表寄存器

 ...(省略一部分)

gdt:
	.word	0,0,0,0		! dummy

	.word	0x07FF		! 8Mb - limit=2047 (2048*4096=8Mb)
	.word	0x0000		! base address=0
	.word	0x9A00		! code read/exec
	.word	0x00C0		! granularity=4096, 386 
 !上面是GDT的第二项，系统代码段

	.word	0x07FF		! 8Mb - limit=2047 (2048*4096=8Mb)
	.word	0x0000		! base address=0
	.word	0x9200		! data read/write
	.word	0x00C0		! granularity=4096, 386
 !上面是GDT的第三项，系统数据段

idt_48:
	.word	0			! idt limit=0
	.word	0,0			! idt base=0L

gdt_48:
	.word	0x800		! gdt limit=2048, 256 GDT entries
	.word	512+gdt,0x9	! gdt base = 0X9xxxx
```

对上面这段代码稍做解读。 首先lidt 和 lgdt 。 这里的l都是load的意思，表示加载，就是把gdt_48代表的6位操作数加载到寄存器中，即gdt的入口地址。那么转到后文去看idt_48段和gdt_48段。他们的格式统一，理解为前两个字节为段限长，后两个字节为段基址，段基址又由基址和偏移量组成。

0x800=8*16^2=2048=2k字节 GDT中8个字节组成一个段描述符项，所以GDT一共256项。      .word 512+gdt,0x9 其实就是 0x90200+gdt 就是setup段中的gdt段，gdt段就是前面的代码。在初始化gdt时，第一个项为空，第二项为系统代码段，第三项为系统数据段。

这里放张图更好理解
​​
![label](/images/Linux-image1.png)

 然后打开A20，CPU可以进行32位寻址，最大寻址空间为4GB，接着setup程序对可编程中断控制器8259A进行重新编程。这里的代码省略，不做过多讲解。

接着，setup程序将cr0寄存器的第0位（PE）置1，设定处理器的工作方式为保护模式。
```nasm
	mov	ax,#0x0001	! protected mode (PE) bit
	lmsw	ax		! This is it!
```
 最后，jmpi 0,8 进入system模块，执行head.s
```nasm
	jmpi	0,8		! jmp offset 0 of segment 8 (cs)
```
这里的0表示的是段内偏移地址，8指的是保护模式下的段选择符。这里的8不能看出十进制的8，而应该看成二进制的1000。 这里段选择符后两位表示特权级，如果是0表示内核特权级，3表示用户特权级，即二进制的11。第三位0表示GDT，1表示LDT。1表示GDT的1项，也就是第2项，就是之前设定的GDT中的内核代码段。这表示setup到此正式结束了，将要跳转到内核代码段system模块中的head.s程序继续执行。
# head.s
随后开始执行head.s(boot/head.s),这里head.s与前两个汇编程序的语法不同，这里采用的是AT&T的汇编格式。

head程序之所以叫head，是因为他位于system模块的最开始的地方，也是整个内存空间的起始位置处。head程序在内存中占有25KB+184B的空间。

首先先做段对齐，让DS、ES、FS、GS等寄存器指向内核代码段的起始地址。但是其实这里不能叫段对齐，此时已经进入保护模式，在实模式下CS是代码段基址，保护模式下CS是代码段选择符。所以下面代码中的0x10也应该看成二进制00010000，最后两位00表示内核特权级，第三位0表示是GDT，第4、第5位的10表示是GDT表的2项。
```nasm
_pg_dir:
startup_32:
	movl $0x10,%eax
	mov %ax,%ds
	mov %ax,%es
	mov %ax,%fs
	mov %ax,%gs
```
同样的思路，SS转变为栈段选择符，esp为栈顶指针。
```nasm
lss _stack_start,%esp
```
这里在汇编中的以下划线开头的变量是C语言中定义的变量。而stack_start是在kernal/shed.c中定义的一个长指针，指向user_stack.
```c
long user_stack [ PAGE_SIZE>>2 ] ;

struct {
	long * a;
	short b;
	} stack_start = { & user_stack [PAGE_SIZE>>2] , 0x10 };
```
接着调用setup_idt，重新建立idt中断描述符表。
```nasm
call setup_idt

setup_idt:
	lea ignore_int,%edx
	movl $0x00080000,%eax
	movw %dx,%ax		/* selector = 0x0008 = cs */
	movw $0x8E00,%dx	/* interrupt gate - dpl=0, present */

	lea _idt,%edi
	mov $256,%ecx
```
要理解上面这段代码首先要了解中断描述符的结构
![label](/images/Linux-image2.png)

 过程入口点偏移值就是中断服务程序的偏移地址。上面的32位要存入eax，下面的32位要存入edx。lea ignore_int，%edx 首先将ignore_int的偏移值存入edx，edx的低4字节是要放入eax的低4字节的，所以后面又movw %dx,%ax，不过这要放在后面执行，不然会被movl $0x00080000,%eax 覆盖。 然后movl $0x00080000,%eax 将0x00080000存入eax，再执行movw %dx,%ax 这样eax就做好了，最后再用0x8E00存入edx的低4字节，这样edx也做好了，整个中断描述符项就做好了。

然后就是循环256次，将中断描述符表中的每一项(共256项)全部指向ignore_int中断门，然后会使用lidt加载中断描述符寄存器。其中ecx可以看成计数器，循环256次。ignore_int 中断门没有实际作用，起到占位的作用，后续main函数中会重新挂载中断程序。
```nasm
rp_sidt:
	movl %eax,(%edi)
	movl %edx,4(%edi)
	addl $8,%edi
	dec %ecx
	jne rp_sidt
	lidt idt_descr
	ret
```
然后就是废除已有的GDT，建立新的GDT。为什么要建立新的GDT，而不用原来setup中设计的GDT呢？这是因为setup程序所在的内存区域在将来设置缓冲区的时候会被覆盖，所以必须要重新建立一套GDT。
```nasm
call setup_gdt

setup_gdt:
	lgdt gdt_descr   !加载全局描述符表寄存器
	ret

idt_descr:
	.word 256*8-1		# idt contains 256 entries  
	.long _idt          ! 6字节操作数，限长和基址

_gdt:	.quad 0x0000000000000000	/* NULL descriptor */
	.quad 0x00c09a0000000fff	/* 16Mb */
	.quad 0x00c0920000000fff	/* 16Mb */
	.quad 0x0000000000000000	/* TEMPORARY - don't use */
	.fill 252,8,0			/* space for LDT's and TSS's etc */

!前四项，空项、内核代码段、内核数据段、空项
!这里和前面setup设置GDT思路是一样的
!不同的是，段线长变成了16MB，扩大了一倍
接着重新做段对齐，就是让各个寄存器重新指向内核代码段，以及重新设置内核栈，因为修改了段限长

movl $0x10,%eax		# reload all the segment registers
	mov %ax,%ds		# after changing gdt. CS was already
	mov %ax,%es		# reloaded in 'setup_gdt'
	mov %ax,%fs
	mov %ax,%gs
    lss _stack_start,%esp
```
接着就是检测A20地址线是否开启。如果A20没有打开，计算机仍然处于实模式下，寻址范围只有1MB，往0x100000写值回回滚到0x00000。所以比较这两个地方写入的值是否一致来检测A20地址先是否打开。
```nasm
xorl %eax,%eax      ！首先异或清零
1:	incl %eax		# check that A20 really IS enabled
	movl %eax,0x000000	# loop forever if it isn't
	cmpl %eax,0x100000
	je 1b
```
然后检测x87协处理器是否打开，如果打开，设置为保护模式。这里不过多阐述。
# 内核分页并跳转main函数
然后head程序要进入最重要的两个步骤，内核分页以及转入main函数执行。head程序首先先将L6段和main函数压栈，这样内核分页执行完成后就可以直接出栈执行main函数。
```nasm
jmp after_page_tables
after_page_tables:
	pushl $0		# These are the parameters to main :-)
	pushl $0
	pushl $0
	pushl $L6		# return address for main, if it decides to.
	pushl $_main
	jmp setup_paging
L6:
	jmp L6			# main should never return here, but
				# just in case, we know what happens.
```
接着要进行内核分页。首先需要对5页内存进行清零，每页有1k个页表项，每项4字节，占4Kb，页目录表从0x000地址开始。
```nasm
setup_paging:
	movl $1024*5,%ecx		/* 5 pages - pg_dir+4 page tables */
	xorl %eax,%eax
	xorl %edi,%edi			/* pg_dir is at 0x000 */
	cld;rep;stosl
```
 接着设置页目录表，$pg1+7表示0x00001007，是页目录表的第一项，则第一个页表所在的地址为：0x00001007 & 0xffff000 = 0x1000,第一个页表的属性为：0x00001007 & 0x 00000fff = 0x07表示该页存在、用户可读可写。
```nasm
.org 0x1000
pg0:

.org 0x2000
pg1:

.org 0x3000
pg2:

.org 0x4000
pg3:

movl $pg0+7,_pg_dir		/* set present bit/user r/w */
	movl $pg1+7,_pg_dir+4		/*  --------- " " --------- */
	movl $pg2+7,_pg_dir+8		/*  --------- " " --------- */
	movl $pg3+7,_pg_dir+12		/*  --------- " " --------- */
```
 然后设置页表项，4个页表，一个页表1024个页表项 ，一共4096个页表项，恰好能够映射4096*4kb=16MB的物理内存。每个页表项的内容是当前项所映射的物理内存地址+该页的标志。从最后一个页表的最后一项开始倒序填写。最后一页的最后一项是4092，所以是$pg3+4092。
```nasm
movl $pg3+4092,%edi
	movl $0xfff007,%eax		/*  16Mb - 4096 + 7 (r/w user,p) */
	std
1:	stosl			/* fill pages backwards - more efficient :-) */
	subl $0x1000,%eax
	jge 1b
```
最后建立分页机制，设置页目录基址寄存器CR3，使之指向页目录表，再将CR0寄存的最高位置为1。因为CRO的最高位，即第32位，PG标志位，是分页机制控制位。当CR0中的PG标志置位时，CPU使用CR3指向的页目录和页表进行虚拟地址到物理地址的映射。
```nasm
xorl %eax,%eax		/* pg_dir is at 0x0000 */
	movl %eax,%cr3		/* cr3 - page directory start */
	movl %cr0,%eax
	orl $0x80000000,%eax
	movl %eax,%cr0		/* set paging (PG) bit */
```
最后，一句ret，跳入main函数执行。至此，汇编代码全部结束。

# 问题解答
下面回答一下这些问题：

1、bootsect、setup、head程序之间是怎么衔接的？给出代码证据。
```nasm
jmpi	0,SETUPSEG    ! bootsect跳转至setup

jmpi	0,8		! jmp offset 0 of segment 8 (cs)
                ! 跳转至内核代码段的初始位置，即head的入口地址
```
2、setup程序里的cli是为了什么？

cli关中断，为了防止在实模式的中断废除之后，保护模式的中断建立之前有中断影响。

3、setup程序的最后是jmpi 0,8 为什么这个8不能简单的当作阿拉伯数字8看待？

此时已经进入保护模式，这里的8是保护模式下的段选择符。8要看成二进制的1000，最后两位00表示内核特权级，第三位0表示GDT表，第四位1表示GDT中的第2项，即内核代码段。这样我们可以通用jmpi 0,8 跳转到内核代码段的起始地址执行head.s程序。

4、打开A20和打开pe究竟是什么关系，保护模式不就是32位的吗？为什么还要打开A20？有必要吗？

有必要。实模式下CPU的最大寻址为1MB，打开A20后CPU可以进行32位寻址，寻址空间扩展为4GB，而打开PE是打开保护模式，物理地址和线性地址一一对应。打开A20是打开PE的必要条件，而打开A20不一定需要打开PE。

5、Linux是用C语言写的，为什么没有从main还是开始，而是先运行3个汇编程序，道理何在？

普通的C语言编写的程序都是用户应用程序，运行在操作系统中计算机刚加电还没有加载操作系统程序，默认为16位实模式，与main函数执行需要的32位保护模式有一定的距离。需要借助bios加载三个汇编程序，然后利用这三个汇编程序完成内存规划、进入保护模式、建立IDT和GDT、设置分页机制等，然后才能在保护模式下调用main函数进行执行。

6、为什么不用call，而是用ret“调用”main函数？画出调用路线图，给出代码证据。

call指令会将EIP的值自动压栈，保护返回现场，然后执行被掉函数的程序。等到执行被调函数ret指令时，自动出栈给EIP并还原现场，继续执行call的下一条指令。但这对操作系统的main函数有点奇怪了，main函数是操作系统的，操作系统已经是最底层的系统，不需要再返回了。所以可以手动模拟call函数，先将main函数压栈，然后ret调用main函数，这样main函数就不需要ret了。

路线图参考P39 in [1] 新设计团队. Linux内核设计的艺术[M]. 北京:机械工业出版社, 2014.

代码：
```nasm
after_page_tables:
	pushl $0		# These are the parameters to main :-)
	pushl $0
	pushl $0
	pushl $L6		# return address for main, if it decides to.
	pushl $_main
	jmp setup_paging
L6:
	jmp L6			# main should never return here, but
				# just in case, we know what happens.


setup_paging:

	...

	ret			/* this also flushes prefetch-queue */
```
7、保护模式的“保护”体现在哪里？

打开保护模式后，CPU的寻址方式发生了变化，CPU寻址使用GDT获取代码段或者数据段的基址，保护模式通过段基址和段限长，防止了代码或者数据段的覆盖与代码段之间的访问越界等问题。同时保护模式引入了特权级的概念，用户特权级的代码无法访问内核特权级的代码和数据。

8、特权级的目的和意义是什么？为什么特权级是基于段的？

特权级是操作系统为了更好地管理内存空间及其访问控制而设的，提高了系统的安全性。

操作系统把内核设计为最高特权级，把用户进程设计为最低特权级。这样操作系统可以访问GDT、LDT，而GDT、LDT是逻辑地址形成线性地址的关键，因为操作系统可以控制线性地址。物理地址是由内核将线性地址转换而成的，所以操作系统可以访问任何物理地址，而用户进程只能使用逻辑地址。

在操作系统中，一般代码归为一个段，数据归为一个段，通过段选择符获取短的基址和特权级信息，特权级基于段，这样当段选择子具有不匹配的特权级时，按照特权级规则判断是否可以访问。特权级基于段可以有效禁止用户访问一些特殊指令。

9、在setup程序里曾经设置过一次gdt，为什么在head程序中将其废弃，又重新设置了一个？为什么折腾两次，而不是一次搞好？

原来的GDT所在的位置是在setup中设置的，将来setup模块的内存区域会在设计缓冲区时被覆盖，所以必须重新设计一套GDT。如果一次搞好，将setup中的GDT之间拷贝到head.s中的位置，那么后面移动system模块时会把他覆盖。如果先移动system模块，再复制GDT的内容，它又会把head.s对应的程序覆盖掉，而此时head.s还没有执行，所以必须要折腾两次。

10、用户进程自己设计一套LDT表，并与GDT挂接，是否可行，为什么？

显然不可以。GDT和LDT是CPU硬件认定的，这两个数据结构的首地址挂接在CPU的GDTR、LDTR两个寄存器上。同时，GDTR和LDTR的指令LGDT、LLDT只能在0特权级下运行，用户自己进程设计的LDT表，属于3特权级。

11、保护模式、分页下，线性地址到物理地址的转化过程是什么？

保护模式、分页下，线性地址通过MMU进行解析，以页目录表、页表、页面三级映射模式映射到物理地址。具体的转换过程为：每个线性地址长度为32位，MMU按照10-10-12的方式来识别地址值，分别解析为页目录号、页表项号、页面内偏移。CR3中存放着页目录表的基址，通过CR3找到页目录表，再找到页目录项，进而找到对应页表，寻找页表项，然后找到页面物理地址，最后加上12位页内偏移地址，形成最后的物理地址。

12、为什么开始启动计算机的时候，执行的是BIOS代码而不是操作系统自身的代码？

通常我们的用户程序是在操作系统上执行，操作系统会为应用程序创建进程并把应用程序的可执行代码加载到内存。但是在刚开始启动计算机时，操作系统还没有在内存中，需要先把操作系统加载到内存中，所以需要首先执行bios代码，来把操作系统加载到内存。

13、为什么BIOS只加载了一个扇区，后续扇区却是由bootsect代码加载？为什么BIOS没有直接把所有需要加载的扇区都加载？

对bios而言，约定在加电启动后，需要把扇区代码加载到0x7c00处，后续扇区则由bootsect代码加载，这些代码由编写系统的用户负责，与bios无关。这样构建的好处是统一设计和统一安排，简单有效。而且，如果需要使用bios全部加载完成后再执行，这样需要很长的时间，所以linux采用边执行边加载。

14、为什么BIOS把bootsect加载到0x07c00，而不是0x00000？加载后又马上挪到0x90000处，是何道理？为什么不一次加载到位？

因为历史原因，bios约定加载到0x07c00处。加载之后，依据系统对内存的规划，内核从0x0000开始，这样0x7c00后续可能会被覆盖，所以需要将其移动到0x90000。为什么不一次加载到位，这是因为历史遗留原因，根据约定，都需要初始加载到0x07c00。

有不对的地方欢迎批评指正！

# References
[1] 新设计团队. Linux内核设计的艺术[M]. 北京:机械工业出版社, 2014.
[2] 赵炯. Linux内核完全剖析[M]. 北京:机械工业出版社, 2008.

​