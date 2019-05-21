# C 成长笔记

## 终端IO设置

```c
/*
在linux中，读写控制可以由 struct termios
这个结构体来控制，在平常我们都是敲下回车，然后
数据才真正写到了缓冲区，唤醒了进程，但是我们可以
通过改变相关属性，来达到不用回车就可以获取数据
*/

#include <unistd.h>
#include <termios.h>
#include <stdio.h>

static struct termios old, new;

/* Initialize new terminal i/o settings */
void initTermios(int echo)
{
  tcgetattr(0, &old); /* grab old terminal i/o settings */
  new = old; /* make new settings same as old settings */
  new.c_lflag &= ~ICANON; /* disable buffered i/o */
  new.c_lflag &= echo ? ECHO : ~ECHO; /* set echo mode */
  tcsetattr(0, TCSANOW, &new); /* use these new terminal i/o settings now */
}

/* Restore old terminal i/o settings */
void resetTermios(void)
{
  tcsetattr(0, TCSANOW, &old);
}

/* Read 1 character - echo defines echo mode */
char getch_(int echo)
{
  char ch;
  initTermios(echo);
  ch = getchar();
  resetTermios();
  return ch;
}

/* Read 1 character without echo */
char getch(void)
{
  return getch_(0);
}

/* Read 1 character with echo */
char getche(void)
{
  return getch_(1);
}

```

## 面向对象的一种方案

在内核代码中，很多结构体都会嵌入一个抽象的结构，来达到面向对象的设计，于是就有了 upcast 的需求，这里标出的是upcast的代码，很有意思

```c

#define container_of(ptr, type, member) ({			\
        const typeof( ((type *)0)->member ) *__mptr = (ptr);	\
        (type *)( (char *)__mptr - offsetof(type,member) );})

#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

// 括号内多条语句，最终值取最后一条语句的值

 #define to_test(ptr, member) \
	container_of(ptr, struct test, member)

struct test {
	int a,b,c;
	int i;
};

int main() {
	struct test t;
	int * p = &t.i;

	struct test * pt;
	pt = to_test(p, i);
	
// 这里只是一个简单的例子，注意的就是，ptr 和 member * 的类型必须一致
}


```

## Include trick

Good example~~~

```c
/* Generate an array of cgroup subsystem pointers */
#define SUBSYS(_x) &_x ## _subsys,
static struct cgroup_subsys *subsys[] = {
#include <linux/cgroup_subsys.h>
};

#define SUBSYS(_x) extern struct cgroup_subsys _x ## _subsys;
#include <linux/cgroup_subsys.h>
#undef SUBSYS

/* Define the enumeration of all cgroup subsystems */
#define SUBSYS(_x) _x ## _subsys_id,
enum cgroup_subsys_id {
#include <linux/cgroup_subsys.h>
	CGROUP_SUBSYS_COUNT
};
#undef SUBSYS


```

{% code-tabs %}
{% code-tabs-item title="linux/cgroup\_subsys.h" %}
```c
#ifdef CONFIG_CPUSETS
SUBSYS(cpuset)
#endif

/* */

#ifdef CONFIG_CGROUP_DEBUG
SUBSYS(debug)
#endif

/* */

#ifdef CONFIG_CGROUP_NS
SUBSYS(ns)
#endif

/* */

#ifdef CONFIG_FAIR_CGROUP_SCHED
SUBSYS(cpu_cgroup)
#endif

/* */

#ifdef CONFIG_CGROUP_CPUACCT
SUBSYS(cpuacct)
#endif
```
{% endcode-tabs-item %}
{% endcode-tabs %}

