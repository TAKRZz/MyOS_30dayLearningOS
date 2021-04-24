# day04study

## 1.用C语言实现内存写入

naskfunc.nas

```
[SECTION .text]                   ; 
_write_mem8:	; void write_mem8(int addr, int data);
		MOV		ECX,[ESP+4]		; [ESP+4]中存放的是addr，将其读入ECX中
		MOV		AL,[ESP+8]		; [ESP+8]中存放的是data，将其读入AL
		MOV		[ECX],AL         ; 将AL 写入 [ECX]
		RET
```

参数指定的数字会被存放到内存中

- 第一个数字放在 [ESP + 4]

- 第二个数字放在 [ESP + 8]

与C语言联合使用 *能自由* 使用的寄存器只有EAX、ECX、EDX，至于其他寄存器只能使用读值而不能写值。

**EAX** Label

```
void io_hlt(void);
void write_mem8(int addr, int data);
void HariMain(void)
{
	int i; /* 変数宣言。iという変数は、32ビットの整数型 */
	for (i = 0xa0000; i <= 0xaffff; i++) {
		write_mem8(i, 15); /* MOV BYTE [i],15  将VRAM 全部写为 15*/ 
	}
	for (;;) {
		io_hlt();
	}
}
```

## 2.条纹图案

bootpack.c :

```
void HariMain(void)
{
	int i; /* 変数宣言。iという変数は、32ビットの整数型 */
	for (i = 0xa0000; i <= 0xaffff; i++) {
		write_mem8(i, i & 0x0f); //写入 i & 0x0f 与运算
	}
	for (;;) {
		io_hlt();
	}
}
```

![image-20210424121525460](C:\Users\TAKR0\AppData\Roaming\Typora\typora-user-images\image-20210424121525460.png)

## 3.挑战指针

C语言没有直接写入指定内存地址的语句，指针 fixed it。

```
*i = i & 0x0f； 			//会报类型错误
MOV [i], (i & 0x0f) 	 // 不知道 i 到底是 BYTE、WORD、DWOR
```

```c
char * p ; 		// 用于BYTE类地址 char 1 BYTE
short * p; 		// 用于WORD类地址 short 2 BYTE
int * p;   		// 用于DWORD类地址 int 4 BYTE
```

但 p 都是 1 BYTE， 因为 p 记录的是地址 Pointer 。

```c
void HariMain(void)
{
	int i; /* 変数宣言。iという変数は、32ビットの整数型 */
	char *p; /* pという変数は、BYTE [...]用の番地 */
	for (i = 0xa0000; i <= 0xaffff; i++) {
		p = i; /* 番地を代入  修改为 p = (char *) i ; */
		*p = i & 0x0f;
		/* これで write_mem8(i, i & 0x0f); の代わりになる */
	}
	for (;;) {
		io_hlt();
	}
}
```

```
bootpack.c:10: warning: assignment makes pointer from integer without a cast
```

```c
char * p ; 			// 声明的是变量 p 
```

## 4.指针的应用（1）

```c
void io_hlt(void);
void HariMain(void)
{
	int i; /* 変数宣言。iという変数は、32ビットの整数型 */
	char *p; /* pという変数は、BYTE [...]用の番地 */
	p = (char *) 0xa0000; /* 番地を代入 */
	for (i = 0; i <= 0xffff; i++) {
		*(p + i) = i & 0x0f; 		// *(p + i)
	}
	for (;;) {
		io_hlt();
	}
}
```

## 5.指针的应用（2）

```c
void io_hlt(void);
void HariMain(void)
{
	int i; /* 変数宣言。iという変数は、32ビットの整数型 */
	char *p; /* pという変数は、BYTE [...]用の番地 */
	p = (char *) 0xa0000; /* 番地を代入 */
	for (i = 0; i <= 0xffff; i++) {
		p[i] = i & 0x0f;			  // p[i] 与 i[p] 相同！
	}
	for (;;) {
		io_hlt();
	}
}

```

## 6.色号设定

处理颜色问题  使用320 * 200 的 8 位色彩模式，色号使用8位二进制数，也就是只能使用0~255的数。

```
OSAKA：
#000000 : 黑		 #00ffff : 浅亮蓝	#000084 : 暗蓝
#ff0000 : 亮红	#ffffff : 白		 #840084 : 暗紫
#00ff00 : 亮绿	#c6c6c6 : 亮灰	#008484 : 浅暗蓝
#ffff00 : 亮黄	#840000 : 暗红	#848484 : 暗灰
#0000ff : 亮蓝	#008400 : 暗绿	#ff00ff : 亮紫
#848400 : 暗黄
```

bootpack.c

