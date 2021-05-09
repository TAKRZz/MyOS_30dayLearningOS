# day07study

## 1.获取按键编码

让按键的编码显示出来 int.c  inthandler21

```C
#define PORT_KEYDAT		0x0060
void inthandler21(int *esp)
{
	struct BOOTINFO *binfo = (struct BOOTINFO *) ADR_BOOTINFO;
	unsigned char data, s[4];
	io_out8(PIC0_OCW2, 0x61);	/* 通知PIC “IRQ-01已经受理完毕” */
	data = io_in8(PORT_KEYDAT);
	sprintf(s, "%02X", data);
	boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
	putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
	return;
}
```

从编号 0x0060 的设备输入的8位信息是按键编码    0x60就是键盘

IRQ1 -- 0x61    IRQ3 -- 0x63

![image-20210506171811807](C:\Users\TAKR0\AppData\Roaming\Typora\typora-user-images\image-20210506171811807.png)

## 2.加快中断处理

目前按键的字符存在了中断程序的局部变量data中，另一方面字符显示需要大量的时间工作，会随时打断正在进行的进程。先将按键的编码接收下来，保存到变量中，然后偶尔查看这个变量。

in.c 

```C
#define PORT_KEYDAT		0x0060
struct KEYBUF {
	unsigned char data, flag;
};
struct KEYBUF keybuf;
void inthandler21(int *esp)
{
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61);	/* IRQ-01受付完了をPICに通知 */
	data = io_in8(PORT_KEYDAT);
	if (keybuf.flag == 0) {
		keybuf.data = data;
		keybuf.flag = 1;
	}
	return;
}
```

利用flag表示缓冲区是否为空（0），如果缓冲区有数据但是又来了新的，只能把新的忽略。

HariMain ： 

```C
void HariMain(void)
{
	struct BOOTINFO *binfo = (struct BOOTINFO *) ADR_BOOTINFO;
	char s[40], mcursor[256];
	int mx, my, i;
	init_gdtidt();
	init_pic();
	io_sti(); /* IDT/PICの初期化が終わったのでCPUの割り込み禁止を解除 */
	io_out8(PIC0_IMR, 0xf9); /* PIC1とキーボードを許可(11111001) */
	io_out8(PIC1_IMR, 0xef); /* マウスを許可(11101111) */
	init_palette();
	init_screen8(binfo->vram, binfo->scrnx, binfo->scrny);
	mx = (binfo->scrnx - 16) / 2; /* 画面中央になるように座標計算 */
	my = (binfo->scrny - 28 - 16) / 2;
	init_mouse_cursor8(mcursor, COL8_008484);
	putblock8_8(binfo->vram, binfo->scrnx, 16, 16, mx, my, mcursor, 16);
	sprintf(s, "(%d, %d)", mx, my);
	putfonts8_asc(binfo->vram, binfo->scrnx, 0, 0, COL8_FFFFFF, s);
	for (;;) {
		io_cli();
		if (keybuf.flag == 0) {
			io_stihlt();
		} else {
			i = keybuf.data;
			keybuf.flag = 0;
			io_sti();
			sprintf(s, "%02X", i);
			boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
			putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
		}
	}
}
```

首先 io_cli() 屏蔽中断，也要 hlt() 和 sti() ，

+ **CLI** 将中断标志 *interrupt flag* 置为0的指令*clear interrupt flag*
+ **STI** 是要将中国中断标志置为1的指令 *set interrupt flag*
+ **HLT** 让CPU待机 *halt* 

```assembly
_io_stihlt:	; void io_stihlt(void);
		STI
		HLT
		RET
```

当按下右CTRL键时，会产生两个字节的键码值“E0 1D”，松开产生“E0 9D”。键盘内部一次只能发送一个字节，所以每按就会产生**两次中断**，第一次E0，第二次1D。但在这个程序中忽略了第二次中断。

## 3.制作FIFO缓冲区

为了解决上面的问题，所以需要一个buffer。而且采用**FIFO**。

