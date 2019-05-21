---
description: ongoing...
---

# 什么是 crt\( on going\)

## 序

嵌入式开发的过程中，有一个特别的过程叫做 交叉编译，意思呢就是我们在自己的电脑上编译后，放在嵌入式设备中可以运行，编译的这一过程就是交叉编译。

如果我们编写一个简单的 helloworld.c 程序，想要在嵌入式设备中运行，我们就必须得知道，一个简单的helloworld程序，到底有哪些是系统帮我们做的，当我们移植的时候，又得做什么。这时候不得不了解的一个东西就是crt，C runtime Library

## 什么是crt

crt 就是C运行库，提供了一个 C程序运行的所需的相关函数，以及初始化的代码。

考虑一个简单的 main 函数

```c

int main(int argc,   char *argv[] ) {
     
     puts("halo");
     
    return 0;
}
```

这里有几个很有意思的地方，首先 main 是一个函数，意味着它是被调用的，所以在初学的时候，我们简单的认为，main就是程序的入口，其实并不是，严谨来说，main 只是程序必须经过的一个**中间过程**，它是有返回值的，程序开始执行的一条代码肯定不是 main 函数里面的语句。

我们需要了解的一点，就是**main函数之前，程序还做了很多事情**。

同时 main 函数还有参数，这些就是调用它的函数赋值给它的，再次体现 main 只是一个中间过程，而不是入口。

#### crt0.o, crti.o, crtbegin.o, crtend.o, and crtn.o

现在是时候介绍这几个 object 文件了，这些 .o 文件都是可以被链接的，熟悉gcc的人肯定都不会陌生。其中， crt0 crti crtn 都是由C库提供，而 crtbegin crtend 由编译器提供，记住一点，编译器，C标准库都是操作系统专用的，不同的操作系统之间的实现是不同的，比如Windows下的C标准库实现起来和Linux不一样，编译器自然更不用说了。

{% code-tabs %}
{% code-tabs-item title="crt0.s" %}
```c
section .text

.global _start
_start:
	# Set up end of the stack frame linked list.
	movq $0, %rbp
	pushq %rbp # rip=0
	pushq %rbp # rbp=0
	movq %rsp, %rbp

	# We need those in a moment when we call main.
	pushq %rsi
	pushq %rdi

	# Prepare signals, memory allocation, stdio and such.
	call initialize_standard_library

	# Run the global constructors.
	call _init

	# Restore argc and argv.
	popq %rdi
	popq %rsi

	# Run main
	call main

	# Terminate the process with the exit code.
	movl %eax, %edi
	call exit
.size _start, . - _start
```
{% endcode-tabs-item %}
{% endcode-tabs %}

这是一个简单的示例，能让我们了解，为什么会有这几个obj文件的出现，其中很关键的一步是什么呢，调用main函数，对于我们用户来说，不需要关心一个程序到底需要做什么样的初始化，我们只需要增加自己的功能，接着这些配套的环境就可以帮助我们将这个程序成功的运行。实际上，这就是一种**抽象**。

一个程序需要运行，需要很多的准备，比如初始化桟以及相关寄存器，还有信号的处理句柄，最后保证程序调用的exit，通知其父进程，此进程已经结束，释放它在内核中的保存的数据（进程数据）。

这也是为什么，我们编写main函数，甚至可以写 void main，我以前就很好奇，如果一个进程不调用exit函数，那么它的资源永远也得不到释放，所以必须得保证**进程最后的一步就是调用exit函数**，也是因为这个原因，所以得把main函数作为一个中间的步骤。

当然，更多的考量一定是为了减少程序员的负担和降低入门门槛，这些对于所有进程都一样的操作不需要人为的干预，而是自动化的处理。信号处理句柄是什么，就是其他进程（或者本身）对它发送相关信号作出的应答，比如我们在任务管理器结束一个进程，其实就是发送相关的信号，然后进程的处理函数保证了**进程调用exit函数，释放了自己的资源**。所以进程也得做这方面的初始化，当然我们还可以人为的设置句柄，这是允许的。

并且，上述的操作都是 OS-specific 的，当然更是 platform-specific 也就是说不同的平台处理一定是不同的。这也解释了为什么交叉编译的时候必须要目标机的编译工具还有C库，因为这些都是平台（操作系统）相关的，只有保证了这些构造函数，析构的函数的准确调用，才能提供C语言编写的程序的运行环境。

> [Here](https://www.embecosm.com/appnotes/ean9/html/ch05s02.html)
>
>  The C Runtime system must carry out the following tasks.
>
> *  Set up the target platform in a consistent state. For example setting up appropriate exception vectors.
> *  Initialize the stack and frame pointers
> *  Invoke the C constructor initialization and ensure destructors are called on exit.
> *  Carry out any further platform specific initialization.
> *  Call the C `main` function.
> *  Exit with the return code supplied if the C `main` function ever terminates.



所有可执行文件，都会一个头部，其中就会表明它的入口偏移地址，操作系统将程序加载到内存之后，就会跳转到这个入口将执行权交给了程序，但是这路口不是main , 而是在 crt0.o 的一个 \_start 函数

```c
.section .text

.global _start
_start:
    # 这里就是函数的入口
```

{% hint style="info" %}
"crt" stands for "C runtime", and the zero stands for "the very beginning" --Wiki
{% endhint %}

crt0 需要做的事情不少，比如

首先先不管交叉编译，这是引出问题一个例子，我们就以本机为例，当我们编译一个简单的程序的时候，gcc 究竟给我们做了哪些处理。



我们想要交叉编译，首先得有目标机的编译器，这里我们以 arm 为例，可以在官网下载到 Linux 下针对 arm 的 GCC，然后目标机的操作系统提供的C标准库，平时我们在编写程序的调用的一些标准函数， printf 等等，都是由C的库提供的动态库，当我们编译的时候编译器自动帮我们链接，简单点理解，就是它提供了函数，我们只需要调用就好啦，这些细节都被屏蔽了，但是我们交叉编译的时候，必须得手动指定它们的位置了，因为默认链接的是我们系统的运行库。

## printf





