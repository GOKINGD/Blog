---
title: Linux源码解析--从main函数初始化到开中断
date: 2024-07-17 12:47:52
tags: [Linux,源码解析]
categories: Linux
---

# 前言
上文讲到了Linux系统启动前执行的三个汇编程序，head.s程序通过将main函数压栈再出栈跳转到main函数执行，此时真正进入由C语言编写的Linux源代码。上一篇文章可以点这里进行跳转[Linux源码解析--从开机到main函数](https://gokingd.github.io/2024/07/12/Linux%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%BB%8E%E5%BC%80%E6%9C%BA%E5%8A%A0%E7%94%B5%E5%88%B0main%E5%87%BD%E6%95%B0/)

# 介绍
本文基于Linux0.11源代码，分析main函数中前几个初始化步骤，直到main函数中打开中断，执行move_to_user_mode()，由内核特权级转为用户特权级。


# main函数
```c
	//init/main.c
	mem_init(main_memory_start,memory_end);
	trap_init();
	blk_dev_init();
	chr_dev_init();
	tty_init();
	time_init();
	sched_init();
	buffer_init(buffer_memory_end);
	hd_init();
	floppy_init();
	sti();
	move_to_user_mode();
```

main函数位于init/main.c
**在进入main函数后，执行mem_init()之前，系统首先对根设备号和硬盘参数表进行备份。**

```c
	ROOT_DEV = ORIG_ROOT_DEV;
 	drive_info = DRIVE_INFO; 
//#define DRIVE_INFO (*(struct drive_info *)0x90080)  复制0x90080处的硬盘参数表
```
为什么这里DRIVE_INFO要宏定义为0x90080呢，原因是这里是之前在setup.s中进行过设置，0x90080~ 0x9008f放了第一个硬盘的参数表，0x90090~ 0x9009f存放了第二个硬盘的参数表，不过在上一篇文章中并没有提到，感兴趣的话可以去看setup.s中从65行开始的汇编代码，这里就不贴出了。

**然后要对物理内存进行规划**。

```c
	//内存大小=1Mb字节+扩展内存*1024字节
	memory_end = (1<<20) + (EXT_MEM_K<<10);
	//#define EXT_MEM_K (*(unsigned short *)0x90002)
	memory_end &= 0xfffff000;  //让内存空间是4kb的倍数，一页4kb
	if (memory_end > 16*1024*1024)
		memory_end = 16*1024*1024;//内存空间最大16Mb
	if (memory_end > 12*1024*1024) 
		buffer_memory_end = 4*1024*1024;//设置内存缓冲区末端为4Mb
	else if (memory_end > 6*1024*1024)
		buffer_memory_end = 2*1024*1024;//设置内存缓冲区末端为2Mb
	else
		buffer_memory_end = 1*1024*1024; //设置内存缓冲区末端为1Mb
	main_memory_start = buffer_memory_end; //主内存起始地址=缓冲区末端地址
#ifdef RAMDISK
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
```
同样，这里将EXT_MEM_K宏定义为0x90002也是因为在setup中进行了设置，将扩展内存数值存在0x90002 处，同样感兴趣可以参考setup第43行代码，具体代码这里也不贴出了，只需要直到这里是扩招内存就行了。
注意到rd_init函数，是对虚拟盘做初始化，如果在makefile文件中进行设置使用虚拟盘，则会定义RAMDISK，那么就会执行rd_init函数，我们假设这里系统需要设置虚拟盘，并将虚拟盘的大小设置为2MB，那么系统将在主内存区处，即内存缓冲区末端为虚拟盘开辟2MB的内存空间。

```cpp
// kernal/blk_dev/ramdisk.c
main_memory_start += rd_init(main_memory_start, RAMDISK*1024);

#define MAJOR_NR 1
#define DEVICE_REQUEST do_rd_request

long rd_init(long mem_start, int length)
{
	int	i;
	char	*cp;

	blk_dev[MAJOR_NR].request_fn = DEVICE_REQUEST;
	rd_start = (char *) mem_start;
	rd_length = length;
	cp = rd_start;
	for (i=0; i < length; i++)
		*cp++ = '\0';
	return(length);
}
```
blk_dev是请求项函数控制结构, 是一个blk_dev_struct 类型的数组，结构体内有两个参数，一个是请求项操作的函数指针，一个是当前请求项的指针。宏定义MAJOR_NR为1是因为请求项函数控制结构中第2项即下标为1的那一项对应内存，因为虚拟盘是利用虚拟内存来模拟硬盘。同时初始化时挂接函数为do_rd_request 。

```c
// kernal/blk_dev/ll_rw_blk.c
#define NR_BLK_DEV	7

struct blk_dev_struct {
	void (*request_fn)(void);
	struct request * current_request;
};

struct blk_dev_struct blk_dev[NR_BLK_DEV] = {
	{ NULL, NULL },		/* no_dev */
	{ NULL, NULL },		/* dev mem */
	{ NULL, NULL },		/* dev fd */
	{ NULL, NULL },		/* dev hd */
	{ NULL, NULL },		/* dev ttyx */
	{ NULL, NULL },		/* dev tty */
	{ NULL, NULL }		/* dev lp */
};

```
请求项函数控制结构挂载好请求处理函数后，剩余的部分就是将虚拟盘的全部区域初始化为'\0' 。
**mem_init()**
```c
// mm/memory.c
#define USED 100
#define MAP_NR(addr) (((addr)-LOW_MEM)>>12)
void mem_init(long start_mem, long end_mem)
{
	int i;

	HIGH_MEMORY = end_mem;
	for (i=0 ; i<PAGING_PAGES ; i++)
		mem_map[i] = USED;
	i = MAP_NR(start_mem);
	end_mem -= start_mem;
	end_mem >>= 12;
	while (end_mem-->0)
		mem_map[i++]=0;
}
```
系统要对除了内核的1MB外的15MB空间进行分页管理，使用mem_map数组记录每一个页的使用次数。先把所有的页置为100，然后再根据主存的起始位置和终止位置把所有页全部清0.
```c
#define PAGING_MEMORY (15*1024*1024)
#define PAGING_PAGES (PAGING_MEMORY>>12)
static unsigned char mem_map [ PAGING_PAGES ] = {0,};
```
**trap_init()**
重建中断体系，挂载中断服务程序。设置中断服务程序的方法都一致，这里只贴出一个作为例子。

```c
// kernal/traps.c
#define set_trap_gate(n,addr) \
	_set_gate(&idt[n],15,0,addr)
	
void trap_init(void)
{
	...
	set_trap_gate(0,&divide_error);
	...
}

```
这里表示把divide_error函数的地址，即除0错误的中断服务程序的地址挂载到idt的第0项。
```c
#define _set_gate(gate_addr,type,dpl,addr) \
__asm__ ("movw %%dx,%%ax\n\t" \
	"movw %0,%%dx\n\t" \
	"movl %%eax,%1\n\t" \
	"movl %%edx,%2" \
	: \
	: "i" ((short) (0x8000+(dpl<<13)+(type<<8))), \
	"o" (*((char *) (gate_addr))), \
	"o" (*(4+(char *) (gate_addr))), \
	"d" ((char *) (addr)),"a" (0x00080000))
```
如何理解_set_gate函数宏展开的汇编代码呢？
%0、%1、%2、%3：0、1、2、3可以看作变量，这些变量在程序的":"之后,程序的两个":",是定义输入、输出项的。针对这段程序这些变量的前面都加了明确的限定，例如"i"(输入项)、"o"(输出项)，剩下的"d"(edx的初始值),"a"(eax的初始值)。而0、1、2、3的概念就是指第几个变量，这里输入项、输出向、寄存器初始混合编号；相应的0（"i"((short)(0x8000+(dpl<<13)+(type<<8))))）;1（(*((char *)(gate_addr)))）;2（(*(4+(char *)(gate_addr)))）;3（"d"((char *)(addr))）;4（"a"(0x00080000)），剩下就按照第一篇文章的重建idt理解即可。这还是再次贴出idt的图。

