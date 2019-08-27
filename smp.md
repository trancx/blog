---
description: smp boot notes
---

# SMP（Pending\)

## Goal

ARM 的多核引导，X86之后再提，我们先关注

## Draft

准备工作：

![](.gitbook/assets/image%20%2846%29.png)

head-common.S

![](.gitbook/assets/image%20%2818%29.png)

proinfo.h

![](.gitbook/assets/image%20%28140%29.png)

asm/mm/Makefile

![](.gitbook/assets/image%20%2813%29.png)

proc-xxx.S

![](.gitbook/assets/image%20%28154%29.png)

head.S

![](.gitbook/assets/image%20%2861%29.png)

head-common.S

![](.gitbook/assets/image%20%2819%29.png)

，但是，这个变量，不是一个指令吗。。。？？

![](.gitbook/assets/image%20%286%29.png)

暂时不太明白，\_\_cpu\_flush 里面不就一条指令吗。。

anyway, 一定是利用了 b 指令的编码，估计和相对偏移有关，注意 lr 最后赋值是 enable\_mmu，所以在那之前，主 CPU 已经初始化完成了，所以其他CPU还要做一个虚拟地址到总线地址的映射，以32bit来说就是 - 3G，不详细展开了。



![](.gitbook/assets/image%20%2864%29.png)

调用之后，回到之前设置的 lr 地址，来到下面的函数

![](.gitbook/assets/image%20%2827%29.png)

这上面 r13 是在最开始赋值的，地址就是下面这个函数

![](.gitbook/assets/image%20%28132%29.png)

![](.gitbook/assets/image%20%2845%29.png)

这时候终于来到了 C 的接口，舒服多了是吧~~

主要在涉及几件事：

1. booting CPU 此时正在等待 其他 CPU 
2. 其他CPU必须要初始化那个 logica map，也可能是 device tree初始化

more drafts

smp.c 中 arm\_dt\_init\_cpu\_maps,  setup\_processors, 另外还有 smp\_operations,  顺带也有 clk\_ops，应该分类在时钟那，但是呢，我怕忘记，所以先弄出来，总之这部分不能含糊，在开一个 GIC 的介绍吧，还没看见一个好点的。

关键是，这里面直接使用了 smp\_processor\_id 就拿到了cpu的id，仿佛从天而降一样，无法理解！！！

## setup\_arch

![](.gitbook/assets/image%20%2891%29.png)

![](.gitbook/assets/image%20%2869%29.png)

这个函数利用了 设备书，构建了这样一个MAP，但是我好奇，是否device tree中一定会有呢？

![](.gitbook/assets/image%20%2862%29.png)

![](.gitbook/assets/image%20%28113%29.png)

这里获取了 hwid，特别注意一下吧

## per-CPU 的建立

![](.gitbook/assets/image%20%2810%29.png)

这对之后的引导应该起到了非常重要的作用，先要理解这个问题！！！ 比如 current 是不是在之前已经被初始化为 cpu++ 所以后面来的 CPU 就必然是不同的等等

## Main CPU 的流程

![](.gitbook/assets/image%20%2829%29.png)

![](.gitbook/assets/image%20%2886%29.png)

![](.gitbook/assets/image%20%28109%29.png)

线程完成剩下的初始化。

![](.gitbook/assets/image%20%2872%29.png)

![](.gitbook/assets/image%20%28129%29.png)

为每个 CPU 初始化

![](.gitbook/assets/image%20%28111%29.png)

再次，改变了它的 CPU，不解释了吧，已经清楚了

![](.gitbook/assets/image%20%28153%29.png)

最终！！！！ 相信关键的一点，肯定是修改了 current-&gt;cpu....

![](.gitbook/assets/image%20%2897%29.png)

{% hint style="info" %}
也就是说，第一个上去的CPU，默认就是0，它唤醒的CPU，是我自己加+1，然后给你创建了 init 进程，那么  thread\_info-&gt;cpu  ,,,,,  it's up to the boot cpu~
{% endhint %}

## End

P.S. 比较关键的几个地方在于，

1.  CPU 可以进入 WFI WFE 等状态，可能是自己进入
2. 这些状态可以通过中断，或者寄存器唤醒
3. 具体措施比如 SGI，或者共享机制寄存器

它们的启动的基址也是可以i确定的，这是非常重要的，这之后我们再来看看，总之 boot cpu给它设定的路线，thread\_info-&gt;cpu 在它出生之前都已经确定，所以它一上来就可以调用 smp\_processor\_id 作为自己在 per-CPU 变量的偏移，这是非常有意思的，然而提到的人似乎并不多。

## reference

{% embed url="https://blog.csdn.net/lory17/article/details/50160303" %}

{% embed url="https://www.cnblogs.com/linhaostudy/p/9371562.html" %}

{% embed url="https://www.programering.com/a/MTMxQDMwATY.html" %}

