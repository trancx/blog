# CFS - 完全公平调度器

## 序

最早接触操作系统这个概念的时候，其中它的一个作用就有 进程调度，这也是最核心的一个地方，但是也是最比较难懂的地方，因为光要理解一个进程调度的切换本质，特别是一个新手，估计就能花费一个星期，如果是对栈很熟悉的可能能很快掌握，但是除了进程切换，进程调度的关键还在于进程调度的算法，这也是最核心的地方，之所以到了如今才来学习，也是有点因为畏难的心理。

不过迟早是要面对的，所以如今让我们来走进一个具体的进程调度的算法，进程切换是涉及到具体的架构的，当然不是这篇文章的主题了。

{% hint style="info" %}
这篇文章不区分进程和线程的区别，线程就当作一个进程，这里所说的进程就是一个可调度的单位。
{% endhint %}

### 调度算法的发展过程

我不是跟着早期Linux一起成长的孩子，这点我也没有发言权，Linux 诞生的时候我还没有出生，但是 IBM 有一篇[文章](https://developer.ibm.com/tutorials/l-completely-fair-scheduler/)介绍的不错。

> Early Linux schedulers used minimal designs, obviously not focused on massive architectures with many processors or even hyperthreading. The 1.2 Linux scheduler used a circular queue for runnable task management that operated with a round-robin scheduling policy. This scheduler was efficient for adding and removing processes \(with a lock to protect the structure\). In short, the scheduler wasn’t complex but was simple and fast.
>
> Linux version 2.2 introduced the idea of scheduling classes, permitting scheduling policies for real-time tasks, non-preemptible tasks, and non-real-time tasks. The 2.2 scheduler also included support for symmetric multiprocessing \(SMP\).
>
> The 2.4 kernel included a relatively simple scheduler that operated in O\(N\) time \(as it iterated over every task during a scheduling event\). The 2.4 scheduler divided time into epochs, and within each epoch, every task was allowed to execute up to its time slice. If a task did not use all of its time slice, then half of the remaining time slice was added to the new time slice to allow it to execute longer in the next epoch. The scheduler would simply iterate over the tasks, applying a goodness function \(metric\) to determine which task to execute next. Although this approach was relatively simple, it was relatively inefficient, lacked scalability, and was weak for real-time systems. It also lacked features to exploit new hardware architectures such as multi-core processors.
>
> The early 2.6 scheduler, called the _O\(1\) scheduler,_ was designed to solve many of the problems with the 2.4 scheduler—namely, the scheduler was not required to iterate the entire task list to identify the next task to schedule \(resulting in its name, _O\(1\),_ which meant that it was much more efficient and much more scalable\). The O\(1\) scheduler kept track of runnable tasks in a run queue \(actually, two run queues for each priority level—one for active and one for expired tasks\), which meant that to identify the task to execute next, the scheduler simply needed to dequeue the next task off the specific active per-priority run queue. The O\(1\) scheduler was much more scalable and incorporated interactivity metrics with numerous heuristics to determine whether tasks were I/O-bound or processor-bound. But the O\(1\) scheduler became unwieldy in the kernel. The large mass of code needed to calculate heuristics was fundamentally difficult to manage and, for the purist, lacked algorithmic substance.
>
> Given the issues facing the O\(1\) scheduler and other external pressures, something needed to change. That change came in the way of a kernel patch from Con Kolivas, with his Rotating Staircase Deadline Scheduler \(RSDL\), which included his earlier work on the staircase scheduler. The result of this work was a simply designed scheduler that incorporated fairness with bounded latency. Kolivas’ scheduler impressed many \(with calls to incorporate it into the current 2.6.21 mainline kernel\), so it was clear that a scheduler change was on the way. Ingo Molnar, the creator of the O\(1\) scheduler, then developed the CFS based around some of the ideas from Kolivas’ work. Let’s dig into the CFS to see how it operates at a high level.

不了解也无伤大雅，因为进程算法都是与时俱进的，都是随着硬件的发展再做出具体的调整，在最早的 Linux，每个进程都拥有一个平等的时间片，但是根据一些优先级，来改变调度顺序，这些概念到如今都没有改变，唯一一直在变的就是，CPU 的性能一直在提升，所以单位时间执行的指令也飞速的提升，这也给了进程的时间片可以被**分割的更短**，所以看起来_**同时执行的进程**_变多了。

对了，维基上有一个[目录](https://en.wikipedia.org/wiki/Category:Linux_kernel_process_schedulers)，是内核存在过的调度算法，可以了解到更多的信息。

## 完全公平调度算法

今天介绍的是在内核2.6.2左右才被正式合并的一个算法，CFS\( completely fair scheduler \) ，它的思想非常的简单，可能具体的实现上已经更新换代很久了，但是不管多么复杂，核心的思想是没有改变的，我们得了解的就是设计的思想，具体的实现是根据实际的需要做改动的。

### 理想的CPU

当时第一个版本的[补丁](https://lwn.net/Articles/230501/)，作者对这个算法的思想做了一个简单的概述，然后笔者在其提到的一个主页上找到了相对详细一点的[概述](http://people.redhat.com/mingo/cfs-scheduler/sched-design-CFS.txt)。

> ```text
> 80% of CFS's design can be summed up in a single sentence: CFS basically
> models an "ideal, precise multi-tasking CPU" on real hardware.
>
> "Ideal multi-tasking CPU" is a (non-existent  :-))  CPU that has 100%
> physical power and which can run each task at precise equal speed, in
> parallel, each at 1/nr_running speed. For example: if there are 2 tasks
> running then it runs each at 50% physical power - totally in parallel.
> ```

作者提到了，CFS 的设计就是一句话，CFS模拟了一个理想化多任务处理器，下面有着详细的解释，就是说如果有 n 个进程，那么在延迟 delay 下，就是每个任务运行 delay/n 的时间。

这个设计，够简单了吧。也就是说内核不会在调度上有任何的偏差，是完全公平的。

### 相对的公平

人生哪有绝对的公平呢，操作系统也是一样的，不然也就不会有 _**nice**_ 调用了，现在在这把这个公平的实质说清楚，首先，公平指的是，调度器在选择进程的时候，_**不会因为你的特权高而选择你，而会因为你没有执行完自己的时间而选择你**_。这句话，就是公平的本质。

举个例子，一个进程nice值为-20，最高特权级，但是它执行完了自己应该有的时间，而有一个nice值为0的进程等待IO被唤醒，那么对于 CFS 来说，它还是会选择后者，虽然说 CFS 并没有交互性进程的识别算法，但是交互性进程本身就有一个显著的特点，那就是经常等待IO 。

也就是说，每个进程根据自己的 nice 值，也是根据自己的特权级高低，会得到**不同长度**的**时间片**，调度器**在乎的是你是不是没有执行完自己的时间，如果没有，而且你还是有最多的可使用的时间，那么我就选择你执行**。

### weight weight weight

谨以此小标题致敬 wuli 坤坤 😀。理解 CFS 的本质，有一个很关键的字段就是 **weight**，我们小学接触 一次函数 的时候表达式为

$$y = kx + b$$

实际上在国外，一种常见的写法是

$$y = wx + b$$

这里的 $$w$$ 就是指代 weight $$b$$ 指代 bias ，每一个进程（调度单元），都有一个 weight 字段，代表当前的进程的“ 重量 ”，非常的形象。nice 值和 weight 的转换如下。

![nice-to-weight-convertion](../.gitbook/assets/image%20%2893%29.png)

为什么是这个规则！肯定有读者就好奇了，现在以 nice 0 和 nice 1 的两个进程为例子。

那么

$$sum = 1024 + 820 = 1844$$

则 $$1024/1844\approx0.555$$

再计算 $$820/1844 \approx 0.445$$

现在我告诉你，这个比例就是俩进程占用 CPU 的比例

两者 $$1024/820 = 1.248$$

而 $$1.248 / （1.248+1）\approx 0.55$$ $$1 / (1.248+1) \approx 0.45$$

如果现在知道 nice = 0 而 weight = 1024，则 nice = -1 的 weight

$$weight = 1024 * 1.248 = 1277.952$$

所以 weight 的作用保证了一点，那就是当他们的数值相差 1 它们占用CPU的时间相差大概是 10%，这就是 weight 存在的作用。

现在考虑我们有一个运行的队列，我们用一个字段来记录当前队列所有进程的总重，每当有新的进程进入或者出列，我们就更新它，假设这个字段为 _**total\_weight**_

而如今我们选择了一个进程 $$t$$ 来执行，那么假设我们可承受的延迟是 100ms

也就是说 100ms内 我们就得把所有的进程调度一次，来保证用户的可使用性，那么你觉得这个进程应该执行多长时间呢？

$$
time\_slice = \frac{t~->weight}{total\_weight} * 100~ms
$$

这样，上面的 nice 和 weight 的转换的作用是不是就解释完了，虽然 CFS 在选择进程上是公平的，但是每个进程能执行的时间是不公平的，这种不公平是用 weight 来代表，一个进程的“重量”，代表的是在一段时间内，**它的CPU使用占比**。

看看内核代码的注释

```c
/*
 * Nice levels are multiplicative, with a gentle 10% change for every
 * nice level changed. I.e. when a CPU-bound task goes from nice 0 to
 * nice 1, it will get ~10% less CPU time than another CPU-bound task
 * that remained on nice 0.
 *
 * The "10% effect" is relative and cumulative: from _any_ nice level,
 * if you go up 1 level, it's -10% CPU usage, if you go down 1 level
 * it's +10% CPU usage. (to achieve that we use a multiplier of 1.25.
 * If a task goes up by ~10% and another task goes down by ~10% then
 * the relative distance between them is ~25%.)
 */
 static const int prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

### 内核的脉搏

em，肯定有读者好奇，执行的时间是怎么记录的呢，嵌入式的同学肯定是不会不知道时间脉冲这个概念，简单的说，内核每一段固定时间就有一次时间中断，我们也称之为脉搏，那么每一次时间中断，我们就在当前进程的记录下加一段时间，如果发现它一旦超过了它所能执行的时间，我们就给进程做一个标记，它的时间到了，然后真正被 schedule（） 函数切换的时候，我们把它当前多出来的时间也从它下一次扣，具体怎么实现等下就明白了，总之这就是设计的思想。

### Fair Clock

我一直在纠结，是以最新的 vruntime 来阐述，还是最初的 Fair Clock 来阐述这个算法，思来想去，我就把 vruntime 相关的博客放上来，然后我自己用最初的设计来阐述就OK

内核是用 **rq （Run Queue）**维护当前可运行的进程，上面的进程按一定的规则来排列，以当前所说的 CFS RQ，就是以一个 Fair Key 的值来排列，这个值记录是最后一次被调离出 CPU 的时候 Fair Clock的值。

Fair Clock， 记录的是使用 CPU 的时间总和，以纳秒为单位，也是说最初的时候是 0，然后每一次时间中断（脉搏），我们就更新它，不仅如此，每当有进程入列或者出列，我们就更新这个值，其实它就是 _**rq**_ 的时间线，但是它屏蔽了时钟，只关心自己运行了多长时间。

现在还要谈谈所有进程都有一个 Fair Key，它的值就是上次执行它的 Fair Clock 的值，实际上是不是就是可以简单理解为，上次执行这个进程的时间。注意，记录的时间一定是最后进程被切出 CPU 的时候。

为了表述方便，之后我说的时间，均指代 Fair Clock 的值，也就是说，每一段时间间隔，我就更新 _**rq**_ 的 Fair Clock（时间线），同时，当前CPU正在执行的进程的 Fair Key 也更新为这个值，最后当进程失去了CPU，我们也更新 Fair Key，代表的是**进程失去CPU资源的最后时刻**。

而我们如何选择，新的进程呢？ 那就是选择 Fair Key 最小的一个进程，是不是很容易想到，也就是说我们选择的是最久没有运行的一个进程，理所当然轮到它了。

那么读者可能会好奇，为什么前面要加一个 Fair 修饰它呢，它不就是一个时间线吗，这个字段存在的意义就是当一个公平的时钟，所以必然有它不同的地方。稍后我们就会揭晓，通过前面几节的叙述，我们知道，不同优先级的进程能获得的时间片长度是不同的，具体有多少得取决于它的 weight 以及 系统所需求的延迟，还有当前所有进程的 weight 之和。

那么，Fair Clock 是怎么增长的？前面我们说每个一段时间 delta，我们就更新它，但是 Fair Clock 实际上不是一一对应。

$$
Fair\_Clock ~+= ~ \frac{1024}{t->\_weight} * delta
$$

也就是说，每过一段时间，Fair Clock 其实会比实际的时间增长的慢一点，这里同时揭示了非常重要一点， **不同特权级的进程，执行相同的时间，对 Fair Clock 的增长是不一样的**！按照常理想象，特权高，理所应当对 Fair Clock 作用小，因为 CFS 选择的是最久之前获得CPU的进程，如此就可以保证特权级高的进程可以在队列的前方，当然了，**当它们都执行了自己的那部分时间，和下面的公式联例，对 Fair Clock 增长就是相同的**。

$$
time\_slice = \frac{t~->weight}{total\_weight} * 100~ms
$$

$$
Fair\_Clock ~+= ~ \frac{1024}{total\_weight} *latency
$$

总结一下， Fair Clock 是一个公平的时钟，目的记录进程离开 CPU 的一个“时间”，同时又根据 nice 的不同，做了不一样的处理，结果就是高特权的进程在相同的时间下对其作用小，也就是为了高特权的进程能再次顺利获取属于它的那份时间，试想一下，如果作用是一样的，那么高特权的进程（先）和普通进程（后）执行了相同的时间，结果因为 Fair Key 的增大，到了队列的后面，而我本来应该拥有更多的时间，但是却因为”公平对待“，导致了不公平。

### 进程爱睡觉 - wait\_runtime

刚才我们说了，当一个进程消耗完了自己的时间，那么它被切出之前，我们会给它的 Fair Key 赋值为当前的 Fair Clock，代表它此时最后一次被调度的时间。考虑一个情况，一个进程再开机就自动运行，然后马上睡眠，接着一小时之后它来执行。

忘了说明一点，睡眠的时候，必然会导致进程离开 _**rq**_，因为 _**rq**_ 维护的队列里储存都是已经就绪的进程，那么 CFS 就会记录它出列时的 Fair Clock 赋值其 Fair Key，并且呢，它知道进程要去睡眠，因为传入的参数会标明，所以它还记录了 **sleep\_start\_fair** 这个值，意味着睡眠开始的 Fair Clock的值。

那么，进程再次唤醒，一个小时最后，重新入列，可想而知，必然是在队列第一位，经过一次调度后，立即得到了 CPU 的使用权，如果照着前几节所说，那么它会获得一段时间，根据自己的 weight ，执行完后 Fair Key 立即更新为 当前的 Fair Clock。

有问题吗？仔细想想，它的 Fair Key 在一段非常短的时间内（进程占用CPU之后的最近一次的更新），立即增长了好几倍，因为它的 Fair Key 在是一小时以前的 Fair Clock，结果执行了一段时间以后，它又跟其他的进程平起平坐了，这有失公允。

所以我们有哪些手段？ 把它的 Fair Key 减掉一部分，也就是说，Fair Key 不在是与 Fair Clock 伪同步了，剪掉的一部分我们称为 wait\_runtime，可以简单的理解为睡眠的时间，其实并不是，为什么呢，如果进程睡眠了一小时，我们还让它 Fair Key 落后 Fair Clock 一个小时么？这样这个进程永远都会是被执行的一个，就没有公平这一说了。

```text
"wait_runtime" is the amount of time the task should now run on 
the CPU for it to become completely fair and balanced.

small detail: on 'ideal' hardware, the p->wait_runtime value would
always be zero - no task would ever get 'out of balance' from the
'ideal' share of CPU time.
```

这个值记录的是一个平衡值，可以是正的也可以是负的，它代表了当前进程一个指标，就是能使它能得到自己那份资源的一个度量，如果是正的，意味着它应该得到更多，如果是负的，意味着它已经得到了超过了自己应该获得。

$$
Fair\_Key = Fair\_Clock - wait\_runtime
$$

如果它是负的，那么它在更新之后就会比 Fair Clock 还要大，如果是正的，意味着它会比它更小。设计的人也留下了注释，在理想的环境下，每个进程执行完自己的时间，就切换下一个进程，这样在一段时间内，它们的 Fair \_Key 一定相同（有时不同，但是大致上一定是一样的）而且 wait\_runtime 为 0，因为没有不公平出现，但是以下几点导致了不公平。

* 进程唤醒和睡眠，导致了 wait\_runtime 不为 0
* 时间片分割粒度不一定是赋予它的时间的公约数，那么进程就会执行比预计的时间多了点

可能还有其他原因，还没想到，其实就是因为不是理想的情况，所以总是有不公平的情况出现，所以需要这个指标，让 CFS 调整。

```text
CFS's task picking logic is based on this p->wait_runtime value and it
is thus very simple: it always tries to run the task with the largest
p->wait_runtime value.
```

Fair Clock 在一定时间内其实没有区别，因为每个进程被调度就会被更新一次，只要时间切割的不是太久，我们就可以认为 Fair Clock 是几乎相同的，正是因为上面说的状况出现，导致了 wait \_runtime 不为0，那么 Fair Key 也就不和 Fair Clock 同步了。

所以，设计的人就认为这个算法本质就是挑选最大的 wait\_runtime 的一个进程。现在回到刚才那个情况，当进程一个进程入列的时候，如果传入参数表明，它是由 **睡眠→就绪** 状态，那么我就得调整它的 wait\_runtime 了。

首先，获取它的睡眠时间 $$delta = Fair\_Clock - sleep\_start\_fair$$ 然后计算它的应该得到多少补偿

$$
wait\_runtime = \frac{delta*~t->weight}{1024}
$$

也就是说，当进程的**优先级**越高，得到的补偿也就越多，当然这个是有_**上限**_的，就是为了防止存在一个睡眠一个小时，然后执行一小时为了破坏系统调度这样的流氓程序，比如这个上限是三个时间片的长度转化的值，已经很长了，为什么？一般存在这种长时间睡眠的程序，有个特点，一定是处理完又继续睡眠的，当然，不可一概而论，所以这个值应该由系统管理员决定。

总之，对于睡眠的进程，CFS 对其是有一定的**属性加成**的，为了平衡它之前收到的不公平对待，不过这种调整是有限的，因为不是调度器本身的原因导致的延迟。

经过如上的分析，我们发现 CFS 虽然没有针对 interactive 的进程优化，但是本身这类进程是 IO-bound 所以，它在 CFS 的反应一定是很快的，这也使得桌面响应的速度加快，但是其他的进程同样也没有落下，所以 CFS 设计简单，但是非常的适合内核调度的需求。

### 进程何时调度

到底什么时候一个进程才需要被调度，这实际上是一个核心的问题，简单的回答可以说是时间片消耗完了就应该被调度，这是毫无疑问的，但是总有进程选择主动释放 CPU 的使用权，转而睡眠，而当其再苏醒的时候，应该什么时候才获得 CPU 的使用权，这是一大难题。

CFS 直接摒弃了这些概念，它只负责记账，谁此刻最需要 CPU，就让谁使用，巧妙之处在于，经常性睡眠的进程在它的眼中正是最需要 CPU 的。

每一次时间中断，内核都会调用调度器的相关函数，来判断当前的进程是否需要被换出，基于 2.6.23 的代码，可以理解为如下。

1. 首先，把当前进程出列并入列，这一步是为了更新 Fair Clock/Key 等信息
2. 然后判断当前 Fair Key 最小的是否仍为当前进程，如果不是，则做如下判断。
3. 先计算候选进程和当前进程的 Fair Key 之差 $$delta$$ 
4. 然后计算当前进程已经执行的时间 $$delta\_exe$$ 
5. 如果进程已经超过了当前当前进程应该执行的时间，那么则可以换出当前进程
6. 否则，如果 $$delta$$ 小于一定的值，则不用换出。

也就是说，只有差值满足了一定的需求，才认为进程需要被调度了，是时候介绍一个概念了。

#### Granularity - 粒度

这篇文章一直在避免涉及源代码，哈哈哈，因为源代码太高深莫测了，很多值都是通过测出来的，这也是机器学习的思想，很多时候没有绝对的最优解，都是通过一步步尝试出来得到当前的最优解，所以我们必须得了解其思想，然后自己再来测试。

颗粒度，描述的是一个进程执行的时间单位，其实很形象，颗粒度本来就是用来描述分割程度的，它的值越小，意味着进程的一段时间片就越小。它正确的理解应该是，描述一个最小单位的值，我感觉是这样 \(￣▽￣\)"

> The CFS scheduler offers a single tunable: a "granularity" value which describes how quickly the scheduler will switch processes in order to maintain fairness. A low granularity gives more frequent switching; this setting translates to lower latency for interactive responses but can lower throughput slightly. Server systems may run better with a higher granularity value. --[cite\_here](https://lwn.net/Articles/230574/)

形象地说，granularity 就是指一个进程使用CPU之后应该执行的时间，我们之间都是用 slice 来表示的，其实是一回事。来看看，内核代码如何计算 gran

```c
/*
 * Calculate the preemption granularity needed to schedule every
 * runnable task once per sysctl_sched_latency amount of time.
 * (down to a sensible low limit on granularity)
 *
 * For example, if there are 2 tasks running and latency is 10 msecs,
 * we switch tasks every 5 msecs. If we have 3 tasks running, we have
 * to switch tasks every 3.33 msecs to get a 10 msecs observed latency
 * for each task. We do finer and finer scheduling up to until we
 * reach the minimum granularity value.
 *
 * To achieve this we use the following dynamic-granularity rule:
 *
 *    gran = lat/nr - lat/nr/nr
 *
 * This comes out of the following equations:
 *
 *    kA1 + gran = kB1
 *    kB2 + gran = kA2
 *    kA2 = kA1
 *    kB2 = kB1 - d + d/nr
 *    lat = d * nr
 *
 * Where 'k' is key, 'A' is task A (waiting), 'B' is task B (running),
 * '1' is start of time, '2' is end of time, 'd' is delay between
 * 1 and 2 (during which task B was running), 'nr' is number of tasks
 * running, 'lat' is the the period of each task. ('lat' is the
 * sched_latency that we aim for.)
 */
static long
sched_granularity(struct cfs_rq *cfs_rq)
{
    unsigned int gran = sysctl_sched_latency;
    unsigned int nr = cfs_rq->nr_running;

    if (nr > 1) {
        gran = gran/nr - gran/nr/nr;
        gran = max(gran, sysctl_sched_min_granularity);
    }

    return gran;
}
```

简单的说呢，就是说这个进程，我现在给你CPU使用，但是 gran 时间过去之后，你就得归还了，因为我得保证系统的响应时间在一定的延迟内。

刚才我们提到的 $$delta$$ 大于一个值的话就得把当前的进程换出了，这个值与它有关，假设计算出来的值为 gran 那么最终这个值是

$$
niced\_granularity = \left\{ \begin{array}{ll}
\frac{gran~*~1024}{weight} & \textrm{if weight>=~1024}\\
& \\
\frac{gran~*~weight}{1024} & \textrm{if weight < 1024}\\
\end{array} \right.
$$

也就是说呢，这个值还取决于当前进程的 weight，得到的 niced\_gran 如果不是1024，都是比其小的，**这里我并没有理解其原因**！！注意，这里的代码都是基于 2.6.23 的，如果查看2.6.27 的代码，其实已经没有这个判断了，我们需要理解的是一点，**只要俩者相差的距离过大**，我们就得重新调度了。

{% hint style="info" %}
重新调度并不是指立刻进行上下文切换，首先这里发生在中断，不可能进行进程调度，所以只能设置标志位，等待下一次 schedule 的调用出现，进程就会被换出。
{% endhint %}

> **What is schedule\(\) function?**  
> It implements the Main Scheduler. It is defined in kernel/sched.c  
> It is called from many points in the kernel to allocate the CPU to a process other than the currently active one  
> Usually called after returning from system calls, if TIF\_NEED\_RESCHED is set for current task -[cite\_here](https://oakbytes.wordpress.com/2012/07/03/cfs-and-periodic-scheduler/)

### 进程的孩子怎么办

操作系统的所有的进程都是由最开始的进程\( init \)衍生出来的，可以说，它是所有进程的父母，试想一下，每次我们 fork 之后，就有一个新的子进程添加到运行队列（子进程肯定是就绪态 ）。大部分情况下，我们 fork 之后要做的事情都是执行一个新的可执行程序，然后父母进程则睡眠或者等待子进程的一些信号，所以我们 CFS 初始化一个新的进程的时候，往往是把 Fair Key 调整的比父进程（也就是当前进程）要小，如此以来子进程就可以得到更快的调度。

考虑一个程序，每次我执行完一段时间 （大概是一个时间片），然后我 fork，让我子进程继续执行，接着我周而复始，那是不是这个进程永远霸占这 CPU 了，在之前的进程调度器都是通过让子进程得到父进程的时间片的一半，防止这种情况的发生。

2.6.23 是通过给 $$wait\_runtime$$ 赋值一个小于 0 的值，这样下一次更新 Fair Key 的时候，就会比 Fair Clock 走得快。**，如果一直有进程一直在 Fork子进程， 那么子进程永远得不到执行。**

```c
/*
     * Child runs first: we let it run before the parent
     * until it reschedules once. We set up the key so that
     * it will preempt the parent:
     */
    se->fair_key = curr->fair_key -
        niced_granularity(curr, sched_granularity(cfs_rq)) - 1;

    /*
     * The statistical average of wait_runtime is about
     * -granularity/2, so initialize the task with that:
     */
    if (sysctl_sched_features & SCHED_FEAT_START_DEBIT)
        se->wait_runtime = -(sched_granularity(cfs_rq) / 2);
```

而 2.6.24 ，对于一个新的初始化的进程，我们当前管理的进程的最小的 Fair Key 的基础上，加上一个时间片的长度，_**子进程优先**_这个理念也得得到体现，所以我们还得比较 此时子进程的 Fair Key 和 父进程的 比较，保证子进程的 Fair Key 比父进程小，如果大了，则两者交换。在倒数第二节有提到。

{% hint style="info" %}
Fair Key 和 vruntime 是可以等价来理解的
{% endhint %}

### 为何选择早期的代码

读者可能发发现，这里选的都是最早 CFS 出现的代码，原因只有一个，文档非常的丰富，一个新兴的事物，设计者为了让更多人的理解，肯定得写不少的解释文档，可是随着后期的丰富，其中原委往往只有最主要的那几个人才看得懂了，只有了解了最初的模样，才能掌握后面的，因为主要的思想不会改变，所以阅读内核的代码也是从早期的代码开始阅读。

至于后期出现的 vruntime 本质其实就是 Fair Clock，它的思想仍然没有改变，可能做了一些修改，经过测试之后得到最简单最优的一个算法。但是如果直接开始看那里的代码，则会非常的难以理解。

### 以 nice = 0 和 nice = 1 的两个进程为例

可能上面的解释仍有读者难以理解，因为我的语文并不好，接下来就以一个实际的例子出发，假设存在 t1 t2 两个进程，它们的 nice 值分别是 0 和 1。

假设现在 CFS 运行了一段时间，现在 Fair Clock 为 100

俩个进程同时进入了队列，现在 _**rq**_ 的 $$total\_weight$$ 为 1844，并且它们的 Fair Key 初始化 为 100（忽略两者进来的先后，都无所谓这里 ）

我们再假设，系统的延迟为 100ms，时间中断是 5ms，只有这两个进程是就绪态

现在 CFS 选择了 t1 运行，其实 t2 也无所谓，都一样

5ms 后，时间中断，t1 的 Fair Key 更新为 105（这里假设初始 wait\_runtime 为0）

$$
Fair\_Clock ~+= ~ \frac{1024}{t->\_weight} * delta
$$

接着 t1 重新插入队列中，发现已经不是 Fair Key 最小的那个进程，于是计算两者 Fair Key 的差，得到 5，而 t1 进程目前执行了 5ms，它应该执行

$$
delta\_mine = \frac{t1->weight}{total\_weight} *100ms
$$

代入得到 55ms，并且现在 Fair Key 的差值仅为 5，我们计算 gran = 25

$$
gran = latency/n - latency/n/n
$$

$$
niced\_granularity = \left\{ \begin{array}{ll}
\frac{gran~*~1024}{weight} & \textrm{if weight>=~1024}\\
& \\
\frac{gran~*~weight}{1024} & \textrm{if weight < 1024}\\
\end{array} \right.
$$

如此循环5次，发现了 两者差大于25，此时 Fair Clock 为 130 = t1-&gt;Fair Key ，而 t2 仍是最初的 100，所以我们认为此时进程可以切换了。其实这里并不太好对吧，明明进程仍有 30ms 可以使用，所以后面 2.6.24.7 的代码，就已经是判断**它是否超过自己应该得到的时间**了，我觉得只有适合不适合，没有好与不好，上面这个判断其实可以让只有俩个进程的运行的CPU的延迟都很低，因为我们在保证了 100ms 延迟的前提下，又进一步**压缩**了进程的时间片。

t1-&gt;fair\_key = 130; t2-&gt;fair\_key = 100

我们假设5ms之后，schedule函数被调用，所以 t1-&gt;fair\_key = 135

CFS 终于选择了 t2 运行，5ms 之后， t2-&gt;weight = 820

$$
Fair\_Clock ~+= ~ \frac{1024}{t->\_weight} * delta
$$

所以 5ms 之后， Fair Clock = 135 + 6 = 141 ，它的 gran 是 25 \* 1024 / 820 = 31

{% hint style="info" %}
同样是运行 5ms，但是低特权的进程却让 Fair Clock “走”的更快
{% endhint %}

注意： 现在 t1 的 Fair Key 已经比 t2 的要小了，但是并没有超过 gran，以及它能运行的时间 45ms

下一个时间中断: t2-&gt;Fair Key = 147 = Fair Clock， 两者差为 $$delta = 147 -135 = 12$$

... t2-&gt;Fair Key = 153 = Fair Clock ， $$delta = 12 +6 = 18$$

... $$delta = 24$$ ，此时进程运行了 20ms

... $$delta = 30$$

... $$delta = 36$$ 进程运行了 30ms，t2-&gt;Fair Key = 171 = Fair Clock

轮到了 t1 假设， 5ms 之后 进程开始调度，则 t2-&gt;Fair Key = 177 = Fair Clock，进程运行了 35ms

此时 CPU 占用比是 1:1 ， t1-&gt;gran = 25，所以 30ms 仍然会轮到 t2 运行

注意到它们的比值其实就根据它们的重量得到，所以这样下去一直会是 1:1，虽然

Fair Clock 增长速度不一样，但是两者能忍受的 差值（niced\_gran） 也不一样，

我们是不是忘了什么，wait\_runtime !

每一个 5ms，进程的 wait\_runtime 会这样增加，delta = 5ms

$$
task->wait\_runtime ~+= ~\frac{task->weight -1024}{total\_weight}*delta
$$

{% hint style="info" %}
CFS 是以纳秒为单位的，所以这看起来数值会很小
{% endhint %}

$$
Fair\_Key = Fair\_Clock - wait\_runtime
$$

对于 t1 他的 weight 正好为 1024，所以没有补偿，但是对于 t2，它的 wait\_runtime 会变小，当然是有个上限，这样它的 Fair Key 就走在了 Fair Clock 的前方。当然这里以 5ms 计算，实际上只有0.5，因为我是以毫秒为单位，但是记住它是_**累计量**_，那么实际上，它除了让 Fair Clock 的更快，它同时还让自己的 Fair Key 一定比 Clock 大一点。

{% hint style="info" %}
如果进程睡眠，那么 wait\_runtime 会变大，这样就走在了 Clock 的后面
{% endhint %}

举这个例子，就是让大家了解 CFS 的实质就是维护了一个 Fair Clock，以及调整公平后的每个进程都会有的 Fair Key。长时间未执行的进程必然马上得到 CPU 的使用权。

实际的实现还有比较多的参数，但是太高深莫测了，但是我已经了解了实际的思想，就是维护一个公平的时钟，如果让我来实现，我会简单的判断当前的进程拥有多少的时间，如果没有超过，接着判断，如果当前已经不是最小的 Fair Key，我会让高特权的进程所能接受的差值大一些，而不是上面只让 nice = 0 的能获得最大的 niced\_gran，毕竟是特权进程，然后 wait\_runtime 设计为一个累计量，每当进程要被切换的时候更新，更新为它这次距离它应该执行的时间的差 对应的 一个权衡的值。

{% hint style="info" %}
上面的代码每次执行都更新一次，既然保证了 Fair Clock 的增长不一致，又设计这个，只存在一个我觉得即可。
{% endhint %}

{% hint style="info" %}
在 2.6.24 之后，就是直接让进程运行到自己的时间片直到结束，并且，特权进程让 vunrime\( Fair Clock\) 增长的慢。并且也没有 wait\_runtime ，因为默认是执行完时间片才切换，然后睡眠的进程得到 vruntime\(Fair Key\) = minimal\_vruntime - 一个时间片 的值，也就是说让睡眠的进程是当前最需要CPU的。
{% endhint %}

## CFS 的实现 - 以 2.6.23 为例

{% hint style="info" %}
Fair Key 维护在一个红黑树，其实是什么结构都无所谓，这里不阐述了。
{% endhint %}

这里不在进行不停的代码复制粘贴，我只会提一些关键的地方，也是自己最开始阅读的代码困惑的几个地方。

每一次时钟中断，tick

```c
/*
 * scheduler tick hitting a task of our scheduling class:
 */
static void task_tick_fair(struct rq *rq, struct task_struct *curr)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &curr->se;

    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);
        entity_tick(cfs_rq, se);
    }
}