```c
void io_hlt(void);
void io_cli(void);
void io_out8(int port, int data);
int io_load_eflags(void);
void io_store_eflags(int eflags);
/* 実は同じソースファイルに書いてあっても、定義する前に使うのなら、
	やっぱり宣言しておかないといけない。 */
void init_palette(void);
void set_palette(int start, int end, unsigned char *rgb);
void HariMain(void)
{
	int i; 					//声明变量 32位整型 i
	char *p; 				//声明变量 BYTE指针 p
	init_palette(); 		 //设定调色板
	p = (char *) 0xa0000;     //指定地址
	for (i = 0; i <= 0xffff; i++) {
		p[i] = i & 0x0f;
	}
	for (;;) {
		io_hlt();
	}
}
void init_palette(void)
{
	static unsigned char table_rgb[16 * 3] = {
		0x00, 0x00, 0x00,	/*  0:黒 */
		0xff, 0x00, 0x00,	/*  1:明るい赤 */
		0x00, 0xff, 0x00,	/*  2:明るい緑 */
		0xff, 0xff, 0x00,	/*  3:明るい黄色 */
		0x00, 0x00, 0xff,	/*  4:明るい青 */
		0xff, 0x00, 0xff,	/*  5:明るい紫 */
		0x00, 0xff, 0xff,	/*  6:明るい水色 */
		0xff, 0xff, 0xff,	/*  7:白 */
		0xc6, 0xc6, 0xc6,	/*  8:明るい灰色 */
		0x84, 0x00, 0x00,	/*  9:暗い赤 */
		0x00, 0x84, 0x00,	/* 10:暗い緑 */
		0x84, 0x84, 0x00,	/* 11:暗い黄色 */
		0x00, 0x00, 0x84,	/* 12:暗い青 */
		0x84, 0x00, 0x84,	/* 13:暗い紫 */
		0x00, 0x84, 0x84,	/* 14:暗い水色 */
		0x84, 0x84, 0x84	/* 15:暗い灰色 */
	};
	set_palette(0, 15, table_rgb);
	return;
	/* static char 命令は、データにしか使えないけどDB命令相当 */
}
void set_palette(int start, int end, unsigned char *rgb)
{
	int i, eflags;
	eflags = io_load_eflags();	 // 记录中断许可标志的值
	io_cli(); 				    // 将中断标志设置为 0， 禁止中断
	io_out8(0x03c8, start);
	for (i = start; i <= end; i++) {
		io_out8(0x03c9, rgb[0] / 4);
		io_out8(0x03c9, rgb[1] / 4);
		io_out8(0x03c9, rgb[2] / 4);
		rgb += 3; 				// 指针自增
	}
	io_store_eflags(eflags);	  //复原中断许可标志 
	return;
}
```

**io_out8** 向 port 8 输出数据，

将想要设定的调色板号码写入0x03c8，接着按R、G、B的顺序写入0x03c9

如果想要读出调色板的状态，首先将调色板的号码写入0x03c7，再从0x03c读取3次，读出的顺序也是R、G、B。

如果最初执行了**CLI**，最后要执行**STI**

**CLI** 将中断标志 *interrupt flag* 置为0的指令*clear interrupt flag*

**STI** 是要将中国中断标志置为1的指令 *set interrupt flag*

当CPU收到中断请求时，可以忽略请求（为0）。

**EFLAGS** 是extend FLAGS 16位 扩展得到的 32位寄存器，FLAGS是存储进位标志和中断标志等标志的寄存器。对于进位标志可以通过JC和JNC等跳转指令判断是否为0、1；但对于中断标志，只能通过读取EFLAGS检查第9位为0、1。

|  15  |  14  | 13   |  12  |  11  |  10  |  9   |  8   |  7   |  6   |  5   |  4   |  3   |  2   | 1    |  0   |
| :--: | :--: | ---- | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | ---- | :--: |
|      |  NT  |      | IOPL |  OF  |  DF  |  IF  |  TF  |  SF  |  ZF  |      |  AF  |      |  PF  |      |  CF  |

set_palette 想在设定调色版之前首先执行CLI，但处理结束后要恢复 *之前的* 中断标志，所以利用io_load_eflags() 读取最初的eflags值，之后利用io_store_eflags(eflags)恢复值。

naskfunc.nas

