---
description: CFS - 续  cgroup - 序
---

# Task Group - 进程组

## 序

人生第一次面试，是网易的基础架构，我还是很后面才知道，原来这个其实就是运维，他们日常的工作有一个就是部署相关的服务，其中就涉及了分发资源给请求，其实涉及的一个思想就是容器，而容器其中的核心就是控制组。控制组当中最关键的是进程组。

## 相关结构

{% hint style="info" %}
上节介绍的是 CFS 的思想，具体的实现可以千变万化。
{% endhint %}

上一节没有介绍调度类的知识，其实非常的简单，就是一个抽象和模块的思想，调度器被抽象为了一个抽象类，提供相关的接口给 schedule\(\) 调用，每一个进程也被抽象为了类，供具体（实例化）的调度类使用。

![Graphical view of scheduling classes](../.gitbook/assets/image%20%2863%29.png)

其中我们关注的是 sched\_fair 这个类，_**schedule**_ 在每次调度就优先从 rt 中挑选新的进程，如果没有就切换别的 类，本质就是调用类实现的函数（地址）。

![Structure hierarchy for tasks and the red-black tree](../.gitbook/assets/image%20%2856%29.png)

上一节没有提到的一件事就是，所有任务抽象成一个 entity，然后以一个红黑树结构来存储，红黑树适合在动态性较强的情况下使用，Key 就是我们上节提到的 vruntime/fair\_clock，每次最左的一个节点就是下一个即将获得调度的 entity