static void entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    struct sched_entity *next;

    /*
     * Dequeue and enqueue the task to update its
     * position within the tree:
     */
    dequeue_entity(cfs_rq, curr, 0);
    enqueue_entity(cfs_rq, curr, 0);

    /*
     * Reschedule if another task tops the current one.
     */
    next = __pick_next_entity(cfs_rq);
    if (next == curr)
        return;

    __check_preempt_curr_fair(cfs_rq, next, curr,
            sched_granularity(cfs_rq));
}

/*
 * Preempt the current task with a newly woken task if needed:
 */
static void
__check_preempt_curr_fair(struct cfs_rq *cfs_rq, struct sched_entity *se,
              struct sched_entity *curr, unsigned long granularity)
{
    s64 __delta = curr->fair_key - se->fair_key;
    unsigned long ideal_runtime, delta_exec;

    /*
     * ideal_runtime is compared against sum_exec_runtime, which is
     * walltime, hence do not scale.
     */
    ideal_runtime = max(sysctl_sched_latency / cfs_rq->nr_running,
            (unsigned long)sysctl_sched_min_granularity);

    /*
     * If we executed more than what the latency constraint suggests,
     * reduce the rescheduling granularity. This way the total latency
     * of how much a task is not scheduled converges to
     * sysctl_sched_latency:
     */
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    if (delta_exec > ideal_runtime)
        granularity = 0;

