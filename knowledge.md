# Binary's knowledge
## アセンブラの読み方 ##  
### 記法について ###
[参考-命令一覧-](http://softwaretechnique.jp/OS_Development/Tips/IA32_Instructions/A.html)
* intel記法  
こちらがよく見る記法  
この記法で確認したい場合は  
``$objdump -M intel -S <実行ファイル>``


* AT&T記法  
レジスタを参照する場合には、レジスタ名の前に「%」をプレフィクスとして付けなくてはいけない。  定数の場合には、プレフィクスとして「$」が必要となる。  
つまりこういうやつだ。  
``objdump``ではデフォルト表示でこの記法になる。
```
        pushl   %ebp
        movl    %esp, %ebp
        subl    $8, %esp
        andl    $-16, %esp
        movl    $0, %eax
        movl    %eax, -4(%ebp)
        movl    -4(%ebp), %eax
        call    __alloca
        call    ___main
        movl    $LC0, (%esp)
        call    _printf
        movl    $0, %eax
        leave
        ret
        .def    _printf;        .scl    2;      .type   32;     .endef
```
[参考](https://freestylewiki.xyz/fswiki/wiki.cgi?page=%E3%82%A2%E3%82%BB%E3%83%B3%E3%83%96%E3%83%A9%E5%85%A5%E9%96%80#p9)
```
AT&T:
アメリカの代表的通信業者由来
Intel:
CPUとかのIntel

一般的にはAT&T記法のほうがとっつきにくいので、Intel記法のほうがよく解説されるが、
GCCなどで標準的に使われるGAS( GNU Assembler )はAT&T記法
```
``call __x86.get_pc_thunk.ax ``

以下の様にmain()よりも上に出てくるこの文字の意味は32bit?

``` c
   0x0000052b <+14>:	push   ecx
   0x0000052c <+15>:	call   0x559 <__x86.get_pc_thunk.ax>
   0x00000531 <+20>:	add    eax,0x1aa7
```

~~これは、コンパイラがデフォルトで``position-independent``で実行可能なファイルを生成するため。~~

### Position-independent ###
本来、実行ファイルが置かれるアドレスは固定である。  
PIE(Position-Independent Executables)を利用することでランダムなアドレスに置くことが可能になる。


___
## セクションについて  ##
プログラムは複数のセクションから成る


#### 特に重要なセクション ####

セクション名|説明
:-:|:-
.text|機械語となる部分を格納。バイナリはここがメイン
.data|初期値を持つ変数を格納。
.bss|0で初期化される変数。ここに格納された値は全て0になる。
.rodata|定数(const)を格納する。ここは書き換えが起こらない
.plt|動的リンクされた関数。リンカにより生成される
.got.plt|動的リンクのアドレステーブル。ここの値をプログラム実行時にインタプリタが書き換える

上記より、`.data`セクションは`書き換わる事が前提` -> この辺はアドレスの書き換えが狙いやすい??  
逆に`.text`、`.rodata`のバイナリが書き変わることは考えにくい?

`.got`セクションはプログラムがキャッシュを作ることを目的に書き込み可能になっていることがある(`No RELRO`もしくは`Partial RELRO`の状態)GOT OVerwriteはこういった場合に成立する。

[pltはここを参照](https://tkmr.hatenablog.com/entry/2017/02/28/030528)  

`readelf -r got` や `objdump -d`で見れる



通常、PIC (位置独立コード) は共有ライブラリに利用されるが、Linux上でのGCC, Glibc および GNU Binutils を使うと、実行ファイルも位置独立にすることができます。  

## 実行ファイル内の情報を調べる ##
宣言された変数などは調べることが可能  
``` c
#include <stdio.h>

char *greeting;
int main(){
...
```
↓
``` sh
$readelf -s test|grep greeting
    54: 00002008     4 OBJECT  GLOBAL DEFAULT   23 greeting
```
`stringsで気になったwordをgrepしたら追加で情報が出るかも。`


* .data セクションの中身を調べる  

stringsで`cat flag.txt`の様な文字列を見つけたらここを参照すると文字列などがあるかも。  
system()が呼べる環境では利用できる。
```
objdump -M intel -s <file>
```
___
## gdb (gdb-peda)コマンド ##
* まずはgdb起動
```sh
$gdb
```
* 実行ファイルを読み込んで
``` sh
gdb-peda$ file runme
```

* ブレークポイントをさします
``` sh
gdb-peda$ b main
Breakpoint 1 at 0x80484ea
```

* レジスタのアドレス表示  
``` c
print/x $rax
```  

こういう書き方も可能  
``` c
gdb-peda$ x $ebp-0x10
0xbffff438:	0x00616161
＃ebpレジスタから0x10減じたアドレスの値を表示
```
* ブレークポイントの指定  
```
b main
```  

* 任意のメモリアドレスに値を格納する。  
``` c
set {<type>}<address> = <value>
例:
gdb-peda$ set {int}0xbffff438 = 0x00414141
#結果
gdb-peda$ x 0xbffff438
0xbffff438:	0x00414141
```

* ステップオーバー/ステップイン  

ステップオーバー: サブルーチンの中はデバッグしない
```
gdb$ ni
```

ステップイン: サブルーチンの中にも入ってデバッグする  
```
gdb$ si
```

* 値の調べ方
```
gdb-peda$ x $ebp
0xbffff428:	0xbffff438
```
こういう書き方も可能
```
(ebpレジスタから-0xc分のアドレス位置から 10Byte分の値を表示)
gdb-peda$ x/10x $ebp-0xc
0xbffff41c:	0x00000000	0x00000001	0x00000000	0xbffff438
0xbffff42c:	0x004005fa	0xb7fe79b0	0xbffff450	0x00000000
0xbffff43c:	0xb7df4e81	0xb7fb4000
```

* 型の調べ方  
``` bash
gdb-peda$ whatis 0xbffff438
type = unsigned int
gdb-peda$ ptype 0xbffff438
type = unsigned int
```

* 型について
``` sh
short        2バイトのデータ型(-32768 ~ 32767)
unsigned short  -の符号を持たないshort型
unsigned int -の符号を持たないint型(0 ~ 4294967295)
long         8バイト
unsigned long int -の符号を持たないlong型
```

* ファイル内で宣言されているグローバル変数のアドレスを調べる
``` bash
gdb-peda$ p &secret
$1 = (<data variable, no debug info> *) 0x402008 <secret>
```

* mainとかでも調べることが可能
```
```

* アドレスの中身を調べる
``` bash
gdb-peda$ x/s 0x8048841
0x8048841:	"/bin/cat flag.txt"
```
`x/<フォーマット> <アドレス|&変数>`  
`x/s`は文字列として表示

### goで書かれたファイルを見る場合 ###

``.gdbinit``に以下の行を追加する。
```
add-auto-load-safe-path /usr/local/go/src/runtime/runtime-gdb.py
```

* checksecもできる
```

```
___
## radare2の使い方 ##

プログラムの保護機能などを見たい(about checksec)
``` c
[0x00400650]> iI
arch     x86
~~
pic      false
relocs   true
relro    partial
rpath    NONE
```

### ASCIIコード  
0x41, 0x42などの表記で``文字``が扱われている場合、それらはASCIIコードである。
``` c
mov ch1, 0x41  # 0x41 は 'A'
mov ch2, 0x42  # 0x42 は 'B'
```

### スタックとレジスタについての理解 ###

#### 特殊レジスタ(32bit) #####
```
push   ebp
mov    ebp,esp
```
※64bitの場合は``ebp``→``rbp``, ``esp``→``rep``となる

#### 汎用レジスタ ####

[参考](http://herumi.in.coocan.jp/prog/x64.html)

x64では引数にレジスタが利用される。  
以下は呼び出される順番とその用途


レジスタ|用途|呼び出された側(callee)での保存の必要性
:-:|:-:|:-:
rax|戻り値/利用したSSEレジスタの数|不要
rdi|1番目の整数型引数|不要
rsi|2番目の整数型引数|不要
rdx|3番目の整数型引数|不要
rcx|4番目の整数型引数|不要
r8|5番目の整数型引数|不要
r9|6番目の整数型引数|不要
r10, r11|-|不要
r12～r15, rbx, rbp, rsp|-|変更するなら必要*
xm0～xm1|1番目の浮動小数型引数 / 戻り値|不要
xm2～xm7|n番目の浮動小数型引数|不要
xm8～xm15|テンポラリ|不要


### アセンブラ 命令について  
よく使われる記法  
* test
* je  
=>こいつらを合わせてif文の様に使う
```
(GAS記法)
test %rax, %rax
je .L0
```

``test %rax %rax``  
test命令は、2オペランドの論理積を計算し、結果がゼロならゼロフラグ(ZF)を立てます。
この例の場合はオペランドが両方同じものなので、raxが1なら結果は1、raxが0なら結果は0になります。  

`` je ラベル ``  -Jump if Equal-  
je命令は、ゼロフラグ(ZF)をチェックして、フラグが立っていればラベルへジャンプします。
注意点は、jeは testの意味や%%raxの内容などは感知していなくて、単にZFの値のみを見ているということです。 
### C言語からアセンブラ ### 
文字列の例(Intel記法)
``` c
char str[] = "aaa";
```

以下の命令は``ebp``から``-0x10減じた番地``に``0x616161``を``DWORD PTR(文字列型)``で格納する。という意味  
``` c
DWORD PTR [ebp-0x10],0x616161
``` 
なぜ減じたアドレス番地になるかというと、プラグラムが利用するスタック領域はメモリの概念では上にスタック（積む）されていき、``ebp``はプログラムが利用しているアドレスの開始番地を指す。 

|ebp-0x10|0x616161|←ebpから-0x10減じたアドレス番地に格納|
|--|--|:-|
|ebp|←プログラムが開始されるアドレス番地|

gdbでは以下の様に確認できる。
``` shell
gdb-peda$ x 0xbffff438
0xbffff438:	0x00616161
gdb-peda$ p $ebp
$9 = (void *) 0xbffff448
```

アドレスの概念
``` as
08048484 <fgets@plt>:
 8048484:	ff 25 e4 99 04 08    	jmp    DWORD PTR ds:0x80499e4
 804848a:	68 10 00 00 00       	push   0x10
 804848f:	e9 c0 ff ff ff       	jmp    8048454 <_init+0x30>
```
この`8048484`には`ff`が入る。`8048485`には`25`が格納されている。objdumpは人間の目に見やすく表示してくれているので  
`8048484:`から`jmp 0x80499e4`を実行した6byte分進んだ`804848a:`が次の行に表示される

___
## C言語について ##

### 覚えること ###
* **ポインタ型変数とは**  
メモリー上のアドレスを記憶する変数の型のこと  
以下の様に宣言する。 
``` c
int p      //int型
int* p     //int型へのポイント型
int *p
```
Usage:
``` c
int main(void)
{
    // int型へのポイント型変数
    int* p = NULL;
    // int型変数
    int i;
    i = 10;
    // iのアドレスを取り出してpに代入(&演算子を使ってiのアドレスを取り出している)
    p = &i;
    // pの値
    printf("p = %p\n",p);
    // iのアドレス
    printf("&i = %p\n",&i);
    return 0;
}
```
Result:
``` c
p = 0x7fff4670e844
&i = 0x7fff4670e844
```

### ltrace ###
``ltrace`` は共有ライブラリの関数呼び出しをトレースする Linux 用のツールです。システムコールをトレースするstrace と同様に、デバッグに大変役立ちます。

### strace ###
``strace`` システムコールをトレースする。

実行アドレスなどがわかる
```
$strace -i ./bof2
[b7fd6d09] execve("./bof2", ["./bof2"],
[b7ff12c7] brk(NULL) 
....
```
___
## 検証環境 ##
[参考](https://docs.oracle.com/cd/E39368_01/E36387/html/ol_aslr_sec.html)
* アドレス空間配置のランダム化  
次の命令のメモリー・アドレスを予測すること難しくなり、バッファオーバーフロー攻撃の防御に役立つ。  
Linuxでは以下の様にする。  
```
sudo sysctl -w kernel.randomize_va_space=0

```
```
$echo <value> > /proc/sys/kernel/randomize_va_space

Examle:
$echo 0 > /proc/sys/kernel/
```
|値|意味|
:-|:-:
0|ASLRを無効化 
1|スタック、仮想動的共有オブジェクト(VDSO)ページ、共有メモリー領域の位置をランダム化。
2|1 + データ・セグメントの位置をランダム化。 デフォルトの設定。  



値を完全に変更するには、次のように設定を/etc/sysctl.confに追加します。
```
[/etc/sysctl.conf]
kernel.randomize_va_space = 0

$sysctl -p
```
以下のコマンドで確認できる。
```
$sysctl  kernel.randomize_va_space
kernel.randomize_va_space = 0
```

また、実行ファイルコンパイル時は、セキュリティオプションを外す。
``` bash
$ gcc bof.c -fno-stack-protector 
(canaryを無効)
```
RELROの無効化
``` bash
$ gcc fsb.c -z norelro
```

### checksec.sh ###
``` bash
cd ~/Download/
$wget https://github.com/slimm609/checksec.sh/archive/1.6.tar.gz
$tar zxvf 1.6.tar.gz
$cp checksec.sh-1.6/checksec $HOME/bin/checksec.sh
```

___
## pwn手法 ##
## ret2libc ###

セキュリティコンテストチャレンジブックのbof3を使った  
buffer変数の領域を超えてスタックを破壊し、関数のリターンアドレスにコマンドを実行させる。


```
buffer
```
```
<buffer変数のスタック領域>
<関数のリターンアドレス> => $eip
```
この状態からbuffer変数を溢れさせる
↓

アドレス|入力値|改ざん後
:-:|:-|:-
buffer変数のスタック|AAAA|何かしらの処理aアドレスでsystem関数に呼ばれるbuffer変数
関数のリターンアドレス|0xb7e19200 (system)|system関数が呼ばれる
何かしらの処理a|BBBB(ダミー)|system関数のリターンアドレス
何かしらの処理b|0x0804a060(buffer変数のアドレス)|system関数の引数として渡される

↓
```
$echo -e '/bin/sh -c ls #AAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x00\x92\xe1\xb7BBBB\x60\xa0\x04\x08'|./bof3
```

### VM ###
Mintのネットワークを固定にするとwifiのみに接続した時、切れてしまう。

### ropgagdet ###
```
wget https://github.com/downloads/0vercl0k/rp/rp-lin-x86
```

### x64のエンディアン ###
* 64bit単位でエンディアンがかかる。  

```
0000000000401800 <printf@plt>:
```
↓
```
\x00\x18\x40\x00\x00\x00\x00\x00,
```