均出自 [\_HERE\_](https://developer.ibm.com/tutorials/l-completely-fair-scheduler/) ，下图出自 [这儿](https://www.cnblogs.com/acool/p/6882644.html)

![structure of group scheduling](../.gitbook/assets/image%20%2835%29.png)

### 调度实体

先前我都是用 entity 来表示一个调度单位，因为这个**调度实体不一定是一个进程（线程）**，考虑一个情况，如果我将一个运行队列\(struct rq \)，抽象成一个 sched\_enity，当调度器选择到它的时候，其实说明，**这个队列的进程都有机会被选择执行**，更重要的一点，这个实体也维护了一个红黑书队列，它接着要做的，就是继续从它的队列里选择最左的一个 sched\_entity，依次选择，直到选择到了一个进程。

{% hint style="info" %}
所以，负责调度的函数，必须要知道，选择到的 sched\_entity 是一个队列，还是一个进程/线程
{% endhint %}

现在好好看看 sched\_entity 的结构。

```c
struct sched_entity {
    struct load_weight    load;        /* for load-balancing */
    struct rb_node        run_node;
    unsigned int        on_rq;

    u64            exec_start;
    u64            sum_exec_runtime;
    u64            vruntime;
    u64            prev_sum_exec_runtime;
    ...
    ...
#ifdef CONFIG_FAIR_GROUP_SCHED
    struct sched_entity    *parent;
    /* rq on which this entity is (to be) queued: */
    struct cfs_rq        *cfs_rq;
    /* rq "owned" by this entity/group: */
    struct cfs_rq        *my_q;
#endif
};
```

省略了一些统计的字段，这些结构光看变量名就可略知一二了，当系统支持组调度的时候，entity 里面还有字段记录了几个重要的值。

* entity 的父母
* entity 所在的队列，也就是说它应该被存放的地方（拥有红黑树的那个队列）
* entity 所代表的队列，也就是说，**队列被抽象成了一个调度实体，伪装成一个“进程”**

现在来看看 cfs\_rq 的结构

```c
/* CFS-related fields in a runqueue */
struct cfs_rq {
    struct load_weight load;
    unsigned long nr_running;
    ...
    ...
    struct rb_root tasks_timeline;
    /* 'curr' points to currently running entity on this cfs_rq.
     * It is set to NULL otherwise (i.e when none are currently running).
     */
    struct sched_entity *curr;
    ...

#ifdef CONFIG_FAIR_GROUP_SCHED
    struct rq *rq;    /* cpu runqueue to which this cfs_rq is attached */

    /*
     * leaf cfs_rqs are those that hold tasks (lowest schedulable entity in
     * a hierarchy). Non-leaf lrqs hold other higher schedulable entities
     * (like users, containers etc.)
     *
     * leaf_cfs_rq_list ties together list of leaf cfs_rq's in a cpu. This
     * list is used during load balance.
     */
    struct list_head leaf_cfs_rq_list;
    struct task_group *tg;    /* group that "owns" this runqueue */
#endif
};
```

原本，我们可能想会是这样一个结构

```c
struct cfs_rq_demo {
    ...
    struct sched_entity se;
    ...
}
```

如此以来，便完成了抽象，但是思考一下，绝大部分的 se 都是单一的进程，而不应该是一个队列，所以这样设计在存在大量进程的时候会出现浪费，所以实际上是用一个指针，存放了 task\_group 的地址，而且，se 中已经存放了指向 cfs\_rq 的指针，大部分我们的代码都只需要找到 cfs\_rq 即可，而从 cfs\_rq 到 se 这一路线，似乎不常见。

{% hint style="info" %}
具体的设计考虑具体需求
{% endhint %}

```c
struct task_group {
#ifdef CONFIG_FAIR_CGROUP_SCHED
    struct cgroup_subsys_state css;
#endif
    /* schedulable entities of this group on each cpu */
    struct sched_entity **se;
    /* runqueue "owned" by this group on each cpu */
    struct cfs_rq **cfs_rq;
    unsigned long shares;
    /* spinlock to serialize modification to shares */
    spinlock_t lock;
    struct rcu_head rcu;
};
```

之所以有数组，是因为如今的CPU都是多核的，必须每一个核都存在一个队列，换句话说，就是每一个核都有一个 se

## Group Schedule

前面提及了一些相关的结构 ，我们现在来看看，组调度的具体实现。

### init\_task\_group

思考一个问题，正常情况下， 用户是不会有创建进程组的需求的，就是所有的进程在调度下，依次运行，每一个核都只有一个队列，旗下是所有的进程。

慢着，我们是不是可以说，现在所有的进程，都正好在一个组内呢，也就说，现在只有一个组，是默认创建的，是最顶级的一个组，所有的进程都是它的内部成员。

实际上，内核也是这样处理的。

```c
/* Default task group's sched entity on each cpu */
static DEFINE_PER_CPU(struct sched_entity, init_sched_entity);
/* Default task group's cfs_rq on each cpu */
static DEFINE_PER_CPU(struct cfs_rq, init_cfs_rq) ____cacheline_aligned_in_smp;

static struct sched_entity *init_sched_entity_p[NR_CPUS];
static struct cfs_rq *init_cfs_rq_p[NR_CPUS];

/* Default task group.
 *    Every task in system belong to this group at bootup.
 */
struct task_group init_task_group = {
    .se     = init_sched_entity_p,
    .cfs_rq = init_cfs_rq_p,
};
```

下面看一个图，自己简单画了一下。

![](../.gitbook/assets/image%20%2892%29.png)

**红色箭头**是调度模块初始化的时候完成的，rq 是 Per CPU 变量，简单点理解就是一个数组，每个CPU都有一给自己的备份。

```c
void __init sched_init(void)
{
            ...
            for_each_cpu(i)
            {
                        struct cfs_rq *cfs_rq = &per_cpu(init_cfs_rq, i);
                        struct sched_entity *se =
                             &per_cpu(init_sched_entity, i);

                        init_cfs_rq_p[i] = cfs_rq;
                        init_cfs_rq(cfs_rq, rq);
                        cfs_rq->tg = &init_task_group;
                        list_add(&cfs_rq->leaf_cfs_rq_list,
                             &rq->leaf_cfs_rq_list);

                        init_sched_entity_p[i] = se;
                        se->cfs_rq = &rq->cfs;
                        se->my_q = cfs_rq;
                        se->load.weight = init_task_group_load;
                        se->load.inv_weight =
                             div64_64(1ULL<<32, init_task_group_load);
                        se->parent = NULL;
                        ...
            }
}
```

### 组调度的实现

看下图，这是一个嵌套的过程，当最高层的进程组\( init task group \)，选择了一个调度实体，发现它是一个队列，就 Down-cast\( se-&gt;my\_q \) 到 cfs\_rq，然后在这个队列选择，直到选择到的是一个进程（线程），然后进行进程上下文的切换。

![](../.gitbook/assets/image%20%2826%29.png)

上一节提到，有一个关键点在于，如何识别一个 se 是一个队列，而不是一个进程，其实简单的判断它的 _**my\_q**_ 字段 便可以知道，如果非空，那么就说明是一个队列了。

```c
/* runqueue "owned" by this group */
static inline struct cfs_rq *group_cfs_rq(struct sched_entity *grp)
{
    return grp->my_q;
}

static struct task_struct *pick_next_task_fair(struct rq *rq)
{
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;

    if (unlikely(!cfs_rq->nr_running))
        return NULL;

    do {
        se = pick_next_entity(cfs_rq);
        cfs_rq = group_cfs_rq(se);
    } while (cfs_rq);

    return task_of(se);
}
```

pick\_next\_entity 做的事情很容易想到，就是选择队列维护的红黑树最左的一个节点。

简单说说调度的过程吧。

* 外部时钟中断，进程超出自己所能执行的时间，进程被设置 换出标识
* schedule\( \) 调用，per-CPU-&gt;rq 可能有几个分队列，首先要处理 rt 进程，如果处理完成，现在开始处理 cfs\_rq 上的进程
* fair\_share\_clss-&gt;pick\_new\_task\_fair\( rq \) 就到了上面的函数
* 递归下降选择一个进程，过程就是这节陈述的，结合上节 cfs 的思想
* 进行进程上下文的切换

{% hint style="info" %}
per-CPU 变量就是全局变量，只是每个CPU有备份，相当于数组。
{% endhint %}

### 组调度的 weight

为什么要分组，光说可能还记忆不深刻，举个例子

![LWN.net](../.gitbook/assets/image%20%2832%29.png)

现在假设进程分成了俩个组，Guests 组内有三个进程， Sys tasks 只有一个进程，也就是说最顶层的 cfs 队列中只有俩个 sched\_entity，再假设这俩个的 entity的 weight 相同，也就是说，俩个 entity 占用的 CPU 的时间是一样的。

但是，需要注意的一点是，每次选到 Guests 这一个 entity，实际上它还要往下分配给三个 entity，我们小时候都做过 比例的 数学题。

$$
A:B = 1:1 \\
B:C = 3:1 \\
so \\
A:C = 3:1
$$

也就是说，单位时间内，Sys tasks 因为只有一个进程，那么它 总 的执行的时间就是 Guests 内的进程 3倍，这正是进程组的作用，**资源分配**。

更重要的一点，是俩个组的进程之间是隔离\( Isolation \)的，比如 G（Guests） 组内的进程无论是睡眠还是一直活跃的执行，都能保证的一点是，**S 组的进程一定能掌握 50% 的CPU使用率**。

{% hint style="info" %}
注意， Guest 和 Sys tasks 是平级的，如果他们的 weight 相同，那么这俩个组占用的 CPU 永远是 1 ：1，当然，如果 Guest 中有 50 个进程而 Sys 中只有一个，那么 Sys 一个进程就可以占用 50% 的 CPU，这是分组的特点
{% endhint %}

可能在这个例子不明显，现在考虑，如果有1000个进程，CFS调度会保证所有的进程占用 0.1%的 CPU，但是我们如果有一个服务型进程，需要把网络来的资源处理，必须保证它的 CPU 的使用率是 5%，这是如果改变它的 weight，将会变得非常大，因为要获得相对较大的使用率，为何不直接设置一个分组，并且可能存在多个进程也有类似的情况，那么分组就是一个非常好的选择。

无论另外一组的进程是睡眠或者就绪，另外一个组的进程的运行都是毫无干扰，一定有 5% 的 CPU 使用率。

实际应用的过程中，一个队列（或者说 task group）抽象而成的 se，其实是它们选出的一个_**代表**_，和其它的同一级 se（可能是进程，也可能是其他的组） 公平竞争，改变了这个代表的 weight，就相当于改变了所有旗下的进程的weight，**因为 se 的 weight，只影响在同一个队列的比重**。

{% hint style="info" %}
所以我们说容器，容器，容器其实就是隔离，分配资源。在大型服务机构，往往有成千上万的资源需要分发，容器的作用体现在这里。
{% endhint %}

现在考虑，俩个组，一个组 A的 weight 为 1024，另一个组 B 的 weight 则是 512，各有一个进程a，b。那么当俩个进程都是就绪态的时候，占用的 CPU的比例是多少？

答案是 a : b = 2：1（66.7%：33.3%），那假设还有一个进程 c不在任何组内，但是和这俩个组平级呢？

![](../.gitbook/assets/image%20%2877%29.png)

答案是

$$
a :b:c=2:2:1
$$

读者还可以考虑如果 B组 中的进程不止一个，有俩个的情况，又当如何？

由于 B 的 weight 比较小，所以在执行相同的时间，它的 vruntime 就会增长的非常快，因为内核从最顶层开始调度，正常情况下，latency 是固定的，即这个时间内，需要调度完队列里所有的 se（注意不一定是进程 ），下面这个函数就是计算 se 的时间片

```c
/*
 * We calculate the wall-time slice from the period by taking a part
 * proportional to the weight.
 *
 * s = p*w/rw
 */
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    u64 slice = __sched_period(cfs_rq->nr_running);

    slice *= se->load.weight;
    do_div(slice, cfs_rq->load.weight);

    return slice;
}

/*
 * Preempt the current task with a newly woken task if needed:
 * 判断进程是否已经使用完了自己的进程片
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

正常情况下，是这样计算

$$
slice = sysctl\_sched\_latency*\frac{se->weight}{cfs\_rq->weight}
$$

{% hint style="info" %}
这里特别用 se 来 替换 task，让读者们了解，一个可调度实体可不一定是一个 process
{% endhint %}

那么继续以刚才的例子分析，一个 slice 中，a:b:c = 2:2:1，并且在这种情况下（ se得到的时间不均 ），它们增加 vruntime 相同，然后我们考虑，如果现在选择的是 B 组所代表的 se

![](../.gitbook/assets/image%20%2891%29.png)

显然，由于 B 组的队列只有一个进程，那么理论上来说呢，那么以 B的角度来说（B也是一个队列，它也要判断自己下面的 se，是否消耗完自己的时间片 ），考虑刚才的公式，b 是不是直接得到了整个 slice？因为这个队列没有其余的进程和它分这一块蛋糕。

答案当然是否定的，我们来看看每一个时钟中断，是如何对这种嵌套的结构进行 slice 判断的。

### 一个调度周期的结束

以 2.6.24 的代码来分析，每一个时钟中断会出现下面的调用。

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
    /*
     * Update run-time statistics of the 'current'.
     */
    update_curr(cfs_rq);

    if (cfs_rq->nr_running > 1 || !sched_feat(WAKEUP_PREEMPT))
        check_preempt_tick(cfs_rq, curr);
}

/*
 * Preempt the current task with a newly woken task if needed:
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

tick\_task\_fair 当每个周期出现的时候，就更新记账信息，注意，se 以及 se 的父母\( 即一个队列 \) 都会更新自己的 vruntime，刚才举例的时候，组 B 的 weight 512，而进程 b 没有说明，默认就是 1024，所以你会发现，**它们俩的 vruntime 增长的速度不一样**。

所以，虽然 b进程得到了整个 slice，但是更新信息，是一个 自底向上 的过程，一段时间后，虽然以 b 的角度来说，它没有超时，但是，队列到了最上层，发现 B 所代表的 se，超时了，这种情况下，b 进程仍然被换出。

这个过程值得好好理解，对于队列来说，它不在乎旗下的 se 是进程或是队列，它保证的是一点，slice 时间内，所有的 se 都得调度一遍，并且更新 vruntime 的时候，如果存在父 se，向上更新，父 se（以这个例子就是 B ） 在那一层来说也是作为 se 存在，并且平级（三个 se，平等对待），上一层的队列，考虑的也是 slice 期间内，要全部 se（挂载它的队列上的se ）调度一遍。

这都是为了 组调度 而服务～

## Control Group

上面一节说明了，进程组存在的关键就是提供了 进程之间的隔离以及资源的分配。现在由于云计算越来越受到重视，对资源分配的需求的也日渐提高，所以容器这几年才会如此流行。

控制组，是任务组的超集，它不只是提供了CPU的占用，这一资源的分配，而是提供了诸如，内存，CPU核以及硬盘等一系列资源的限制，实际上这是不是模拟了一个简单的虚拟机，让一个进程生活在所有的资源都能被限制的环境下，不错，这就是 CG 的设计理念，轻量型虚拟化。

一个进程所需要的全部资源都能被限制，就可以说做到了这一点。会有这个需求，是因为如今的企业会提供云服务，也就是你付钱，我给你一台服务器，你来设计服务端给自己的客户提供需求。但是，怎么可能真的提供一台独立的服务器呢，所以就需要虚拟一台，企业往往有成千上万的 CPU，内存还有硬盘等等资源，所以把一整块虚拟化成多个 服务单元，并且可以方便的给管理员统一管理。

这是我个人理解的好处，肯定不止这些，比如这对于服务器的扩建也是非常容易的，只需要额外的分配一些 CPU 还有内存即可，因为往往这个服务器占用的资源只是一部分，这也是虚拟化的一大优点。

### TG & CG

TG\( Task Group \) 是 CG（Control Group）的一个子部件，CG 的实现的一部分是 TG，但是 CG 额外提供了上层的接口，比如，让用户可以新建一个 Task Group，然后可以将相关的任务添加到新建的组内等等。

如果非要把两者区分开，其实就是超类和基类的区别。之前我们所说的 TG，只包括了内核的任务分组的思想，最终是需要 CG 实现的完成的接口才能说完整。

TG 是作为 CPU 的资源控制存在，也就是说对进程所在的组的 CPU 资源的控制由 TG 实现了核心模块，我们在讲解 CFS 的时候，已经知道通过 weight 来限制 CPU 资源，本质就是那儿~

现在我们来关心的问题是，上层的提供的接口是如何与这个模块对接的。

### CG 的调用链

这里阐述的是，从上层接口到最后与 TG 模块对接那一部分，其他的子模块也是类似的，所以这里一次性讲清楚，日后分析其他模块也更加清晰。

CG 上层接口是通过一个文件系统实现的，类似 proc，sysfs等，并不是真正才能在于硬盘的一个文件系统的接口，利用的是 VFS 模块，这样的优点就是上层的操作又变成了简单的 Read/Write/Open，这也是\*nix 类操作系统的一大特色，其实 Windows 也是这样，只是参数比较复杂。

早期的是挂载在 proc ，后来都是依靠 sysfs，但是核心思想几乎没有改变。

![Whole Picture](../.gitbook/assets/image%20%2819%29.png)

CG 暴露给的接口给用户，通过 VFS 流程，来到了内核，最终和其子模块对接，我们以进程组为例子，上层添加一个 pid 到一个组的调用过程如下。

```bash
echo $$ >>  /mnt/point/mycg/tasks

## 添加当前 shell 的 pid 到 mycg（cgroup）的那一组
```

{% hint style="info" %}
mycg 是先前创建的一个 cgroup，当然在那时会创建一个 task\_group 并且抽象为 se 添加到与它同级的 cfs\_rq 中，也就是上一节提到的 cfs\_rq 的 “伪装”
{% endhint %}

![~~~](../.gitbook/assets/image%20%2861%29.png)

## 总结

这一节的关键在于理解，一个组如何伪装成一个可调度实体（se），并且不同级的进程占用 CPU 的比例是如何造成的，这非常有助于理解分组的思想，这里对于 CG 的讲解非常省略，因为它可以单独开一章节了，这里的重点应该是 组调度。

## References

* [CFS group scheduling](https://lwn.net/Articles/240474/)
* [Inside the Linux 2.6 Completely Fair Scheduler](https://developer.ibm.com/tutorials/l-completely-fair-scheduler/)
* [Process Container](https://lwn.net/Articles/236038/)

