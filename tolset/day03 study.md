# day 03 MyOS study TAKR



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

INT 0x02——读盘  0x03——写盘 0x04——校验 0x0c——寻道  

CH  CL DH DL

软盘有80个柱面（半径），每个柱面有18个扇区（角度） sector

容量 80 * 2 * 18 * 512=1440 KB

BX表示从软盘读取数据存放在内存中的地址，但只能表示 0xffff 64k的数字， （后期） EBX 寄存器扩展为32位，可以处理 4Gb 的内存。

段寄存器 segment register   起辅助作用，指定内存地址。

ES:BX 表示地址[ES:BX]表示 ES * 16 + BX 的地址 可以达到 1M 的地址。

0x8000~0x81ff 这**512**字节是留给启动区的。

DS：被省略，一般将DS置为0

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
		MOV		AH,0x02			; AH=0x02 : ディスク読み込み
		MOV		AL,1			; 1セクタ
		MOV		BX,0
		MOV		DL,0x00			; Aドライブ
		INT		0x13			; ディスクBIOS呼び出し
		JNC		fin				; エラーがおきなければfinへ
		ADD		SI,1			; SIに1を足す
		CMP		SI,5			; SIと5を比較
		JAE		error			; SI >= 5 だったらerrorへ
		MOV		AH,0x00
		MOV		DL,0x00			; Aドライブ
		INT		0x13			; ドライブのリセット
		JMP		retry
~~~

**JNC** *jump if not carry* 

JAE