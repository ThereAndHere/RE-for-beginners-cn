#第三章 Hello,world!

让我们来看一下《The C programming Language》书中的著名例子：

```cpp
#include <stdio.h>
int main()
{
	printf("hello, world\n");
	return 0;
}
```

##3.1 x86

###3.1.1 MSVC

让我们用 MSVC 2010编译：

```
cl 1.cpp /Fa 1.asm
```

（/Fa 选项是编译器生成汇编清单文件）

```cpp
CONST SEGMENT
$SG3830 DB 'hello, world', 0AH, 00H
CONST ENDS
PUBLIC _main
EXTRN _printf:PROC
; Function compile flags: /Odtp
_TEXT SEGMENT
_main PROC
    push ebp
    mov ebp, esp
    push OFFSET $SG3830
    call _printf
    add esp, 4
    xor eax, eax
    pop ebp
    ret 0
_main ENDP
_TEXT ENDS
```

MSVC生成的汇编使用Intel语法。Intel语法和AT&T语法的区别以后再讨论。

编译器生成1.obj文件，用来之后链接到1.exe。在我们的例子中，该文件包含两个段：CONST（存放常数）和 _TEXT（存放代码）。

字符串“hello, world”在C/C++里的类型是const char[]，它没有自己的名字。编译器需要总要以某种方法处理该字符串，所以给它定义了内部名字 $SG3830。

这就是为什么该例子可以被改写成这样：

```cpp
#include <stdio.h>
const char $SG3830[]="hello, world\n";
int main()
{
    printf($SG3830);
    return 0;
}
```

让我们回到汇编清单。我们可以看到，该字符串以0字节结尾，是一个标准的C/C++字符串。

在代码段_TEXT中，目前只有一个函数：main()。main()函数以序言代码开始，以结尾代码结束（像其他大部分函数一样）。

在函数序言之后我们可以看到调用了printf()函数：`CALL _printf`。在函数调用之前，包含我们问候的字符串地址通过PUSH指令被放在了栈上。

当printf()函数返回到main()函数，字符串的地址依然还在栈上。当我们不再需要它时，栈指针（ESP 寄存器）需要被校正。

`ADD ESP, 4` 意思是ESP寄存器的值加4。为什么是4？因为这是一个32位程序，地址长度为4字节，我们正好需要4个字节来恢复堆栈。如果是x64的代码，我们需要8字节。`ADD ESP, 4`和`POP register`的效果相同但不需要额外的寄存器。

一些编译器（比如Intel C\++ 编译器）会使用`POP ECX`而不是`ADD`。这个指令几乎有着同样的效果除了ECX寄存器会被覆盖。Intel C\++ 编译器使用`POP ECX`可能是因为POP指令比`ADD ESP`指令短。（POP 1字节，ADD 3字节）。

这是使用 `POP` 替代 `ADD` 的例子，来自Oracle RDBMS:

```
.text:0800029A         push ebx
.text:0800029B         call qksfroChild
.text:080002A0         pop ecx
```

在printf()之后，原始的C/C\++ 代码包含了语句 `return 0` ——返回0作为main()的结果。在生成的代码里这同通过指令`XOR EAX, EAX`实现，`XOR`实际是“异或”指令，但编译器经常用它来代替`MOV EAX, 0`，这同样是因为它是一个比较短的指令（XOR 2字节，MOV 5字节）。

一些编译器使用 `SUB EAX, EAX`，意思是用EAX中的值减去EAX中的值，同样会得到结果0。

最后一个指令`RET`返回到调用者中，通常是C/C\++ CRT（C Runtime Library）代码，然后进一步返回到操作体统的领空。

###3.1.2 GCC

现在我们在Linux下使用GCC编译器编译相同的C/C\++代码：`gcc 1.c -o 1`，然后再IDA的帮助下看看main()函数是怎样生成的。IDA和MSVC一样使用Intel语法。

```
main proc near
var_10 = dword ptr -10h
	push ebp
	mov ebp, esp
	and esp, 0FFFFFFF0h
	sub esp, 10h
	mov eax, offset aHelloWorld ; "hello, world\n"
    mov [esp+10h+var_10], eax
	call _printf
	mov eax, 0
	leave
	retn
main endp
```
结果几乎是相同的。字符串“hello, world”的地址先被加载到EAX寄存器中然后保存到栈顶。此外，函数序言包含了`AND ESP, 0FFFFFFF0h`，这个指令将ESP寄存器的值对齐到16字节边界。这使栈中的值都以相同的方式对齐（使用4字节或16字节对齐的值使CPU的性能更好）。

`SUB ESP, 10h`在栈上分配了16字节。然而，我们在之后会看到只有4字节是有用的。这是因为在栈上分配的大小也是16字节对齐的。

字符串地址被直接放在了栈上而不使用PUSH指令。var_10是一个局部变量也是printf()的参数。

接着printf()函数被调用。

