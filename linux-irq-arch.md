---
description: OOP ~
---

# Linux 中断架构

## Preface

这里准备介绍一下 Linux 中断的设计思想，当然，也会涉及到具体的中断过程，但是前人已经总结的非常好了，我等就不必重复劳动了。

内核设计结构的时候，我认为是区分哪些是改变的，哪些是不变的，然后把变化的抽象为一个不变的结果，把它们抽象为结构，和不变的固化在内核中，然后不断的修正，优化。

这也说明了一点，没有一开始的设计就是完美的，我们只有一个大概的理念，然后不断地完善它，所以能一开始就设计好的，必然是对整体都了解非常透彻的，这类人我们也称为 SE，当然他们的代码实力不一定强到哪里去就是了。

一个比较典型的就是 `VFS` 的设计，它将所有的文件系统的操作全都抽象到 file\_operations, 并且抽象出 `inode`, `dentry`, `superblock`, 有些文件系统本身设计上并没有这些结构，但是这些就要求驱动必须实现转换，这也是为什么 Linux 原生的文件系统会更有优势就在于此，当然，对于文件操作函数那些有些都是必须实现的，这样设计的好处就在于内核可以不做修改，专注于逻辑上的设计，而不是专注于不同的文件系统的操作上的不同，这就是框架的作用，先将不同给屏蔽，然后专注于整体的逻辑上的设计。

文件系统设备节点，以及自己的块设备。

再一个是驱动的架构，内核专注的一点，应该是在驱动的匹配，管理上，而不应该是由于不同的驱动而来的特殊操作，所以一个优秀的内核就应该先将不同的部分给抽象，然后专注于一些更加重要的部分。这部分可能是出错的恢复，日志，使用优化，以及管理上的需求。

OOP 的思想正是如此，抽象出不同的部分，内核正是无处不体现的着 OOP 的思想。

## 中断的大致设计

首先区分有哪些是变化的，

1. 中断控制器如何应答中断，如何获取中断号
2. 无数使用中断的设备，以及它们配置中断的方式
3. 中断向量表的配置（架构相关）

{% hint style="info" %}
为什么说只有代码量大了，才来说设计，因为没有功能，就没有设计一说，我是这么认为的，OS 非常多的模块混杂在一起，设计是慢慢的优化的，没有一开始就有非常牛的设计，当然，里面总的接口设计沿用的是最早 Unix 的内核，还是非常的简约，好用的
{% endhint %}

但这些变化的实现的目标都是一样的，得到中断号，开关中断，特权级设置等等，但是它们的具体实现都是不一样的，这时候内核就会把目标抽象为一个函数指针，让不同的中断控制器来注册，内核本质就是一个 Abstraction Core

所以内核设计一系列的接口，`request_irq` `enable_irq` `disable_irq` ... 这些都是抽象出来的接口，让需要使用中断的模块注册回调函数，实现起来就类似这样

```c

int request_irq(u32 irq, void * callback)
{
    get_irq_chip(irq)->request_irq(irq)
    set_call_back(irq, callback);
    return SUCCESS;
}

```

对于调用函数的模块来说，`irq_chip` 是透明的，可能是 `GIC v3 v2`... 内核关键需要抽象出一个接口，上面有需要 `irq_chip` 注册的函数指针~ ，借此内核就屏蔽了具体的中断控制器，然后对于 `irq -> callback` 这一层映射，对于 `irq_chip` 也是透明的，来看看一些具体的代码吧。