    /*
     * Take scheduling granularity into account - do not
     * preempt the current task unless the best task has
     * a larger than sched_granularity fairness advantage:
     *
     * scale granularity as key space is in fair_clock.
     */
    if (__delta > niced_granularity(curr, granularity))
        resched_task(rq_of(cfs_rq)->curr);
}
```

可以看到判断非常的复杂，跟我们上面说的类似，就不解释了，当 **fork\(\)** 时

```c
/*
 * Share the fairness runtime between parent and child, thus the
 * total amount of pressure for CPU stays equal - new tasks
 * get a chance to run but frequent forkers are not allowed to
 * monopolize the CPU. Note: the parent runqueue is locked,
 * the child is not running yet.
 */
static void task_new_fair(struct rq *rq, struct task_struct *p)
{
    struct cfs_rq *cfs_rq = task_cfs_rq(p);
    struct sched_entity *se = &p->se, *curr = cfs_rq_curr(cfs_rq);

    sched_info_queued(p);

    update_curr(cfs_rq);
    update_stats_enqueue(cfs_rq, se);
    /*
     * Child runs first: we let it run before the parent
     * until it reschedules once. We set up the key so that
     * it will preempt the parent:
     */
    se->fair_key = curr->fair_key -
        niced_granularity(curr, sched_granularity(cfs_rq)) - 1;
    /*
     * The first wait is dominated by the child-runs-first logic,
     * so do not credit it with that waiting time yet:
     */
    if (sysctl_sched_features & SCHED_FEAT_SKIP_INITIAL)
        se->wait_start_fair = 0;

    /*
     * The statistical average of wait_runtime is about
     * -granularity/2, so initialize the task with that:
     */
    if (sysctl_sched_features & SCHED_FEAT_START_DEBIT)
        se->wait_runtime = -(sched_granularity(cfs_rq) / 2);

    __enqueue_entity(cfs_rq, se);
}
```

最重要的记账函数，每一次插入和出列，都会调用

```c
/*
 * Update the current task's runtime statistics. Skip current tasks that
 * are not in our scheduling class.
 */