不同于MSVC，当GCC编译时优化选项没开时，它使用`MOV EAX, 0`而不是更短的操作码。

最后一条指令`LEAVE`，等同于`MOV ESP, EBP`和`POP EBP`。换句话说，该指令恢复了栈寄存器(ESP)，并把EBP寄存器恢复到初始状态，由于我们在函数开始时修改了这些寄存器（ESP和EBP），所以这是很有必要的。

###3.1.3 GCC:AT&T 语法

让我们看看AT&T语法汇编语言是怎样的，这种语法在UNIX世界里更加流行。

```
gcc -S 1_1.c
```

我们可以得到

```
	.file "1_1.c"
	.section .rodata
.LC0:
	.string "hello, world\n
    .text
    .globl main
    .type main, @function
main:
    .LFB0:
    .cfi_startproc
    pushl %ebp
    .cfi_def_cfa_offset 8
    .cfi_offset 5, -8
    movl %esp, %ebp
    .cfi_def_cfa_register 5
    andl $-16, %esp
    subl $16, %esp
    movl $.LC0, (%esp)
    call printf
    movl $0, %eax
    leave
    .cfi_restore 5
    .cfi_def_cfa 4, 4
    ret
    .cfi_endproc
.LFE0:
    .size main, .-main
    .ident "GCC: (Ubuntu/Linaro 4.7.3-1ubuntu1) 4.7.3"
    .section .note.GNU-stack,"",@progbits
```

这份清单包含许多宏（以点开头）。现在他们并不是我们要关注的。为了简单起见，我们可以忽略他们（除了 .string宏，它们表示了一个null结尾的字符串就像C字符串一样），我们就能看到：

```
.LC0:
	.string "hello, world\n"
main:
    pushl %ebp
    movl %esp, %ebp
    andl $-16, %esp
    subl $16, %esp
    movl $.LC0, (%esp)
    call printf
    movl $0, %eax
    leave
    ret
```

Intel和AT&T语法的一些主要区别在于：

* 源和目的操作数的位置相反。  
Intel语法：<指令> <目的操作数> <源操作数>  
AT&T 语法：<指令> <源操作数> <目的操作数>  
有一个简单的方法可以记住这个区别：当你使用Intel语法时，你可以想象两个操作数之间有一个等号(=)，当你使用AT&T语法时则可以想象成右箭头(→)。  
* AT&T：在寄存器名字前必须有百分号(%)，数字前有美元符号($)。圆括号代替了方括号。
* AT&T：指令要添加后缀表示操作数的大小：  

> – q — quad (64 bits)  
>– l — long (32 bits)  
>– w — word (16 bits)  
>– b — byte (8 bits)  

让我们回到编译后的结果：和我们在IDA中看到的相同。只有一个微妙的区别：0FFFFFFF00h被写成了$-16，他们是相同的：10进制的16在16进制中为0x10。-0x10等于0xFFFFFFF0（对于32位数据类型来说）。

还有：返回值通过MOV被置0,而不是使用XOR。MOV只是将值加载到寄存器，他的名字是误称（数据不是被移动而是被复制）。在其他的架构中，这个指令被命名为“LOAD”或“STORE”或其他类似的名字。

##3.2 x86-64

###3.2.1 MSVC-x86-64

让我们尝试一下64位的MSVC

```
$SG2989 DB 'hello, world', 0AH, 00H

main PROC
    sub rsp, 40
    lea rcx, OFFSET FLAT:$SG2989
    call printf
    xor eax, eax
    add rsp, 40
    ret 0
main ENDP
```

在x86-64中，所有寄存器被扩展到64位，他们的名字都使用R-前缀。为了更少地使用栈（或者说更少地访问外部存储/缓存），有一个流行的方法来通过寄存器传递参数。也就是说，一部分函数参数通过寄存器传递，剩下的才使用栈。在Win64中，4个函数参数使用RCX, RDX, R8, R9寄存器传递。我们可以看到，printf()中指向字符串的指针不用栈传递而是使用RCX寄存器。

现在指针变为64位了，所以他们通过64位寄存器（R-前缀）传递。但是，为了向后兼容，同样还是可以访问32位的部分，使用E-前缀。

这就是x86-64中的 RAX/EAX/AX/AL:

main()函数返回一个整数类型的值(int)，在C/C++中，为了保持向后兼容和可移植性，依然还是32位，所以这就是为什么EAX在函数最后被清空而不是RAX。

本地栈上还有40字节被分配，这叫做 "shadow space"，这将在后面讨论。

###3.2.2 GCC-x86-64

我们再试下64位Linux中的GCC

```
.string "hello, world\n"
main:
    sub rsp, 8
    mov edi, OFFSET FLAT:.LC0 ; "hello, world\n"
    xor eax, eax ; number of vector registers passed
    call printf
    xor eax, eax
    add rsp, 8
    ret
```

使用寄存器传递参数的方法同样应用在Linux, BSD和Mac OS X中。前6个参数通过RDI, RSI, RDX, RCX, R8, R9传递，其他则通过栈传递。

