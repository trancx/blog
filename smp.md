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

### 具体的唤醒流程

```c
void __init smp_set_ops(struct smp_operations *ops)
{
	if (ops)
		smp_ops = *ops;
};
...
...
int __cpuinit boot_secondary(unsigned int cpu, struct task_struct *idle)
{
	if (smp_ops.smp_boot_secondary)
		return smp_ops.smp_boot_secondary(cpu, idle);
	return -ENOSYS;
}

```

而在另外一处

{% code-tabs %}
{% code-tabs-item title="/arch/arm/kernel/setup.c" %}
```c
	if (is_smp()) {
		if (!mdesc->smp_init || !mdesc->smp_init()) {
			if (psci_smp_available())
				smp_set_ops(&psci_smp_ops);
			else if (mdesc->smp)
				smp_set_ops(mdesc->smp);
		}
		smp_init_cpus();
		smp_build_mpidr_hash();
	}
	...
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% code-tabs %}
{% code-tabs-item title="/arch/arm/kernel/psci\_smp.c" %}
```c

static int psci_boot_secondary(unsigned int cpu, struct task_struct *idle)
{
	if (psci_ops.cpu_on)
		return psci_ops.cpu_on(cpu_logical_map(cpu),
					virt_to_idmap(&secondary_startup));
	return -ENODEV;
}

const struct smp_operations psci_smp_ops __initconst = {
	.smp_boot_secondary	= psci_boot_secondary,
#ifdef CONFIG_HOTPLUG_CPU
	.cpu_disable		= psci_cpu_disable,
	.cpu_die		= psci_cpu_die,
	.cpu_kill		= psci_cpu_kill,
#endif
};
```
{% endcode-tabs-item %}
{% endcode-tabs %}



```c
static void __init psci_0_2_set_functions(void)
{
	...
	psci_function_id[PSCI_FN_CPU_OFF] = PSCI_0_2_FN_CPU_OFF;
	psci_ops.cpu_off = psci_cpu_off;

	psci_function_id[PSCI_FN_CPU_ON] = PSCI_FN_NATIVE(0_2, CPU_ON);
	psci_ops.cpu_on = psci_cpu_on;
	...
}

int __init psci_dt_init(void)
{
	struct device_node *np;
	const struct of_device_id *matched_np;
	psci_initcall_t init_fn;

	np = of_find_matching_node_and_match(NULL, psci_of_match, &matched_np);

	if (!np || !of_device_is_available(np))
		return -ENODEV;

	init_fn = (psci_initcall_t)matched_np->data;
	return init_fn(np);
}
```

设备树初始化的，关注关键的几个

```c
static psci_fn *invoke_psci_fn;
...

static unsigned long __invoke_psci_fn_smc(unsigned long function_id,
			unsigned long arg0, unsigned long arg1,
			unsigned long arg2)
{
	struct arm_smccc_res res;

	arm_smccc_smc(function_id, arg0, arg1, arg2, 0, 0, 0, 0, &res);
	return res.a0;
}

static void set_conduit(enum psci_conduit conduit)
{
	switch (conduit) {
	case PSCI_CONDUIT_HVC:
		invoke_psci_fn = __invoke_psci_fn_hvc;
		break;
	case PSCI_CONDUIT_SMC:
		invoke_psci_fn = __invoke_psci_fn_smc;
		break;
	default:
		WARN(1, "Unexpected PSCI conduit %d\n", conduit);
	}

	psci_ops.conduit = conduit;
}

static int psci_cpu_on(unsigned long cpuid, unsigned long entry_point)
{
	int err;
	u32 fn;

	fn = psci_function_id[PSCI_FN_CPU_ON];
	err = invoke_psci_fn(fn, cpuid, entry_point, 0);
	return psci_to_linux_errno(err);
}

```

所以关键就来到了这

{% code-tabs %}
{% code-tabs-item title="/include/linux/arm-smccc.h" %}
```c
/**
 * __arm_smccc_hvc() - make HVC calls
 * @a0-a7: arguments passed in registers 0 to 7
 * @res: result values from registers 0 to 3
 * @quirk: points to an arm_smccc_quirk, or NULL when no quirks are required.
 *
 * This function is used to make HVC calls following SMC Calling
 * Convention.  The content of the supplied param are copied to registers 0
 * to 7 prior to the HVC instruction. The return values are updated with
 * the content from register 0 to 3 on return from the HVC instruction.  An
 * optional quirk structure provides vendor specific behavior.
 */
asmlinkage void __arm_smccc_hvc(unsigned long a0, unsigned long a1,
			unsigned long a2, unsigned long a3, unsigned long a4,
			unsigned long a5, unsigned long a6, unsigned long a7,
			struct arm_smccc_res *res, struct arm_smccc_quirk *quirk);

#define arm_smccc_smc(...) __arm_smccc_smc(__VA_ARGS__, NULL)
```
{% endcode-tabs-item %}
{% endcode-tabs %}

应该可以猜测的到，是汇编实现的了

{% code-tabs %}
{% code-tabs-item title="/arch/arm/kernel/smccc-call.S" %}
```c
	/*
	 * Wrap c macros in asm macros to delay expansion until after the
	 * SMCCC asm macro is expanded.
	 */
	.macro SMCCC_SMC
	__SMC(0)
	.endm

	.macro SMCCC instr
UNWIND(	.fnstart)
	mov	r12, sp
	push	{r4-r7}
UNWIND(	.save	{r4-r7})
	ldm	r12, {r4-r7}   @ r0-r7 8 args
	\instr
	pop	{r4-r7}
	ldr	r12, [sp, #(4 * 4)]  @ now r12 points at &res
	stm	r12, {r0-r3}
	bx	lr
UNWIND(	.fnend)
	.endm

/*
 * void smccc_smc(unsigned long a0, unsigned long a1, unsigned long a2,
 *		  unsigned long a3, unsigned long a4, unsigned long a5,
 *		  unsigned long a6, unsigned long a7, struct arm_smccc_res *res,
 *		  struct arm_smccc_quirk *quirk)
 */
ENTRY(__arm_smccc_smc)
	SMCCC SMCCC_SMC
ENDPROC(__arm_smccc_smc)
```
{% endcode-tabs-item %}
{% endcode-tabs %}

`SMC` 是一个宏，分别针对 `Thumb` 和 `ARM` 下做了区分，这两者的指令集不一样。上面的操作想要理解，需要知道 `ARM Calling Convention`，我以前写过一篇只是简单的介绍了，但是最后的一张图便可以理解，当 `bl` 指令调用的时候，`sp` 指向的是 第五个参数的地址，理解这个，上述的操作就很自然了

## End

P.S. 比较关键的几个地方在于，

1.  CPU 可以进入 WFI WFE 等状态，可能是自己进入
2. 这些状态可以通过中断，或者寄存器唤醒
3. 具体措施比如 SGI，或者共享机制寄存器

它们的启动的基址也是可以i确定的，这是非常重要的，这之后我们再来看看，总之 boot cpu给它设定的路线，thread\_info-&gt;cpu 在它出生之前都已经确定，所以它一上来就可以调用 smp\_processor\_id 作为自己在 per-CPU 变量的偏移，这是非常有意思的，然而提到的人似乎并不多。

---- 补充于 2019年9月17日22:16:12

实际上，在 ATF 作用下，唤醒 CPU 需要用 smc  exception call

## reference

{% embed url="https://blog.csdn.net/lory17/article/details/50160303" %}

{% embed url="https://www.cnblogs.com/linhaostudy/p/9371562.html" %}

{% embed url="https://www.programering.com/a/MTMxQDMwATY.html" %}

