第三章 Hello,world!

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
– l — long (32 bits)
– w — word (16 bits)
– b — byte (8 bits)

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
