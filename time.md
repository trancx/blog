# Time Keeping

## 什么是 jiffies

一个全局变量，系统初始化时钟之后，每一个时钟中断到来就会增加`1`，记载着启动的时间，也可以作为简单的时钟计数



## 序

以 x86 和 ARM 两个角度出发，说说时钟（SoC，Arch timer， DTS的交互），就是说，理一理这个框架，config-&gt;driver/clocksource/-&gt;of-&gt;kernel\_init-&gt;probe-&gt;match\_ids

## Draft

timer\_init 

of\_probe

of\_xx\_init -&gt; DECLARE-&gt; init\_func

interfacing the kernel framework~

clock\_source: 提供日期，是个长期的时钟

clock\_event:  reverse of ..source, 这就是个 timer，周期性的调用

## tick device

```c
/*
 * Tick devices
 */
DEFINE_PER_CPU(struct tick_device, tick_cpu_device);
/*
 * Tick next event: keeps track of the tick time
 */
ktime_t tick_next_period;
ktime_t tick_period;

/*
 * tick_do_timer_cpu is a timer core internal variable which holds the CPU NR
 * which is responsible for calling do_timer(), i.e. the timekeeping stuff. This
 * variable has two functions:
 *
 * 1) Prevent a thundering herd issue of a gazillion of CPUs trying to grab the
 *    timekeeping lock all at once. Only the CPU which is assigned to do the
 *    update is handling it.
 *
 * 2) Hand off the duty in the NOHZ idle case by setting the value to
 *    TICK_DO_TIMER_NONE, i.e. a non existing CPU. So the next cpu which looks
 *    at it will take over and keep the time keeping alive.  The handover
 *    procedure also covers cpu hotplug.
 */
int tick_do_timer_cpu __read_mostly = TICK_DO_TIMER_BOOT;
```

clock\_event 是否会被接受，看的是 tick\_device set\_up  之类的，对比的是，是否支持 oneshot, 也就是手动启动下一次，以及 rating 的大小，越大越好。

这样就是说，处理时钟中断的过程，没有唤醒时钟，不在计数范围内，这对调度来说是公平的，但是我目前还不清楚，为什么 tick 的时候，需要判断是否本CPU的中断，难以理解。

很简单，因为只有一个`cpu`负责增加 `jiffies`，除了引导的 CPU 其它都是被唤醒的，并不能作为 jiifies，但是每个进程的计数却是分开的，所以才有了那段经典的判断。

logical\_id 就是 sched\_init 递增的而已，而其它 CPU 都是由引导 CPU 唤醒的，唤醒的位置也不同，看看 secondary\_startup  

## tick 的矫正

![](.gitbook/assets/image%20%28128%29.png)

![](.gitbook/assets/image%20%28104%29.png)

ktime\_get 在 update\_wall\_time 更新

![](.gitbook/assets/image%20%28141%29.png)

所以说有俩个 clock

![](.gitbook/assets/image%20%28126%29.png)

一个是 tick\_device 一个是 clock\_source，后者让前者更加的精确

![](.gitbook/assets/image%20%28164%29.png)

## mul & shift

$$
(B~*~\lfloor \frac{A~*2^{shift} + \lfloor B/2\rfloor}{B}\rfloor \gg~shift) ~\approx A
$$

![](.gitbook/assets/image%20%28172%29.png)

这里前面计算 sftacc 就是为了限制 mul 的大小，如果 mul 太大，我们转换的公式就会出问题

$$
to = max ~*mul~\gg shift
$$

`tmp`  如果超过了 32位，那么超过的位数就得从 `mul` 中拿去，因为 `max = maxsec * from` 假设，此时的 `mul` 的位数 + max 的位数 超过`64`，那么出现了数据丢失

所以前面进行这一步计算是为了这个考虑，当然，从公式中我们也可以看出，`mul` 小了一些，那么 `shift` 也小一些就好了，所以这是双向的限制，保证了两点

`from * mul`  以及 `to << shift` 不会越界，当然是是 `64 bit` 的界

## END

