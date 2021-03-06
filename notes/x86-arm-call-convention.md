---
description: 桟的一些知识 - 以前的笔记
---

# X86 ARM 下的函数调用规则

## 序

刚学计机的第一年，虽然 “桟” 这个词经常从老师嘴里冒出，其实他们都没有真正解释过这个东西到底是什么，当然对于大部分学上层的人来说，不知道是无所谓，特别是如果学 Java ，基本上接触不到这个概念。

这篇文章呢就以桟展开，然后说说函数之前的参数是怎么配合桟来实现的，看完这篇文章，无论是在 x86 还是 ARM 架构下面，都应该可以写出汇编和C语言的混合调用的代码，这也是在内核实现中不可缺少的一部分。

## 桟的本质

桟，就是一块可用的内存，它只是规定了数据拿出和放进的顺序，这个顺序就是我们经常说的，先进后出，在上课的时候老师总会用一个桶来比喻，先放进去的东西总是在下面，所以得把后面放的东西拿出来，才能拿最早放进去的。

其实这个顺序为什么是这样，根本不重要，关键在于什么呢，数据是有序放进去的，也就是说拿出来的时候一定也是有序的，它跟数组没有区别，只是我们要拿到第一个元素的时候，是到最后面去拿。但是，**实际上我们也不一定遵循这个顺序**，有时候我们甚至可以直接抛弃一块连续的空间，比如后面抛弃局部变量空间的时候。

在 x86 的体系下，有一个专门的寄存器叫，sp\( stack pointer \)，它存储了一个地址，就是桟顶的地址，更准确的说，是**最后一个放进去的数据**的地址

```text
sp[0] = 最后一个放进去的元素
sp[1] = 倒数第二个放进去的元素
...
```

在初始化的时候，无论是操作系统的初始化，又或是普通程序初始化，其中要做一件事是，分配一块区域，让 sp 指向区域最末端

```c
void * ptr = malloc(4KB);
sp  =  (int)ptr + (1 << 22);

// sp = 0x400000, 假设分配的区域就是 0～0x3fffff

那么 sp[0] = ? 
！注意 这里会造成内存越界

我们说了，sp[0] 指向的是最后一个放进去的数据，现在放数据进去了吗？
很简单的道理， sp[0] 访问的是 0x400000~0x400003 这4 字节
这里假设机器是 32bit

而一块区域并不在我们分配的区域里面
int i = 1;
int * s_sp = (int *）sp；
s_sp--;
*s_sp = 1;
sp = (int)s_sp;

注意这里把 sp 看成是一个整形的变量，实际上
也是如此，指针这个概念只有C才存在
上面代码等同于

push $1  // 把立即数 1 入栈

也等同于 

sub $4, %sp  // sp = sp - 4;
mov $1, (%sp)

此时的 sp == 0x3fffffc
```

上面的代码说明了 sp 永远指向的地方都是存放数据的地方，我们用 \[sp\] 访问的是存在数据的地方，当我们想加入新数据的话，也就是 push 操作，**必须得把 sp -= 4** 也就让它的地址指向一块当前没有数据的地方，然后在放入数据，这样 sp 再次指向了**最后一次放入的数据**。

![push](../.gitbook/assets/image%20%28135%29.png)

那么同理也可以知道，如果执行 pop 操作

{% hint style="info" %}
sp 和 esp 的意义是一样的，读者可以暂时不要区分
{% endhint %}

```c
pop():
    int * p = (int *)sp;
    int data = *p;
    sp = sp - 4; // or  sp = (int)(--p);

pop:
    movl (%esp), %reg
    subl $4, %esp;
<=>
    popl %reg
```

也就是说，先取数据在改变 sp

**pop** 之后，并不会请空无效数据\(data\)的值，stack pointer已经标示了当前有效数据就是在它之上的数据 其余数据是什么并不关心，所以在C中，一些局部变量，我们虽然在函数体之外，仍然可以访问到。

![pop](../.gitbook/assets/image%20%28181%29.png)