```C
struct KEYBUF {
	unsigned char data[32];
	int next;
};
struct KEYBUF keybuf;
void inthandler21(int *esp)
{
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61);	/* IRQ-01受付完了をPICに通知 */
	data = io_in8(PORT_KEYDAT);
	if (keybuf.next < 32) {
		keybuf.data[keybuf.next] = data;
		keybuf.next++;
	}
	return;
}
```

容量为32

```C
for (;;) {
		io_cli();
		if (keybuf.next == 0) {
			io_stihlt();
		} else {
			i = keybuf.data[0];
			keybuf.next--;
			for (j = 0; j < keybuf.next; j++) {  //每执行一个就把所有前移，效率较低。
				keybuf.data[j] = keybuf.data[j + 1];
			}
			io_sti();
			sprintf(s, "%02X", i);
			boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
			putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
		}
	}
```



## 4.改善FIFO缓冲区

将数组理解为环形数组，用双指针和一个记录长度。

```C
struct KEYBUF {
	unsigned char data[32];
	int next_r, next_w, len;
};
struct KEYBUF keybuf;
void inthandler21(int *esp)
{
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61);	/* IRQ-01受付完了をPICに通知 */
	data = io_in8(PORT_KEYDAT);
	if (keybuf.len < 32) {      // len = 32 就不动了，直接忽略。
		keybuf.data[keybuf.next_w] = data;
		keybuf.len++;
		keybuf.next_w++;
		if (keybuf.next_w == 32) {
			keybuf.next_w = 0;
		}
	}
	return;
}
```

减少了数据移动，加快了速度。

## 5.整理FIFO缓冲区

```C
struct FIFO8 {     //buffer 大小可变  size
	unsigned char *buf;
	int p, q, size, free, flags;
};
#define FLAGS_OVERRUN		0x0001
void fifo8_init(struct FIFO8 *fifo, int size, unsigned char *buf)
{
	fifo->size = size;
	fifo->buf = buf;
	fifo->free = size; 			/* 缓冲区大小*/
	fifo->flags = 0;
	fifo->p = 0; 			    /* 下一个数据写入位置 */
	fifo->q = 0; 			    /* 下一个数据读出位置 */
	return;
}
int fifo8_put(struct FIFO8 *fifo, unsigned char data)
/* 向FIFO传值 */
{
	if (fifo->free == 0) {
		/* 空きがなくてあふれた */
		fifo->flags |= FLAGS_OVERRUN; // 变成 1
		return -1;
	}
	fifo->buf[fifo->p] = data;
	fifo->p++;
	if (fifo->p == fifo->size) {
		fifo->p = 0;
	}
	fifo->free--;
	return 0;
}
int fifo8_get(struct FIFO8 *fifo)
/* FIFOからデータを一つとってくる */
{
	int data;
	if (fifo->free == fifo->size) { // 全空
		/* バッファが空っぽのときは、とりあえず-1が返される */
		return -1;    // 全空返回 -1
	}
	data = fifo->buf[fifo->q];
	fifo->q++;
	if (fifo->q == fifo->size) {
		fifo->q = 0;   /
	}
	fifo->free++;
	return data;
}
int fifo8_status(struct FIFO8 *fifo)
/* どのくらいデータが溜まっているかを報告する */
{
	return fifo->size - fifo->free;
}
```

int.c :

```C
struct FIFO8 keyfifo;
void inthandler21(int *esp)
{
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61);		/* 通知PIC0，说IRQ-01的受理已完成 */
	data = io_in8(PORT_KEYDAT);
	fifo8_put(&keyfifo, data);
	return;
}
```

HariMain() :

```C
	char s[40], mcursor[256], keybuf[32];
	fifo8_init(&keyfifo, 32, keybuf);
	for (;;) {
		io_cli();
		if (fifo8_status(&keyfifo) == 0) {
			io_stihlt();
		} else {
			i = fifo8_get(&keyfifo);
			io_sti();
			sprintf(s, "%02X", i);
			boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
			putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
		}
	}
```

## 6.鼠标

鼠标 IRQ12   键盘 IRQ1

鼠标的中断次数远大于键盘，IBM认为这不方便，于是有**激活鼠标**的指令。

