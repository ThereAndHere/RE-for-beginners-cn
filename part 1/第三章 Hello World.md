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

MSVC生成的汇编使用Intel语法。Intel语法和A&T语法的区别以后再讨论。

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

在函数序言之后我们可以看到调用了printf()函数：CALL _printf。
