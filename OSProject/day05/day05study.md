# day05study

# 结构化、文字显示与GDT/IDT初始化

## 1.接收启动信息

**binfo** *bootinfo*  **scrn** *screen*

```c
void HariMain(void)
{
	char *vram;
	int xsize, ysize;
	short *binfo_scrnx, *binfo_scrny;
	int *binfo_vram;
	init_palette();
	binfo_scrnx = (short *) 0x0ff4;
	binfo_scrny = (short *) 0x0ff6;
	binfo_vram = (int *) 0x0ff8;
	xsize = *binfo_scrnx;
	ysize = *binfo_scrny;
	vram = (char *) *binfo_vram;
	init_screen(vram, xsize, ysize);
	for (;;) {
		io_hlt();
	}
}
```

## 2.试用结构体

```c
struct BOOTINFO {
	char cyls, leds, vmode, reserve; // 1 BYPE / per 
	short scrnx, scrny;			    // 2 BYTE / per
	char *vram;					   // 4 BYTE 
};
void HariMain(void)
{
	char *vram;
	int xsize, ysize;
	struct BOOTINFO *binfo;
	init_palette();
	binfo = (struct BOOTINFO *) 0x0ff0;
	xsize = (*binfo).scrnx;
	ysize = (*binfo).scrny;
	vram = (*binfo).vram;
	init_screen(vram, xsize, ysize);
	for (;;) {
		io_hlt();
	}
}
```

## 3.试用箭头 ->

```c
void HariMain(void)
{
	struct BOOTINFO *binfo = (struct BOOTINFO *) 0x0ff0;
	init_palette();
	init_screen(binfo->vram, binfo->scrnx, binfo->scrny);
	for (;;) {
		io_hlt();
	}
}
```

## 4.显示字符

描画文字现状的数据 字体(*font*)数据 

```
static char font_A[16] = {	
	0x00, 0x18, 0x18, 0x18, 0x18, 0x24, 0x24, 0x24,
	0x24, 0x7e, 0x42, 0x42, 0x42, 0xe7, 0x00, 0x00
};
putfont8(binfo->vram, binfo->scrnx, 10, 10, COL8_FFFFFF, font_A);
	
	void putfont8(char *vram, int xsize, int x, int y, char c, char *font)
{
	int i;
	char *p, d /* data */;
	for (i = 0; i < 16; i++) {
		p = vram + (y + i) * xsize + x;
		d = font[i];
		if ((d & 0x80) != 0) { p[0] = c; }
		if ((d & 0x40) != 0) { p[1] = c; }
		if ((d & 0x20) != 0) { p[2] = c; }
		if ((d & 0x10) != 0) { p[3] = c; }
		if ((d & 0x08) != 0) { p[4] = c; }
		if ((d & 0x04) != 0) { p[5] = c; } //与运算，判断第5位是否为1
		if ((d & 0x02) != 0) { p[6] = c; } //是1的话填为c COL8_FFFFFF白色
		if ((d & 0x01) != 0) { p[7] = c; }
	}
	return;
}
```

![image-20210426121806040](C:\Users\TAKR0\AppData\Roaming\Typora\typora-user-images\image-20210426121806040.png)

## 5.增加字体

沿用OSASK的字体数据 hankaku.txt  编译器 makefont.exe ，可以将256个字符读进来，然后输出成16 * 256 = 4096。

A的字符编码是0x41，放在 hankaku + 0x41 * 16 开始的16字节中，C语言中的 ‘A’ 也是0x41 65 ASCII ， 所以可以写作 “ hankaku + ‘A' * 16 ”。

```c
void HariMain(void)
{
	struct BOOTINFO *binfo = (struct BOOTINFO *) 0x0ff0;
	extern char hankaku[4096];
	init_palette();
	init_screen(binfo->vram, binfo->scrnx, binfo->scrny);
	putfont8(binfo->vram, binfo->scrnx,  8, 8, COL8_FFFFFF, hankaku + 'A' * 16);
	putfont8(binfo->vram, binfo->scrnx, 16, 8, COL8_FFFFFF, hankaku + 'B' * 16);
	putfont8(binfo->vram, binfo->scrnx, 24, 8, COL8_FFFFFF, hankaku + 'C' * 16);
	putfont8(binfo->vram, binfo->scrnx, 40, 8, COL8_FFFFFF, hankaku + '1' * 16);
	putfont8(binfo->vram, binfo->scrnx, 48, 8, COL8_FFFFFF, hankaku + '2' * 16);
	putfont8(binfo->vram, binfo->scrnx, 56, 8, COL8_FFFFFF, hankaku + '3' * 16);
	for (;;) {
		io_hlt();
	}
}
```