bootpack.c :

```C
#define PORT_KEYDAT				0x0060
#define PORT_KEYSTA				0x0064
#define PORT_KEYCMD				0x0064
#define KEYSTA_SEND_NOTREADY	 0x02
#define KEYCMD_WRITE_MODE		 0x60
#define KBC_MODE				0x47
void wait_KBC_sendready(void)
{
	/* 等待键盘控制电路准备完毕 */
	for (;;) {
		if ((io_in8(PORT_KEYSTA) & KEYSTA_SEND_NOTREADY) == 0) {
			break;
    }}
	return;
}
void init_keyboard(void)     	//初始化键盘控制电路
{
	wait_KBC_sendready();
	io_out8(PORT_KEYCMD, KEYCMD_WRITE_MODE);
	wait_KBC_sendready();
	io_out8(PORT_KEYDAT, KBC_MODE);
	return;
}
```

wait_KBC_sendready() 让键盘控制电路( Keyboard controller, **KBC**) 做好准备。

如果KBC可以接受CPU指令了，CPU从设备号0x0064处所读取的数据的倒数第二位（0x02），应该是0。

init_keyboard() 一边确认KBC做好准备，一边发送模式设定指令（0x60），利用鼠标模式的号码是 0x47。

```C
#define KEYCMD_SENDTO_MOUSE		0xd4
#define MOUSECMD_ENABLE			0xf4
void enable_mouse(void)
{
	/* 激活鼠标 */
	wait_KBC_sendready();
	io_out8(PORT_KEYCMD, KEYCMD_SENDTO_MOUSE);  // 往KBC发送0xd4，下一个数据就会自动发给鼠标
	wait_KBC_sendready();
	io_out8(PORT_KEYDAT, MOUSECMD_ENABLE);  
	return; /* 顺利的话，键盘控制其会返送回ACK(0xfa) */
}
```

## 7.从鼠标接受数据

int.c :

```C
struct FIFO8 mousefifo;
void inthandler2c(int *esp)		/* 来自PS/2鼠标的中断 */
{
	unsigned char data;
	io_out8(PIC1_OCW2, 0x64);	/* 通知PIC1，IRQ-12的受理已完成 */
	io_out8(PIC0_OCW2, 0x62);	/* 通知PIC0，IRQ-02的受理已完成*/
	data = io_in8(PORT_KEYDAT);
	fifo8_put(&mousefifo, data);
	return;
}
//   键盘   作对比
struct FIFO8 keyfifo;
void inthandler21(int *esp)
{
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61);		/* 通知PIC0，说IRQ-01的受理已完成 */
	data = io_in8(PORT_KEYDAT);
	fifo8_put(&keyfifo, data);
	return;
}
```

IRQ-12是**从PIC**的第4号（从PIC相当于从 IRQ - 08 到 IRQ - 15）。首先要通知 IRQ - 12受理完成，再通知**主PIC**。这是因为主/从PIC的协调不能够自动完成，需要程序教主PIC如何处理，否则会忽视从PIC的下一个interrupt。

bootpack.c : 

```C
fifo8_init(&mousefifo, 128, mousebuf);  //鼠标的缓冲区更大 128BYTE
for (;;) {
		io_cli();
		if (fifo8_status(&keyfifo) + fifo8_status(&mousefifo) == 0) {
			io_stihlt();
		} else {
			if (fifo8_status(&keyfifo) != 0) {  //先处理键盘再处理鼠标
				i = fifo8_get(&keyfifo);
				io_sti();
				sprintf(s, "%02X", i);
				boxfill8(binfo->vram, binfo->scrnx, COL8_008484,  0, 16, 15, 31);
				putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
			} else if (fifo8_status(&mousefifo) != 0) {
				i = fifo8_get(&mousefifo);
				io_sti();
				sprintf(s, "%02X", i);     		//输出后移了32个像素
				boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 32, 16, 47, 31);
				putfonts8_asc(binfo->vram, binfo->scrnx, 32, 16, COL8_FFFFFF, s);
			}
		}
	}
```

day06 finish  TAKR    