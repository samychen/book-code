# crack-tutorial

- 用 masm(radasm) 编译 exe，用 ollydbg 来反汇编学习
- 用 vc 编译 exe，用 ollydbg 来反汇编学习
- 用 vc 反汇编 debug 版 exe 来学习

https://github.com/Apress/modern-x86-assembly-language-programming


人类正常阅读方式是从左上到右下，16 进制编辑器也是从左上到右下，内存地址空间也是从左上到右下。那么文件中数据的存储方式是小端存储，即最低位在左边。

C++ 提供的整数数据类型有三种:int、long、short。在 vc6 中，int 类型与 long 类型在内存中都占 4 个字节，short 类型在内存中占两个字节。

由于二进制数不方便显示和阅读，因此内容中的数据采用 16 进制数显示。1 个字节有 2 个 16 进制数组成，在进制转换中，1 个 16 进制数可用 4 个 2 进制数表示，每个 2 进制数表示 1 位， 因此 1 个字节在内存中占 8 位。

## 无符号整数

在内存中，无符号整数的所有位都用来表示数值。以无符号整数数据 unsigned int 为例，此类型的变量在内存中占 4 字节，由 8 个 16 进制数组成，取值范围为 0x00000000 ~ 0xFFFFFFFF，如果转换为十进制数，则表示范围为 0 ~ 4294967295。

当无符号整型不足 32 位时，用 0 来填充剩余高位，直到占满 4 字节内存空间为止。例如，数字 5 对应的二进制数为 101，只占了 3 位，按 4 字节大小保存，剩余 29 个高位将用 0 填充，填充后结果为：00000000000000000000000000000101；转换成 16 进制数 0x00000005 之后，在内存中以“小尾方式”存放。“小尾方式”存放是以字节为单位，按照数据类型长度，高数据位对高地址，低数据位对低地址，如 0x12345678 将会存储为 78 56 34 12。相应地，在其他计算机体系中，也有“大尾方式”，其数据存储方式和“小尾方式”相反，高数据位放在内存的低端，低数据位放在内存的高端，如 0x12345678 将会存储为 12 34 56 78。

由于是无符号整数，不存在正负之分，都是正数，故无符号整数在内存中都是以真值的形式存放的，每一位都可以参与数据表达。无符号整数可表示的正数范围是补码的一倍。


## 有符号整数

有符号整数中用来表示符号的是最高位——符号位。最高位为 0 表示整数，最高位为 1 表示负数。有符号整数在内存中同样占 4 字节，但由于最高位为符号位，不用用来表示数值，因此有符号整数的取值范围要比无符号整数取值范围少 1 位，即 0x80000000 ~ 0x7FFFFFFF，如果转换为 10 进制数，则表示范围为 - 2 147 483 648 ~ 2 147 483 647。

在有符号整数中，正数的表示区间为：0x00000000 ~ 0x7FFFFFFF；负数的表示区间为：0x80000000 ~ 0xFFFFFFFF。

负数在内存中都是以补码形式存放的，补码的规则是用 0 减去这个数的绝对值，也可以简单地表达为对这个数值取反加 1。例如，对于 -3，可以表达为 0 - 3，而 0xFFFFFFFD + 3 等于 0 （进位丢失），所以 -3 的补码也就是 0xFFFFFFFD 了。相应地，0xFFFFFFFD 作为一个补码，最高位为 1，视为负数，转换回真值同样也可以用 0 - 0xFFFFFFFD 的方式，于是得到 -3。为了计算方便，人们也常用取反加一的方式来求得补码，因为对于任何 4 字节的数值 x，都有 x + x(反) = 0xFFFFFFFF，于是 x + x(反) + 1 = 0，接下来就可以推导出 0 - x = x(反) + 1 了。


## 浮点数类型



研究用高级语言编写的程序的算法，按照惯例是从重建源代码中诸如函数、局部与全局变量、分支、循环等关键结构开始的，这使得反汇编器（相对于调试器）列出的内容更容易阅读，并极大地简化了它的分析工作。

现代反汇编器是很智能化的，并在识别关键结构方面享有很高的声誉。特别是，IDA Pro 能够成功地识别标准库函数、友 ESP 分配的局部变量以及 CASE 分支等。然而，IDA 偶尔也会出错，从而误导使用者。

## EBP 和 ESP 寄存器的用途

EBP 和 ESP 常配合使用完成堆栈的访问，下面是一段常见的堆栈访问指令。

``` asm
push    ebp
move    ebp, esp
sub     esp, 78
push    esi
push    edi
cmp     dword ptr [ebp+8], 0
```

