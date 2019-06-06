---
description: 续
---

# Control Group - 控制组

## E.G.

先从一个例子出发，w表示的是weight，现在最顶层的 cfs\_rq 中维护着俩个活跃的进程，现在假设它们在同**一个 CPU 核**运行，那么我们运行 top 的结果应该是怎么样的？

![](../.gitbook/assets/image%20%28110%29.png)

应该是这样的，b c俩个进程占用的CPU的资源之和应该是 另外一个 se 的 1/2，实际也是如此，而 b c 俩个进程公平竞争，Group B 得到的时间片。

{% hint style="info" %}
b c 本身也是一个 sched\_entity ，忘记在图片中提及。
{% endhint %}

![](../.gitbook/assets/image%20%2870%29.png)

说明一点，三个进程做的都是 while\(true\) 死循环，所以不存在 yield 的可能～

## 实现的过程

{% hint style="info" %}
注意，所有的注释图，但凡涉及到 list\_head 都是双向链表，当时没有注意到，所以读者得留心，大部分都是双向索引的～
{% endhint %}

实现的本质在组调度已经说明了，现在说的得是上层接口了。

```bash
## 新建俩个 cgroup
cgcreate -g cpu,cpuset:mycg 
```

上面使用的只是工具，完成的工作很简单，俩个 mkdir 指令

```bash
[root@centos ~] cgcreate -g cpu,cpuset:mycg 
[root@centos ~] 
[root@centos ~] cd /sys/fs/cgroup/
[root@centos cgroup] find . -name "mycg"  -print
./cpuset/mycg
./cpu,cpuacct/mycg
[root@centos cgroup]
```

所以我们可以，手动进入这俩个目录使用 mkdir 命令，做的事情是一样的。

{% hint style="info" %}
cpu 指的就是 之前说的 组调度， cpuset 是改变进程运行的所在 CPU 核
{% endhint %}

## Cgroup 文件系统

代码出自内核 2.6.24，Cgroup v1.0 那时的代码

### 关键的数据结构

#### cgroup

说核心把，这个结构其实只起了连接的作用，所以不要觉得它能具体处理什么，它只是一个统筹的结构。当我们在 /sys/fs/cgroup/cpu 下 mkdir 的时候其实就创建了这么一个结构，它代表的就是一个控制单元，里面可能受到一个或多个 资源控制器 的限制。也可以理解为，它代表了一块受限的资源。   

```c
struct cgroup {
	unsigned long flags;		/* "unsigned long" so bitops work */

	/* count users of this cgroup. >0 means busy, but doesn't
	 * necessarily indicate the number of tasks in the
	 * cgroup */
	atomic_t count;

	/*
	 * We link our 'sibling' struct into our parent's 'children'.
	 * Our children link their 'sibling' into our 'children'.
	 */
	struct list_head sibling;	/* my parent's children */
	struct list_head children;	/* my children */

	struct cgroup *parent;	/* my parent */
	struct dentry *dentry;	  	/* cgroup fs entry */

	/* Private pointers for each registered subsystem */
	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];

	struct cgroupfs_root *root;
	struct cgroup *top_cgroup;
	
	
	
	/*
	 * List of cg_cgroup_links pointing at css_sets with
	 * tasks in this cgroup. Protected by css_set_lock
	 */
	struct list_head css_sets;

	/*
	 * Linked list running through all cgroups that can
	 * potentially be reaped by the release agent. Protected by
	 * release_list_lock
	 */
	struct list_head release_list;
};

```

#### cgroupfs\_root -- Hierarchy

```c
/*
 * A cgroupfs_root represents the root of a cgroup hierarchy,
 * and may be associated with a superblock to form an active
 * hierarchy
 */
struct cgroupfs_root {
	struct super_block *sb;

	/*
	 * The bitmask of subsystems intended to be attached to this
	 * hierarchy
	 */
	unsigned long subsys_bits;

	/* The bitmask of subsystems currently attached to this hierarchy */
	unsigned long actual_subsys_bits;

	/* A list running through the attached subsystems */
	struct list_head subsys_list;

	/* The root cgroup for this hierarchy */
	struct cgroup top_cgroup;

	/* Tracks how many cgroups are currently defined in hierarchy.*/
	int number_of_cgroups;

	/* A list running through the mounted hierarchies */
	struct list_head root_list;

	/* Hierarchy-specific flags */
	unsigned long flags;

	/* The path to use for release notifications. No locking
	 * between setting and use - so if userspace updates this
	 * while child cgroups exist, you could miss a
	 * notification. We ensure that it's always a valid
	 * NUL-terminated string */
	char release_agent_path[PATH_MAX];
};
```

每当用户挂载了一个新的 cgroup 文件系统的时候，就会创建这个结构，注意它其实**内嵌**了一个 cgroup，也可以称为**最高层**的 cgroup。

![embedded cgroup  ](../.gitbook/assets/image%20%2816%29.png)

注意，用户的每一次 mount 操作，其实就创建了一个 Hierarchy，其实就是创建了一棵资源管理树，别的地方还是直译为层级，实际上非常难以理解。

值得一提的是，**每个资源管理器\( subsystem \)，只能绑定在一个 Hierarchy** ，比如 cpu 这个 资源管理器，如果你在 mount 的时候指定了 你所需要的 资源管理器，那么它就不能属于别的 资源管理树了。

```bash
$  mount -t cgroup -o cpu,cpuset hier1 /mnt/point
```

这可以理解为，创建了一个名字叫 hier1 的 Hierarchy，然后它绑定了 cpu cpuset 俩个资源管理器（sub- system） ，所以保证的一点就是，这俩个 subsystems 没有被其它已经创建的 hierarchy 所使用。

```c
/*
 * The "rootnode" hierarchy is the "dummy hierarchy", reserved for the
 * subsystems that are otherwise unattached - it never has more than a
 * single cgroup, and all tasks are part of that cgroup.
 */
static struct cgroupfs_root rootnode;
 虚拟根节点，用来绑定全部的资源管理器

/* The list of hierarchy roots */
 每次 mount 就会创建一个 cgroup rootfs hierachy)， 挂载在这，最开始的 dummy top也在
static LIST_HEAD(roots);
static int root_count;

/* dummytop is a shorthand for the dummy hierarchy's top cgroup */
#define dummytop (&rootnode.top_cgroup)
```

值得一提的是，最开始初始化的时候，创建了一个虚拟的 root\( dummy top \)，用来挂载内核**全部**的资源管理器。