static inline void
__update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long delta, delta_exec, delta_fair, delta_mine;
    struct load_weight *lw = &cfs_rq->load;
    unsigned long load = lw->weight;

    delta_exec = curr->delta_exec;
    schedstat_set(curr->exec_max, max((u64)delta_exec, curr->exec_max));

    curr->sum_exec_runtime += delta_exec;
    cfs_rq->exec_clock += delta_exec;

    if (unlikely(!load))
        return;

    delta_fair = calc_delta_fair(delta_exec, lw);
    delta_mine = calc_delta_mine(delta_exec, curr->load.weight, lw);

    if (cfs_rq->sleeper_bonus > sysctl_sched_min_granularity) {
        delta = min((u64)delta_mine, cfs_rq->sleeper_bonus);
        delta = min(delta, (unsigned long)(
            (long)sysctl_sched_runtime_limit - curr->wait_runtime));
        cfs_rq->sleeper_bonus -= delta;
        delta_mine -= delta;
    }

    cfs_rq->fair_clock += delta_fair;
    /*
     * We executed delta_exec amount of time on the CPU,
     * but we were only entitled to delta_mine amount of
     * time during that period (if nr_running == 1 then
     * the two values are equal)
     * [Note: delta_mine - delta_exec is negative]:
     */
    add_wait_runtime(cfs_rq, curr, delta_mine - delta_exec);
}
```

{% hint style="info" %}
每次时间中断，当前进程都会重新插入队列，以更新信息。
{% endhint %}

## vruntime 时期的 CFS

前面也不止一次的提到，vruntime 和 Fair Key 是一个概念，从 2.6.24 之后 CFS 得到了简化，去除了挺多不必要的比较，也增加了一些安全检查，总体来说更容易阅读。

1. 首先在调度方面，正常情况，只有当进程的时间片消耗完毕的情况下才会重新调度。
2. 当一个进程苏醒，或者一个新的线程创建会调用 preemption 相关函数
3. 睡眠的进程得到的 bonus，都通过 vruntime 实现

```c
/*
 * Preempt the current task with a newly woken task if needed:
 *   
 */
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long ideal_runtime, delta_exec;

    ideal_runtime = sched_slice(cfs_rq, curr);
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    if (delta_exec > ideal_runtime)
        resched_task(rq_of(cfs_rq)->curr);
}
```

sched\_slice 是根据 weight 获取进程应该运行的时间片长度，参考 Fair Clock那一节提到的计算方法，在正常的调度情况下进程都耗尽自己的 CPU 时间。

```c
/*
 * Preempt the current task with a newly woken task if needed:
 */
