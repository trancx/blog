---
description: smp boot notes
---

# SMP 引导流程

## Preface

多核的出现使得引导的过程有些许不同，但是主线其实和以前是类似的，在实习的过程中也遇到了需要多核引导的需求，所以研究了一阵子

### 大致过程

在之前都是类似的，由 `boot cpu` 完成引导，对于其他核，可能系统架构中存在电源管理的模块，需要写相应的寄存器来达到唤醒的操作，至于使能之后的默认地址，也一定是架构相关的，比如默认是物理地址的 `0x0000000`

那么当`boot cpu` 完成基础的模块初始化之后，就可以尝试唤醒其他核了，这部分暂时我还没细究，但是关键一定是操作寄存器来使能，可能还有寄存器用来填写使能之后的初始指令的地址，因为这都是可以实现的。

为了能让 `smp_processor_id` 起作用，跳转到 C 代码之前的关键一步就是得设置栈寄存器指向 init\_task ，那么其他的核引导的时候，关键的一点就在于让后面引导的 CPU 拿到自己的栈页面，所以 `boot cpu` 做的事情就是分配一个页面，然后把地址放在固定的地方，然后唤醒一个核，让它能顺利的引导。

其他的核仍要做一些相应的初始化的，因为很多的寄存器都是每个核都有一个**备份**，比如仍要打开`MMU`，以及配置中断模式下的栈，初始化自己的时钟（还包括对中断控制器一些配置）等等，这些初始化函数，都是在 `boot cpu` 做初始化的代码注册的，让后面的核只需要做一些配置，大部分的工作已经完成了，当然了，它还得通知 `boot cpu` 完成了初始化，它还得继续唤醒下一个核。

## Main CPU

### 汇编过程

下面从 `boot cpu` 的角度出发，这类的代码非常多的分析，我会尽量简单的描述，这里的关键有几点

1. 判断处理器类型，完成特定的初始化
2. 使能 `MMU`
3. 解压 `.data`  的数据
4. 初始化 `bss`
5. 初始化栈寄存器（指向 `init_task`）

这几点的关键就不需要赘述了吧，提一提第一点是如何实现的吧

{% code-tabs %}
{% code-tabs-item title="/arch/arm/kernel/head.S" %}
```c
__HEAD
ENTRY(stext)

mrc	p15, 0, r9, c0, c0		@ get processor id
bl	__lookup_processor_type		@ r5=procinfo r9=cpuid
movs	r10, r5				@ invalid processor (r5=0)?
...
@ before turn on MMU
add	pc, r10, #PROCINFO_INITFUNC
....
```
{% endcode-tabs-item %}
{% endcode-tabs %}

关键是 `procinfo` 这个结构，至于那个宏正是用 `offset_of` 计算出来的，`pc` 的指令现在正是在这个结构的 `__cpu_flush` 这里存放的指令是什么呢？关键得知道，`r5` 是怎么得到的

```c
struct proc_info_list {
	...
	unsigned long		__cpu_mm_mmu_flags;	/* used by head.S */
	unsigned long		__cpu_io_mmu_flags;	/* used by head.S */
	unsigned long		__cpu_flush;		/* used by head.S */
	...
};

```

```c
/*
 * Read processor ID register (CP#15, CR0), and look up in the linker-built
 * supported processor list.  Note that we can't use the absolute addresses
 * for the __proc_info lists since we aren't running with the MMU on
 * (and therefore, we are not in the correct address space).  We have to
 * calculate the offset.
 *
 *	r9 = cpuid
 * Returns:
 *	r3, r4, r6 corrupted
 *	r5 = proc_info pointer in physical address space
 *	r9 = cpuid (preserved)
 */
	__CPUINIT
__lookup_processor_type:
	adr	r3, __lookup_processor_type_data
	ldmia	r3, {r4 - r6}
	sub	r3, r3, r4			@ get offset between virt&phys
	add	r5, r5, r3			@ convert virt addresses to
	add	r6, r6, r3			@ physical address space
1:	ldmia	r5, {r3, r4}			@ value, mask
	and	r4, r4, r9			@ mask wanted bits
	teq	r3, r4
	beq	2f
	add	r5, r5, #PROC_INFO_SZ		@ sizeof(proc_info_list)
	cmp	r5, r6
	blo	1b
	mov	r5, #0				@ unknown processor
2:	mov	pc, lr
ENDPROC(__lookup_processor_type)
 
 /*
 * Look in <asm/procinfo.h> for information about the __proc_info structure.
 */
	.align	2
	.type	__lookup_processor_type_data, %object
__lookup_processor_type_data:
	.long	.
	.long	__proc_info_begin
	.long	__proc_info_end
	.size	__lookup_processor_type_data, . - __lookup_processor_type_dat
```

