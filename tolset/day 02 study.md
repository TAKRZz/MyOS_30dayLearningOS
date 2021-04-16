# day 02 study

**ORG** *origin* 在开始执行的时候，将机器语言指令装载到内存中的地址 $ 含义改变，指代内存地址

**JMP** *jump* 类似于goto 跳转

**entry** 入口，指定JMP的跳转目的地 c++ FLAG

**MOV** *move* 赋值指令

AX —— accumulator 累加寄存器

CX —— counter		计数寄存器

DX —— data 			数据寄存器

BX —— base			基址寄存器

SP —— stack pointer 栈指针寄存器

BP ——  base pointer 基址指针寄存器

SI —— source index	源编制寄存器

DI —— destination index 目的变址寄存器

*以上均为16为寄存器*  **X** 表示 extend，以前是8位，现在扩展为16位

AL —— accumulator low 累加寄存器低位

CL —— counter low		计数寄存器低位

DL —— data low			数据寄存器低位

BL —— base low			基址寄存器低位

AH —— accumulator high 累加寄存器高位

CH —— counter high		计数寄存器高位

DH —— data high 			数据寄存器高位

BH —— base high			基址寄存器高位

*以上均为8位寄存器* ， AX = AL + AH， 即前8位后8位， CPU只能存储16字节  8 * 16

Extend EAX ECX EDX EBX ESP EBP ESI EDI 32位寄存器

**段寄存器**

ES —— extra segment 	附加段寄存器

CS —— code segment 	代码段寄存器

SS —— stack segment 	栈段寄存器

DS —— data segment	  数据段寄存器

FS —— segment part2	无名

GS —— segment part3	无名