## 6.显示字符串

```
void putfonts8_asc(char *vram, int xsize, int x, int y, char c, unsigned char *s)
{
	extern char hankaku[4096];
	for (; *s != 0x00; s++) {
		putfont8(vram, xsize, x, y, c, hankaku + *s * 16);
		x += 8;
	}
	return;
}
```

字符串最后有一个 \0 (0x00)

## 7.显示变量值

使用 sprintf 函数，与 printf 不同的是 sprintf 不是按照指定格式输出，而会将输出内容作为字符串写在内存中。GO的C编译器附带的函数，因为它只对 *内存* 进行操作，所以可以应用于所有操作系统。

```c
#include <stdio.h>
void HariMain(void)
{
	struct BOOTINFO *binfo = (struct BOOTINFO *) 0x0ff0;
	char s[40];
	init_palette();
	init_screen(binfo->vram, binfo->scrnx, binfo->scrny);
	putfonts8_asc(binfo->vram, binfo->scrnx,  8, 8, COL8_FFFFFF, "ABC 123");
	putfonts8_asc(binfo->vram, binfo->scrnx, 31, 31, COL8_000000, "Haribote OS.");
	putfonts8_asc(binfo->vram, binfo->scrnx, 30, 30, COL8_FFFFFF, "Haribote OS.");
	sprintf(s, "scrnx = %d", binfo->scrnx); 
	putfonts8_asc(binfo->vram, binfo->scrnx, 16, 64, COL8_FFFFFF, s);
	for (;;) {
		io_hlt();
	}
}
```

sprintf（地址，格式，值，值，值，……）地址即生成字符串的存放地址，后面的值为置换的变量。

|  %d  |   单纯的十进制数   | %5d  |        5位十进制数         |
| :--: | :----------------: | :--: | :------------------------: |
| %05d |  强制5位十进制数   |  %x  | 单纯的十六进制数，字母小写 |
|  %X  | 单纯十六，字母大写 | %5x  |       5位十六进制数        |
| %05x |   5位十六进制数    |      |                            |

![image-20210426140503899](C:\Users\TAKR0\AppData\Roaming\Typora\typora-user-images\image-20210426140503899.png)

## 8.显示鼠标指针

绘制鼠标指针，鼠标指针的大小为16 * 16，

```c
void init_mouse_cursor8(char *mouse, char bc)
/* マウスカーソルを準備（16x16） */
{
	static char cursor[16][16] = {
		"**************..",
		"*OOOOOOOOOOO*...",
		"*OOOOOOOOOO*....",
		"*OOOOOOOOO*.....",
		"*OOOOOOOO*......",
		"*OOOOOOO*.......",
		"*OOOOOOO*.......",
		"*OOOOOOOO*......",
		"*OOOO**OOO*.....",
		"*OOO*..*OOO*....",
		"*OO*....*OOO*...",
		"*O*......*OOO*..",
		"**........*OOO*.",
		"*..........*OOO*",
		"............*OO*",
		".............***"
	};
	int x, y;
	for (y = 0; y < 16; y++) {
		for (x = 0; x < 16; x++) {
			if (cursor[y][x] == '*') {  //外圈黑
				mouse[y * 16 + x] = COL8_000000;
			}
			if (cursor[y][x] == 'O') {  //内侧白
				mouse[y * 16 + x] = COL8_FFFFFF;
			}
			if (cursor[y][x] == '.') {
				mouse[y * 16 + x] = bc; //background color
			}
		}
	}
	return;
}
void HariMain(void)
{
	struct BOOTINFO *binfo = (struct BOOTINFO *) 0x0ff0;
	char s[40], mcursor[256];
	int mx, my;

	init_palette();
	init_screen8(binfo->vram, binfo->scrnx, binfo->scrny);
	mx = (binfo->scrnx - 16) / 2; /* 画面中央になるように座標計算 */
	my = (binfo->scrny - 28 - 16) / 2;
	init_mouse_cursor8(mcursor, COL8_008484);
	putblock8_8(binfo->vram, binfo->scrnx, 16, 16, mx, my, mcursor, 16);
	sprintf(s, "(%d, %d)", mx, my);
	putfonts8_asc(binfo->vram, binfo->scrnx, 0, 0, COL8_FFFFFF, s);
	for (;;) {
		io_hlt();
	}
}
void putblock8_8(char *vram, int vxsize, int pxsize,
	int pysize, int px0, int py0, char *buf, int bxsize)
{
	int x, y;
	for (y = 0; y < pysize; y++) {
		for (x = 0; x < pxsize; x++) {
			vram[(py0 + y) * vxsize + (px0 + x)] = buf[y * bxsize + x];
		}
	}
	return;
}
```