其实就是预先所有的结构都存在了一块区域，相当于数组，然后根据里面预先定义的值和处理器的标识符比较，如果相等就说明符合了，那么 `r5` 也就理所当然的指向对应的 `procinfo` 了，涉及到的两个变量是由链接脚本指定的

{% code-tabs %}
{% code-tabs-item title="vmlinux.lds.S" %}
```c
. = ALIGN(4);							\
	VMLINUX_SYMBOL(__proc_info_begin) = .;				\
	*(.proc.info.init)						\
	VMLINUX_SYMBOL(__proc_info_end) = .;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

以 v7 处理器为例子，

{% code-tabs %}
{% code-tabs-item title="/arch/arm/mm/proc-v7.S" %}
```c
.section ".proc.info.init", #alloc, #execinstr

	/*
	 * Standard v7 proc info content
	 */
.macro __v7_proc initfunc, mm_mmuflags = 0, io_mmuflags = 0, hwcaps = 0, proc_fns = v7_processor_functions
	ALT_SMP(.long	PMD_TYPE_SECT | PMD_SECT_AP_WRITE | PMD_SECT_AP_READ | \
			PMD_SECT_AF | PMD_FLAGS_SMP | \mm_mmuflags)
	ALT_UP(.long	PMD_TYPE_SECT | PMD_SECT_AP_WRITE | PMD_SECT_AP_READ | \
			PMD_SECT_AF | PMD_FLAGS_UP | \mm_mmuflags)
	.long	PMD_TYPE_SECT | PMD_SECT_AP_WRITE | \
		PMD_SECT_AP_READ | PMD_SECT_AF | \io_mmuflags
	W(b)	\initfunc
	.long	cpu_arch_name
	.long	cpu_elf_name
	.long	HWCAP_SWP | HWCAP_HALF | HWCAP_THUMB | HWCAP_FAST_MULT | \
		HWCAP_EDSP | HWCAP_TLS | \hwcaps
	.long	cpu_v7_name
	.long	\proc_fns
	.long	v7wbi_tlb_fns
	.long	v6_user_fns
	.long	v7_cache_fns
.endm

/*
	 * Match any ARMv7 processor core.
	 */
	.type	__v7_proc_info, #object
__v7_proc_info:
	.long	0x000f0000		@ Required ID value
	.long	0x000f0000		@ Mask for ID
	__v7_proc __v7_setup
	.size	__v7_proc_info, . - __v7_proc_info
```
{% endcode-tabs-item %}
{% endcode-tabs %}

所以下面这条指令会跳转到`__v7_setup`这个地方

```c
add	pc, r10, #PROCINFO_INITFUNC
```

里面的代码现在我还看不出名堂，都是一些寄存器的配置，不是我们这里讨论的重点了

### C 代码

不再分析 `start_kernel`，这里的重点是 `smp`

其他核心的唤醒其实在非常后，其实都是由创建的线程来完成初始化的，

{% code-tabs %}
{% code-tabs-item title="/init/main.c" %}
```c
static int __ref kernel_init(void *unused)
{
	kernel_init_freeable();
	...
}