一直对寄存器 ESP 和 EBP 的概念总是有些混淆，查看定义 ESP 是栈顶指针，EBP 是存取堆栈指针。还是不能很透彻理解。之后借于一段汇编代码，总算是对两者有个比较清晰的理解。
下面是按调用约定__stdcall 调用函数 test(int p1,int p2) 的汇编代码
假设执行函数前堆栈指针 ESP 为 NN
``` asm
push   p2    ; 参数 2 入栈, ESP -= 4h , ESP = NN - 4h
push   p1    ; 参数 1 入栈, ESP -= 4h , ESP = NN - 8h
call test    ; 压入返回地址 ESP -= 4h, ESP = NN - 0Ch  
;// 进入函数内
{
push   ebp                        ; 保护先前 EBP 指针， EBP 入栈， ESP-=4h, ESP = NN - 10h
mov    ebp, esp                   ; 设置 EBP 指针指向栈顶 NN-10h
mov    eax, dword ptr  [ebp+0ch]  ;ebp+0ch 为 NN-4h, 即参数 2 的位置
mov    ebx, dword ptr  [ebp+08h]  ;ebp+08h 为 NN-8h, 即参数 1 的位置
sub    esp, 8                     ; 局部变量所占空间 ESP-=8, ESP = NN-18h
...
add    esp, 8                     ; 释放局部变量, ESP+=8, ESP = NN-10h
pop    ebp                        ; 出栈, 恢复 EBP, ESP+=4, ESP = NN-0Ch
ret    8                          ;ret 返回, 弹出返回地址, ESP+=4, ESP=NN-08h, 后面加操作数 8 为平衡堆栈, ESP+=8,ESP=NN, 恢复进入函数前的堆栈.
}
```
看完汇编后, 再看 EBP 和 ESP 的定义, 哦, 豁然开朗,
原来 ESP 就是一直指向栈顶的指针, 而 EBP 只是存取某时刻的栈顶指针, 以方便对栈的操作, 如获取函数参数、局部变量等。

- [函数调用（ebp，esp，堆栈快照）](http://www.cnblogs.com/mayingkun/p/5792055.html)


EIP，EBP，ESP 都是系统的寄存器，里面存的都是些地址。为什么要说这三个指针，是因为我们系统中栈的实现上离不开他们三个。

其实它还有以下两个作用：
 1. 栈是用来存储临时变量，函数传递的中间结果。
 2. 操作系统维护的，对于程序员是透明的。
我们可能只强调了它的后进先出的特点，至于栈实现的原理，没怎么讲？下面我们就通过一个小例子说说栈的原理。
先写个小程序：
``` cpp
void fun(void)
{
   printf("hello world")；
}

void main(void)
{
  fun()
  printf("函数调用结束");
}
```

这是一个再简单不过的函数调用的例子了。
当程序进行函数调用的时候，我们经常说的是先将函数压栈，当函数调用结束后，再出栈。这一切的工作都是系统帮我们自动完成的。
但在完成的过程中，系统会用到下面三种寄存器：
1.EIP
2.ESP
3.EBP
当调用 fun 函数开始时，三者的作用。

1. EIP 寄存器里存储的是 CPU 下次要执行的指令的地址。
 也就是调用完 fun 函数后，让 CPU 知道应该执行 main 函数中的 printf（"函数调用结束"）语句了。
2. EBP 寄存器里存储的是是栈的栈底指针，通常叫栈基址，这个是一开始进行 fun() 函数调用之前，由 ESP 传递给 EBP 的。（在函数调用前你可以这么理解：ESP 存储的是栈顶地址，也是栈底地址。）
3. ESP 寄存器里存储的是在调用函数 fun() 之后，栈的栈顶。并且始终指向栈顶。
 
当调用 fun 函数结束后，三者的作用：
1. 系统根据 EIP 寄存器里存储的地址，CPU 就能够知道函数调用完，下一步应该做什么，也就是应该执行 main 函数中的 printf（“函数调用结束”）。
2.EBP 寄存器存储的是栈底地址，而这个地址是由 ESP 在函数调用前传递给 EBP 的。等到调用结束，EBP 会把其地址再次传回给 ESP。所以 ESP 又一次指向了函数调用结束后，栈顶的地址。
其实我们对这个只需要知道三个指针是什么就可以，可能对我们以后学习栈溢出的问题以及看栈这方面的书籍有些帮助。当有人再给你说 EIP,ESP,EBP 的时候，你不能一头雾水，那你水平就显得洼了许多。其实不知道我们照样可以编程, 因为我们是 C 级别的程序员，而不是 ASM 级别的程序员。

ebp 和 esp 的主要区别在于：一个函数内部 ebp 指向的是函数入口处栈指针，而 esp 是当前指令的栈指针，所以一般来说，esp 是变化的，他随着 push,pop,add esp,0xXX,sub esp,0xXX 等的指令的变化而变化，但在一个函数内部 ebp 通常是不变的（除非人为的用汇编指令对 ebp 操作），所以通常用 ebp 来保存函数内部的参数，局部变量等重要的内容（由于 esp 可变，不能用 esp 访问），并且在函数结束的时候将 ebp 的值赋值给 esp 来维持栈的平衡。
实际上 ebp 和 esp 都能访问到整个栈的内容，但是由于 esp 的可变性，通常并不用 esp 来访问，而用 ebp 访问。


## 函数

## 启动函数

## 虚函数

## 构造函数与析构函数

## 对象、结构体与数组

## this 指针

## new 操作符与 delete 操作符

## 库函数

## 函数的参数

### 函数的返回值

### 局部堆栈变量

### 寄存器与临时变量

### 全局变量

### 常量与偏移量

### 文本与字符串

### IF-THEN-ELSE 条件语句

### SWITCH-CASE-BREAK 语句

### 循环语句

### 数学运算符