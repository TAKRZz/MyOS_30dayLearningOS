

# 分割编译与中断处理

## 1.分割源文件

优点

+ 按照内容进行分类，容易找到地方
+ Makefile写的好，提高make速度
+ 多个小文件比大文件好处理

缺点

+ 源文件数量增加
+ 分类分的不好的话容易找不到文件

bootpack.c 划分位 graphic.c 图像处理  dsctbl.c 关于GDT、IDT的descriptor table 处理 bootpack.c 其他处理

## 2.整理Makefile

```
%.gas : %.c Makefile
	$(CC1) -o $*.gas $*.c
%.nas : %.gas Makefile
	$(GAS2NASK) $*.gas $*.nas
%.obj : %.nas Makefile
	$(NASK) $*.nas $*.obj $*.lst
```

make.exe先寻找普通的生成规则，再寻找一般规则。

## 3.整理头文件

1）将头文件都放在bootpack.h中，

```C
/* asmhead.nas */
struct BOOTINFO { /* 0x0ff0-0x0fff */
	char cyls; /* ブートセクタはどこまでディスクを読んだのか */
	char leds; /* ブート時のキーボードのLEDの状態 */
	char vmode; /* ビデオモード  何ビットカラーか */
	char reserve;
	short scrnx, scrny; /* 画面解像度 */
	char *vram;
};
#define ADR_BOOTINFO	0x00000ff0

/* naskfunc.nas */
void io_hlt(void);
void io_cli(void);
void io_out8(int port, int data);
int io_load_eflags(void);
void io_store_eflags(int eflags);
void load_gdtr(int limit, int addr);
void load_idtr(int limit, int addr);

/* graphic.c */
void init_palette(void);
void set_palette(int start, int end, unsigned char *rgb);
void boxfill8(unsigned char *vram, int xsize, unsigned char c, int x0, int y0, int x1, int y1);
void init_screen8(char *vram, int x, int y);
void putfont8(char *vram, int xsize, int x, int y, char c, char *font);
void putfonts8_asc(char *vram, int xsize, int x, int y, char c, unsigned char *s);
void init_mouse_cursor8(char *mouse, char bc);
void putblock8_8(char *vram, int vxsize, int pxsize,
	int pysize, int px0, int py0, char *buf, int bxsize);
#define COL8_000000		0
#define COL8_FF0000		1
#define COL8_00FF00		2
#define COL8_FFFF00		3
#define COL8_0000FF		4
#define COL8_FF00FF		5
#define COL8_00FFFF		6
#define COL8_FFFFFF		7
#define COL8_C6C6C6		8
#define COL8_840000		9
#define COL8_008400		10
#define COL8_848400		11
#define COL8_000084		12
#define COL8_840084		13
#define COL8_008484		14
#define COL8_848484		15

/* dsctbl.c */
struct SEGMENT_DESCRIPTOR {
	short limit_low, base_low;
	char base_mid, access_right;
	char limit_high, base_high;
};
struct GATE_DESCRIPTOR {
	short offset_low, selector;
	char dw_count, access_right;
	short offset_high;
};
void init_gdtidt(void);
void set_segmdesc(struct SEGMENT_DESCRIPTOR *sd, unsigned int limit, int base, int ar);
void set_gatedesc(struct GATE_DESCRIPTOR *gd, int offset, int selector, int ar);
#define ADR_IDT			0x0026f800
#define LIMIT_IDT		0x000007ff
#define ADR_GDT			0x00270000
#define LIMIT_GDT		0x0000ffff
#define ADR_BOTPAK		0x00280000
#define LIMIT_BOTPAK	0x0007ffff
#define AR_DATA32_RW	0x4092
#define AR_CODE32_ER	0x409a
```

.h文件中罗列出了函数的定义，还在注释中写明了函数的定义在哪一个源文件。

要在graphic.c中加上

```c
#include "bootpack.h"
```

头文件header  " " 表明该头文件和源文件位于同一个文件夹中，<>表示该头文件位于编译器提供的文件夹中。

## 4.意犹未尽

naskfunc.nas 的 _load_gdtr:

```
_load_gdtr:		; void load_gdtr(int limit, int addr);
		MOV		AX,[ESP+4]		; limit
		MOV		[ESP+6],AX
		LGDT	[ESP+6]
		RET

_load_idtr:		; void load_idtr(int limit, int addr);
		MOV		AX,[ESP+4]		; limit
		MOV		[ESP+6],AX
		LIDT	[ESP+6]
		RET
```