static noinline void __init kernel_init_freeable(void)
{
	...	
	smp_init();
	...
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

上面的函数一路往下就会来到

```c
static int __cpuinit _cpu_up(unsigned int cpu, int tasks_frozen)
{
	int ret, nr_calls = 0;
	void *hcpu = (void *)(long)cpu;
	unsigned long mod = tasks_frozen ? CPU_TASKS_FROZEN : 0;
	struct task_struct *idle;

	idle = idle_thread_get(cpu);
	...
	/* Arch-specific enabling code. */
	ret = __cpu_up(cpu, idle);

	/* Wake the per cpu threads */
	smpboot_unpark_threads(cpu);

	...

	return ret;
}
```

每一个`cpu`都会创建一个 `idle thread`，不仅如此，`task` 和 `stack` 本就是密不可分的，所以也可以说，分配了内核栈区域

```c
int __cpuinit __cpu_up(unsigned int cpu, struct task_struct *idle)
{
	int ret;

	/*
	 * We need to tell the secondary core where to find
	 * its stack and the page tables.
	 */
	secondary_data.stack = task_stack_page(idle) + THREAD_START_SP;
	secondary_data.pgdir = virt_to_phys(idmap_pgd);
	secondary_data.swapper_pg_dir = virt_to_phys(swapper_pg_dir);
	__cpuc_flush_dcache_area(&secondary_data, sizeof(secondary_data));
	outer_clean_range(__pa(&secondary_data), __pa(&secondary_data + 1));

	/*
	 * Now bring the CPU into our world.
	 */
	ret = boot_secondary(cpu, idle);
	if (ret == 0) {
		/*
		 * CPU was successfully started, wait for it
		 * to come online or time out.
		 */
		wait_for_completion_timeout(&cpu_running,
						 msecs_to_jiffies(1000));
	} 

	secondary_data.stack = NULL;
	secondary_data.pgdir = 0;

	return ret;
}
```

上面赋值的全局变量，正是当其他的核引导的时候会访问的，当然了，我们需要注意一点，`Linux` 正是用栈来判断 `CPU` 的 `id` 的，而这个 `id` 其实都是 `boot CPU` 设置的，因为它们的栈都是它分配（也可以说 `init_task`，这两者其实紧密相连）

```c
int __cpuinit boot_secondary(unsigned int cpu, struct task_struct *idle)
{
	if (smp_ops.smp_boot_secondary)
		return smp_ops.smp_boot_secondary(cpu, idle);
	return -ENOSYS;
}
```

接下来就涉及到了具体的架构了，但最后都会让 `cpu` 执行到 `secondary_startup` 这个函数，先不看具体的架构，来关注一下其他核

## Secondary CPU

{% hint style="info" %}
代码有些更改了顺序，更好理解
{% endhint %}

```c
/*
 * Initial data for bringing up a secondary CPU.
 */
struct secondary_data {
	unsigned long pgdir;
	unsigned long swapper_pg_dir;
	void *stack;
};
extern struct secondary_data secondary_data;
```

```c
.type	__secondary_data, %object
__secondary_data:
	.long	.
	.long	secondary_data
	.long	__secondary_switched

ENTRY(secondary_startup)
	/*
	 * Common entry point for secondary CPUs.
	 *
	 * Ensure that we're in SVC mode, and IRQs are disabled.  Lookup
	 * the processor type - there is no need to check the machine type
	 * as it has already been validated by the primary processor.
	 */
	safe_svcmode_maskall r9

	mrc	p15, 0, r9, c0, c0		@ get processor id
	bl	__lookup_processor_type
	movs	r10, r5				@ invalid processor?
	moveq	r0, #'p'			@ yes, error 'p'
	beq	__error_p

	/*
	 * Use the page tables supplied from  __cpu_up.
	 */
	adr	r4, __secondary_data
	ldmia	r4, {r5, r7, r12}		@ address to jump to after
	sub	lr, r4, r5			@ mmu has been enabled
	ldr	r4, [r7, lr]			@ get secondary_data.pgdir
	add	r7, r7, #4
	ldr	r8, [r7, lr]			@ get secondary_data.swapper_pg_dir
	adr	lr, BSYM(__enable_mmu)		@ return address
	mov	r13, r12			@ __secondary_switched address
 ARM(	add	pc, r10, #PROCINFO_INITFUNC	) @ initialise processor
						  				@ (return control reg)
ENDPROC(secondary_startup)

ENTRY(__secondary_switched)
	ldr	sp, [r7, #4]			@ get secondary_data.stack
	mov	fp, #0
	b	secondary_start_kernel
ENDPROC(__secondary_switched)
```

所以，启用了 `MMU` 之后，最后来到了 `secondary_start_kernel`

```c
/*
 * This is the secondary CPU boot entry.  We're using this CPUs
 * idle thread stack, but a set of temporary page tables.
 */
asmlinkage void __cpuinit secondary_start_kernel(void)
{
	struct mm_struct *mm = &init_mm;
	unsigned int cpu;

	/*
	 * The identity mapping is uncached (strongly ordered), so
	 * switch away from it before attempting any exclusive accesses.
	 */
	cpu_switch_mm(mm->pgd, mm);
	local_flush_bp_all();
	enter_lazy_tlb(mm, current);
	local_flush_tlb_all();

	cpu = smp_processor_id();

	...
	cpu_init();
	/*
	 * Give the platform a chance to do its own initialisation.
	 */
	if (smp_ops.smp_secondary_init)
		smp_ops.smp_secondary_init(cpu);

	notify_cpu_starting(cpu);

	/*
	 * OK, now it's safe to let the boot CPU continue.  Wait for
	 * the CPU migration code to notice that the CPU is online
	 * before we continue - which happens after __cpu_up returns.
	 */
	set_cpu_online(cpu, true);
	complete(&cpu_running);

	/*
	 * Setup the percpu timer for this CPU.
	 */
	percpu_timer_setup();

	local_irq_enable();
	local_fiq_enable();

	/*
	 * OK, it's off to the idle thread for us
	 */
	cpu_startup_entry(CPUHP_ONLINE);
}

```

几个比较关键的点

1. `MMU` 相关
2. 对于  `ARM` 还需要初始化其他模式的`sp` 寄存器
3. 调用 `notify_cpu_starting` 调用其他模块注册的初始化函数
4. 通知 `boot cpu` 完成初始化，注意此时 `boot cpu` 正在等待
5. 进入 `idle` 循环，`idle` 框架也是值得探索的，但这不是重点

这些初始化，保证了后面起来的 `CPU`，也有自己时钟，也能完成进程调度等功能，已经和主 `CPU` 没有任何区别

这里也有一个有意思的问题，就是为什么刚起来的 `CPU`，就可以使用 `smp_processor_id`，正是因为它的栈（或者说 `idle task`）是由 `boot cpu` 所指定，`CPU id` 也正是如此，当然这个 `id` 是虚拟 `id`，`CPU` 也有自己的 `id` 寄存器，Linux 使用的是前者，因为这屏蔽了架构，不同的架构的 `CPU id` 可能千奇百怪（`ARM` 就是 `Mpidr` 寄存器）

{% hint style="info" %}
也就是说，引导`CPU`，默认 `id`就是 0，它唤醒的`CPU`，是我自己加`+1`，然后给你创建了 `init` 进程，那么  `thread_info->cpu` `it's up to the boot cpu~`
{% endhint %}

## PSCI 平台的唤醒流程

下面以实习公司的架构唤醒流程为例，看看到底是怎么个玩法~ 接着刚刚分析的 `smp_ops`

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

这里嵌套了几层

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

来到了这

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

{% hint style="info" %}
实际操作的时候发现，`SMC` 系统调用进入 `EL3` 后挂死，最后发现问题是因为之前的代码手动写了对应的`CPU`的内部使能寄存器
{% endhint %}

## ATF 

虽然还没分析过它的存在，其实跟 `x86` 的操作系统非常的相似，以前的 `ARM` 并不会对内存寄存器等资源设定屏蔽层，如今引入了，而实现就是一个 `eret` 指令，那不是跟 `x86` 不谋而合吗，[这里](https://github.com/ARM-software/arm-trusted-firmware)是它的 `git`

```c
/*
 * Top-level Standard Service SMC handler. This handler will in turn dispatch
 * calls to PSCI SMC handler
 */
static uintptr_t std_svc_smc_handler(uint32_t smc_fid,
			     u_register_t x1,
			     u_register_t x2,
			     u_register_t x3,
			     u_register_t x4,
			     void *cookie,
			     void *handle,
			     u_register_t flags)
{
	/*
	 * Dispatch PSCI calls to PSCI SMC handler and return its return
	 * value
	 */
	if (is_psci_fid(smc_fid)) {
		uint64_t ret;

		ret = psci_smc_handler(smc_fid, x1, x2, x3, x4,
		    cookie, handle, flags);

#if ENABLE_RUNTIME_INSTRUMENTATION
		PMF_CAPTURE_TIMESTAMP(rt_instr_svc,
		    RT_INSTR_EXIT_PSCI,
		    PMF_NO_CACHE_MAINT);
#endif

		SMC_RET1(handle, ret);
	}
	...
}
```

{% code-tabs %}
{% code-tabs-item title="arm-trusted-firmware/lib/psci/psci\_main.c" %}
```c
/*******************************************************************************
 * PSCI top level handler for servicing SMCs.
 ******************************************************************************/
u_register_t psci_smc_handler(uint32_t smc_fid,
			  u_register_t x1,
			  u_register_t x2,
			  u_register_t x3,
			  u_register_t x4,
			  void *cookie,
			  void *handle,
			  u_register_t flags)
{
	u_register_t ret;

	if (is_caller_secure(flags))
		return (u_register_t)SMC_UNK;

	/* Check the fid against the capabilities */
	if ((psci_caps & define_psci_cap(smc_fid)) == 0U)
		return (u_register_t)SMC_UNK;

	if (((smc_fid >> FUNCID_CC_SHIFT) & FUNCID_CC_MASK) == SMC_32) {
		/* 32-bit PSCI function, clear top parameter bits */

		uint32_t r1 = (uint32_t)x1;
		uint32_t r2 = (uint32_t)x2;
		uint32_t r3 = (uint32_t)x3;

		switch (smc_fid) {
		case PSCI_VERSION:
			ret = (u_register_t)psci_version();
			break;

		case PSCI_CPU_OFF:
			ret = (u_register_t)psci_cpu_off();
			break;

		case PSCI_CPU_SUSPEND_AARCH32:
			ret = (u_register_t)psci_cpu_suspend(r1, r2, r3);
			break;

		case PSCI_CPU_ON_AARCH32:
			ret = (u_register_t)psci_cpu_on(r1, r2, r3);
			break;

		...
		...
		}
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

同一个文件中，还可以找到

```c
/*******************************************************************************
 * PSCI frontend api for servicing SMCs. Described in the PSCI spec.
 ******************************************************************************/
int psci_cpu_on(u_register_t target_cpu,
		uintptr_t entrypoint,
		u_register_t context_id)

{
	int rc;
	entry_point_info_t ep;

	/* Determine if the cpu exists of not */
	rc = psci_validate_mpidr(target_cpu);
	if (rc != PSCI_E_SUCCESS)
		return PSCI_E_INVALID_PARAMS;

	/* Validate the entry point and get the entry_point_info */
	rc = psci_validate_entry_point(&ep, entrypoint, context_id);
	if (rc != PSCI_E_SUCCESS)
		return rc;

	/*
	 * To turn this cpu on, specify which power
	 * levels need to be turned on
	 */
	return psci_cpu_on_start(target_cpu, &ep);
}
```

{% code-tabs %}
{% code-tabs-item title="/lib/psci/psci\_on.c" %}
```c
/*******************************************************************************
 * Generic handler which is called to physically power on a cpu identified by
 * its mpidr. It performs the generic, architectural, platform setup and state
 * management to power on the target cpu e.g. it will ensure that
 * enough information is stashed for it to resume execution in the non-secure
 * security state.
 *
 * The state of all the relevant power domains are changed after calling the
 * platform handler as it can return error.
 ******************************************************************************/
int psci_cpu_on_start(u_register_t target_cpu,
		      const entry_point_info_t *ep)
{
	int rc;
	aff_info_state_t target_aff_state;
	int target_idx = plat_core_pos_by_mpidr(target_cpu);


	/* Protect against multiple CPUs trying to turn ON the same target CPU */
	psci_spin_lock_cpu(target_idx);

	/*
	 * Generic management: Ensure that the cpu is off to be
	 * turned on.
	 * Perform cache maintanence ahead of reading the target CPU state to
	 * ensure that the data is not stale.
	 * There is a theoretical edge case where the cache may contain stale
	 * data for the target CPU data - this can occur under the following
	 * conditions:
	 * - the target CPU is in another cluster from the current
	 * - the target CPU was the last CPU to shutdown on its cluster
	 * - the cluster was removed from coherency as part of the CPU shutdown
	 *
	 * In this case the cache maintenace that was performed as part of the
	 * target CPUs shutdown was not seen by the current CPU's cluster. And
	 * so the cache may contain stale data for the target CPU.
	 */
	flush_cpu_data_by_index((unsigned int)target_idx,
				psci_svc_cpu_data.aff_info_state);
	rc = cpu_on_validate_state(psci_get_aff_info_state_by_idx(target_idx));
	if (rc != PSCI_E_SUCCESS)
		goto exit;

	/*
	 * Call the cpu on handler registered by the Secure Payload Dispatcher
	 * to let it do any bookeeping. If the handler encounters an error, it's
	 * expected to assert within
	 */
	if ((psci_spd_pm != NULL) && (psci_spd_pm->svc_on != NULL))
		psci_spd_pm->svc_on(target_cpu);

	/*
	 * Set the Affinity info state of the target cpu to ON_PENDING.
	 * Flush aff_info_state as it will be accessed with caches
	 * turned OFF.
	 */
	psci_set_aff_info_state_by_idx(target_idx, AFF_STATE_ON_PENDING);
	flush_cpu_data_by_index((unsigned int)target_idx,
				psci_svc_cpu_data.aff_info_state);

	/*
	 * The cache line invalidation by the target CPU after setting the
	 * state to OFF (see psci_do_cpu_off()), could cause the update to
	 * aff_info_state to be invalidated. Retry the update if the target
	 * CPU aff_info_state is not ON_PENDING.
	 */
	target_aff_state = psci_get_aff_info_state_by_idx(target_idx);
	if (target_aff_state != AFF_STATE_ON_PENDING) {
		assert(target_aff_state == AFF_STATE_OFF);
		psci_set_aff_info_state_by_idx(target_idx, AFF_STATE_ON_PENDING);
		flush_cpu_data_by_index((unsigned int)target_idx,
					psci_svc_cpu_data.aff_info_state);

		assert(psci_get_aff_info_state_by_idx(target_idx) ==
		       AFF_STATE_ON_PENDING);
	}

	/*
	 * Perform generic, architecture and platform specific handling.
	 */
	/*
	 * Plat. management: Give the platform the current state
	 * of the target cpu to allow it to perform the necessary
	 * steps to power on.
	 */
	rc = psci_plat_pm_ops->pwr_domain_on(target_cpu);
	assert((rc == PSCI_E_SUCCESS) || (rc == PSCI_E_INTERN_FAIL));

	if (rc == PSCI_E_SUCCESS)
		/* Store the re-entry information for the non-secure world. */
		cm_init_context_by_index((unsigned int)target_idx, ep);
	else {
		/* Restore the state on error. */
		psci_set_aff_info_state_by_idx(target_idx, AFF_STATE_OFF);
		flush_cpu_data_by_index((unsigned int)target_idx,
					psci_svc_cpu_data.aff_info_state);
	}

exit:
	psci_spin_unlock_cpu(target_idx);
	return rc;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

所以目前 ARMv8 情况下，对于 CPU 的唤醒，是由 ATF 实现的，本质应该与电源管理相关，有空这里接着补上

## 总结

比较关键的地方在于，后续被唤醒的`CPU`的启动的基址是可以确定的，这是非常重要的，照着 `boot cpu` 给它设定的路线，`thread_info->cpu` 在它唤醒之前已经确定，所以它一上来就可以调用 `smp_processor_id` 作为自己在 `per-CPU` 变量的偏移，这是非常有意思的，然而提到的人似乎并不多

## References

[ARM多核引导过程](https://blog.csdn.net/cs0301lm/article/details/41078599)