```c
/**
 * struct irq_chip - hardware interrupt chip descriptor
 *
 * @name:		name for /proc/interrupts
 * @irq_startup:	start up the interrupt (defaults to ->enable if NULL)
 * @irq_shutdown:	shut down the interrupt (defaults to ->disable if NULL)
 * @irq_enable:		enable the interrupt (defaults to chip->unmask if NULL)
 * @irq_disable:	disable the interrupt
 * @irq_ack:		start of a new interrupt
 * @irq_mask:		mask an interrupt source
 * @irq_mask_ack:	ack and mask an interrupt source
 * @irq_unmask:		unmask an interrupt source
 * @irq_eoi:		end of interrupt
 * @irq_set_affinity:	set the CPU affinity on SMP machines
 * @irq_retrigger:	resend an IRQ to the CPU
 * @irq_set_type:	set the flow type (IRQ_TYPE_LEVEL/etc.) of an IRQ
 * @irq_set_wake:	enable/disable power-management wake-on of an IRQ
 * @irq_bus_lock:	function to lock access to slow bus (i2c) chips
 * @irq_bus_sync_unlock:function to sync and unlock slow bus (i2c) chips
 * @irq_cpu_online:	configure an interrupt source for a secondary CPU
 * @irq_cpu_offline:	un-configure an interrupt source for a secondary CPU
 * @irq_suspend:	function called from core code on suspend once per chip
 * @irq_resume:		function called from core code on resume once per chip
 * @irq_pm_shutdown:	function called from core code on shutdown once per chip
 * @irq_calc_mask:	Optional function to set irq_data.mask for special cases
 * @irq_print_chip:	optional to print special chip info in show_interrupts
 * @irq_request_resources:	optional to request resources before calling
 *				any other callback related to this irq
 * @irq_release_resources:	optional to release resources acquired with
 *				irq_request_resources
 * @flags:		chip specific flags
 */
struct irq_chip {
	const char	*name;
	unsigned int	(*irq_startup)(struct irq_data *data);
	void		(*irq_shutdown)(struct irq_data *data);
	void		(*irq_enable)(struct irq_data *data);
	void		(*irq_disable)(struct irq_data *data);

	void		(*irq_ack)(struct irq_data *data);
	void		(*irq_mask)(struct irq_data *data);
	void		(*irq_mask_ack)(struct irq_data *data);
	void		(*irq_unmask)(struct irq_data *data);
	void		(*irq_eoi)(struct irq_data *data);

	int		(*irq_set_affinity)(struct irq_data *data, const struct cpumask *dest, bool force);
	int		(*irq_retrigger)(struct irq_data *data);
	int		(*irq_set_type)(struct irq_data *data, unsigned int flow_type);
	int		(*irq_set_wake)(struct irq_data *data, unsigned int on);
    ....

	unsigned long	flags;
};
```

这个结构就是内核抽象出来的，而实际我们如何设计回调函数有一万种方法，我们关键需要理解的是这种透明化的思想，**对于变化的，我们抽象出它们的目标，让变化的来注册，内核专注于统筹，专注于流程，而不是专注于变化**。

内核只需要提供一个注册函数，让控制器模块把这个结构体的地址传给内核，之后内核只需要建立某一个区域的 `irq =>` 这个地址的映射，即可完成操作

```c
0 - 255 | => irq_chip1
256 - 511 | => irq_chip2
```

比如在注册的时候加上一个范围即可，实际实现是通过注册 `irq_domain` 因为内核还实现了 `irq` 的虚拟化，比如每个控制器都支持 `0-255` 中断号，实际上会冲突，但是我们再加一层 `hw_irq` 到 `virt_irq` 的映射即可实现屏蔽，来看一个实际的例子

{% code title="/drivers/irqchip/irq-gic-v3.c" %}
```c
gic_data.domain = irq_domain_add_tree(node, &gic_irq_domain_ops,
					      &gic_data);
					
static const struct irq_domain_ops gic_irq_domain_ops = {
	...
	.alloc = gic_irq_domain_alloc,
	...
};

static struct irq_chip gic_chip = {
	.name			= "GICv3",
	.irq_mask		= gic_mask_irq,
	.irq_unmask		= gic_unmask_irq,
	.irq_eoi		= gic_eoi_irq,
	.irq_set_type		= gic_set_type,
	.irq_set_affinity	= gic_set_affinity,
};

static int gic_irq_domain_map(struct irq_domain *d, unsigned int irq,
			      irq_hw_number_t hw)
{
	...
	irq_set_chip_and_handler(irq, &gic_chip,
					 handle_percpu_devid_irq);
	...
	return 0;
}

static int gic_irq_domain_alloc(struct irq_domain *domain, unsigned int virq,
				unsigned int nr_irqs, void *arg)
{
	int i, ret;
	irq_hw_number_t hwirq;

	ret = gic_irq_domain_translate(domain, fwspec, &hwirq, &type);

	for (i = 0; i < nr_irqs; i++) {
		ret = gic_irq_domain_map(domain, virq + i, hwirq + i);
	}

	return 0;
}
```
{% endcode %}