这个函数将指定的段上限 ( limit )和地址值赋值给名位**GDTR**的48位寄存器。但是它不能使用MOV指令来赋值。赋值的方法是：指定一个内存地址，从指定的地址读取6个字节，然后赋值给GDTR寄存器 —— LGDT。

该寄存器的低16位是段上限，它等于GDT的有效字节数 - 1，剩下的高32位，代表GDT的开始地址。

ADD AX，0x1234  编译    05 34 12，低位放在内存地址小的字节里。

dsctbl.c : 

```C
struct SEGMENT_DESCRIPTOR {
	short limit_low, base_low;
	char base_mid, access_right;
	char limit_high, base_high;
};
void set_segmdesc(struct SEGMENT_DESCRIPTOR *sd, unsigned int limit, int base, int ar)
{
	if (limit > 0xfffff) {
		ar |= 0x8000; /* G_bit = 1 */
		limit /= 0x1000;
	}
	sd->limit_low    = limit & 0xffff;
	sd->base_low     = base & 0xffff;
	sd->base_mid     = (base >> 16) & 0xff;
	sd->access_right = ar & 0xff;
	sd->limit_high   = ((limit >> 16) & 0x0f) | ((ar >> 8) & 0xf0);
	sd->base_high    = (base >> 24) & 0xff;
	return;
}
```

SEGMENT_DESCRIPTOR:

+ 段的大小   	   20b
+ 段的起始地址   4BYTE
+ 段的管理属性    12b

**段的地址base**分为3段，low、mid、high，合起来刚好32位，分为3段是为了与80286时代的程序兼容，多CPU可运行。

**段的上限limit**，32位可表示4GB，但这个数值本身需要4BYTE，加上base就已经8BYTE了，就会把整个结构体占满没有地方保存段的管理属性了。因此段的上限只能使用20位，这样段的上限最大也只能到1MB为止，Intel为了解决这个问题，在段的属性里加入一个标志位**Gbit**，这个标志位：

+ 是 1 的时候，limit的单位不解释为BYTE而是PAGE，在CPU中一页通常是4KB，这样4KB * 1M = 4GB   granularity

这20位的段上限分别写到 limit_low 16位 和 limit_high 的低4位中。

[^note]: 书中写的是上4位，我感觉看代码是低4位



**12位的段属性** ： 段属性又称：段的访问权属性，access_right，其中的高4位放在了limit_high的高4位中，所以程序将ar当作xxxx0000xxxxxxxx 16位结构来处理。

ar的高4位被称为”扩展访问权”，是一开始预留的。这四位由**GD00**构成，

+ Gbit
+ D指段的模式：1是指32位模式，0是指16位模式

ar的低8位：

+ 0x00 ： 未使用的记录表 descriptor table
+ 0x92 ： 系统专用，可读写的段。不可执行。
+ 0x9a ： 系统专用，可执行的段。可读不可写。
+ 0xf2 ： 应用程序用，可读写的段。不可执行。
+ 0xfa ： 应用程序用，可执行的段。可读不可写。

在32位模式下，CPU有系统模式 ring0，应用模式 ring3 之分 。操作系统“管理用”的程序和应用程序“被管理”的程序的运行时模式是不同的。例如应用程序在应用模式下试图调用LGDT等指令的话，CPU则对该指令不予执行，并会告知操作系统。另外，应用程序访问操作系统专用的段时，CPU也会中断执行向操作系统报告。

**//就像操作系统的那个多个环组成的圆一样，外层无法调用内层。**

## 5.初始化 PIC

为了实现鼠标移动，必须使用中断。

**PIC** *programmable interrupt controller* 可编程中断控制器

在设计上，CPU单独只能处理一个中断，所以在主板上有几个辅助芯片。

PIC是将8个中断信号集合成一个中断信号的装置，进来一个就将输出管脚信号变成ON。IBM的大叔认为电脑会有8个以上的外设，便将中断信号设置了15个，增设了2给PIC。

与CPU直接相连的PIC称为 主PIC master PIC，与主PIC相连的称为 从PIC slave PIC，主PIC处理0~7号信号，从PIC处理8~15号信号。并且，从PIC通过第2号IRQ与主PIC相连。

**IRQ**  *interrupt request*

int.c : 