最后总结一下，栈就是指数据取出和放入遵循**一定规则**的区域，没有什么神秘的地方。然后，sp 指针指向的是最后一个放入数据，如果没有放入数据就访问，会导致内存越界。

## 栈的作用

在理解栈这个东西之后，我们就可以来阐述它存在的意义了。

在程序中，我们会用到很多的变量，也有很多的函数调用，其中绝大多数的变量都是局部变量，这些变量的存在周期只存在于函数调用期间，我们把这种变量称之为局部变量，接下来我们了解到，函数调用是怎么实现返回的，以及为什么局部变量只会在函数内有效，还有函数调用之间的参数如何传递。

### 局部变量

变量，就是一块内存空间，大小取决于它的数据类型本身的大小，而局部变量，其实就是它所在的空间正好就是桟里面。

```c
void foo()
{
    int i=0,j=1,k=2;
    i--;
    j++;
    k--;
    ...
    ...
    return; 
}
 ​
foo:
    push $0    
    push $1
    push $2
    decl (%esp+8)
    incl (%esp+4)
    decl (%esp)
    ...
    ...     
    addl $12, %esp  // 此时桟顶数据已经不再包括i,j,k了
    ret          // 函数返回
```

代码不难理解，下面就是汇编代码，为什么局部变量在函数体之外就不能访问呢，原因很简单，当我们函数返回之前，分配给局部变量的空间我们回收了，回收是通过改变桟指针，sp，来实现的（注意我们的桟是从高地址到低地址），之前也提到了，sp指向的空间以下的空间都是无效的数据，所以我们认为 i,j,k 这几个变量已经无效了，所以它们被称为局部变量。

所谓局部变量，只是桟的一块空间，当我们离开函数之后，桟顶指针就不在指向它们了，读者可能会发现，其实离开函数之后，用 \[sp-4\] 不就可以访问了，对的，所以我们说这不是严格意义上的回收，但是访问局部变量是有自己的**规范**的。

### 函数参数传递

函数的参数和局部变量是等价的，当我们调用一个函数之前，就把它的参数放在了桟里，让调用的函数可以使用。

```c
 ​
void bar(int ​i, ​int ​j, ​int ​k);

void foo()
{
    int i=0,j=1,k=2;
    ...
    bar(i,j,k);
    ...
    ...
    return; 
}

foo:
// 局部变量的入桟
    push $0    
    push $1
    push $2
    ...
// 参数入桟 拷贝了一遍局部变量 
    movl (%esp), %reg  
    push %reg  //k入桟
    movl (%esp+4), %reg
    push %reg //j入桟
    mov (%sp+8), %reg
    push %reg //i入桟
// 调用函数
    call bar
    ...
```

所以说局部变量的本质和传递的参数都一样，它们都是**桟上的空间**。有趣的一个地方是，参数是**从右至左**的顺序入桟。其实就是一个习惯，稍后会解释。

上面的代码很清楚解释了一个道理，就是为什么函数里面修改传递来的参数对调用方是没有影响的，因为在调用之间，_**要传递的参数都做了拷贝，被调用的函数只是对拷贝的那一份做了修改，并不改变原来的那一份**_。有人说引用可以，其实**引用本质就是指针**。

### 函数调用的本质

**函数就是一块代码的首地址，和我们C语言使用的 goto 是没有区别的**。

这是必须得明白的一件事，当我们函数返回的时候，就是 goto 到了原来调用它的地方的下一条代码。那么调用之前，我们得做的事情有什么呢，首先是调用的参数得入桟，然后把函数返回需要到的地方的地址入桟。接着，跳转到目标函数的地址执行。