中断控制器注册函数，当内核分配中断号（也就是其它模块调用 `request_irq`  的时候），内核就找到了这个 `domain`，然后调用 `alloc` 函数，内核给它的只是一个虚拟中断号，内核只需要知道虚拟中断号对应哪个 `domain`，即可，然后 `virt_irq` 对应的 `hw_irq` 只需要中断控制器来维护。

你只要明白了内核设计的思想，上面的代码实现有非常多方式，映射的话数组都可以实现。

总结一下

1. 中断控制器完成 `virt_irq -> hw_irq`，向内核注册函数指针
2. 内核封装接口，提供给其它所有的模块（包括中断控制器）

还有提一点，不同的架构中断向量表初始化方式不一样，所以初始化在不同的目录下

`arch/arm/kernel/irq.c` 比如这个目录，忘记是不是这个了，但是我阐述的只是分开实现的这一个事实，然后其中个比如实现了一个 `early_irq_init`

```c

vector_irq:
...

adr r1, arch_handler_irq
ldr pc, [r1]
...
```

`arch_handler_irq` 只是一个函数指针，这时候我们只需要中断控制器来注册，那么中断到来它就可以获取中断号了，它还需要转换为 `virt_irq` 然后内核就可以根据它来查找自己维护的表，调用其他模块注册的回调函数

所以，**中断控制器对于向量表的设置仍然是透明的**，**对于其它模块注册的回调函数仍然是透明的**，它只需要告诉内核，只跟内核互动，这种解耦非常适合学习。

内核需要这种 decoupling 来维持稳定，模块一旦耦合严重，就非常不好维护，那你就不能专心专注逻辑了， 别如多核访问的并发处理，首先我们把耦合带来的复杂给解决，那么其它一些问题才能集中解决，问题只在它的模块解决，也是一种抽象。

不管你看驱动架构，看文件系统架构（VFS）还是到现在服务器后端的一些框架，都有着非常好的设计思想，都会把变化的模块抽象，然后设计一个 Core，专门负责它们的交互，负责它们的解耦，然后自己在专注于逻辑，这里的专注，指所有的参与模块，比如中断控制器可以集中处理我们自己硬件的初始化，中断应答，而其他模块怎么使用，我不需要在乎，因为这不我们应该关心的，其他模块以此类推。

## 中断虚拟化\( Pending \)

刚刚的 `domain`，其实就是一个虚拟化的一个苗头，这是服务器的需求，大型机的中断成千上万，必须得有办法支持中断的虚拟化，其实最初的`8259-A` 只支持15个中断口，后面公用中断线不也是虚拟的一种手段？只是如今可能在接线上更加灵活配置，比如支持写某一个寄存器来触发，即采用  Message 的中断方式。

... on going

## ARM 32位 中断过程

References 里面作者介绍的非常清楚，我这里做一些补充吧，在 x86 架构下，中断到来的时候是非常的简单的，首先如果判断 cs 目前不在特权级，那么就根据 `tss` 中提供的 `sp0` 提供的栈基址，push 进 `ss sp eflags cs ip`（否则根据只入栈 `eflags cs ip`），这样的做法非常符合常理，也就是说， x86 不会提供多套寄存器\( ARM 我们也叫 banked \)，而是想办法利用栈来备份，ARM 则是有点不同，它会提供多套备份，中断来临的时候利用的是 banked 寄存器备份

```c
CPSR[7] = 1  关闭中断
CPSR[4:0] = IRQ_MODE
mov lr_irq, pc
mov spsr, cpsr
mov pc, vector_addr


总之就是备份了 pc, cpsr
切换到 irq 模式，并关闭中断，
跳转到响应函数

SCTLR 的 V bit 可以决定基址
其实有个 VBAR 寄存器也可以决定
所以说 ARM 和 x86 是非常类似的
```

{% hint style="info" %}
注意此时的 `sp` 也会切换到 `sp_irq`
{% endhint %}

你会发现，当我们进入了向量表指定的函数之后，当前的 `lr` 和 `cpsr` 都不是我们自己的，并且 PC 指针也被修改了，现在思考一下我们的目标是什么，备份！现在改变了什么？

1. `pc` 寄存器
2. `cpsr` 寄存器

这俩个寄存器的值跑哪儿了？ 答案是 `lr_irq spsr_irq`，所以我们跟在 x86 的操作是一样的，只是现在需要入栈在 `pc` 和 `cpsr` 对应的位置的，是这俩个寄存器。 所以，在中断处理函数一系列复杂的操作，都是为了备份在 中断之前的所有的寄存器。