```C
#include "bootpack.h"
void init_pic(void)
/* PIC的初始化 */
{
	io_out8(PIC0_IMR,  0xff  ); /* 禁止所有中断 */
	io_out8(PIC1_IMR,  0xff  ); /* 禁止所有中断 */
    // 配置PIC0
	io_out8(PIC0_ICW1, 0x11  ); /* 边沿触发模式 edge trigger mode */
	io_out8(PIC0_ICW2, 0x20  ); /* IRQ0~7由INT20~27接受 */
	io_out8(PIC0_ICW3, 1 << 2); /* PIC1由IRQ2连接 */
	io_out8(PIC0_ICW4, 0x01  ); /* 无缓冲区模式 */
    // 配置PIC1
	io_out8(PIC1_ICW1, 0x11  ); /* 边沿触发模式 */
	io_out8(PIC1_ICW2, 0x28  ); /* IRQ8-15由INT28-2f接受 */
	io_out8(PIC1_ICW3, 2     ); /* PIC1由IRQ2连接 */
	io_out8(PIC1_ICW4, 0x01  ); /* 五缓冲区模式 */
    
	io_out8(PIC0_IMR,  0xfb  ); /* 11111011 PIC1以外全部禁止 */
	io_out8(PIC1_IMR,  0xff  ); /* 11111111 禁止所有中断 */
	return;
}
```

PIC0 ： 主PIC    PIC1 ：从PIC

端口号码：

```c
#define PIC0_ICW1		0x0020   //00100000
#define PIC0_OCW2		0x0020   //00100000
#define PIC0_IMR		0x0021   //00100001
#define PIC0_ICW2		0x0021   //00100001
#define PIC0_ICW3		0x0021   //00100001
#define PIC0_ICW4		0x0021   //00100001
#define PIC1_ICW1		0x00a0   //10100000
#define PIC1_OCW2		0x00a0   //10100000
#define PIC1_IMR		0x00a1   //10100001
#define PIC1_ICW2		0x00a1   //10100001
#define PIC1_ICW3		0x00a1   //10100001
#define PIC1_ICW4		0x00a1   //10100001
```

PIC的寄存器都是8位寄存器；

+ **IMR** *interrupt mask register* 中断屏蔽寄存器  8位对应8路IRQ信号，如果某一位为1，则该为所对应的IRQ信号被忽视。可能正在对中断设定进行更改或没有连接设备。
+ **ICW** *initial control word* 初始化控制数据  ICW有4个，编号1~4。ICW1和ICW4与PIC主板配线方式、中断信号的电器特征等有关；ICW3是有关主-从连接的，对主PIC来说，每一个IRQ都能和从PIC连接，所以最多可以有64个IRQ8个PIC，但显示是只能设定为一个1；对从PIC来说，该从PIC与主PIC的第几号连接，用3位来设定，这些都是硬件上决定的不可更改；因此只有ICW2可以针对不同的操作系统进行设定，ICW2决定了IRQ以哪一号中断通知CPU。中断发生后，PIC发送2个字节的数据，发来的数据为 0xcd 0x？？，这在CPU看来和读内存的程序是完全一样的，所以CPU将它当作机器语言执行；0xcd是调用BIOS的INT指令，之前的 INT 0x10编译后就是0xcd 0x10，所以CPU将PIC的中断信号执行成了INT指令。

使用INT 0x20~0x2f接收中断信号IRQ0~15     不能使用INT0x00~0x1f在IRQ上，因为CPU内部产生指令会使用0x00~0x1f

## 6.中断处理程序的制作

鼠标是IRQ12，键盘是IRQ1

```c
struct BOOTINFO { /* 0x0ff0-0x0fff */
	char cyls; /* ブートセクタはどこまでディスクを読んだのか */
	char leds; /* ブート時のキーボードのLEDの状態 */
	char vmode; /* ビデオモード  何ビットカラーか */
	char reserve;
	short scrnx, scrny; /* 画面解像度 */
	char *vram;
};
void inthandler21(int *esp)
/* PS/2 键盘的中断 */
{
	struct BOOTINFO *binfo = (struct BOOTINFO *) ADR_BOOTINFO;
	boxfill8(binfo->vram, binfo->scrnx, COL8_000000, 0, 0, 32 * 8 - 1, 15); //方块打印
	putfonts8_asc(binfo->vram, binfo->scrnx, 0, 0, COL8_FFFFFF, "INT 21 (IRQ-1) : PS/2 keyboard");
	for (;;) {
		io_hlt();
	}
}
```

当按了键盘时！，在（0，0）到（255，15）打印了黑色的方块，然后在（0，0）位置输出了一行白色文字。

inthandler21接收esp指针的指，但是在函数中并没有使用esp的值。

```assembly
		EXTERN	_inthandler21, _inthandler27, _inthandler2c
_asm_inthandler21:
		PUSH	ES
		PUSH	DS
		PUSHAD
		MOV		EAX,ESP
		PUSH	EAX
		MOV		AX,SS
		MOV		DS,AX
		MOV		ES,AX
		CALL	_inthandler21
		POP		EAX
		POPAD
		POP		DS
		POP		ES
		IRETD
```

