# day 03 MyOS study 

## 1.IPL Initial Program Loader 启动程序装载器

~~~
MOV		AX,0x0820
		MOV		ES,AX
		MOV		CH,0			; シリンダ0
		MOV		DH,0			; ヘッド0
		MOV		CL,2			; セクタ2

		MOV		AH,0x02			; AH=0x02 : ディスク読み込み
		MOV		AL,1			; 1セクタ
		MOV		BX,0
		MOV		DL,0x00			; Aドライブ
		INT		0x13			; ディスクBIOS呼び出し
		JC		error
~~~

**JC** jump if carry(进位标志)  FLACS.CF  

INT 0x02——读盘  0x03——写盘 0x04——校验 0x0c——寻道  0x13——系统复位

CH  CL DH DL

软盘有80个柱面（半径），每个柱面有18个扇区（角度） sector

容量 80 * 2 * 18 * 512=1440 KB

BX表示从软盘读取数据存放在内存中的地址，但只能表示 0xffff 64k的数字， *后期*  EBX 寄存器扩展为32位，可以处理 4Gb 的内存。

段寄存器 segment register   起辅助作用，指定内存地址。

ES:BX 表示地址[ES:BX]表示 ES * 16 + BX 的地址 可以达到 1M 的地址。

0x8000~0x81ff 这**512**字节是留给启动区的。

DS：被省略，一般将DS置为0

## 2.试错

~~~
entry:
		MOV		AX,0			; レジスタ初期化
		MOV		SS,AX
		MOV		SP,0x7c00
		MOV		DS,AX

; ディスクを読む

		MOV		AX,0x0820
		MOV		ES,AX
		MOV		CH,0			; シリンダ0
		MOV		DH,0			; ヘッド0
		MOV		CL,2			; セクタ2

		MOV		SI,0			; 失敗回数を数えるレジスタ
retry:
		MOV		AH,0x02			; AH=0x02 : 读入磁盘
		MOV		AL,1			; 1个扇区
		MOV		BX,0
		MOV		DL,0x00			; A驱动器
		INT		0x13			; 调用磁盘BIOS
		JNC		fin				; 没出错的话跳转到fin
		ADD		SI,1			; 出错的话：SIに1を足す
		CMP		SI,5			; SIと5を比較
		JAE		error			; SI >= 5 だったらerrorへ
		MOV		AH,0x00
		MOV		DL,0x00			; Aドライブ
		INT		0x13			; ドライブのリセット
		JMP		retry
~~~

**JNC** *jump if not carry* 进位标志是0则跳转

**JAE** *jump if above or equal*  大于等于跳转

## 3.读到18扇区

~~~
MOV		AX,0x0820			 	; AX = 0x0820
		MOV		ES,AX			; ES = AX
		MOV		CH,0			; シリンダ0 柱面
		MOV		DH,0			; ヘッド0   磁头 Head
		MOV		CL,2			; セクタ2   扇区
readloop:
		MOV		SI,0			; 失败次数记录
retry:
		MOV		AH,0x02			; AH = 0x02 : 读入磁盘？
		MOV		AL,1			; AL = 1  1セクタ
		MOV		BX,0			; BX = 0
		MOV		DL,0x00			; DL = 0x00  A驱动器
		INT		0x13			; 调用磁盘BIOS
		//这是如何读取的？ 只是说调用BIOS就结束了？
		JNC		next			; 没出错时跳转到next
		ADD		SI,1			; SIに1を足す SI++
		CMP		SI,5			; SIと5を比較
		JAE		error			; SI >= 5 だったらerrorへ
		MOV		AH,0x00			
		MOV		DL,0x00			; Aドライブ
		INT		0x13			; ドライブのリセット
		JMP		retry
next:
		MOV		AX,ES			; AX = ES
		ADD		AX,0x0020        ; AX += 0x0020
		MOV		ES,AX			; ADD ES,0x020  ES = AX
		ADD		CL,1			; CLに1を足す  进入下一扇区
		CMP		CL,18			; CLと18を比較 扇区是否达到18
		JBE		readloop		; CL <= 18 だったらreadloopへ

