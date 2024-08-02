---
layout: post
title: C中的编译器优化关键字
subtitle: restrict关键字，以及简单的汇编小记
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

&emsp;&emsp;rbp寄存器一开始指向的是当前栈帧的基地址，也就是调用main函数的栈帧的地址，我们将其压入到当前栈帧的栈顶————这意味着main函数栈帧的上面还存储了上一个栈帧的基地址，这样我们才能知道main函数如何返回。然后我们将当前的栈顶指针（rsp）移到栈基地址（rbp）里面，这样我们就确立了main函数栈帧的基地址。

&emsp;&emsp;然后是a++：a++先是读取a到edi（Extended Destination Index，扩展目标索引寄存器）里,然后++之，最后存回去。

&emsp;&emsp;由此可知，即使是最简单的++，电脑操作一个数的流程是：读取，计算，存回。

## 说明

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

&emsp;&emsp;哇，这不就是我们理想的代码吗？或者说，这才是这一段代码的真正逻辑啊！我一开始有一个疑问，这都是O2级别优化了，编译器难道不知道还有没有指针指向这个内存空间吗？我后来一想，编译器虽然知道编译的时候没有重复指针指向这个空间，但是编译器不能知道，运行之后会不会有代码改变某个指针的值，导致重复指向！所以，这是我们人类需要向其保证的。

&emsp;&emsp;那么，优化的原理也十分显然了，因为减少了内存读取的操作，将加速运算。我想，这样的关键字在计算密集型函数内应该十分有用，有兴趣的读者可以自行编写程序测试这样的优化到底有多少效果。