`push { r0 - r15 } + cpsr`，只是其中俩个需要做些处理~ 现在看代码就会有种豁然开朗的感觉，现在我们来关注  SVC 状态下触发的中断，那些基址什么的就不细说了，看主线吧。

```c
.align       5
vector_irq:
       sub  lr, lr, 4    //因为异常发生时，cpu将pc地址+4赋值给lr，这里做修正。     
      
       @ Save r0, lr_<exception> (parent PC) and spsr_<exception>
       @ (parent CPSR)
       @    
       stmia       sp, {r0, lr}       //保存r0, lr，到irq堆栈中（每个异常都有属于自己的堆栈）
       mrs  lr, spsr     //lr保存spsr_irq的值，即usr状态的cpsr的值（见2.1）
       str   lr, [sp, #8]             // 保存 spsr到[sp+8]处
       
       @ Prepare for SVC32 mode.  IRQs remain disabled.
       @
       mrs  r0, cpsr
       eor   r0, r0,#( IRQ_MODE ^ SVC_MODE| PSR_ISETSTATE)    //  PSR_ISETSTATE：选择ARM/Thumb指令集                                             
       msr  spsr_cxsf, r0  //这里的cxsf表示从低到高分别占用的4个8bit的数据域

       @ the branch table must immediately follow this code

       and  lr, lr, #0x0f  
       mov  r0, sp              
       ldr   lr, [pc, lr, lsl #2]                                             
       movs      pc, lr      

       @ 最终就来到下面这个地方
__irq_svc:
	svc_entry
	irq_handler
	
```

注意上面使用的 `sp` 是 `sp_irq`，每次中断触发都是一样的，这个值在  `cpu_init()` 那里初始化，这里我们在 SMP 那一章节来讲讲吧，总之`sp`指向一个 12 字节的临时栈，上面备份了`r0`，`lr, cpsr` 寄存器，`r0` 备份是因为我们需要使用它，而 `lr, spsr_irq` 不正是我们需要备份的 `pc, cpsr` 吗？读者好好理解这句话，那么对这段代码就会有很清晰的理解。

```c
.macro	svc_entry, stack_hole=0
	sub	sp, sp, #(S_FRAME_SIZE + \stack_hole - 4)

	stmia	sp, {r1 - r12}

	ldmia	r0, {r3 - r5}
	add	r7, sp, #S_SP - 4	@ here for interlock avoidance
	mov	r6, #-1			@  ""  ""      ""       ""
	add	r2, sp, #(S_FRAME_SIZE + \stack_hole - 4)

	str	r3, [sp, #-4]!		@ save the "real" r0 copied
					@ from the exception stack

	mov	r3, lr

	@
	@ We are now ready to fill in the remaining blanks on the stack:
	@
	@  r2 - sp_svc
	@  r3 - lr_svc
	@  r4 - lr_<exception>, already fixed up for correct return/restart
	@  r5 - spsr_<exception>
	@  r6 - orig_r0 (see pt_regs definition in ptrace.h)
	@
	stmia	r7, {r2 - r6}

	.endm
```

{% hint style="info" %}
此时我们进入了 svc 模式，`sp lr cpsr` 都会被更换，但是  `r0` 指向的区域都将进入中断之前的寄存器备份了
{% endhint %}

上面的代码首先备份了 `r1 - r12`，`r0` 目前指向的是  `IRQ_MODE`的临时栈，而它的备份也在栈里面，备份了这些寄存器之后，我们就可以肆无忌惮的使用它们了，关键就是 `r7` 指向了寄存器备份区域的最顶端，它目前的任务是备份，`r13 - r15， cpsr， old_r0`，读者可以看看 `struct pt_regs` 结构，其实 72 个字节区域，正好可以保存 18 个寄存器，所以上面的代码就在获取剩下的 5 个寄存器，仔细看就可以知道，`lr` 是没有被使用的，所以可以直接备份，这里必须得明白一点，我们在中断模式操作的是另外一个 `lr_irq`，然后 `pc` 也就是在中断模式备份的那个 `lr_irq`，`sp` 指向没有开辟这段区域的位置，`cpsr` 类似，不解释了，`r6` 直接赋值为 -1

