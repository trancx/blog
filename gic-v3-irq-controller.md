# GIC V3 中断控制器

## 序

作为学习的记录吧，网上分析源码还是比较少的，大部分都是阐述一些架构的知识，我在这个基础上结合一下，因为最近需要移植这部分的代码。

## Draft

![](.gitbook/assets/image%20%28173%29.png)

概述： TODO

### GICD

Distributor 顾名思义把，它的作用就是管理第 32 - 1020 SPI 中断，那么它的作用有哪些呢

1. SPI 中断的特权级
2. SPI 中断的开关，Enable Disable
3. SPI 所属组，0 还是 1
4. SPI 中断路由，Affinity 设置
5. SPI 中断触发方式，低电平，边缘触发...

{% hint style="info" %}
注意 SGI 和 PPI 我们一般只会在 GICR 那里处理，因为它是属于每一个CPU的，可以这么理解，但是 GICD 也能配置，应该对所有的 Redistributor 都起作用，还没测试过

一般 GICD\_xxx&lt;n&gt; 在每个 Redistributor 区域，都会有一个 GICR\_xxx0 其实就是说，它的作用就是移动到了 GICR 实现，每个核都能独立配置
{% endhint %}

对应的寄存器如下：

![](.gitbook/assets/image%20%2876%29.png)

Lower means Priority Higher

![](.gitbook/assets/image%20%2831%29.png)

这里就不具体截图了，值得注意的是，想要保证 IRQ 关闭，必须得写 GICD\_ICENABLER 然后保证开启就得写 GICD\_ISENABLER，当然有的时候我们可能不在乎别的中断，只要保证一个中断开启或者关闭

#### 组 和 Affinity

![](.gitbook/assets/image%20%2812%29.png)

比较特别的是 GICD\_IROUTER 64位， 并且  n = 32 - 1020，0-31 也是存在的，只是被保留了

`The maximum value of n is given by (32*(GICD_TYPER.ITLinesNumber+1) - 1). GICD_IROUTER registers where n=0 to 31 are reserved.`

![](.gitbook/assets/image%20%28132%29.png)

而其中的 Mode 字段

![](.gitbook/assets/image%20%28161%29.png)

32位的 ARM 使用的时候会有一个问题，那就是 AArch32 根本不存在 Aff3 字段，这个也是之后我需要面对的问题。

中断触发的设置是 GICD\_ICFGR&lt;n&gt; 一样，具体的在每个CPU保留的去看白皮书吧，这里不详细介绍了~

![](.gitbook/assets/image%20%2827%29.png)

#### GICD\_TYPER GICD\_CTLR

![](.gitbook/assets/image%20%2832%29.png)

注意不同的模式下这个位有点不同，这个根据具体代码介绍，然后这里有非常关键的

Affinity Enable 以及 Group Interrupt，理解了这俩个概念，才好掌握这个寄存器的作用

![](.gitbook/assets/image%20%28116%29.png)

### GICR

![](.gitbook/assets/image%20%2815%29.png)

清晰明了，首先每一个 Core（即 CPU interface）有一个 Redistributor Space，然后其中又可以分为 RD\_base SGI\_base， 来看看它的基本作用有哪些，

1. 那些在GICD没有配置的，尤其是 GICD\_XXX&lt;n&gt;的，这里一般都有 GICR\_xxx0 其实就是和它对应，比如中断开关，
2. Redistributor 很重要的作用就是支持 LPI，因为 LPI 就是导向到 Rdisttibutor 
3. RD\_base 的配置不多，主要还是在 SGI\_base，那里是和 SGI PPI 打交道的

白皮书的每一个区域，都有不同的介绍，下面只讲 SGI\_BASE 这是我们关注的重点

![](.gitbook/assets/image%20%2848%29.png)

### GICC & System Register

![](.gitbook/assets/image%20%28124%29.png)

GICC 支持 MMIO 也支持 System Register 俩种方式，之所以把 CPU interface 集成进来，就是因为 CPU 是要经常访问这些寄存器的，所以速度也是一个重要的指标，具体看refs\[2\]，那里有一些解释。

![](.gitbook/assets/image%20%2867%29.png)

中断处理过程中，涉及到的主要是这三个寄存器

* GICC\_IAR 读取一个中断号并 Acknowledge
* GICC\_EOIR 标记一个中断的结束，实际上并没有处理完成
* GICC\_DIR   Deactivate 一个中断，这标记这一个中断的结束

当然，这取决于模式

`A write to this register performs priority drop for the specified interrupt and, if the appropriate GICC_CTLR.EOImodeS or GICC_CTLR.EOImodeNS field == 0, also deactivates the interrupt.`

不过，永远是 Deactivate 才意味着中断结束，不同的模式只是自动的处理了某一过程。

现在来关注俩个控制寄存器

GICC\_CTLR

![](.gitbook/assets/image%20%28184%29.png)

我们考虑的情况都是 GICD\_CTLR.DS == 1 情况，也就是关闭了 Secure 的模式

![](.gitbook/assets/image%20%2897%29.png)

此时比较重要的是几个位

![](.gitbook/assets/image%20%28105%29.png)

![](.gitbook/assets/image%20%281%29.png)

这其实说明，真正我我们只需要将所有中断归为一个类，当不需要路由到某一个 CPU 的时候，只需要将 CPU 的 CTRL 那一位置0，即可屏蔽掉某一组中断，这有一个重要的思想，那就是中断可以在多处被限制，Distributor, Redistributor and CPU Interface，而对于CPU而言，大多数时候只需要管好自己的 Interface 即可，其他控制都是在初始化的时候才有，或者是因为要修改 affinity routing

GICC\_PMR

![](.gitbook/assets/image%20%2862%29.png)

猜想: 设成0，即可屏蔽所有中断

## References

{% embed url="https://blog.csdn.net/yhb1047818384/article/details/86708769" %}

{% embed url="http://www.lujun.org.cn/?p=3874/" %}

{% embed url="https://blog.csdn.net/sunsissy/article/details/73842533" %}