```c
void bar(int i) {
    ...
}

void foo() {
    ...
    bar(2);
    ...
}

bar:
    0xb00    ...
    ....
    0xbef    ret

foo:
    0xa00    ...
    0xa04    push $2
    0xa08    call bar
    0xa0c    ...

上述的意思就是 foo 的函数代码所在地址 0xa00 
同理  bar 在 0xb00

那么 call 到底做了什么事情
首先  push $0xa0c  也就是返回的时候要执行跳转到的地方（ goto ）
然后  jmp   0xb00    // goto bar
即直接 goto 目标代码所在地方执行

ret 呢
    应该不难猜出 应为已经知道要回到哪个地方了
    首先在 ret 之前得保证 sp 跟进来的时候指向的是同一个地方

明确一点， 栈 是 所 有 函 数 公 用 的， bar函数也会使用 sp
那么如何保证它一定回到最初的位置呢，答案是做备份

下面介绍一个新的寄存器 bp 
在所有函数入口和出口都有这样类似的指令

function:
    pushl   %ebp        // bp 也是全局的 要备份
    movl    %esp, %ebp  // bp = sp   
    ...
    ...    中间局部变量的存在导致 sp 必然会改变
    ...
    movl    %ebp, %esp    // 让 sp = old_sp， 复原
    popl    %ebp        // bp = old_bp
    ret
```

也就是说，bp 寄存器的作用就是备份函数进入时候的栈顶指针，**然后 bp 本身放进了栈里面备份**。然后无论我们函数中间 sp 怎么变，我们都可以保证最后它一定是进来时的位置。

```c
了解了 bp 和 sp 的备份机制
ret 的作用现在继续

刚才说了，调用函数之前除了要将函数要的参数入栈,
call 还会把调用函数的下一条指令入栈
这一步是call 指令自动完成的

所以 ret 之前，我们恢复了 sp 最开始的位置

sp 指向的是 最 后 一 个 入 栈 的 元 素
即调用函数前，最后一个 自动入栈的  下 一 条 指 令 地  址

ret::
    popl  %reg    // 把地址拿出来
    jmp  %reg    // goto 到目标地址

    以上面的例子， reg  = 0xa0c
```

_**call**_ 和 _**ret**_ 做的事情，利用无条件跳转就可以轻松实现对吧，关键得理解，**函数地址就是一个代码的起始地址，每一条指令都是有地址的**。

现在，我们又要介绍一个新的寄存器，它储存的地址就是 将要执行的代码的地址

```c
foo:
    0xa00    ...
    0xa04    push $2
    0xa08    call bar
    0xa0c    ...

    当 CPU 执行 0xa04 指令的时候
    还有一个 ip ( instruct pointer ) 寄存器，指向的就是 0xa08
    我们上课会学到的 PC 指针，它就是本尊啦

也就说 call 其实是这么做的
    pushl  %eip
    jmp    0xxx  目标代码的地址
```

这些寄存器是什么都不重要，关键我们得理解其中栈的作用，就是作为一个**备份的区域**，存函数参数，放了调用函数之后该回到的地方 以及 函数本身要使用的局部变量。

接下来 以下一个例子 来一个总结

```c
int func(int arg1) {
    int local;
    arg1++;
    return arg1;
}

void caller() {
    int ret;
    ret = func(1);
}

caller:
    // 准备阶段
    pushl   %ebp
    movl    %esp, %ebp

    // 开辟 ret 局部变量的空间
    subl    $4, %esp

    // 参数入栈
    pushl   $1
    call   func

    //  存储返回值到本地变量中
    movl    %eax, [sp+4]

   //  还原状态
    movl    %ebp, %esp
    popl    %ebp
    ret
```

![stack](../.gitbook/assets/image%20%28174%29.png)

调用 **函数之前 和 函数之后 的 sp 一定是一样**的，所以 sp 指向的一定是最后一次 push 导致 sp 下移的位置。

还有一些前面没有说过的东西，返回值是怎么实现的？就是通过一个 ax 的寄存器，然后发现了吗，我们直接用 _**mov**_ 指令来索引本地变量，再次说明，栈只是**遵循一个特定顺序的存储空间**，甚至，假设还有一个本地变量，我们还可以，用

```text
mov  reg, [sp+4]
```

之间我们说的，**入栈出栈的顺序根本没有遵守，它就是一块可用的空间。**