中断处理完成后，不能RET而要使用IRETD，这个指令不能使用C语言写，只能借助汇编。

[^1]: 在中断处理程序中无限循环，IRETD指令得不到执行，所以怎么都行，但为了后续的程序，不能用C语言写。

EXTERN在调用CALL的地方再进行说明。

栈*stack* 

**BUFFER**写程序的时候经常有一个需求，虽然不用永久记忆，但需要暂时记住某些东西以备后用，缓冲区*buffer*，收到一大堆信息，先存在缓冲区里，再慢慢处理，这就是缓冲区的意思。

+ **FIFO** *first in first out* 先进先出，*LILO* last in last out
+ **FILO** *first in last out* 先进后出，LIFO last in first out

栈STACK FILO PUSH POP

```assembly
PUSH EAX
;相当于
ADD ESP, -4
MOV [SS:ESP], EAX
```

```assembly
POP EAX
;相当于
MOV EAX, [SS:ESP]
ADD ESP, 4
```

```assembly
PUSHAD
;相当于
PUSH EAX
PUSH ECX
PUSH EDX
PUSH EBX
PUSH ESP
PUSH EBP
PUSH ESI
PUSH EDI
```

```assembly
POPAD
;相当于
POP EDI
POP ESI
POP EBP
POP ESP
POP EBX
POP EDX
POP ECX
POP EAX
```

所以这个函数，是将寄存器的值保存在栈中，然后将DS和ES调整到与SS相等，再调用_inthandler21，返回以后再将寄存器的值复原，最后执行IRETD。中断处理发生在函数处理的途中，所以一定要还原寄存器的值。

将SS赋值给DS和ES是因为在C语言中，它们指向同一个段，如果不这么做，inthandler21就不能顺利执行。

**CALL** 调用指令，可以调用函数，开头的EXTERN类似于include

dsctbl.c : 

```c
void init_gdtidt(void)
{
	struct SEGMENT_DESCRIPTOR *gdt = (struct SEGMENT_DESCRIPTOR *) ADR_GDT;
	struct GATE_DESCRIPTOR    *idt = (struct GATE_DESCRIPTOR    *) ADR_IDT;
	int i;

	/* GDTの初期化 */
	for (i = 0; i <= LIMIT_GDT / 8; i++) {
		set_segmdesc(gdt + i, 0, 0, 0);
	}
	set_segmdesc(gdt + 1, 0xffffffff,   0x00000000, AR_DATA32_RW);
	set_segmdesc(gdt + 2, LIMIT_BOTPAK, ADR_BOTPAK, AR_CODE32_ER);
	load_gdtr(LIMIT_GDT, ADR_GDT);

	/* IDTの初期化 */
	for (i = 0; i <= LIMIT_IDT / 8; i++) {
		set_gatedesc(idt + i, 0, 0, 0);
	}
	load_idtr(LIMIT_IDT, ADR_IDT);
			//新加入的，将asm_inthandler21注册到IDT中
    /* IDTの設定 */
	set_gatedesc(idt + 0x21, (int) asm_inthandler21, 2 * 8, AR_INTGATE32);
	set_gatedesc(idt + 0x27, (int) asm_inthandler27, 2 * 8, AR_INTGATE32);
	set_gatedesc(idt + 0x2c, (int) asm_inthandler2c, 2 * 8, AR_INTGATE32);

	return;
}
```

将asm_inthandler21放在 idt +0x21的位置，就可以自动调用函数。

2 * 8表示的是asm_inthandler21属于段号是2，* 8 是为了让低三位为0   2<<3

```C
	set_segmdesc(gdt + 2, LIMIT_BOTPAK, ADR_BOTPAK, AR_CODE32_ER);
```

说明号码位2的段覆盖了整个bootpack.h，AR_CODE32_ER将IDT的属性设置为0x008e  它表示这是用于中断处理的有效设定。

HariMain（）中io_sti()仅仅是执行STI指令，它是CLI的逆指令

**CLI** 将中断标志 *interrupt flag* 置为0的指令*clear interrupt flag*

**STI** 是要将中国中断标志置为1的指令 *set interrupt flag*

```assembly
_io_cli:	; void io_cli(void);
		CLI
		RET
_io_sti:	; void io_sti(void);
		STI
		RET
```

执行STI指令后，IF *interrupt flag* 置为1，CPU接受来自外部设备的中断。IF只有一个。

虽然写了鼠标的中断指令，但实际上无法触发中断。

day06finish

感觉每次学习效率都好低啊。