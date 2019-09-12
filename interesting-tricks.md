# Interesting Tricks

## Preface

其实内核有非常有意思的宏，函数。它们的实现都值得好好学习，这里把遇到的整合在一起吧，也做一个参考~

## Hoooh

### 大小端

```c
first, uapi/linux/byteorder/little_endian.h
or, xxx/big_endian.h

会定义不同的宏，比如 __le32_to_cpu 对于小端的CPU
自然就是不做处理，反而则做处理
这个宏会在这俩个文件都定义，然后只会 include 一个
很容易理解  so, xxx_to_cpu 具体的宏在编译的时候就确定了~
```

```c
对于 小端
#define __cpu_to_le64s(x) do { (void)(x); } while (0)
#define __le64_to_cpus(x) do { (void)(x); } while (0)
#define __cpu_to_le32s(x) do { (void)(x); } while (0)
#define __le32_to_cpus(x) do { (void)(x); } while (0)
#define __cpu_to_le16s(x) do { (void)(x); } while (0)
#define __le16_to_cpus(x) do { (void)(x); } while (0)
#define __cpu_to_be64s(x) __swab64s((x))
#define __be64_to_cpus(x) __swab64s((x))
#define __cpu_to_be32s(x) __swab32s((x))
#define __be32_to_cpus(x) __swab32s((x))
#define __cpu_to_be16s(x) __swab16s((x))
#define __be16_to_cpus(x) __swab16s((x))

而 大端
#define __cpu_to_le64s(x) __swab64s((x))
#define __le64_to_cpus(x) __swab64s((x))
#define __cpu_to_le32s(x) __swab32s((x))
#define __le32_to_cpus(x) __swab32s((x))
#define __cpu_to_le16s(x) __swab16s((x))
#define __le16_to_cpus(x) __swab16s((x))
#define __cpu_to_be64s(x) do { (void)(x); } while (0)
#define __be64_to_cpus(x) do { (void)(x); } while (0)
#define __cpu_to_be32s(x) do { (void)(x); } while (0)
#define __be32_to_cpus(x) do { (void)(x); } while (0)
#define __cpu_to_be16s(x) do { (void)(x); } while (0)
#define __be16_to_cpus(x) do { (void)(x); } while (0)

正好相反
```

大小端的切换是一样的操作，所以两者函数都调用的一样，最终来到这，如果没有CPU自带的指令切换大小端的话（有些架构有自带的，就采用内嵌汇编）

```c
/**
 * __swab64 - return a byteswapped 64-bit value
 * @x: value to byteswap
 */
#define __swab64(x)				\
	(__builtin_constant_p((__u64)(x)) ?	\
	___constant_swab64(x) :			\
	__fswab64(x))

以 64bit 作为例子，其他都是一样的

#define ___constant_swab64(x) ((__u64)(				\
	(((__u64)(x) & (__u64)0x00000000000000ffULL) << 56) |	\
	(((__u64)(x) & (__u64)0x000000000000ff00ULL) << 40) |	\
	(((__u64)(x) & (__u64)0x0000000000ff0000ULL) << 24) |	\
	(((__u64)(x) & (__u64)0x00000000ff000000ULL) <<  8) |	\
	(((__u64)(x) & (__u64)0x000000ff00000000ULL) >>  8) |	\
	(((__u64)(x) & (__u64)0x0000ff0000000000ULL) >> 24) |	\
	(((__u64)(x) & (__u64)0x00ff000000000000ULL) >> 40) |	\
	(((__u64)(x) & (__u64)0xff00000000000000ULL) >> 56)))
	
static inline __attribute_const__ __u64 __fswab64(__u64 val)
{
#ifdef __HAVE_BUILTIN_BSWAP64__
	return __builtin_bswap64(val);
#elif defined (__arch_swab64)
	return __arch_swab64(val);
#elif defined(__SWAB_64_THRU_32__)
	__u32 h = val >> 32;
	__u32 l = val & ((1ULL << 32) - 1);
	return (((__u64)__fswab32(l)) << 32) | ((__u64)(__fswab32(h)));
#else
	return ___constant_swab64(val);
#endif
}

static inline __attribute_const__ __u32 __fswab32(__u32 val)
{
#ifdef __HAVE_BUILTIN_BSWAP32__
	return __builtin_bswap32(val);
#elif defined(__arch_swab32)
	return __arch_swab32(val);
#else
	return ___constant_swab32(val);
#endif
}
```

