---
layout: post
title: 简单汇编/restrict/宏粘连符/(un)likely笔记
subtitle: restrict关键字，简单的汇编小记，likely和unlikely；封面：Twitter@fleia0124
categories: cpp
banner:
  image: /assets/images/banners/12.jpg
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: cpp
---

## 汇编小引

> 引入汇编的基础介绍，熟悉的人不用看

&emsp;&emsp;对于下面一个，简单的a+b的代码

```c
int add_a(int a) {
    a = a + 1;
   return a;
}

int main() {
   return add_a(8);
}
```

其汇编是：

```nasm
add_a:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi  // 读取a
        add     DWORD PTR [rbp-4], 1    // ++
        mov     eax, DWORD PTR [rbp-4]  // 存回去
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        mov     edi, 8
        call    add_a
        pop     rbp
        ret
```

&emsp;&emsp;注意到两个函数开始的命令都是两个有关`rbp`寄存器的命令。`rbp`寄存器一开始指向的是当前栈帧的基地址，也就是调用`main`函数的栈帧的地址，我们将其压入到当前栈帧的栈顶————这意味着`main`函数栈帧的上面还存储了上一个栈帧的基地址，这样我们才能知道main函数如何返回。然后我们将当前的栈顶指针（rsp）移到栈基地址（rbp）里面，这样我们就确立了main函数栈帧的基地址。

&emsp;&emsp;然后看到a++的函数：a++先是读取a到edi（Extended Destination Index，扩展目标索引寄存器）里,然后++之，最后存回去。

&emsp;&emsp;由此可知，即使是最简单的++，电脑操作一个数的流程是：读取，计算，存回。

## restrict说明

&emsp;&emsp;使用restrict修饰的指针，代表像编译器保证，当前作用域内不会再有另一个指针指向同一个内存地址（怎么有点c++里unique_ptr之类的感觉）。这样的好处是，在编译的时候，可以提前将这个值加载进入CPU寄存器里面，而不是反复从内存中读取。