只要理解了这段代码的目的就是为了保存中断之前的所有寄存器，那么其实非常好阅读的，不然你不明白为什么会这么的绕，下面我们来看看恢复的操作，其实不就是直接弹出嘛。

```c
#ifndef CONFIG_THUMB2_KERNEL
	.macro	svc_exit, rpsr, irq = 0
	msr	spsr_cxsf, \rpsr
	
#if defined(CONFIG_CPU_V6) || defined(CONFIG_CPU_32v6K)
	@ We must avoid clrex due to Cortex-A15 erratum #830321
	sub	r0, sp, #4			@ uninhabited address
	strex	r1, r2, [r0]			@ clear the exclusive monitor
#endif

	ldmia	sp, {r0 - pc}^			@ load r0 - pc, cpsr
	.endm
```

对于 Thumb 2 的宏还有另外一个版本，那个版本应该是不支持，直接加载内存到 `pc`，所以使用了 `rfe` 指令，实现原理是一样的，我也没有遇到过那个情况。这个代码是非常的简单的，之前的备份可能还复杂，但这里却是一目了然，中间那个特殊情况可以忽略，应该是处理器的 BUG

###  为什么要在 SVC 模式

之所以备份的时候有点复杂，都是因为我们需要切换到 svc 模式，以及 ARM 在中断到来的时候自动帮我们修改了寄存器，而不是帮我们备份到栈中，切换到 svc 模式的时候又带来了寄存器的变换，所以又增加了点难度，那么究竟为何要切换呢？

原因是 `irq` 模式并没有什么特别，我们没有一定要为它准备一个 4KB 的栈空间，x86 模式也是这样，都是利用当前进程的内核栈，既然都有现成的，为何不用？所以我认为总共的原因有俩点

1. 统一，简化操作
2. 节省中断栈的管理

### ARM 汇编的一些知识

```c
ldr ip,[sp], #4 将sp中内容存入ip,之后sp=sp+4;
ldr ip,[sp, #4] 将sp+4这个新地址下内容存入ip,之后sp值保持不变
ldr ip,[sp, #4]!将sp+4这个新地址下内容存入ip,之后sp=sp+4将新地址值赋给sp
str ip,[sp], #4 将ip存入sp地址处,之后sp=sp+4;
str ip,[sp, #4] 将ip存入sp+4这个新地址,之后sp值保持不变
str ip,[sp, #4]!将ip存入sp+4这个新地址,之后sp=sp+4将新地址值赋给sp
```

```c
^'的理解
'^'是一个后缀标志,不能在User模式和Sys系统模式下使用该标志.该标志有两个存在目的:

1.对于LDM操作,同时恢复的寄存器中含有pc(r15)寄存器,那么指令执行的同时cpu自动
将spsr拷贝到cpsr中
如:在IRQ中断返回代码中
ldmfd {r4}           //读取sp中保存的的spsr值到r4中
msr spsr_cxsf,r4     //对spsr的所有控制为进行写操作,将r4的值全部注入spsr
ldmfd {r0-r12,lr,pc}^//当指令执行完毕,pc跳转之前,将spsr的值自动拷贝到cpsr中
 
2.数据的送入、送出发生在User用户模式下的寄存器,而非当前模式寄存器
如:ldmdb sp,{r0 - lr}^;表示sp栈中的数据回复到User分组寄存器r0-lr中,而不是恢复到当
前模式寄存器r0-lr,当然对于User,System,IRQ,SVC,Abort,Undefined这6种模式来说
r0-r12是共用的,只是r13和r14为分别独有,对于FIQ模式,仅仅r0-r7是和前6中模式的r0-r7共用,
r8-r14都是FIQ模式下专有.
```

那些 IA FA，就不说了，英文简写很容易记住，然后说说 `cxfs`

```c
c － control field maskbyte(PSR[7:0])
x － extension field maskbyte(PSR[15:8])
s － status field maskbyte(PSR[23:16)
f － flags field maskbyte(PSR[31:24]).

老式声明方式:cpsr_flg,cpsr_all在ADS中已经不在支持
cpsr_flg对应cpsr_f
cpsr_all对应cpsr_cxsf
```

## References

{% embed url="https://blog.csdn.net/qianlong4526888/article/details/27695583" %}

{% embed url="https://blog.csdn.net/qianlong4526888/article/details/8598524" %}

{% embed url="https://blog.csdn.net/sddzycnqjn/article/details/8220875" %}



