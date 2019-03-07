---
description: 参考书籍 《Android 系统源代码分析 》-  李俊
---

# 安卓系统入门

## 序

大概很早之前，我就有学习安卓的想法了，那时候知道它的内核是Linux 内核，也对 Linux 更加敬佩了，所以很好奇它的上层是怎么搭建起来的，所以这里的教程不是教你怎么编写安卓程序，首先我不会，其次肯定也不难，但是我会了解为什么可以这样编程。

## 安卓的架构与JNI

![architecture](.gitbook/assets/tu-pian%20%282%29.png)

这是很抽象的说法，我以Linux 的角度来说，首先内核就是 Linux 内核，运行的第一个用户程序之后就是安卓的部分了，但是安卓对内核也做了适应性的优化，然后关键的一点，后台有一个虚拟机JVM进程，运行上层的Java 程序，并且和下层的Native Libarary交互。

因为内核的原生支持一定是 C/C++ ，但是上层的程序又想用 Java 编写，所以安卓系统的大量工作就在于一点，搭建一个中间库，我们称它为 JNI

![](.gitbook/assets/tu-pian%20%285%29.png)

比如最简单的一个 System.out.printl 函数，最终肯定是调用了 Linux 系统调用的 write 函数，但是 Java 如何和 C 交互呢，它们就不是一个时代的产物，在参考书里这么说的：

> _“绝大数的脚本引擎都支持一个显著的特性，一定要提供一种脚本中调用 C/C++ 编写的模块的机制，才能称得上是一个相对完善的脚本引擎”_

所有的程序都离不开操作系统的支持，而这些支持都是以 C的规范暴露出来的接口，所以必须得支持！然后 Java，利用了一个中间库的机制，建立一个 Java 可以识别的（规范的）中间库，然后中间库（C/C++）编写，又可以直接与C/C++的底层库交互。

中间库的存在就是作为一个**承上启下**的作用，因为**Java有自己的规范，而C也有自己的规范**，那怎么办，好，找一个代理人，这个代理人就是 **JNI** 

我这里以安卓的设计的JNI为例子，安卓改进之后与传统的有点区别，但是本质估计都没有却别，只要理解原理就是，首先读者先理解一点，_**函数（方法）只是内存中一个代码段，我们只要知道了它的地址，往栈里放进相关的变量，那么这个函数不管是用什么语言编写的都是无所谓的，只要你遵循了这个函数调用所需的参数。**_

所以对于Java程序，关键是什么呢，**得到自己类里Native关键字声明的方法的目标函数（库）地址**，

{% code-tabs %}
{% code-tabs-item title="Foo.java" %}
```java

package test.test;

public class Foo {
    static {
        System.loadLibaray("test_jni");
    }
    public static native void bar();
    public native void emm();
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% code-tabs %}
{% code-tabs-item title="test\_jni.c" %}
```c
中间JNI规范的代码

static void test_test_bar(JNIEnv * env ) {
    // call func in Library
}

static void test_test_emm(JNIEnv * env, jobject thiz ) {
     // call xx(xx)
}

```
{% endcode-tabs-item %}
{% endcode-tabs %}

如果了解 C++ 实现机制的人肯定不觉得奇怪，就算是Java中声明的函数，到了C这里也一定是有参数，因为_**所谓的面向对象特性（多态性）都是通过指针实现**_的，对于静态函数，调用的时候没有具体的对象指针（C++的this），其实说明一点，这就是全局函数，而 env 参数，我的理解就是一个线程的标识符，每个线程有不同的值，因为**不同的线程会加载相同的动态库**，必须得有自己的备份，不能共享，比如那些全局数据，必须有自己的一份，如果有同学了解 mmap 机制，在实现映射的时候，有一个标识符是不能更改，一更改相关数据，马上就执行复制一份新的数据，从而不在共享（类似COW机制）。

注意函数名，必须跟Java的包名相关，这应该是为了防止同名函数的一个措施，那么问题来啦，Java 如何调用 JNI 的函数呢。

动态库的设计很简单，就是一个**暴露函数（以及变量）的地址**的一段小程序，所以 Java 只需要把它加载到内存当中，分析它的符号表，就可以找到它的函数所在地址，当然 JNI 实现了一个更简单的操作。

```c
typedef struct {
    const char * name;    // Java 中函数的名字
    const char * signature;  // 函数的参数和返回值
    void * fnPtr;     // 函数指针
} JNINativeMethod;
```

在编写完 JNI 的中间函数之后，给每个函数填写一个结构体，用来实现映射

```c

static JNINativeMethod gMethods [] = {
    {"bar", "()V", (void *)test_test_bar },
    {"foo", "()V", (void *)test_test_foo },
}


int register_test_test_Foo(JNIEnv * env) {
    
    return AndroidRuntime::registerNativeMethods(env, "test/test/Foo",
                            gMethods, NELEM(gMethods));    
}

jint JNI_OnLoad( JavaVM * jvm, void * reserve ) {
    ...
     JNIEnv * env = NULL;
     vm->GetEnv( (void **)&env, JNI_VERSION_x_x);
     
     register_test_test_Foo(env);      
    
    ...
}

```

上述的函数省略了一些，在书本p70可以找到详细的，这里关键的一步就是调用了安卓提供了一个接口，将这个表和类结合了起来，其实很容易想到，无非就是 Java 类里面的函数（方法）修改了地址，其值为 gMethods 提供的表，当然 Java 不允许任意修改其函数地址，但是提供了这样一个接口，就好像**反射**的机制一样，在 C++ 很容易就能修改函数的地址，因为我们拿到指针以后就可以为所欲为了，私有的也不再是私有的。

JNI\_OnLoad 函数是 Java 加载运行库时自动调用的，这样的好处就是以后加载运行库之后，一次性把其他 Native 的函数映射全部解决了，不再需要下次再来找。这种思想很常见，还有一种思想就是，不到用的时候都不加载，参考 ld 程序链接动态库的时候，或者内核 COW 还有 page fault 的处理。

总之一句话，**JNI 是为了承接 Java 的规范同时又可以调用 C/C++ 的库（模块），然后呢，其实现的原理，就是地址（指针）的查找与赋值，表的存在加速了这一过程**。

{% hint style="info" %}
其实了解函数的本质之后，那些动态库的原理，驱动加载的原理，虽然很复杂，但是本质其实就是一个指针的查找以及挂钩的过程
{% endhint %}

##  