## 9.GDT与IDT的初始化

移动鼠标指针：GDT与IDT是和CPU有关的设定

**分段 Segmentation**  ORG指令明确程序要读入的内存地址。分段可以解决不同程序使用同一内存区域的问题，将内存分块 block ，每一块的起始地址都看作 0 。这样任何程序都可以以 ORG 0 开始。像这样分割出来的块，称为段 segment，**分页** 也可以解决问题。

MOV AL, [DS : EBX]  CPU会往EBX里加上某个值的16倍但不是DS的16倍，而是DS所表示的段的起始地址。即使省略段寄存器segment register的地址，也会自动任务是指定了DS

表示一个段：

+ 段的大小
+ 段的起始地址
+ 段的管理属性（禁止写入、禁止执行、系统专用等）

CPU用8 BYTE的数据表示这些信息，但是由于指定的寄存器只有16位，所以使用短号segment selector，存放在段寄存器里，然后预先设定好段号与段寄存器的对应关系。  段号可以使用 0 ~ 2^13 - 1 (8191) 因为CPU设计上的原因，段寄存器的低3位不能使用。

可以定义8192个段，所以设定这些段就需要 8192 * 8 = 65536  64KB的空间。CPU没有那么大的存储能力，所以这64KB的数据存在内存中并称为GDT。

**GDT** *global (segment) descriptor table* 全局段号记录表，这些数据整齐的排列在内存中，然后将内存的*起始地址*和*有效设计个数*放在CPU内被称作GDTR的特殊寄存器中。

**IDT** *interrupt descriptor table* 中断记录表，CPU会临时切换处理中断。

总结：要使用鼠标，就必须产生中断。IDT中记录了0~255的中断号码和调用函数的对应关系。

**段GDT的设定优于设定IDT**

```c
struct SEGMENT_DESCRIPTOR {     		// 存放GDT的8字节内容
	short limit_low, base_low;
	char base_mid, access_right;
	char limit_high, base_high;
};
struct GATE_DESCRIPTOR {				// 存放IDT的8字节内容
	short offset_low, selector;
	char dw_count, access_right;
	short offset_high;
};
void init_gdtidt(void)
{
	struct SEGMENT_DESCRIPTOR *gdt = (struct SEGMENT_DESCRIPTOR *) 0x00270000;  				//将0x270000~0x27ffff设为GDT 无意义
	struct GATE_DESCRIPTOR    *idt = (struct GATE_DESCRIPTOR    *) 0x0026f800;					//将0x26f800~0x26ffff设为IDT 
	int i;
	/* GDTの初期化 */
	for (i = 0; i < 8192; i++) {
		set_segmdesc(gdt + i, 0, 0, 0); // gdt + 8 全部置为0
	}
	set_segmdesc(gdt + 1, 0xffffffff, 0x00000000, 0x4092);
	set_segmdesc(gdt + 2, 0x0007ffff, 0x00280000, 0x409a);
    //段号为1的段，上限0xffffffff，2^32 4GB，地址是0，段的属性0x4092
    //段号为2的段，上限0x0007ffff，512KB ,地址是0x280000，正好为bootpack.hrb准备，因为bootpack.hrb是以ORG 0为前提翻译成的机器语言
	load_gdtr(0xffff, 0x00270000); 
    //C语言不能给GDTR赋值，所以要借助汇编语言
	/* IDTの初期化 */
	for (i = 0; i < 256; i++) {
		set_gatedesc(idt + i, 0, 0, 0);
	}
	load_idtr(0x7ff, 0x0026f800);
	return;
}
void set_segmdesc(struct SEGMENT_DESCRIPTOR *sd, unsigned int limit, int base, int ar)
{
	if (limit > 0xfffff) {
		ar |= 0x8000; /* G_bit = 1 */
		limit /= 0x1000;
	}
	sd->limit_low    = limit & 0xffff;//16bit
	sd->base_low     = base & 0xffff; //16bit
	sd->base_mid     = (base >> 16) & 0xff;//8bit
	sd->access_right = ar & 0xff;//8bit
	sd->limit_high   = ((limit >> 16) & 0x0f) | ((ar >> 8) & 0xf0);//8
	sd->base_high    = (base >> 24) & 0xff;//8bit
	return;
}
void set_gatedesc(struct GATE_DESCRIPTOR *gd, int offset, int selector, int ar)
{
	gd->offset_low   = offset & 0xffff;
	gd->selector     = selector;
	gd->dw_count     = (ar >> 8) & 0xff;
	gd->access_right = ar & 0xff;
	gd->offset_high  = (offset >> 16) & 0xffff;
	return;
}
```

day05finished！