```c
int func(int arg1) {
    int local;
    arg1++;
    return arg1;
}

func:
    // 准备阶段
    pushl   %ebp
    movl    %esp, %ebp

   // 开辟 local 局部变量的空间
    subl    $4, %esp

   //  [bp] 是 bp的备份， [bp+4]  是函数返回地址
    incl  (%ebp+8) 

    movl  %(ebp+8), %eax   // 返回值

    //  还原状态
    movl    %ebp, %esp
    popl    %ebp
    ret
```

很明显啦， **bp 用来访问传入的参数，sp 用来访问局部变量，但是它们都是栈上的变量**。

![stack](../.gitbook/assets/image%20%2830%29.png)

传入 arg1 = 1 即是 bp + 8 指向的位置

## x86 arm 的区别

前面说的参数的传入以及局部变量的储存，都是 x86 上的函数调用规则，是有一套协议的，下面给上具体的反汇编代码了，之前的都是伪代码。

```c
int a[5] = {1,2,3,4,5};    /* 数组， 全局变量 */

int  func(int a) {
    return a;
}

int  d(int * a, int b, int c, int d, int e, int f) {
    (*a)++;
    func(b);    
    return e+f;
}

int main(void) {

    d(a, 1,2,3,4, 5);

    return 0;
}
```

![1](../.gitbook/assets/image%20%2839%29.png)

![2](../.gitbook/assets/image%20%28186%29.png)

![3](../.gitbook/assets/image%20%28145%29.png)

![4](../.gitbook/assets/image%20%28149%29.png)

![5](../.gitbook/assets/image%20%2856%29.png)

ARM 就不一个一个分析了，关键在于参数直接用寄存器和桟传递的，然后返回值也是如此，然后局部变量的位置有所不同，其实就是**函数调用的规范**不一样。

```c
.data
a:
    .word    1
    .word    2
    .word    3
    .word    4
    .word    5

    .text
func:
    str    fp, [sp, #-4]!
    add    fp, sp, #0
    sub    sp, sp, #12
    str    r0, [fp, #-8]
    ldr    r3, [fp, #-8]
    mov    r0, r3
    add    sp, fp, #0

    ldr    fp, [sp], #4
    bx    lr

d:
    push    {fp, lr}
    add    fp, sp, #4
    sub    sp, sp, #16
    str    r0, [fp, #-8]
    str    r1, [fp, #-12]
    str    r2, [fp, #-16]
    str    r3, [fp, #-20]
    ldr    r3, [fp, #-8]
    ldr    r3, [r3]
    add    r2, r3, #1
    ldr    r3, [fp, #-8]
    str    r2, [r3]
    ldr    r0, [fp, #-12]
    bl    func
    ldr    r2, [fp, #4]
    ldr    r3, [fp, #8]
    add    r3, r2, r3
    mov    r0, r3
    sub    sp, fp, #4

    pop    {fp, lr}
    bx    lr


main:
    push    {fp, lr}
    add    fp, sp, #4
    sub    sp, sp, #8
    mov    r3, #5
    str    r3, [sp, #4]
    mov    r3, #4
    str    r3, [sp]
    mov    r3, #3
    mov    r2, #2
    mov    r1, #1
    ldr    r0, .L7
    bl    d
    mov    r3, #0
    mov    r0, r3
    sub    sp, fp, #4

    pop    {fp, lr}
    bx    lr

.L7:
    .word    a
```

给出调用的桟

![](../.gitbook/assets/image%20%28150%29.png)

在x86当中，默认的**push**指令就是**Decreasing Before**模式，但是在Arm中没有原生，对于**push, pop**的指令，实际上还是会转换到**ldr，str**或者它的衍生指令。之所以出现STMDA的栈指令，就是为了操作方便，比如你存进去栈的时候是 Full Descending 那么取出来的时候就是 Empty Ascending

![form Net](../.gitbook/assets/image%20%2820%29.png)

## 总结

到如此，我的耐心已经消耗殆尽拉0.0，给出俩张图，读者自己好好理解，所以我很佩服热衷分享的人，并且还能说的很详细的人！

x86

![](../.gitbook/assets/image%20%2893%29.png)

ARM

![](../.gitbook/assets/image%20%28111%29.png)