![&#x6CE8;&#x610F; dummytop &#x6CA1;&#x6709;&#x8FDE;&#x63A5; subsystem](../.gitbook/assets/image%20%28101%29.png)

其实，最开始的创建的一个 dummy top 就可以认为是一个 Hierarchy，只是它连接了全部的资源管理器。

{% hint style="info" %}
这里只画出了俩个 subsystems
{% endhint %}

当用户挂载一个新的 Hierarchy的时候，会指定它需要的 资源管理器（subsystem）。后来新建的 Hierarchy 和 最开始的 dummy top 是平级的，挂载在一个全局的链表上。

![Hierarchy is a  mount point ~](../.gitbook/assets/image%20%2869%29.png)

{% hint style="info" %}
渐渐的把 资源管理器用 subsystem 替代，资源管理树用 Hierarchy 替代，读者必须理解的就是，subsystem 与 Hierarchy 的附属关系。
{% endhint %}

#### cgroup\_subsys

刚才提及的 subsystem 登场了。

```c
/* Control Group subsystem type. See Documentation/cgroups.txt for details */
struct cgroup_subsys {
	struct cgroup_subsys_state *(*create)(struct cgroup_subsys *ss,
						  struct cgroup *cont);
	void (*destroy)(struct cgroup_subsys *ss, struct cgroup *cont);
	int (*can_attach)(struct cgroup_subsys *ss,
			  struct cgroup *cont, struct task_struct *tsk);
	void (*attach)(struct cgroup_subsys *ss, struct cgroup *cont,
			struct cgroup *old_cont, struct task_struct *tsk);
	void (*fork)(struct cgroup_subsys *ss, struct task_struct *task);
	void (*exit)(struct cgroup_subsys *ss, struct task_struct *task);
	int (*populate)(struct cgroup_subsys *ss,
			struct cgroup *cont);
	void (*post_clone)(struct cgroup_subsys *ss, struct cgroup *cont);
	void (*bind)(struct cgroup_subsys *ss, struct cgroup *root);
	int subsys_id;
	int active;
	int early_init;
#define MAX_CGROUP_TYPE_NAMELEN 32
	const char *name;

	/* Protected by RCU */ 
	指向它所绑定的 Hierarchy，注意一开始都在dummytop上	
	struct cgroupfs_root *root;

	struct list_head sibling; // Hierarchy用它来遍历旗下所有的 Subsystems

	void *private;
};
```

这么多指针函数，一看就是类定义把～，subsystem 这个词太容易让人误解了，以前的驱动架构里面也有这个说法，后来移除了，中文我习惯把它叫为 **资源管理器**，它就是管理资源的，但是国内很多人都直接直译了它的英文，实在让人难懂。前面我们提到的，cpu, cpuset... 都是一个具体的 \( subsystem \)资源管理器。

 当用户 mount 时，即创建了一个 Hierarchy，就会要求 资源管理器 和 它绑定，即此时就会从 dummy top 上脱离开来。

![](../.gitbook/assets/image%20%2879%29.png)

{% hint style="info" %}
注意  cgfs\_root 的 subsys\_list 用来连接绑定的 subsystem，subsys 中则有 cgrootfs 的指针
{% endhint %}

来看看 cpu 的实现

```c
struct cgroup_subsys cpu_cgroup_subsys = {
	.name		= "cpu",
	.create		= cpu_cgroup_create,
	.destroy	= cpu_cgroup_destroy,
	.can_attach	= cpu_cgroup_can_attach,
	.attach		= cpu_cgroup_attach,
	.populate	= cpu_cgroup_populate,
	.subsys_id	= cpu_cgroup_subsys_id,
	.early_init	= 1,
};
```

都是具体指向的函数，对了，值得一提的是什么呢，内核设置了一个全局的数组指针，指向了目前的所有的 资源控制器。

```c
/* Generate an array of cgroup subsystem pointers */
#define SUBSYS(_x) &_x ## _subsys,

static struct cgroup_subsys *subsys[] = {
#include <linux/cgroup_subsys.h>
};


拷贝自  "cgroup_subsys.h"
#ifdef CONFIG_CPUSETS
SUBSYS(cpuset)
#endif

#ifdef CONFIG_FAIR_CGROUP_SCHED
SUBSYS(cpu_cgroup)
#endif
...
```

macro 展开后，就是一个取址的操作了～

{% hint style="info" %}
每个 subsystem 都有一个 unique id，对于 cpu 来说 这就是 cpu\_cgroup\_subsys\_id，为的就是辨别，并且，在下面提到的全局数组，这个 id 其实就是它的 index
{% endhint %}

#### cgroup\_subsys\_state

在最开始接触 cgroup 的时候， 最容易弄混的俩个结构，就是它和 刚才提到的 subsystem，刚才我们说了，subsystem 是一个资源管理器，它是一个类，会绑定在一个 Hierarchy（资源管理树）上，表达的意思是，这棵上的进程都受这个资源管理器所限制，思考一下，Hierarchy 是可以有子树，假设我们最顶层的那个资源管理器，以cpuset为例子，我们给了它 1-4 的 CPU核，然后我创建了一个子树（子 Hier），我们现在想分配它 1个核。

现在是不是出现了资源不一样的情况，一个 subsystem 只能绑定在一个 Hierarchy，意味着它只能有一份，然后我们又需求的是 同一个 Hierarchy 也会有资源不一样的情况，所以有了它的存在。

我习惯把它理解为，一个具体的 “资源”，subsystem 只是强调了 资源管理器 的存在，而到底拥有多少资源，是这个结构说了算，通过 suffix “ state ”也能略晓一二，描述的是资源的状态。

{% hint style="info" %}
以 cpu 这个 subsys 来说，就是同一个 Hierarchy 中的进程可能也不在一个 进程组，所以有了这个结构的存在。
{% endhint %}

```c
/* Per-subsystem/per-cgroup state maintained by the system. */
struct cgroup_subsys_state {
	/* The cgroup that this subsystem is attached to. Useful
	 * for subsystems that want to know about the cgroup
	 * hierarchy structure */
	struct cgroup *cgroup;

	/* State maintained by the cgroup system to allow
	 * subsystems to be "busy". Should be accessed via css_get()
	 * and css_put() */

	atomic_t refcnt;

	unsigned long flags;
};
```

注意它有一个 create 的接口，目的就是让一个资源管理器 创建 一个具体的 **资源**，

```c
struct cgroup_subsys {
	struct cgroup_subsys_state *(*create)(struct cgroup_subsys *ss,
						  struct cgroup *cont);
	...
}
```

cgroup 和这些 “资源挂钩”，用一个数组连接到具体的资源，index 就是之前介绍的 unique 的 subsys\_id，用来区分具体是哪个资源。

```c
struct cgroup {
			...
			/* Private pointers for each registered subsystem */
			struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
			...
}
```

前面我们说，默认存在的 dummy top 也是一个 Hierarchy ，所以在最开始初始化的时候，也会给他分配一个这样的 css\( 简称 \)。

![css &#x548C; cg &#x53EF;&#x4EE5;&#x4E92;&#x76F8;&#x7D22;&#x5F15;](../.gitbook/assets/image%20%28106%29.png)

当然，对于 dummy top，默认的资源当然是无限制，回想我们在组调度提到的，init\_task\_cgroup，一开始就是全部进程都在一个组内，所以它其实拥有全部的系统资源。当用户自己新建一个 Hierarchy 的时候，即 mount ，也不会调用 subsys-&gt;create\(\) 函数，只有当用户新建一个子cgroup的时候，才会调用，以获取一个新的 css。

最开始，所有的进程都在一个组，emm，意思就是，最开始所有的 subsystem 都属于一个dummy top，当用户 mount 一个 hierarchy 时，这个 Hierarchy 就必须作为 最顶层 Cgroup 了，那它就必须保证，现在**所有受到这个资源控制的进程都必须在最顶层的那一组**，简单点理解，就是所有进程在没有添加到子 Hierarchy\( Cgroup \) 时，都属于最高层的那一个cgroup，而初始化的时候，就已经全部添加到一个组内了，所以现在还不需要新建一个 css 。

具体的实现，就是 mount 时候，新建的 cgfs\_root 内嵌的cg，会接替 dummy top 的 css

![](../.gitbook/assets/image%20%2848%29.png)

其实关键就在于，**Hierarchy 必须要把之前 dummy top 那个组的全部的进程 接手**，接手的过程就是接替那个 css，还有 subsystem，至于为什么能，就要看下面的 css\_set 了。

{% hint style="info" %}
mount 就是新建一个 Hierarchy
{% endhint %}

#### css\_set

来看看，比较难懂的一个结构，我放在了最后解释它，它是 css\( cgroup\_subsys\_state \)\_set

```c
struct css_set {

	/* Reference count */
	struct kref ref;

	/*
	 * List running through all cgroup groups. Protected by
	 * css_set_lock
	 */
	struct list_head list;

	/*
	 * List running through all tasks using this cgroup
	 * group. Protected by css_set_lock
	 */
	struct list_head tasks;

	/*
	 * List of cg_cgroup_link objects on link chains from
	 * cgroups referenced from this css_set. Protected by
	 * css_set_lock
	 */
	struct list_head cg_links;

	/*
	 * Set of subsystem states, one for each subsystem. This array
	 * is immutable after creation apart from the init_css_set
	 * during subsystem registration (at boot time).
	 */
	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];

};

/* Link structure for associating css_set objects with cgroups */
struct cg_cgroup_link {
	/*
	 * List running through cg_cgroup_links associated with a
	 * cgroup, anchored on cgroup->css_sets
	 */
	struct list_head cgrp_link_list;
	/*
	 * List running through cg_cgroup_links pointing at a
	 * single css_set object, anchored on css_set->cg_links
	 */
	struct list_head cg_link_list;
	struct css_set *cg;
};
```

css\_set 是个什么玩意儿呢？其实就是如它的名字所暗示的，它是一个集合，代表的是一个进程所受哪些具体的 **资源** 限制，现在看起来很复杂，现在要知道的就是，是一个 set！

> * Each task in the system has a reference-counted pointer to a css\_set.
> * A css\_set contains a set of reference-counted pointers to cgroup\_subsys\_state objects, one for each cgroup subsystem registered in the system. There is no direct link from a task to the cgroup of which it's a member in each hierarchy, but this can be determined by following pointers through the cgroup\_subsys\_state objects. This is because accessing the subsystem state is something that's expected to happen frequently and in performance-critical code, whereas operations that require a task's actual cgroup assignments \(in particular, moving between cgroups\) are less common. A linked list runs through the cg\_list field of each task\_struct using the css\_set, anchored at css\_set-&gt;tasks.

比如，我们最开始的例子，进程就受到 cpuset cpu 俩个 **资源** 控制，限制了它的 CPU 核 以及 CPU 的资源占用，那么可以说对于 b c 俩个进程，它们都在一个 css\_set，如果还有其它进程也在跟他们在一个组内（受同样的 **资源**  控制），那么只需要增加 css\_set 的一个引用计数就好了，它是为了节约资源才设计的。

可能还有一个进程，只受到 cpuset 的限制，但是它并不受到 另外一个组的 cpu 资源的限制，所以它们不在一个 set，为了应对各种复杂的情况，所以出现了这个 css\_set

注意，我们常见的操作是，为每一个 subsystem 挂载一个 Hierarchy，也就是说， subsystem 对应一个 Hierarchy，然后，具体在根据子类分配资源。如下图，三个 task 只需要共用一个 css\_set

![css\_set &#x548C; cgroup &#x7684;&#x8FDE;&#x63A5;&#x6BD4;&#x8F83;&#x9690;&#x6666;&#xFF0C;&#x7B49;&#x4F1A;&#x8BF4;&#x660E;&#xFF0C;&#x56FE; A](../.gitbook/assets/image%20%28116%29.png)

内核也有默认的css\_set，目的是为了连接 dummy top 上的 css 还有上面所有的进程，以及其余所有的 css\_set，这里没有画出来，剩余的全部进程可能都在最开始的 css\_set 上。

```c
/* The default css_set - used by init and its children prior to any
 * hierarchies being mounted. It contains a pointer to the root state
 * for each subsystem. Also used to anchor the list of css_sets. Not
 * reference-counted, to improve performance when child cgroups
 * haven't been created.
 */

static struct css_set init_css_set;
static struct cg_cgroup_link init_css_set_link;

/* css_set_lock protects the list of css_set objects, and the
 * chain of tasks off each css_set.  Nests outside task->alloc_lock
 * due to cgroup_iter_start() */
static DEFINE_RWLOCK(css_set_lock);
static int css_set_count;

/* A css_set is a structure holding pointers to a set of
 * cgroup_subsys_state objects. This saves space in the task struct
 * object and speeds up fork()/exit(), since a single inc/dec and a
 * list_add()/del() can bump the reference count on the entire
 * cgroup set for a task.
 */

```

所以最一开始，init\_css\_set 是这样的：（假设只有俩个 subsystems ）

![&#x56FE; B](../.gitbook/assets/image%20%2878%29.png)

所以，最开始所有的 task 都共用一个 css\_set，css\_set 会把全部的任务串连起来，图上很多指针没有画出来，因为太复杂了，不如突出重点。

css set 的存在，就是把拥有同一个 **资源（state）** 的任务连接在了一起，注意，不是 **subsystem**，可以参考上上张图，当创建了一个 cgroup\( 注意，不是 mount，在 vfs 上体现为 mkdir \)，就会创建一个 子css，这时候添加新的 task 进去，就会导致 进程脱离原来的 set。**css\_set 就是资源集合，在一个 set 上的进程共享相同的资源**。

现在来看看，cg\_link 这个结构，目的是为了连接与它挂钩的 cgroup，我们现在要知道一点，首先，**一个 cgroup可能连接多个 css\_set，一个 css\_set 可能与多个 cgroup 有关**，这里我用 cgroup，实际上也可以理解为 css\( 资源 \)，对于 cgroup\_roofs\( top cgroup/ hierarchy \)拥有全部的资源（最早 dummy top 建立的），而后来创建的可能只拥有部分。

```c
/* Link structure for associating css_set objects with cgroups */
struct cg_cgroup_link {
	/*
	 * List running through cg_cgroup_links associated with a
	 * cgroup, anchored on cgroup->css_sets
	 */
	struct list_head cgrp_link_list;
	/*
	 * List running through cg_cgroup_links pointing at a
	 * single css_set object, anchored on css_set->cg_links
	 */
	struct list_head cg_link_list;
	struct css_set *cg;
};
```

以图 A（上上张图），为例子，会是这样的。

![&#x5BF9;&#x4E8E; list\_head &#x7BAD;&#x5934;&#x90FD;&#x662F;&#x53CC;&#x5411;&#x7684;&#xFF0C;&#x8FD9;&#x91CC;&#x6CA1;&#x6709;&#x753B;&#x51FA;](../.gitbook/assets/image%20%2846%29.png)

从 css\_set 的角度来说，cg\_links字段，可以遍历全部的 cg\_links，cg\_links 的 cgrp\_link\_list 就指向的是对应的拥有 css（资源） 的那个 cgroup。

从 cgroup 的角度来说，css\_sets 字段，可以遍历全部的 cg\_links，代表的是 想要分享我资源的 css\_set，着是非常重要的！！！

![](../.gitbook/assets/image%20%2856%29.png)

如果在加上，css ，那必然是，css\_set1 会指向俩个资源，而 2，3分别拥有 cg1 2的资源，注意 cg\_links 都是 二进二出。

![&#x4E0B;&#x9762;&#x8FD8;&#x6709;&#x4E00;&#x4E2A;&#x6307;&#x9488;&#x4F1A;&#x6307;&#x5411;&#x522B;&#x7684; cg](../.gitbook/assets/image%20%28100%29.png)

这张图应该更好理解，这也是比较复杂的结构了。总之，我们要知道，css\_set 里进程，拥有相同的**资源**，这个结构是让进程更方便知道自己到底在哪个 cgroup，到底拥有多少资源。

 现在 review 一下这个结构！

```c
struct css_set {

	/* Reference count */
	struct kref ref;

	/*
	 * List running through all cgroup groups. Protected by
	 * css_set_lock
	 */
	struct list_head list;

	/*
	 * List running through all tasks using this cgroup
	 * group. Protected by css_set_lock
	 */
	struct list_head tasks;

	/*
	 * List of cg_cgroup_link objects on link chains from
	 * cgroups referenced from this css_set. Protected by
	 * css_set_lock
	 */
	struct list_head cg_links;

	/*
	 * Set of subsystem states, one for each subsystem. This array
	 * is immutable after creation apart from the init_css_set
	 * during subsystem registration (at boot time).
	 */
	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];

};
```

### 总结一下

怎么样，是不是够复杂！现在做个总结，

cgroup 就是 掌握若干 资源 的一个集合。

![](../.gitbook/assets/image%20%2810%29.png)

cgroupfs\_root 是一个 Hierarchy，它内嵌了一个 cgroup，还可以创建子group，mount 操作会创建新的 Hierarchy，并且会绑定若干个 subsystem，（一个 subsystem 只能绑定一个 hier），当创建了子cgroup，cgroup 会调用，subsys-&gt;create\(\) 生成新的 资源\( css \)，更重要的是，Hierarchy 会继承最开始为dunmmy top 创建的 css，就是说，接管了全部的资源（当然，指的是绑定了它的 subsystem的资源）。

```c
static int rebind_subsystems(struct cgroupfs_root *root,
			      unsigned long final_bits)
{
	unsigned long added_bits, removed_bits;
	struct cgroup *cgrp = &root->top_cgroup;
	int i;

	/* Process each subsystem */
	for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
		struct cgroup_subsys *ss = subsys[i];
		unsigned long bit = 1UL << i;
		if (bit & added_bits) {
			/* We're binding this subsystem to this hierarchy */
			cgrp->subsys[i] = dummytop->subsys[i];
			cgrp->subsys[i]->cgroup = cgrp;
		// 这句话就可以实现接管，因为 css->cgroup 就表示了它所在cgroup
			list_add(&ss->sibling, &root->subsys_list);
			rcu_assign_pointer(ss->root, root);
			if (ss->bind)
				ss->bind(ss, cgrp);
		} 
	}
	root->subsys_bits = root->actual_subsys_bits = final_bits;
	synchronize_rcu();

	return 0;
}
```

因为最初的初始化，init\_css\_set 将所有的资源都掌握了，如今这里只需要改变了 css-&gt;cgroup 就可以保证，一个 Hierarchy 接管了原来掌握该资源所有的进程。

![hier1\_cpu &#x63A5;&#x7BA1;&#x4E86;&#x5168;&#x90E8; cpu &#x8D44;&#x6E90;](../.gitbook/assets/image%20%2842%29.png)

subsystem 和 cgroup\_subsys\_state，一个是资源管理器，一个是具体的资源，前者提供了 create 接口，用以获取相关的资源。

css\_set 连接若干的 css，忘了提示一点，set 中有 css 数组，**这个数组指向的资源\(css\)是不会重复**的，**一个进程可以受多个 Hierarchy 控制，因为不同的 Hierarchy 对应的不同的资源类型**，但是不可能说拥有俩个资源，这俩个资源都是在一个 Hierarchy，因为它们在数组的 index 是相同的，从常理上也理解不通，同样的资源如何还能分开计算呢？

css\_set 和 cgroup 是多对多的关系，这很重要。上图没有画出，css 连接那俩个 cgroup，读者自己要心里有数，它们是靠 cgroup\_links 实现的。

### 初始化与 mount 过程

本来还想提提 VFS 流程，想想都说烂了，而且这个篇幅，已经有点超出我的想象了，本来想讲的更详细一些，0.0，还是只提关键部分好吧，VFS 可以参考之前的文章。

下面就是最早的初始化代码，后续还有一个注册 proc 文件，就不贴出了。

```c
/**
 * cgroup_init_early - initialize cgroups at system boot, and
 * initialize any subsystems that request early init.
 */
int __init cgroup_init_early(void)
{
	int i;
	kref_init(&init_css_set.ref);
	kref_get(&init_css_set.ref);
	INIT_LIST_HEAD(&init_css_set.list);
	INIT_LIST_HEAD(&init_css_set.cg_links);
	INIT_LIST_HEAD(&init_css_set.tasks);
	css_set_count = 1;
	init_cgroup_root(&rootnode);
	list_add(&rootnode.root_list, &roots);
	root_count = 1;
	
	/******************************/
	init_task.cgroups = &init_css_set;
	/******************************/

	init_css_set_link.cg = &init_css_set;
	list_add(&init_css_set_link.cgrp_link_list,
		 &rootnode.top_cgroup.css_sets);
	list_add(&init_css_set_link.cg_link_list,
		 &init_css_set.cg_links);

	for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
		struct cgroup_subsys *ss = subsys[i];
		...

		if (ss->early_init)
			cgroup_init_subsys(ss);
	}
	return 0;
}

static void cgroup_init_subsys(struct cgroup_subsys *ss)
{
	struct cgroup_subsys_state *css;
	struct list_head *l;

	printk(KERN_INFO "Initializing cgroup subsys %s\n", ss->name);

	/* Create the top cgroup state for this subsystem */
	ss->root = &rootnode;
	css = ss->create(ss, dummytop);
	/* We don't handle early failures gracefully */
	BUG_ON(IS_ERR(css));
	init_cgroup_css(css, ss, dummytop);

	/* Update all cgroup groups to contain a subsys
	 * pointer to this state - since the subsystem is
	 * newly registered, all tasks and hence all cgroup
	 * groups are in the subsystem's top cgroup. */
	write_lock(&css_set_lock);
	l = &init_css_set.list;
	do {
		struct css_set *cg =
			list_entry(l, struct css_set, list);
		cg->subsys[ss->subsys_id] = dummytop->subsys[ss->subsys_id];
		l = l->next;
	} while (l != &init_css_set.list);
	write_unlock(&css_set_lock);

	....
	need_forkexit_callback |= ss->fork || ss->exit;

	ss->active = 1;
}
```

此处，将全部的 subsystem 与 dummy top 连\( 还没有绑定 \)，并且调用 ss-&gt;create\(\) 得到一个 css，同时init\_css\_set 通过 init\_css\_set\_link 连接 top\_cgroup\( 内嵌在 cg\_roofs \)，并且 css\_set 连接了全部 css。其中很关键的一步，就是将 init.cgroup 指向了init\_ css\_set ，这说明这个任务拥有所有的 “资源“，那么从它延伸出来的进程（ fork 出来的 ）都跟享受同样的资源。

![emmm](../.gitbook/assets/image%20%282%29.png)

#### mount 

这里就不解释系统的注册了，这里不是重点～

```c
static struct file_system_type cgroup_fs_type = {
	.name = "cgroup",
	.get_sb = cgroup_get_sb,
	.kill_sb = cgroup_kill_sb,
};
```

{% hint style="info" %}
后面版本的 cgroup 改为了  .mount 其实一样， VFS 可以保证 get\_sb 被调用
{% endhint %}

```c
static int cgroup_get_sb(struct file_system_type *fs_type,
			 int flags, const char *unused_dev_name,
			 void *data, struct vfsmount *mnt)
{
	struct cgroup_sb_opts opts;
	int ret = 0;
	struct super_block *sb;
	struct cgroupfs_root *root;
	struct list_head tmp_cg_links, *l;
	INIT_LIST_HEAD(&tmp_cg_links);

	/* First find the desired set of subsystems */
	ret = parse_cgroupfs_options(data, &opts);
	if (ret) {
		if (opts.release_agent)
			kfree(opts.release_agent);
		return ret;
	}

	root = kzalloc(sizeof(*root), GFP_KERNEL);
	if (!root)
		return -ENOMEM;

	init_cgroup_root(root);
	root->subsys_bits = opts.subsys_bits;
	root->flags = opts.flags;
	if (opts.release_agent) {
		strcpy(root->release_agent_path, opts.release_agent);
		kfree(opts.release_agent);
	}

	sb = sget(fs_type, cgroup_test_super, cgroup_set_super, root);

	if (sb->s_fs_info != root) {
		/* Reusing an existing superblock */
		BUG_ON(sb->s_root == NULL);
		kfree(root);
		root = NULL;
	} else {
		/* New superblock */
		struct cgroup *cgrp = &root->top_cgroup;
		struct inode *inode;

		ret = cgroup_get_rootdir(sb);

		inode = sb->s_root->d_inode;

		mutex_lock(&inode->i_mutex);
		mutex_lock(&cgroup_mutex);

		/*
		 * We're accessing css_set_count without locking
		 * css_set_lock here, but that's OK - it can only be
		 * increased by someone holding cgroup_lock, and
		 * that's us. The worst that can happen is that we
		 * have some link structures left over
		 */
		ret = allocate_cg_links(css_set_count, &tmp_cg_links);

		ret = rebind_subsystems(root, root->subsys_bits);

		list_add(&root->root_list, &roots);
		root_count++;

		sb->s_root->d_fsdata = &root->top_cgroup;
		root->top_cgroup.dentry = sb->s_root;

		/* Link the top cgroup in this hierarchy into all
		 * the css_set objects */
		write_lock(&css_set_lock);
		l = &init_css_set.list;
		do {
			struct css_set *cg;
			struct cg_cgroup_link *link;
			cg = list_entry(l, struct css_set, list);
			BUG_ON(list_empty(&tmp_cg_links));
			link = list_entry(tmp_cg_links.next,
					  struct cg_cgroup_link,
					  cgrp_link_list);
			list_del(&link->cgrp_link_list);
			link->cg = cg;
			list_add(&link->cgrp_link_list,
				 &root->top_cgroup.css_sets);
			list_add(&link->cg_link_list, &cg->cg_links);
			l = l->next;
		} while (l != &init_css_set.list);
		write_unlock(&css_set_lock);

		free_cg_links(&tmp_cg_links);


		cgroup_populate_dir(cgrp);
		mutex_unlock(&inode->i_mutex);
		mutex_unlock(&cgroup_mutex);
	}

	return simple_set_mnt(mnt, sb);

}
```

比较长的一个函数，大致的过程有，创建 cgroup\_rootfs\( Hierarchy \)，一个超级块\( 常规的三板斧 sb inode dentry\)，rebind\_subsystems 会将它需要的 subsystem 绑定（此时并不会创建 css ），并接替 dummy top 的 css，注意上述的代码还将 **top\_cgroup 与全部的 css\_set 连接在一起**，其实目前只有一个。

{% hint style="info" %}
parse\_cgroup\_options 是解析参数的，mount -t cgroup -o opts，用来确定需要哪些 subsystem
{% endhint %}

cgroup\_populate\_dir 目的是为了生成一些配置文件，其实就是分配denty还有 inode，并设置它的 f\_ops，下文详细解释一下就明白了。

## CPU 资源管理器的实现

```bash

$ mount -t cgroup -o cpu cpu_hier /mnt/point/cpu_hier
```

这一步执行的过程，就是上述 mount 的过程，建立一个 hierarchy ，绑定了 cpu 这个 subsystem，然后理所当然，继承了dummy top 的 css

![](../.gitbook/assets/image%20%2828%29.png)

```bash

$  cd /mnt/point/cpu_hier && mkdir mycg
```

在 mount 的时候，对于目录的 inode，进行了如下赋值

```c
static int cgroup_get_rootdir(struct super_block *sb)
{
	struct inode *inode =
		cgroup_new_inode(S_IFDIR | S_IRUGO | S_IXUGO | S_IWUSR, sb);
	struct dentry *dentry;

	if (!inode)
		return -ENOMEM;

	inode->i_op = &simple_dir_inode_operations;
	inode->i_fop = &simple_dir_operations;
	inode->i_op = &cgroup_dir_inode_operations;
	/* directories start off with i_nlink == 2 (for "." entry) */
	inc_nlink(inode);
	dentry = d_alloc_root(inode);
	if (!dentry) {
		iput(inode);
		return -ENOMEM;
	}
	sb->s_root = dentry;
	return 0;
}

static struct inode_operations cgroup_dir_inode_operations = {
	.lookup = simple_lookup,
	.mkdir = cgroup_mkdir,
	.rmdir = cgroup_rmdir,
	.rename = cgroup_rename,
};

static int cgroup_mkdir(struct inode *dir, struct dentry *dentry, int mode)
{
	struct cgroup *c_parent = dentry->d_parent->d_fsdata;

	/* the vfs holds inode->i_mutex already */
	return cgroup_create(c_parent, dentry, mode | S_IFDIR);
}

/*
 * A couple of forward declarations required, due to cyclic reference loop:
 * cgroup_mkdir -> cgroup_create -> cgroup_populate_dir ->
 * cgroup_add_file -> cgroup_create_file -> cgroup_dir_inode_operations
 * -> cgroup_mkdir.
 */
 
static long cgroup_create(struct cgroup *parent, struct dentry *dentry,
			     int mode)
{
	struct cgroup *cgrp;
	struct cgroupfs_root *root = parent->root;
	int err = 0;
	struct cgroup_subsys *ss;
	struct super_block *sb = root->sb;

	cgrp = kzalloc(sizeof(*cgrp), GFP_KERNEL);
	if (!cgrp)
		return -ENOMEM;

	/* Grab a reference on the superblock so the hierarchy doesn't
	 * get deleted on unmount if there are child cgroups.  This
	 * can be done outside cgroup_mutex, since the sb can't
	 * disappear while someone has an open control file on the
	 * fs */
	atomic_inc(&sb->s_active);

	mutex_lock(&cgroup_mutex);

	cgrp->flags = 0;

	cgrp->parent = parent;
	cgrp->root = parent->root;
	cgrp->top_cgroup = parent->top_cgroup;

	for_each_subsys(root, ss) {
		struct cgroup_subsys_state *css = ss->create(ss, cgrp);
		if (IS_ERR(css)) {
			err = PTR_ERR(css);
			goto err_destroy;
		}
		init_cgroup_css(css, ss, cgrp); 
	}

	list_add(&cgrp->sibling, &cgrp->parent->children);
	root->number_of_cgroups++;

	err = cgroup_create_dir(cgrp, dentry, mode);
	if (err < 0)
		goto err_remove;

	/* The cgroup directory was pre-locked for us */
	BUG_ON(!mutex_is_locked(&cgrp->dentry->d_inode->i_mutex));

	err = cgroup_populate_dir(cgrp);
	/* If err < 0, we have a half-filled directory - oh well ;) */

	mutex_unlock(&cgroup_mutex);
	mutex_unlock(&cgrp->dentry->d_inode->i_mutex);

	return 0;
}
```

层层包装，最后新建了一个 cgroup 并且调用了 ss-&gt;create 得到了一个 css，最后调用了 populate\_dir，我们现在来看看，到底干了什么事情。

首先是，ss-&gt;create，以 cpu 这个 subsystem 为例子，这个函数是

{% code-tabs %}
{% code-tabs-item title="sched.c" %}
```c
struct cgroup_subsys cpu_cgroup_subsys = {
	.name		= "cpu",
	.create		= cpu_cgroup_create,
	.destroy	= cpu_cgroup_destroy,
	.can_attach	= cpu_cgroup_can_attach,
	.attach		= cpu_cgroup_attach,
	.populate	= cpu_cgroup_populate,
	.subsys_id	= cpu_cgroup_subsys_id,
	.early_init	= 1,
};
```
{% endcode-tabs-item %}
{% endcode-tabs %}

看到这，眼尖的读者肯定已经发现，现在这些函数，已经是在 sched.c 下了，这意味着这已经是进程管理方面的处理了。

```c
static struct cgroup_subsys_state *
cpu_cgroup_create(struct cgroup_subsys *ss, struct cgroup *cgrp)
{
	struct task_group *tg;

	if (!cgrp->parent) {
		/* This is early initialization for the top cgroup */
		init_task_group.css.cgroup = cgrp;
		return &init_task_group.css;
	}

	/* we support only 1-level deep hierarchical scheduler atm */
	if (cgrp->parent->parent)
		return ERR_PTR(-EINVAL);

	tg = sched_create_group();
	if (IS_ERR(tg))
		return ERR_PTR(-ENOMEM);

	/* Bind the cgroup to task_group object we just created */
	tg->css.cgroup = cgrp;

	return &tg->css;
}
```

现在在来看看，task\_group 这个结构，上面调用的函数无非是分配了一个 task\_group

```c
/* task group related information */
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

注意它内嵌了一个css，典型的继承的思想

```c
/* allocate runqueue etc for a new task group */
struct task_group *sched_create_group(void)
{
	struct task_group *tg;
	struct cfs_rq *cfs_rq;
	struct sched_entity *se;
	struct rq *rq;
	int i;

	tg = kzalloc(sizeof(*tg), GFP_KERNEL);

	tg->cfs_rq = kzalloc(sizeof(cfs_rq) * NR_CPUS, GFP_KERNEL);

	tg->se = kzalloc(sizeof(se) * NR_CPUS, GFP_KERNEL);


	for_each_possible_cpu(i) {
		rq = cpu_rq(i);

		cfs_rq = kmalloc_node(sizeof(struct cfs_rq), GFP_KERNEL,
							 cpu_to_node(i));

		se = kmalloc_node(sizeof(struct sched_entity), GFP_KERNEL,

		tg->cfs_rq[i] = cfs_rq;
		init_cfs_rq(cfs_rq, rq);
		cfs_rq->tg = tg;

		tg->se[i] = se;
		se->cfs_rq = &rq->cfs;
		se->my_q = cfs_rq;
		se->load.weight = NICE_0_LOAD;
		se->load.inv_weight = div64_64(1ULL<<32, NICE_0_LOAD);
		se->parent = NULL;
	}

	for_each_possible_cpu(i) {
		rq = cpu_rq(i);
		cfs_rq = tg->cfs_rq[i];
		list_add_rcu(&cfs_rq->leaf_cfs_rq_list, &rq->leaf_cfs_rq_list);
	}

	tg->shares = NICE_0_LOAD;
	spin_lock_init(&tg->lock);

	return tg;
}
```

实际的将 task\_group 作为一个 se，挂载在最高层的 cfs\_rq，则是发生在添加到实际的任务pid 进去

```bash
$ echo $$ >> /mnt/point/cpu_hier/tasks
```

至于这个文件怎么出现，其实就是 mount 的时候，自动生成的

```c

/*
 * for the common functions, 'private' gives the type of file
 */
static struct cftype files[] = {
	{
		.name = "tasks",
		.open = cgroup_tasks_open,
		.read = cgroup_tasks_read,
		.write = cgroup_common_file_write,
		.release = cgroup_tasks_release,
		.private = FILE_TASKLIST,
	},

	{
		.name = "notify_on_release",
		.read_uint = cgroup_read_notify_on_release,
		.write = cgroup_common_file_write,
		.private = FILE_NOTIFY_ON_RELEASE,
	},

	{
		.name = "releasable",
		.read_uint = cgroup_read_releasable,
		.private = FILE_RELEASABLE,
	}
};

static int cgroup_populate_dir(struct cgroup *cgrp)
{
	int err;
	struct cgroup_subsys *ss;

	/* First clear out any existing files */
	cgroup_clear_directory(cgrp->dentry);

	err = cgroup_add_files(cgrp, NULL, files, ARRAY_SIZE(files));
	if (err < 0)
		return err;

	if (cgrp == cgrp->top_cgroup) {
		if ((err = cgroup_add_file(cgrp, NULL, &cft_release_agent)) < 0)
			return err;
	}

	for_each_subsys(cgrp->root, ss) {
		if (ss->populate && (err = ss->populate(ss, cgrp)) < 0)
			return err;
	}

	return 0;
}
```

这里不讲解 cftype 了，其实就是 dispatcher ，注意， 还调用了 ss-&gt;populate，对于 cpu 这个 subystem，还会生成一个 cpu.shares 文件！**本质就是 se 的 weight**！

```c
static struct cftype cpu_files[] = {
	{
		.name = "shares",
		.read_uint = cpu_shares_read_uint,
		.write_uint = cpu_shares_write_uint,
	},
};

static int cpu_cgroup_populate(struct cgroup_subsys *ss, struct cgroup *cont)
{
	return cgroup_add_files(cont, ss, cpu_files, ARRAY_SIZE(cpu_files));
}

```

所以对 tasks 写会调用，common\_file\_write

```c
static ssize_t cgroup_common_file_write(struct cgroup *cgrp,
					   struct cftype *cft,
					   struct file *file,
					   const char __user *userbuf,
					   size_t nbytes, loff_t *unused_ppos)
{
	enum cgroup_filetype type = cft->private;
	char *buffer;
	int retval = 0;
	....

	switch (type) {
	case FILE_TASKLIST:
		retval = attach_task_by_pid(cgrp, buffer);
		break;
	....
out1:
	kfree(buffer);
	return retval;
}
```

这个函数如下，

```c
/*
 * Attach task with pid 'pid' to cgroup 'cgrp'. Call with
 * cgroup_mutex, may take task_lock of task
 */
static int attach_task_by_pid(struct cgroup *cgrp, char *pidbuf)
{
	pid_t pid;
	struct task_struct *tsk;
	int ret;

	if (sscanf(pidbuf, "%d", &pid) != 1)
		return -EIO;

	if (pid) {
		rcu_read_lock();
		tsk = find_task_by_pid(pid);
		if (!tsk || tsk->flags & PF_EXITING) {
			rcu_read_unlock();
			return -ESRCH;
		}
		get_task_struct(tsk);
		rcu_read_unlock();

		if ((current->euid) && (current->euid != tsk->uid)
		    && (current->euid != tsk->suid)) {
			put_task_struct(tsk);
			return -EACCES;
		}
	} else {
		tsk = current;
		get_task_struct(tsk);
	}

	ret = attach_task(cgrp, tsk);
	put_task_struct(tsk);
	return ret;
}
```

关键在于，attach\_task

```c
/*
 * Attach task 'tsk' to cgroup 'cgrp'
 *
 * Call holding cgroup_mutex.  May take task_lock of
 * the task 'pid' during call.
 */
static int attach_task(struct cgroup *cgrp, struct task_struct *tsk)
{
	int retval = 0;
	struct cgroup_subsys *ss;
	struct cgroup *oldcgrp;
	struct css_set *cg = tsk->cgroups;
	struct css_set *newcg;
	struct cgroupfs_root *root = cgrp->root;
	int subsys_id;

	...
	for_each_subsys(root, ss) {
		if (ss->can_attach) {
			retval = ss->can_attach(ss, cgrp, tsk);
			if (retval) {
				return retval;
			}
		}
	}
	/*
	 * Locate or allocate a new css_set for this task,
	 * based on its final set of cgroups
	 */
	newcg = find_css_set(cg, cgrp);
	if (!newcg) {
		return -ENOMEM;
	}

	task_lock(tsk);
	if (tsk->flags & PF_EXITING) {
		task_unlock(tsk);
		put_css_set(newcg);
		return -ESRCH;
	}
	rcu_assign_pointer(tsk->cgroups, newcg);
	task_unlock(tsk);

	/* Update the css_set linked lists if we're using them */
	write_lock(&css_set_lock);
	if (!list_empty(&tsk->cg_list)) {
		list_del(&tsk->cg_list);
		list_add(&tsk->cg_list, &newcg->tasks);
	}
	write_unlock(&css_set_lock);

	for_each_subsys(root, ss) {
		if (ss->attach) {
			ss->attach(ss, cgrp, oldcgrp, tsk);
		}
	}
	set_bit(CGRP_RELEASABLE, &oldcgrp->flags);
	synchronize_rcu();
	put_css_set(cg);
	return 0;
}

```

这个函数非常的关键，调用了can\_attach 和 attach 接口，前者做一些判断，后者真正的进行操作，以 cpu 为例，就是将刚才创建 task\_group 的 se 添加到上层队列，并把进程的 se 添加到这个 task\_group 的队列下。并且它更新了 css\_set，假设最早期所有的进程都在一个 init\_css\_set，也就是说，现在是用户主动attach 的第一个进程，那么 css\_set 会是:

![new css\_set &#x548C; init\_css\_set &#x8FD8;&#x4F1A;&#x4E32;&#x8FDE;](../.gitbook/assets/image%20%2890%29.png)

并且 top\_cgroup 也会串连全部 css\_set，这里 cglinks 只是简单释义，值得注意是，task\_group 内嵌了了一个 css，表明它就是 资源。

```c
static void
cpu_cgroup_attach(struct cgroup_subsys *ss, struct cgroup *cgrp,
			struct cgroup *old_cont, struct task_struct *tsk)
{
	sched_move_task(tsk);
}

void sched_move_task(struct task_struct *tsk)
{
	int on_rq, running;
	unsigned long flags;
	struct rq *rq;

	rq = task_rq_lock(tsk, &flags);

	if (tsk->sched_class != &fair_sched_class) {
		set_task_cfs_rq(tsk, task_cpu(tsk));
		goto done;
	}

	update_rq_clock(rq);

	running = task_current(rq, tsk);
	on_rq = tsk->se.on_rq;

	if (on_rq) {
		dequeue_task(rq, tsk, 0);
		if (unlikely(running))
			tsk->sched_class->put_prev_task(rq, tsk);
	}

	set_task_cfs_rq(tsk, task_cpu(tsk));

	if (on_rq) {
		if (unlikely(running))
			tsk->sched_class->set_curr_task(rq);
		enqueue_task(rq, tsk, 0);
	}

done:
	task_rq_unlock(rq, &flags);
}

/* Change a task's cfs_rq and parent entity if it moves across CPUs/groups */
static inline void set_task_cfs_rq(struct task_struct *p, unsigned int cpu)
{
	p->se.cfs_rq = task_group(p)->cfs_rq[cpu];
	p->se.parent = task_group(p)->se[cpu];
}

```

当改变了 se.cfs\_rq 指针，调用 enqueue\_task，就会添加到 task\_group 对应的 cfs\_rq 那里了。当然，此时的 task\_group 的 se，也不在队列上

```c
/*
 * The enqueue_task method is called before nr_running is
 * increased. Here we update the fair scheduling stats and
 * then put the task into the rbtree:
 */
static void enqueue_task_fair(struct rq *rq, struct task_struct *p, int wakeup)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &p->se;

	for_each_sched_entity(se) {
		if (se->on_rq)
			break;
		cfs_rq = cfs_rq_of(se);
		enqueue_entity(cfs_rq, se, wakeup);
		wakeup = 1;
	}
}

```

该函数会将se 一层层 加到自己的所在的 cfs\_rq，通过 se-&gt;cfs\_rq 来判断它的所属的队列

![](../.gitbook/assets/image%20%2884%29.png)

全局来看，大致就是这样。

![](../.gitbook/assets/image%20%283%29.png)

## 回到最初的例子

那么，文章开头的例子，是怎么样的呢？这里也给出一张图，首先，全部进程都只能在一个核上运行，这意味着它们都受到一个 cpuset 的资源控制\( css \)，我们假设 cpuset 绑定在了一个 Hierarchy，同时，b，c被限制在一个组内，所以它们受 cpu 资源控制，注意进程 a 不受 cpu 控制，这意味着它还是拥有最初的 也就是 dummy top 创建的 css（cpu）的资源。

![](../.gitbook/assets/image%20%28105%29.png)

所以，不难得出下面这个图。

![&#x6CA1;&#x6709;&#x63D0;&#x53CA;init\_css\_set &#x4EE5;&#x53CA;cg\_links &#x8FD8;&#x6709;subsystem](../.gitbook/assets/image%20%2832%29.png)

提几点， css\_set\_A 表明资源有 一个 CPU核，一个进程组（task\_group）还有其他类型的全部资源\( 比如内存，网卡，硬盘\)， css\_set\_B表明，一个CPU核，还有其他类型全部的资源\( 最初的进程组，即在最高层的队列，内存等等\)。

要强调的就是 **css\_set 表明的就是一个资源集合**，然后解释为什么 cgroup css\_set 都有css 指针数组。

```c
struct cgroup {
	...
	/* Private pointers for each registered subsystem */
	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
};

struct css_set {

	...

	/*
	 * Set of subsystem states, one for each subsystem. This array
	 * is immutable after creation apart from the init_css_set
	 * during subsystem registration (at boot time).
	 */
	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];

};
```

Cgroup 也是一个资源集，但是它是固定的！**css\_set 是 cgroup 集**，源代码注释把它称为，cgroup group，我觉得也不太好理解，总之，进程可以在多个 cgroup 中，比如 b,c 就同时在三个 cgroup 中，所以生成了一个css\_set\_A ，a 也是一样，但是不可能同时在有父子关系的俩个cgroup，原因就是在同一个 Hierarchy中，资源都是互斥的，也就说 数组的索引，会冲突，所以一定不能在，但是可以在父子之间移动。

现在来看看，是如何修改，task\_group 的 se 的weight。

```bash
$ echo "512" >> /grouB/cpu.shares
```

```c
static struct cftype cpu_files[] = {
	{
		.name = "shares",
		.read_uint = cpu_shares_read_uint,
		.write_uint = cpu_shares_write_uint,
	},
};

static int cpu_shares_write_uint(struct cgroup *cgrp, struct cftype *cftype,
				u64 shareval)
{
	return sched_group_set_shares(cgroup_tg(cgrp), shareval);
}

int sched_group_set_shares(struct task_group *tg, unsigned long shares)
{
	int i;

	/*
	 * A weight of 0 or 1 can cause arithmetics problems.
	 * (The default weight is 1024 - so there's no practical
	 *  limitation from this.)
	 */
	if (shares < 2)
		shares = 2;

	spin_lock(&tg->lock);
	if (tg->shares == shares)
		goto done;

	tg->shares = shares;
	for_each_possible_cpu(i)
		set_se_shares(tg->se[i], shares);

done:
	spin_unlock(&tg->lock);
	return 0;
}
```

这便是，文件系统的威力，值得一提，刚才我们调用 ss-&gt;attach 的时候，其实只添加了 task\_group 进入当前的 cpu 的核的队列，而不是每个 cpu 核的队列，那么到底其他队列是如何添加的。

答案是进程会在CPU之间，迁移

```c
/*
 * Move (not current) task off this cpu, onto dest cpu. We're doing
 * this because either it can't run here any more (set_cpus_allowed()
 * away from this CPU, or CPU going down), or because we're
 * attempting to rebalance this task on exec (sched_exec).
 *
 * So we race with normal scheduler movements, but that's OK, as long
 * as the task is no longer on this CPU.
 *
 * Returns non-zero if task was successfully migrated.
 */
static int __migrate_task(struct task_struct *p, int src_cpu, int dest_cpu)
{
	struct rq *rq_dest, *rq_src;
	int ret = 0, on_rq;

	if (unlikely(cpu_is_offline(dest_cpu)))
		return ret;

	rq_src = cpu_rq(src_cpu);
	rq_dest = cpu_rq(dest_cpu);

	double_rq_lock(rq_src, rq_dest);
	/* Already moved. */
	if (task_cpu(p) != src_cpu)
		goto out;
	/* Affinity changed (again). */
	if (!cpu_isset(dest_cpu, p->cpus_allowed))
		goto out;

	on_rq = p->se.on_rq;
	if (on_rq)
		deactivate_task(rq_src, p, 0);

	set_task_cpu(p, dest_cpu);
	if (on_rq) {
		activate_task(rq_dest, p, 0);
		check_preempt_curr(rq_dest, p);
	}
	ret = 1;
out:
	double_rq_unlock(rq_src, rq_dest);
	return ret;
}

static inline void __set_task_cpu(struct task_struct *p, unsigned int cpu)
{
	set_task_cfs_rq(p, cpu);
#ifdef CONFIG_SMP
	/*
	 * After ->cpu is set up to a new value, task_rq_lock(p, ...) can be
	 * successfuly executed on another CPU. We must ensure that updates of
	 * per-task data have been completed by this moment.
	 */
	smp_wmb();
	task_thread_info(p)->cpu = cpu;
#endif
}

/* Change a task's cfs_rq and parent entity if it moves across CPUs/groups */
static inline void set_task_cfs_rq(struct task_struct *p, unsigned int cpu)
{
	p->se.cfs_rq = task_group(p)->cfs_rq[cpu];
	p->se.parent = task_group(p)->se[cpu];
}

```

注释其实写的很明白，就是在 groups 或者 CPUs 之间迁移，中间调用可能有些复杂，但是本质就是这几个函数。

## 末

非常长的一篇笔记，因为实现起来确实有些复杂，其实还有 cpuset 这个资源管理器没有提到，但是本质不难猜测，就是控制进程的在 CPU 之间的迁移，中间加一些判断，就可以知道是否有目标CPU的资源，就实现了。

关键在于理解，Hierarchy Subsystem css css\_set 这几个结构，接口由 VFS 实现～ 恩，暂时就到这，最近得忙编译器以及动态加载那方面的知识～ 容器就到这儿告一段落拉。