所以指向字符串的指针在EDI（寄存器的32位部分）中传递。但为什么不使用64位部分RDI呢?

有很重要的一点需要记住，在64位模式中，所有的MOV指令会写入寄存器的低32位，同时也会清空高32位。也就是说，`MOV EAX, 011223344h`会正确地写入RAX寄存器，因为高位会被清空。

如果我们打开已编译的对象文件(.o)，我们也可以看到所有指令的操作码：

```
.text:00000000004004D0 					main proc near
.text:00000000004004D0 48 83 EC 08 		sub rsp, 8
.text:00000000004004D4 BF E8 05 40 00 	mov edi, offset format ; "hello, world\n"
.text:00000000004004D9 31 C0 			xor eax, eax
.text:00000000004004DB E8 D8 FE FF FF 	call _printf
.text:00000000004004E0 31 C0 			xor eax, eax
.text:00000000004004E2 48 83 C4 08 		add rsp, 8
.text:00000000004004E6 C3 				retn
.text:00000000004004E6 					main endp
```

我们可以看到，在0x4004D4上的指令写入EDI占用5字节，同样的指令写入64位的值到RDI占用7字节。显然，GCC会尝试节省空间，此外，它可以确信包含字符串的数据段不会被分配到4GiB以上。

我们还可以看到EAX寄存器在调用printf()函数前被清空。这是因为使用向量寄存器的数量会通过EAX传递。

##3.3 GCC-还有一件事

匿名的C字符串有着const属性，而且分配在常数短的C字符串是保证不可变的，所以有一个有趣的结果：编译器会使用字符串的特定部分。

我们试一下下面这个例子：

```cpp
#include <stdio.h>

int f1()
{
	printf ("world\n");
}

int f2()
{
	printf ("hello world\n");
}

int main()
{
	f1();
	f2();
}
```

一般的C/C++编译器（包括MSVC）分配两个字符串，但我们来看看GCC 4.8.1是怎么做的：

```
f1 			proc near

s 			= dword ptr -1Ch
			sub esp, 1Ch
			mov [esp+1Ch+s], offset s ; "world\n"
            call _puts
            add esp, 1Ch
			retn
f1 			endp

f2 			proc near

s 			= dword ptr -1Ch
            sub esp, 1Ch
            mov [esp+1Ch+s], offset aHello ; "hello "
            call _puts
            add esp, 1Ch
            retn
f2 			endp

aHello 		db 'hello '
s 			db 'world',0xa,0
```

实际上：当我们打印字符串“hello world”，这两个单词放在内存相邻的位置，f2()函数调用的puts()并不在意字符串是分开的。实际上它并不是分开的，只是“无形中”分开了。

当f1()调用puts()，它使用了“world”字符串加零字节。puts()并不在意字符串之前还有什么。

这个聪明的技巧在最新的GCC中经常被使用并且能节省一些内存。


##3.4 ARM

为了在ARM处理器上实验，需要用到几个编译器：
* 在嵌入式领域著名的 Keil Release 6/2013
* 苹果Xcode 4.6.3 IDE(LLVM-GCC 4.2 编译器)
* GCC 4.9 (Lianro)(for ARM64)

如果没有特别的说明，本书所有的例子都使用32位ARM代码（包括Thumb和Thumb-2模式）。当我们谈论64位ARM时，会把它称为ARM64。

###3.4.1 无优化的 Keil 6/2013(ARM 模式)

让我们在Keil中编译我们的例子：

```
armcc.exe --arm --c90 -O0 1.c
```

armcc编译器使用intel语法生成汇编清单，但包含了许多高层次的ARM处理器相关的宏。但对我们更重要的是指令“原来”的样子，所以让我们看看IDA里反编译的结果。

```
.text:00000000 main
.text:00000000 10 40 2D E9 STMFD SP!, {R4,LR}
.text:00000004 1E 0E 8F E2 ADR R0, aHelloWorld ; "hello, world"
.text:00000008 15 19 00 EB BL __2printf
.text:0000000C 00 00 A0 E3 MOV R0, #0
.text:00000010 10 80 BD E8 LDMFD SP!, {R4,PC}
.text:000001EC 68 65 6C 6C+aHelloWorld DCB "hello, world",0 ; DATA XREF: main+4
```

在这个例子里，我们能容易地发现每个指令都是4字节。的确，我们只用ARM模式编译而不是Thumb。

最开始的指令`STMFD SP!, {R4,LR}`,相当于x86的`PUSH`指令，将两个寄存器里（R4 和 LR）的值放入栈中。在armcc编译器的输出清单中，为了简单起见，确实显示了`PUSH {r4, lr}`指令。但这是不准确的。`PUSH`指令只有在Thumb模式中才会出现。所以为了避免混淆，我们使用IDA的结果。

这条指令首先减小了SP的值使它指向栈里空闲的位置。