是很粗暴的就解决，对于常数，也就是说编译就确定的数值，直接处理就好了，对于传进来的参数是不确定的，我们调用函数处理，但是本质还是一样的，都是做位移处理，也就是不是我们想象的做什么交换，那些都太复杂了，直接用位与操作即可解决，这也说明了，对于一些复杂的位操作，就应该直接暴力解决，而不是思考算法。

### IO\_操作

这里提提 io mmio  那些resource吧，其实只是设置一块内存是不可用的而已（page结构），因为页表做映射不可能再页内继续修改映射的，而物理内存的映射，一般也是以页为单位的，x86 的 IO 地址一般也是固定的，对于 ARM 来说，没有IO地址，一般都是硬布线改变的，而且所谓的 IO 地址，只是地址线上加一根使能线而已，独立或者统一编址都差不多。

> ```text
> /*
>  *  IO port access primitives
>  *  -------------------------
>  *
>  * The ARM doesn't have special IO access instructions; all IO is memory
>  * mapped.  Note that these are defined to perform little endian accesses
>  * only.  Their primary purpose is to access PCI and ISA peripherals.
>  *
>  */
> ```

先看 ARM 吧，最近再看这方面的，本质就是下面几个函数，可能加上一些内存屏障，当然，上面也说了，ARM 也对 x86 特有的 inb\(\) outb\(\) 等也定义了相关宏。

```c
/arch/arm/include/asm/io.h

/*
 * When running under a hypervisor, we want to avoid I/O accesses with
 * writeback addressing modes as these incur a significant performance
 * overhead (the address generation must be emulated in software).
 */
static inline void __raw_writew(u16 val, volatile void __iomem *addr)
{
	asm volatile("strh %1, %0"
		     : "+Q" (*(volatile u16 __force *)addr)
		     : "r" (val));
}

static inline u16 __raw_readw(const volatile void __iomem *addr)
{
	u16 val;
	asm volatile("ldrh %1, %0"
		     : "+Q" (*(volatile u16 __force *)addr),
		       "=r" (val));
	return val;
}
#endif

static inline void __raw_writeb(u8 val, volatile void __iomem *addr)
{
	asm volatile("strb %1, %0"
		     : "+Qo" (*(volatile u8 __force *)addr)
		     : "r" (val));
}

static inline void __raw_writel(u32 val, volatile void __iomem *addr)
{
	asm volatile("str %1, %0"
		     : "+Qo" (*(volatile u32 __force *)addr)
		     : "r" (val));
}

static inline u8 __raw_readb(const volatile void __iomem *addr)
{
	u8 val;
	asm volatile("ldrb %1, %0"
		     : "+Qo" (*(volatile u8 __force *)addr),
		       "=r" (val));
	return val;
}

static inline u32 __raw_readl(const volatile void __iomem *addr)
{
	u32 val;
	asm volatile("ldr %1, %0"
		     : "+Qo" (*(volatile u32 __force *)addr),
		       "=r" (val));
	return val;
}
```

### Inline Assembly - msr\_s 

一些内置汇编的 Tricks 也是宏用到的

```c
/* Indirect stringification.  Doing two levels allows the parameter to be a
 * macro itself.  For example, compile with -DFOO=bar, __stringify(FOO)
 * converts to "bar".
 */

#define __stringify_1(x...)	#x
#define __stringify(x...)	__stringify_1(x)
```