![alt text](/images/Linux2-image1.png)


如果还不理解，可以看这篇文章，讲的很详细 [_set_gate宏](https://www.docin.com/p-1533412800.html)
**blk_dev_init()**
初始化块设备请求项结构。

```c
struct request {
	int dev;		/* -1 if no request */
	int cmd;		/* READ or WRITE */
	int errors;
	unsigned long sector;
	unsigned long nr_sectors;
	char * buffer;
	struct task_struct * waiting;
	struct buffer_head * bh;
	struct request * next;
};

void blk_dev_init(void)
{
	int i;

	for (i=0 ; i<NR_REQUEST ; i++) {
		request[i].dev = -1;
		request[i].next = NULL;
	}
}

```
这里的工作比较简单，就是将所有的请求项都置为空闲项（dev=-1）
**tty_init();**
初始化外设，并将与外设相关的中断服务程序与idt进行挂接，这里不过多说明。
**time_init()**
设置开机时间，通过读取主板上的一块CMOS芯片,对时间数据进行采集。这里也不过多说明。
**sched_init()**
激活进程0。这是非常重要的一步。进程0的task_struct代码已经提前设计好了，但是要能运行进程0，还需要将进程0的管理结构中的数据结构与gdt挂接，并对gdt，进程槽，以及相关寄存器进行设置。

```c
// include/linux/head.h
typedef struct desc_struct {
	unsigned long a,b;
} desc_table[256];
extern desc_table idt,gdt;
// include/kernel/sched.c
void sched_init(void)
{
	int i;
	struct desc_struct * p;
	...
}
```
首先定义了段描述符表，起始就是结构体数组，共256项，每个描述符由8个字节构成。然后用描述符表定义了idt和gdt，所以idt和gdt就是一个256项的结构体数组。sched_init开始就定义了一个指向段描述符表的指针p，后面要用p来清空gdt。然后需要把tss和ldt挂接到gdt上，tss是任务状态描述符表，由tss_struct结构体组成，记录着当前进程的状态。
```c
// include/kernel/sched.c
#define FIRST_TSS_ENTRY 4
set_tss_desc(gdt+FIRST_TSS_ENTRY,&(init_task.task.tss));
set_ldt_desc(gdt+FIRST_LDT_ENTRY,&(init_task.task.ldt));

// include/asm/system.c
#define set_tss_desc(n,addr) _set_tssldt_desc(((char *) (n)),addr,"0x89")
#define set_ldt_desc(n,addr) _set_tssldt_desc(((char *) (n)),addr,"0x82")

#define _set_tssldt_desc(n,addr,type) \
__asm__ ("movw $104,%1\n\t" \
	"movw %%ax,%2\n\t" \
	"rorl $16,%%eax\n\t" \
	"movb %%al,%3\n\t" \
	"movb $" type ",%4\n\t" \
	"movb $0x00,%5\n\t" \
	"movb %%ah,%6\n\t" \
	"rorl $16,%%eax" \
	::"a" (addr), "m" (*(n)), "m" (*(n+2)), "m" (*(n+4)), \
	 "m" (*(n+5)), "m" (*(n+6)), "m" (*(n+7)) \
	)
```
这里的init_task是初始化宏定义好的内核task_union，可以看到task_union是一个task_struct 和 内核栈共用的联合体

```c
union task_union {
	struct task_struct task;
	char stack[PAGE_SIZE];
};
static union task_union init_task = {INIT_TASK,};
#define INIT_TASK \
/* state etc */	{ 0,15,15, \
/* signals */	0,{{},},0, \
/* ec,brk... */	0,0,0,0,0,0, \
/* pid etc.. */	0,-1,0,0,0, \
/* uid etc */	0,0,0,0,0,0, \
/* alarm */	0,0,0,0,0,0, \
/* math */	0, \
/* fs info */	-1,0022,NULL,NULL,NULL,0, \
/* filp */	{NULL,}, \
	{ \
		{0,0}, \
/* ldt */	{0x9f,0xc0fa00}, \
		{0x9f,0xc0f200}, \
	}, \
/*tss*/	{0,PAGE_SIZE+(long)&init_task,0x10,0,0,0,0,(long)&pg_dir,\
	 0,0,0,0,0,0,0,0, \
	 0,0,0x17,0x17,0x17,0x17,0x17,0x17, \
	 _LDT(0),0x80000000, \
		{} \
	}, \
}

```

set_tss_desc和set_ldt_desc都是宏函数，简单理解就是把tss挂接到gdt[FIRST_TSS_ENTRY]的位置，addr表示tss的地址。汇编代码的载入方式和挂载idt的方式类似。
值得注意的是这里FIRST_TSS_ENTRY宏定义为4，FIRST_LDT_ENTRY宏定义为5，这是因为，gdt的0项表示没有用，1项表示代码段，2项表示数据段，3项表示系统段，那么tss就要放到4项，ldt紧随其后放到5项，后面第6项放进程1的tss，以此类推。
接着让p指向gdt中进程0的ldt的后一项，然后把后面的所有项以及task数组全部清0。令p->a=p->b=0的原因是，p是一个指向段描述符的指针，其中只有两个元素，a和b。
```c
	p = gdt+2+FIRST_TSS_ENTRY;
	for(i=1;i<NR_TASKS;i++) {
		task[i] = NULL;
		p->a=p->b=0;
		p++;
		p->a=p->b=0;
		p++;
	}
```
然后将tss和ldt记录在对应的寄存器中
```c
	ltr(0);
	lldt(0);
```
然后设置时钟中断，这里分为三个步骤，分别是对8253定时器进行设置、对轮询相关的服务程序进行设置、以及打开8259A中与时钟中段相关的屏蔽码，这样就可以产生时钟中断了，这是后面进程轮询的基础。这里也不过多说明。
接着将系统调用处理函数set_system_gate与idt进行挂接。这里的步骤和之前的挂接中断服务程序的过程是一样的，不同的是优先级不同，这里是用户优先级3，这是因为这是给用户使用的系统调用软中断，用户进程想要和内核打交道，就要通过系统调用。
**buffer_init(buffer_memory_end)**

首先需要认识缓冲区，图片引用自赵炯的Linux内核完全剖析。高速缓冲区的起始位置从内核模块末端end标号开始，这体现在struct buffer_head * start_buffer = (struct buffer_head *) &end; end是内核模块链接期间由链接程序设置的一个值。
![alt text](/images/Linux2-iamge2.png)

首先，如果缓冲区高端等于1Mb，则由于从640Kb-1Mb被显存和BIOS占用，则实际可用缓冲区高端应该调整为640Kb，否则内存高端一定大于1MB。
```c
// fs/buffer.c
void buffer_init(long buffer_end)
{
	if (buffer_end == 1<<20)
		b = (void *) (640*1024);
	else
		b = (void *) buffer_end;
	...
}
```
接着就是对各个缓冲区的buffer_head进行初始化，设置各种标志位等为0，同时将buffer_head链接成一个双向环链表。整个高速缓冲区被划分为1024字节大小的缓冲块，与块设备上的磁盘逻辑块大小相同。缓冲区的低端设置缓冲头结构，链接高端的缓冲块。下面的图画的很好，引用自赵炯的Linux内核完全剖析
```c
while ( (b -= BLOCK_SIZE) >= ((void *) (h+1)) ) {
		h->b_dev = 0;
		h->b_dirt = 0;
		h->b_count = 0;
		h->b_lock = 0;
		h->b_uptodate = 0;
		h->b_wait = NULL;
		h->b_next = NULL;
		h->b_prev = NULL;
		h->b_data = (char *) b;
		h->b_prev_free = h-1;
		h->b_next_free = h+1;
		h++;
		NR_BUFFERS++;
		if (b == (void *) 0x100000)
			b = (void *) 0xA0000;
	}
	h--;
	free_list = start_buffer;
	free_list->b_prev_free = h;
	h->b_next_free = free_list;
```
![alt text](/images/Linux2-image3.png)
最后在将哈希表控制数组初始化为NULL。hash_table共307项。
```c
for (i=0;i<NR_HASH;i++)
		hash_table[i]=NULL;
```
**hd_init()**
硬盘初始化。
**floppy_init()**
软盘初始化。


最后再打开中断，然后由内核态切换到用户态。

```c
	sti();
	move_to_user_mode();
```

至此，Linux将进入最难理解的部分，进程0将fork进程1并切换到进程1执行，后面的部分将在另外一篇文章中说明。


有不对的地方欢迎批评指正！

# References
[1] 新设计团队. Linux内核设计的艺术[M]. 北京:机械工业出版社, 2014.
[2] 赵炯. Linux内核完全剖析[M]. 北京:机械工业出版社, 2008.