~~~

**JBE** *jump if below or equal* 小于等于跳转

~~~
		MOV		AX,ES			
		ADD		AX,0x0020        ; アドレスを0x200進める
		MOV		ES,AX			; ADD ES,0x020 という命令がないのでこうしている
~~~

  *没有 ADD ES， 0x0020 指令*

**一个扇区一个扇区的读盘** 

和写作 AL = 17 结果一样，**原因后续解释** ！

C0-H0-S1  Cylinder Head Segment  CH DH CL

## 4.读入10个柱面

读完 C0-H0-S1 → C0-H0-S18 →**反面** C0-H1-S1 → C0-H1-S18 → C1-H0-S1 

```
CYLS	EQU		10				; どこまで読み込むか

; ディスクを読む

		MOV		AX,0x0820		; AX = 0x0820
		MOV		ES,AX			; ES = AX
		MOV		CH,0			; シリンダ0 柱面 0
		MOV		DH,0			; ヘッド0   磁头 0
		MOV		CL,2			; セクタ2   扇区 2
readloop:
		MOV		SI,0			; 失敗回数を数えるレジスタ
retry:
		MOV		AH,0x02			; AH = 0x02 : 读入磁盘
		MOV		AL,1			; AL = 1
		MOV		BX,0			; BX = 0
		MOV		DL,0x00			; DL = 0x00
		INT		0x13			; ディスクBIOS呼び出し
		JNC		next			; エラーがおきなければnextへ
		ADD		SI,1			; SIに1を足す
		CMP		SI,5			; SIと5を比較
		JAE		error			; SI >= 5 だったらerrorへ
		MOV		AH,0x00			; AH = 0x00
		MOV		DL,0x00			; DL = 0x00
		INT		0x13			; 重置驱动器?
		JMP		retry
next:
		MOV		AX,ES			; アドレスを0x200進める
		ADD		AX,0x0020
		MOV		ES,AX			; ADD ES,0x0020
		
		ADD		CL,1			; CLに1を足す 扇区++
		CMP		CL,18			; CLと18を比較 18
		JBE		readloop		; CL <= 18 だったらreadloopへ
		MOV		CL,1			; CL = 1 置为 1
		ADD		DH,1			; DH++  磁头换面 
		CMP		DH,2			
		JB		readloop		; DH < 2 だったらreadloopへ
		MOV		DH,0
		ADD		CH,1			; 柱面 ++
		CMP		CH,CYLS			; CYLS
		JB		readloop		; CH < CYLS だったらreadloopへ

```

**JB** *jump if below* 小于跳转 

```
CYLS	EQU		10				; どこまで読み込むか
```

类似于C语言的define， CYLS（cylinders） equal  10 

目前已经写入了10 * 2 * 18 * 512 = 180KB 。

## 5.着手开发操作系统

```
fin:
		HLT
		JMP		fin

```

![image-20210422163605788](C:\Users\TAKR0\AppData\Roaming\Typora\typora-user-images\image-20210422163605788.png)

![image-20210422163635468](C:\Users\TAKR0\AppData\Roaming\Typora\typora-user-images\image-20210422163635468.png)

![image-20210422163658546](C:\Users\TAKR0\AppData\Roaming\Typora\typora-user-images\image-20210422163658546.png)

向一个空软盘保存文件时，

1）文件名会写在0x002600以后的地方

2）文件的内容会写在0x004200以后的地方

## 6.从启动区执行操作系统

磁盘中的内容装载在内存的0x8000开始，所以磁盘的0x4200 即 相加得 0xc200 。

```
; haribote-os
; TAB=4
		ORG		0xc200
fin:
		HLT
		JMP		fin

```

## 7.确认操作系统的执行情况

haribote.nas :