static void check_preempt_wakeup(struct rq *rq, struct task_struct *p)
{
    struct task_struct *curr = rq->curr;
    struct cfs_rq *cfs_rq = task_cfs_rq(curr);
    struct sched_entity *se = &curr->se, *pse = &p->se;
    unsigned long gran;
    ...

    gran = sysctl_sched_wakeup_granularity;  // default = 10ms
    if (unlikely(se->load.weight != NICE_0_LOAD))
        gran = calc_delta_fair(gran, &se->load);

    if (pse->vruntime + gran < se->vruntime)
        resched_task(curr);
}
```

简单点理解，如果这个进程已经睡眠超过 10ms 了，那么我们就马上标记这个进程是可切换的，等下次 schedule\(\) 调用就会将这个进程切换出CPU。这个函数在 **try\_to\_wake\_up\(\)** 和 **wake\_up\_new\_task\(\)** 中均有调用，设计抢占是为了提升 interactivity，不赘述了。

```c
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
    u64 vruntime;
    ...
    ...        
    // for fork, initial = 1, else = 0
    vruntime = cfs_rq->min_vruntime;
    /*
     * The 'current' period is already promised to the current tasks,
     * however the extra weight of the new task will slow them down a
     * little, place the new task so that it fits in the slot that
     * stays open at the end.
     */
    if (initial && sched_feat(START_DEBIT))
        vruntime += sched_vslice_add(cfs_rq, se);

    if (!initial) {
        /* sleeps upto a single latency don't count. */
        if (sched_feat(NEW_FAIR_SLEEPERS) && entity_is_task(se))
            vruntime -= sysctl_sched_latency;

        /* ensure we never gain time by being placed backwards. */
        vruntime = max_vruntime(se->vruntime, vruntime);
    }

    se->vruntime = vruntime;
}
```

看，当进程苏醒的时候，进程的 vrunime 极大可能比队列里最小的 vruntime 还要小，注意此时进程还没有入列，所以它在下次调度之后可以得到执行。而普通的进程必然是比最小的 vruntime 要大的。注释也很清楚

```c
vruntime = max_vruntime(se->vruntime, vruntime);
```

记账函数是类似的，就不复制了。

## 末 - 亦是序

花这么长的篇幅介绍，其实最终目的只有一个，理解为什么可以精确控制进程使用CPU的时间，这里是通过 weight 实现的，下一章控制组相关知识，主角终于来了。

### References

* [the plot thicken](https://lwn.net/Articles/230574/)
* [FAQ style of CFS](https://oakbytes.wordpress.com/2012/06/06/linux-scheduler-introduction/)
* [CFS - 调度](https://juejin.im/post/5a97c9025188255579180e43#heading-13)
* [Inside the Linux 2.6 Completely Fair Scheduler](https://developer.ibm.com/tutorials/l-completely-fair-scheduler/)
* [Linux kernel scheduler](https://helix979.github.io/jkoo/post/os-scheduler/)
* [CFS design](http://people.redhat.com/mingo/cfs-scheduler/sched-design-CFS.txt)