```
; naskfunc
; TAB=4
[FORMAT "WCOFF"]				; オブジェクトファイルを作るモード	
[INSTRSET "i486p"]				; 486の命令まで使いたいという記述
[BITS 32]						; 32ビットモード用の機械語を作らせる
[FILE "naskfunc.nas"]			; ソースファイル名情報
		GLOBAL	_io_hlt, _io_cli, _io_sti, _io_stihlt
		GLOBAL	_io_in8,  _io_in16,  _io_in32
		GLOBAL	_io_out8, _io_out16, _io_out32
		GLOBAL	_io_load_eflags, _io_store_eflags
[SECTION .text]
_io_hlt:	; void io_hlt(void);
		HLT
		RET
_io_cli:	; void io_cli(void);
		CLI
		RET
_io_sti:	; void io_sti(void);
		STI
		RET
_io_stihlt:	; void io_stihlt(void);
		STI
		HLT
		RET
_io_in8:	; int io_in8(int port);
		MOV		EDX,[ESP+4]		; port
		MOV		EAX,0
		IN		AL,DX
		RET
_io_in16:	; int io_in16(int port);
		MOV		EDX,[ESP+4]		; port
		MOV		EAX,0
		IN		AX,DX
		RET
_io_in32:	; int io_in32(int port);
		MOV		EDX,[ESP+4]		; port
		IN		EAX,DX
		RET
_io_out8:	; void io_out8(int port, int data);
		MOV		EDX,[ESP+4]		; port
		MOV		AL,[ESP+8]		; data
		OUT		DX,AL
		RET
_io_out16:	; void io_out16(int port, int data);
		MOV		EDX,[ESP+4]		; port
		MOV		EAX,[ESP+8]		; data
		OUT		DX,AX
		RET
_io_out32:	; void io_out32(int port, int data);
		MOV		EDX,[ESP+4]		; port
		MOV		EAX,[ESP+8]		; data
		OUT		DX,EAX
		RET
_io_load_eflags:	; int io_load_eflags(void);
		PUSHFD		; PUSH EFLAGS という意味
		POP		EAX  
		RET
_io_store_eflags:	; void io_store_eflags(int eflags);
		MOV		EAX,[ESP+4]
		PUSH	EAX
		POPFD		; POP EFLAGS という意味
		RET
```

CPU指令能够用于读写EFLAGS的只有PUSHFD和POPFD指令。

**PUSHFD** *push flags double-word* 将标志位的值按双字长压入栈；

**POPFD** *pop flags double-word* 按双字节长将标志位弹出栈。

## 7.绘制矩形

VRAM 与 画面上的“点”的关系：当前共有320 * 200 = 64000 个像素 左上角为(0, 0)，右下角为（319，199）（x，y）对应的VRAM地址为 0xa0000 + x + y * 320

往该“点”的内存中存入对应的颜色的代码，就会显示对应的颜色。

```c
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
void boxfill8(unsigned char *vram, int xsize, unsigned char c, int x0, int y0, int x1, int y1)
{
	int x, y;
	for (y = y0; y <= y1; y++) {
		for (x = x0; x <= x1; x++)
			vram[y * xsize + x] = c;
	}
	return;
}
```

将（x0，y0）到（x1，y1）的方块用颜色c填满。

## 8.今天的成果

bootpack.c :

```C
void HariMain(void)
{
	char *vram;
	int xsize, ysize;

	init_palette();
	vram = (char *) 0xa0000;
	xsize = 320;
	ysize = 200;

	boxfill8(vram, xsize, COL8_008484,  0,         0,          xsize -  1, ysize - 29);
    // 背景
	boxfill8(vram, xsize, COL8_C6C6C6,  0,         ysize - 28, xsize -  1, ysize - 28);
	boxfill8(vram, xsize, COL8_FFFFFF,  0,         ysize - 27, xsize -  1, ysize - 27);
	boxfill8(vram, xsize, COL8_C6C6C6,  0,         ysize - 26, xsize -  1, ysize -  1);
	boxfill8(vram, xsize, COL8_FFFFFF,  3,         ysize - 24, 59,         ysize - 24);
	boxfill8(vram, xsize, COL8_FFFFFF,  2,         ysize - 24,  2,         ysize -  4);
	boxfill8(vram, xsize, COL8_848484,  3,         ysize -  4, 59,         ysize -  4);
	boxfill8(vram, xsize, COL8_848484, 59,         ysize - 23, 59,         ysize -  5);
	boxfill8(vram, xsize, COL8_000000,  2,         ysize -  3, 59,         ysize -  3);
	boxfill8(vram, xsize, COL8_000000, 60,         ysize - 24, 60,         ysize -  3);
	boxfill8(vram, xsize, COL8_848484, xsize - 47, ysize - 24, xsize -  4, ysize - 24);
	boxfill8(vram, xsize, COL8_848484, xsize - 47, ysize - 23, xsize - 47, ysize -  4);
	boxfill8(vram, xsize, COL8_FFFFFF, xsize - 47, ysize -  3, xsize -  4, ysize -  3);
	boxfill8(vram, xsize, COL8_FFFFFF, xsize -  3, ysize - 24, xsize -  3, ysize -  3);
    //画线 灰线、白线、黑线……
	for (;;) {
		io_hlt();
	}
}
```