```
; haribote-os
; TAB=4
		ORG		0xc200			; origin
		MOV		AL,0x13			; AL = 0x13 VGA显卡 320*200*8色
		MOV		AH,0x00			; AH = 0x00
		INT		0x10			; 0x10
fin:
		HLT
		JMP		fin
```

设置显卡模式（video mode）

AH = 0x00

AL = 模式：

​	0x03 ： 16色字符模式， 80 * 25

​	0x12 ： VGA图形模式， 640 * 480 * 4位彩色模式，独特的4面存储技术

​				*Video Graphics Array*

​	0x13 ： VGA图形模式， 320 * 200 * 8位彩色模式，调色板模式

​	0x6a ： 扩展VGA图形模式，800 * 600 * 4位彩色模式，独特的4面存储模式

返回值：无

make run会显示一片漆黑

## 8.32位（CPU）模式前期准备

C语言开发

haribote.nas :

```
; haribote-os
; TAB=4
; BOOT_INFO関係
CYLS	EQU		0x0ff0			; Cylinder启动区设定
LEDS	EQU		0x0ff1
VMODE	EQU		0x0ff2			; 颜色的位数
SCRNX	EQU		0x0ff4			; 分辨率的X
SCRNY	EQU		0x0ff6			; 分辨率的Y
VRAM	EQU		0x0ff8			; 图像缓冲区的开始地址

		ORG		0xc200			; このプログラムがどこに読み込まれるのか
		MOV		AL,0x13			; VGA模式 320x200x8bit
		MOV		AH,0x00
		INT		0x10			
		MOV		BYTE [VMODE],8	; 记录画面模式
		MOV		WORD [SCRNX],320
		MOV		WORD [SCRNY],200
		MOV		DWORD [VRAM],0x000a0000
; 利用BIOS取得键盘上各种LED指示灯的状态
		MOV		AH,0x02
		INT		0x16 			; keyboard BIOS
		MOV		[LEDS],AL
fin:
		HLT
		JMP		fin
```

从BIOS获取键盘状态 NumLock ON/OFF  

启动时的信息 BOOT_INFO

[VRAM] 保存的 0x000a0000 VRAM 指显卡内存 *Video RAM* 

在INT 0x10 画面模式下，VRAM是 0xa0000 ~ 0xaffff 的64KB

## 9.开始导入C语言

bootpack.c :

```c
void HariMain(void)
{
fin:
	/* ここにHLTを入れたいのだが、C言語ではHLTが使えない！ */
	goto fin;
}
```

cc1.exe .c .gas

gas2nask.exe .gas .nas

nask.exe .nas .obj

obi2bim.exe .obj .bim

bim2hrb.exe .bim .hrb

## 10.实现HLT

naskfunc.nas :

```
; naskfunc
; TAB=4
[FORMAT "WCOFF"]				; オブジェクトファイルを作るモード	
[BITS 32]						; 32ビットモード用の機械語を作らせる
; オブジェクトファイルのための情報
[FILE "naskfunc.nas"]			; ソースファイル名情報
		GLOBAL	_io_hlt			; このプログラムに含まれる関数名
; 以下は実際の関数
[SECTION .text]		; オブジェクトファイルではこれを書いてからプログラムを書く
_io_hlt:	; void io_hlt(void);
		HLT
		RET
```

用汇编语言写的全局函数 io_hlt （要在函数名前加_），后续与 .obj 链接，所以也需要编译成目标文件，所以将格式设置位**"WCOFF"**，另外，还要设置32位机器语言模式。

**RET** 相当于 C语言的 *return* 

bootpack.c : 

```
/* 他のファイルで作った関数がありますとCコンパイラに教える */
void io_hlt(void);
/* 関数宣言なのに、{}がなくていきなり;を書くと、
	他のファイルにあるからよろしくね、という意味になるのです。 */
void HariMain(void)
{
fin:
	io_hlt(); /* これでnaskfunc.nasの_io_hltが実行されます */
	goto fin;
}
```



加入了HLT语句后，CPU的占用明显下降。

day 03 Over！