&emsp;&emsp;下面是在[这个网站](https://godbolt.org/)上测试的结果。编译选项加了`-O2`优化，gcc14.1。这是个很简单的代码，把c加到a+b上。

```c
int square(int* a, int* b, int* c) {
    *a += *c;
    *b += *c;
}
```

&emsp;&emsp;那么，汇编是怎么样的呢？

```nasm
square:
        // 读取c
        mov     eax, DWORD PTR [rdx]
        //读取a
        add     DWORD PTR [rdi], eax

        // 读取c
        mov     eax, DWORD PTR [rdx]
        // 读取b
        add     DWORD PTR [rsi], eax

        ret
```

&emsp;&emsp;我们看到，变量c被读取到寄存器读取了两次，即`mov     eax, DWORD PTR [rdx]`这一段。但是我们知道，完全没这个必要，让人类写肯定不会这么干。这里是使用`restrict`修饰的代码：

```c
int square(int* restrict a, int* restrict b, int* restrict c) {
    *a += *c;
    *b += *c;
}
```

&emsp;&emsp;对应的汇编是：

```nasm
square:
        // 读取c
        mov     eax, DWORD PTR [rdx]
        // 加a
        add     DWORD PTR [rdi], eax
        // 加b
        add     DWORD PTR [rsi], eax

        ret
```

&emsp;&emsp;哇，这不就是我们理想的代码吗？或者说，这才是这一段代码的真正逻辑啊！我一开始有一个疑问，这都是O2级别优化了，编译器难道不知道还有没有指针指向这个内存空间吗？我后来一想，编译器虽然知道编译的时候没有重复指针指向这个空间，但是编译器不能知道，运行之后会不会有代码改变某个指针的值，导致重复指向！所以，这是我们人类需要向其保证的。另外，如果你知道多线程操作不加锁的变量为什么会错误，你就能理解这个保证的意义了，这两个问题是一个道理。

&emsp;&emsp;那么，优化的原理也十分显然了，因为减少了内存读取的操作，将加速运算。我想，这样的关键字在计算密集型函数内应该十分有用，有兴趣的读者可以自行编写程序测试这样的优化到底有多少效果。

## unlikely和likely

&emsp;&emsp;此二者经常出现在linux内核代码中，是何意思？请看代码：

```c
# define likely(x)	__builtin_expect(!!(x), 1)
# define unlikely(x)	__builtin_expect(!!(x), 0)
```

&emsp;&emsp;`__builtin_expect(foo, s)`是一个编译器优化关键字，代表表达式foo的值大概率是s，其中，s应当是一个`const`，例子中`!!(x)`是为了让其计算结果确实是一个布尔值，不论之前是什么，例如之前是999，反转再反转就是布尔值1了。

&emsp;&emsp;所以，`likely(x)`的意思就是x大概率成立，反之就是x大概率不成立。此宏意义何在呢？用来辅助编译器优化分支流程。例如在大流量的网络接口中，好的封包显然占大多数，那么对于异常包的处理就并不总是被触发。我们知道if是一个分支命令，翻译成汇编（不优化的话）会有jmp命令存在，而这并不利于CPU流水线运算（不是jmp不利于，而是jmp之后的代码有可能不利于提高缓存命中率，不能很好的使用寄存器）。使用`likely(x)`宏告诉编译器，你把我这个大概率执行的代码放在正下方，把小概率执行的代码使用jmp命令跳转，从而提升执行速度。

```c
if (likely(packet_is_valid(packet))) {
    // 快速路径：处理有效的网络包
} else {
    // 慢速路径：处理无效或损坏的网络包
}
```

&emsp;&emsp;这个宏在内核中处处可见，用于异常值的处理。内核中处理异常情况全都不是直接写`(foo == false)`，而是使用`unlikely`明确指出当前情况大概率不发生，也就是告诉编译器这里是异常处理，你塞一边去，别影响主要流程的执行效率。我其实编写了示例代码进行测试，发现在逻辑简单的时候，这个宏的作用并不能很好的触发，不过我记录了下面的现象，编译器是gcc，开了O2：

```c
# define likely(x)	__builtin_expect(!!(x), 1)
# define unlikely(x)	__builtin_expect(!!(x), 0)

int square(int num) {
    int t = 0;
    // if (num == 777){
    if (unlikely(num)){
        t = 999;
    }
    else
        t = 888;

    return t;
}
```

不使用`unlikely`:

```nasm
square:
        cmp     edi, 777
        mov     edx, 999
        mov     eax, 888
        cmove   eax, edx
        ret
```

使用`unlikely`:

```nasm
square:
        cmp     edi, 1
        sbb     eax, eax
        and     eax, -111
        add     eax, 999
        ret
```

&emsp;&emsp;第一段代码使用的`cmove`是指，如果比较结果为相等 `(edi == 777)`，则将` edx `的值`（999）`移动到 `eax`。是条件执行指令，有点类似于c++里面的三目表达式。不过`unlikely`是使用了位运算。我多次测试发现使用`unlikely`指定的分支语句，编译器将会更多的使用位运算，甚至是尽可能的使用位运算。可是`AND`和`ADD`都是一个指令周期的，这是为啥呢？

## 宏粘连符`##`

&emsp;&emsp;这个很简单，直接上代码就看懂了，假设我有这样的代码：

```c
#include <stdio.h>

#define __cmp_op_gt >
#define __cmp_op_lt <
#define __cmp_op_ge >=
#define __cmp_op_le <=

#define __cmp(op, x, y) ((x) __cmp_op_##op (y) ? (x) : (y))

int main() {
    int a = 10, b = 20;
    
    int max = __cmp(gt, a, b);
    int min = __cmp(lt, a, b);

    printf("max: %d\n", max); // 输出: max: 20
    printf("min: %d\n", min); // 输出: min: 10

    return 0;
}

```

&emsp;&emsp;注意到了吗，`__cmp(gt, a, b)`可以传入运算符，而`((x) __cmp_op_##op (y) ? (x) : (y))`里面，`op`作为传入的参数，粘在了`__cmp_op_`的后面，例如这里，就粘连成了`__cmp_op_gt`，也就是`>`。

&emsp;&emsp;可是，我的疑问是，为什么不直接定义`#define cmp_gt(a, b) (a > b ? a : b)`，这样不也是一样的？对此，GPT给出的答案是，这样有利于维护，因为你只需要维护`#define __cmp(op, x, y) ((x) __cmp_op_##op (y) ? (x) : (y))`一处就可以了，换句话说，这里仅仅执行比较，而不是在四个地方执行比较！听起来很有道理的样子。不可否认，这样的思想确实是很高明。