之所以要俩层的原因已经解释了，下次来试试

```c
#define __ACCESS_CP15(CRn, Op1, CRm, Op2)	\
	"mrc", "mcr", __stringify(p15, Op1, %0, CRn, CRm, Op2), u32
#define __ACCESS_CP15_64(Op1, CRm)		\
	"mrrc", "mcrr", __stringify(p15, Op1, %Q0, %R0, CRm), u64

#define __read_sysreg(r, w, c, t) ({				\
	t __val;						\
	asm volatile(r " " c : "=r" (__val));			\
	__val;							\
})
#define read_sysreg(...)		__read_sysreg(__VA_ARGS__)

#define __write_sysreg(v, r, w, c, t)	asm volatile(w " " c : : "r" ((t)(v)))
#define write_sysreg(v, ...)		__write_sysreg(v, __VA_ARGS__)

```

这个 Macro 写的非常的漂亮，这是 4.19 的代码，在最早阅读的 3.17 的时候，使用的是自己定义的宏命令，非常难阅读，似乎是利用了硬指令编码来生成的。

```c


/* Low level accessors */
static u64 __maybe_unused gic_read_iar(void)
{
	u64 irqstat;

	asm volatile("mrs_s %0, " __stringify(ICC_IAR1_EL1) : "=r" (irqstat));
	return irqstat;
}

#define ICC_IAR1_EL1			sys_reg(3, 0, 12, 12, 0)


arm64/include/asm/sysreg.h

#define sys_reg(op0, op1, crn, crm, op2) \
	((((op0)-2)<<19)|((op1)<<16)|((crn)<<12)|((crm)<<8)|((op2)<<5))

#ifdef __ASSEMBLY__

	.irp	num,0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30
	.equ	__reg_num_x\num, \num
	.endr
	.equ	__reg_num_xzr, 31

	.macro	mrs_s, rt, sreg
	.inst	0xd5300000|(\sreg)|(__reg_num_\rt)
	.endm

	.macro	msr_s, sreg, rt
	.inst	0xd5100000|(\sreg)|(__reg_num_\rt)
	.endm

#else

asm(
"	.irp	num,0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30\n"
"	.equ	__reg_num_x\\num, \\num\n"
"	.endr\n"
"	.equ	__reg_num_xzr, 31\n"
"\n"
"	.macro	mrs_s, rt, sreg\n"
"	.inst	0xd5300000|(\\sreg)|(__reg_num_\\rt)\n"
"	.endm\n"
"\n"
"	.macro	msr_s, sreg, rt\n"
"	.inst	0xd5100000|(\\sreg)|(__reg_num_\\rt)\n"
"	.endm\n"
);

#endif
```

对比另外一个版本，只需要

```text
#define ICC_IAR1			__ACCESS_CP15(c12, 0, c12, 0)
```

非常容易阅读，所以宏也是一门技术啊~~

### CPU id 

### \# \#\# \_\_VA\_ARGS\_\_

```c

#define AA aa
#define BB bb

#define M_STR(A) #A
// #define M_STR(...) #__VA_ARGS__  it's the same~
#define M_STR_1(A) M_STR(A)

#define M_CAT(B) M_STR(test##B)
#define M_CAT_1(B) M_CAT(B)

#define STR_VAR(...) M_STR_1(__VA_ARGS__)
#define CAT_VAR(...) M_CAT_1(__VA_ARGS__)

M_STR(AA) == "AA"
M_CAT(BB)  == "testBB"

STR_VAR(AA) == M_STR_1(AA) == "aa"
CAT_VAR(BB) == M_CAT_1(BB) == "testbb"

```

{% hint style="info" %}
特殊的处理\# \#\# \_\_VA\_ARGS\_\_ 都是不会对宏进行展开处理的，所以想要使用这些功能同时又要展开宏，就要嵌套一层
{% endhint %}

