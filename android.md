---
description: 参考书籍 《Android 系统源代码分析 》-  李俊
---

# 安卓系统入门

## 序

大概很早之前，我就有学习安卓的想法了，那时候知道它的内核是Linux 内核，也对 Linux 更加敬佩了，所以很好奇它的上层是怎么搭建起来的，所以这里的教程不是教你怎么编写安卓程序，首先我不会，其次肯定也不难，但是我会了解为什么可以这样编程。

## 安卓的架构与JNI

![architecture](.gitbook/assets/tu-pian%20%284%29.png)

这是很抽象的说法，我以Linux 的角度来说，首先内核就是 Linux 内核，运行的第一个用户程序之后就是安卓的部分了，但是安卓对内核也做了适应性的优化，然后关键的一点，后台有一个虚拟机JVM进程，运行上层的Java 程序，并且和下层的Native Libarary交互。

因为内核的原生支持一定是 C/C++ ，但是上层的程序又想用 Java 编写，所以安卓系统的大量工作就在于一点，搭建一个中间库，我们称它为 JNI

![](.gitbook/assets/tu-pian%20%287%29.png)

比如最简单的一个 System.out.printl 函数，最终肯定是调用了 Linux 系统调用的 write 函数，但是 Java 如何和 C 交互呢，它们就不是一个时代的产物，在参考书里这么说的：

> _“绝大数的脚本引擎都支持一个显著的特性，一定要提供一种脚本中调用 C/C++ 编写的模块的机制，才能称得上是一个相对完善的脚本引擎”_

所有的程序都离不开操作系统的支持，操作系统的接口都是以 C的规范暴露出来，所以其他语言必须得支持调用C/C++模块！Java当然也不例外，它利用了一个中间库的机制，建立一个 Java 可以识别的（规范的）中间库，然后中间库（C/C++）编写，又可以直接与C/C++的底层库交互。

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

##  设备的访问

在linux，编写一个驱动是非常容易的，复杂的地方都在于怎么实现如何于设备正常交互，Linux提供的接口非常容易使用，访问的时候也是抽象为 字符，块以及网络设备，最终都以文件的形式展现出来，安卓在这方面也是类似，但做了一些修改。

### 安卓系统所使用的内核与传统内核的区别

> “ 传统的Linux系统把对硬件的支持完全实现在内核空间中，即把对硬件的支持完全实现在硬件驱动模块中” 
>
> “如果安卓系统像Linux系统一样，把对硬件的支持完全实现在硬件驱动模块中，那么必须把这些硬件驱动模块源代码公开，这样可能会损害移动设备厂商的利益，因为这相当于暴露了这些硬件的实现细节和参数”

上述引用中，说明了安卓与Linux的内核其实在驱动实现上是有区别的，但是上述的话可能是作者自己的理解，但是Linux同样有很多手段不暴露驱动的源代码，比如在线更新的时候，是直接加载一个模块的，只要不是要求集成在内核中，或者考虑实现在 initramfs 中，都是一种方案，再者，暴露了驱动的参数其实没有问题，很多设备的设备参数都是开源的，欢迎别人修改和移植，这样才能使自己设备更多人用啊。

安卓在驱动的实现上把需要放置在内核的部分可以屏蔽设备的参数，然后把实现交由用户空间中，然后用户空间以硬件抽象模块（Hardware Abstract Layer）的形式来支持，它封装了实现的细节和参数，这样就可以保护移动设备厂商的利益。（总之，想办法不开源就是）。

这只是作者的理解，其实很多驱动也是要有程序在应用程配套使用的，比如显卡驱动，就不是简单的实现在内核 。但是，在我这几天的阅读看来，Linux 驱动和安卓驱动没有太大的区别，只是安卓下的驱动，除了要包括内核的，还得在 HAL 层**封装**一个 相当于 Linux **用户层的支持程序**，然后在构建 JNI 以及对应的 Java 的基类，让相关的服务可以被用户使用。

### 安卓程序如何与设备交互

前面提到了，尽管作者说了安卓是改动了，但是我至今还没发现本质的区别，所以我们只需要考虑，内核空间和用户空间是怎么交互就知道答案了。

* sysfs 
* proc
* device\_node

其实上面这些的本质，都是 VFS 的实现，再本质一点，就是系统调用，只有这个东西，可以改变特权级，mmap 其实也是系统调用，但是它有一劳永逸的效果，所以适合大数据传输。

知道了上面几点，我们就会发现，安卓驱动程序（linux 亦是如此）本质上就是利用了文件操作函数和设备交互的，这里所说的文件操作函数，包括 _**read,write,open,close,poll,ioctl**_ .... **无论底层怎么包装，最后一定是落实到了这里**，只是它们有可能是通过直接读写设备文件，或者读写 sys 文件。

## HAL 的封装

### HAL 层为什么存在

Hardware Abstract Layer 名字的存在就暗示了它的作用就是要抽象，这是直观的映像，驱动开发人员会把 跟内核设备文件交互的底层函数封装在一个结构体中（以函数指针的形式），最后这个函数的代码以及数据都变成一个**动态库**，上层访问到相关类的时候，就动态的加载，并完成相关的初始化。

上层只需要调用相关的类就好了，中间 JNI 会完成和这个动态库的交互，所以这一层就是封装了驱动的实现，提供相关的接口，供上层 Java 相关的服务调用。

### HAL 的总体设计 - 以 mokoid 为例子

首先了解三个结构体。

{% code-tabs %}
{% code-tabs-item title="include / hardware / hardware.h" %}
```c

/**
 * Every hardware module must have a data structure named HAL_MODULE_INFO_SYM
 * and the fields of this data structure must begin with hw_module_t
 * followed by module specific information.
 */
typedef struct hw_module_t {
    /** tag must be initialized to HARDWARE_MODULE_TAG */
    uint32_t tag;
    /**
     * The API version of the implemented module. The module owner is
     * responsible for updating the version when a module interface has
     * changed.
     *
     * The derived modules such as gralloc and audio own and manage this field.
     * The module user must interpret the version field to decide whether or
     * not to inter-operate with the supplied module implementation.
     * For example, SurfaceFlinger is responsible for making sure that
     * it knows how to manage different versions of the gralloc-module API,
     * and AudioFlinger must know how to do the same for audio-module API.
     *
     * The module API version should include a major and a minor component.
     * For example, version 1.0 could be represented as 0x0100. This format
     * implies that versions 0x0100-0x01ff are all API-compatible.
     *
     * In the future, libhardware will expose a hw_get_module_version()
     * (or equivalent) function that will take minimum/maximum supported
     * versions as arguments and would be able to reject modules with
     * versions outside of the supplied range.
     */
    uint16_t module_api_version;
#define version_major module_api_version
    /**
     * version_major/version_minor defines are supplied here for temporary
     * source code compatibility. They will be removed in the next version.
     * ALL clients must convert to the new version format.
     */
    /**
     * The API version of the HAL module interface. This is meant to
     * version the hw_module_t, hw_module_methods_t, and hw_device_t
     * structures and definitions.
     *
     * The HAL interface owns this field. Module users/implementations
     * must NOT rely on this value for version information.
     *
     * Presently, 0 is the only valid value.
     */
    uint16_t hal_api_version;
#define version_minor hal_api_version
    /** Identifier of module */
    const char *id;
    /** Name of this module */
    const char *name;
    /** Author/owner/implementor of the module */
    const char *author;
    /** Modules methods */
    struct hw_module_methods_t* methods;
    /** module's dso */
    void* dso;
#ifdef __LP64__
    uint64_t reserved[32-7];
#else
    /** padding to 128 bytes, reserved for future use */
    uint32_t reserved[32-7];
#endif
} hw_module_t;


typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module, const char* id,
            struct hw_device_t** device);
} hw_module_methods_t;

/**
 * Every device data structure must begin with hw_device_t
 * followed by module specific public methods and attributes.
 */
typedef struct hw_device_t {
    /** tag must be initialized to HARDWARE_DEVICE_TAG */
    uint32_t tag;
    /**
     * Version of the module-specific device API. This value is used by
     * the derived-module user to manage different device implementations.
     *
     * The module user is responsible for checking the module_api_version
     * and device version fields to ensure that the user is capable of
     * communicating with the specific module implementation.
     *
     * One module can support multiple devices with different versions. This
     * can be useful when a device interface changes in an incompatible way
     * but it is still necessary to support older implementations at the same
     * time. One such example is the Camera 2.0 API.
     *
     * This field is interpreted by the module user and is ignored by the
     * HAL interface itself.
     */
    uint32_t version;
    /** reference to the module this device belongs to */
    struct hw_module_t* module;
    /** padding reserved for future use */
#ifdef __LP64__
    uint64_t reserved[12];
#else
    uint32_t reserved[12];
#endif
    /** Close this device */
    int (*close)(struct hw_device_t* device);
} hw_device_t;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

注意最上面的注释，所有的 HAL 模块，可以有自己的结构体，但是结构体的第一个域，必须是 struct hw\_module\_t 而且实例化的那个结构体的名字也必须是 HAL\_MODULE\_INFO\_SYM 这里的作用很明显，首先必须得是这个结构体开头，其实就是一种 继承的关系，而名字必须是规定的，原因在于，动态加载的时候，找到了这个变量名字所在的地方，就知道了模块的全部信息，这俩者是相辅相成的。

 struct hw\_module\_t 大部分都是存储版本信息，看注释就可以理解了，现在来看一个实际的例子。struct hw\_device\_t 设计同样如此，也是一种继承关系。

```c
struct led_module_t {
   struct hw_module_t common;
};

const struct led_module_t HAL_MODULE_INFO_SYM = {
    common: {
        tag: HARDWARE_MODULE_TAG,
        version_major: 1,
        version_minor: 0,
        id: LED_HARDWARE_MODULE_ID,
        name: "Sample LED Stub",
        author: "The Mokoid Open Source Project",
        methods: &led_module_methods,
    }
    /* supporting APIs go here */
};

static struct hw_module_methods_t led_module_methods = {
    open: led_device_open
};
```

上面这个例子就很明显了说明了要求，刚才还有一个 hw\_module\_method 忘记解释了，里面只有一个 open 的函数，很容易想到，是这个模块被加载的时候调用的。open 函数在下面放出，不然太乱了。

```c
static int led_device_open(const struct hw_module_t* module, const char* name,
        struct hw_device_t** device) 
{
	struct led_control_device_t *dev;

	dev = (struct led_control_device_t *)malloc(sizeof(*dev));
	memset(dev, 0, sizeof(*dev));

	dev->common.tag =  HARDWARE_DEVICE_TAG;
	dev->common.version = 0;
	dev->common.module = module;
	dev->common.close = led_device_close;

	dev->set_on = led_on;
	dev->set_off = led_off;

	*device = &dev->common;

success:
	return 0;
}

struct led_control_device_t {
   struct hw_device_t common;
   /* attributes */
   int fd;
   /* supporting control APIs go here */
   int (*set_on)(struct led_control_device_t *dev, int32_t led);
   int (*set_off)(struct led_control_device_t *dev, int32_t led);
};
```

下面说说自己的理解，module 这个结构的存在是为了让这个动态库（模块）可以被识别，它的固定变量名字就有这个好处，然后可以顺利的调用，open 函数，调用了open函数**动态分配一个 “设备”** 返回给调用它的函数，得到了这个设备，里面就有和内核接口交互的函数了，相当于得到了一个操作句柄。

以上面为例子，就是这个过程

* 找到 HAL\_MODULE\_INFO\_SYM 变量所在地址
* upcast 为 hw\_module\_t 调用 open 函数
* open 函数返回一个设备，上层可以 downcast

限制了第一个域必须是 hw\_module\_t hw\_device\_t 目的就是为了方便操作，当然也可以像内核那样，使用一些宏就可以解决，这个其实就是_**继承的实质**_。

## 上层是什么？

注意到这篇文章的结构是自底向上的，因为针对的读者不一样，针对的读者已经学习过传统 Linux 驱动，但是想要了解安卓驱动的朋友。所以从熟悉到陌生，会容易掌握。

上层的存在就是想方设法让 Java 能够调用，第二节的讲述了 JNI 的原理，就是为了这种情况设计的。

![ architecture](.gitbook/assets/tu-pian%20%281%29.png)

上面这张图 有很多名词不懂，没关系我也不懂，来慢慢分析，现在我们到了 系统服务这一层了，直接甩代码，再理解。首先，为了上层 Java 程序可以调用，必须建立 jni 中间层。

### Java 和 C 之间桥梁 - JNI

现在我们假设，上面的 HAL 层代码已经被编译成一个动态库，名字叫 libled.so

来看看 Java 和 C 之间的设计

```java
public final class LedService extends ILedService.Stub {
    static {
        System.load("/system/lib/libmokoid_runtime.so");
    }   // 加载 jni 的动态库

    public LedService() {
        Log.i("LedService", "Go to get LED Stub...");
	_init();
    }

    public boolean setOn(int led) {
        Log.i("MokoidPlatform", "LED On");
	return _set_on(led);
    }

    public boolean setOff(int led) {
        Log.i("MokoidPlatform", "LED Off");
	return _set_off(led);
    }

    private static native boolean _init();
    private static native boolean _set_on(int led);
    private static native boolean _set_off(int led);
}
```

有人问 aidl 的作用，请出门维基，这里没有这么多篇幅啦，总之就是一个接口的设计。

{% code-tabs %}
{% code-tabs-item title="frameworks/base/core/java/mokoid/hardware/ILedService.aidl" %}
```java
package mokoid.hardware;

interface ILedService
{
    boolean setOn(int led);
    boolean setOff(int led);
}

```
{% endcode-tabs-item %}
{% endcode-tabs %}

下面来看  JNI 的中间代码，也就是上面 加载的 libmokoid\_runtime.so

```cpp
struct led_control_device_t *sLedDevice = NULL;

/** helper APIs */
static inline int led_control_open(const struct hw_module_t* module,
        struct led_control_device_t** device) {
    return module->methods->open(module,
            LED_HARDWARE_MODULE_ID, (struct hw_device_t**)device);
}

static jboolean mokoid_init(JNIEnv *env, jclass clazz)
{
    led_module_t* module;

    if (hw_get_module(LED_HARDWARE_MODULE_ID, (const hw_module_t**)&module) == 0) {
        LOGI("LedService JNI: LED Stub found.");
        if (led_control_open(&module->common, &sLedDevice) == 0) {
    	    LOGI("LedService JNI: Got Stub operations.");
            return 0;
        }
    }
    LOGE("LedService JNI: Get Stub operations failed.");
    return -1;
}

static const JNINativeMethod gMethods[] = {
    { "_init",	  	"()Z",	(void *)mokoid_init },
    { "_set_on",        "(I)Z", (void *)mokoid_setOn },
    { "_set_off",       "(I)Z", (void *)mokoid_setOff },
};

static jboolean mokoid_setOn(JNIEnv* env, jobject thiz, jint led) 
{
    LOGI("LedService JNI: mokoid_setOn() is invoked.");

    if (sLedDevice == NULL) {
        LOGI("LedService JNI: sLedDevice was not fetched correctly.");
        return -1;
    } else {
        return sLedDevice->set_on(sLedDevice, led);
    }
}
...
```

省略了一下加载的代码，关键我们要理解，Java 里面的 Native 关键字表面的意思就是一个外部的函数的指针，这里一个映射表表明了他们的关系。

关键的一个函数， init（） 里面调用的 _**hw\_get\_module** ta_他的作用，是通过传进去的参数，找到模块的动态库的路径，然后 dlopen，然后就是我们在上一节说的 HAL 的作用里面的事情了。

{% code-tabs %}
{% code-tabs-item title="libhardware / master / . / hardware.c" %}
```c

// "led.h"
#define LED_HARDWARE_MODULE_ID "led"

hw_get_module(LED_HARDWARE_MODULE_ID, (const hw_module_t**)&module);

int hw_get_module(const char *id, const struct hw_module_t **module)
{
    return hw_get_module_by_class(id, NULL, module);
}

其实就是一些固定的路径，也就是说 模块的动态库的路径是有一定规律的
{
    snprintf(path, path_len, "%s/%s.%s.so",
             HAL_LIBRARY_PATH3, name, subname);
    if (access(path, R_OK) == 0)
        return 0;
    snprintf(path, path_len, "%s/%s.%s.so",
             HAL_LIBRARY_PATH2, name, subname);
    if (access(path, R_OK) == 0)
        return 0;

}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

假设上面根据规则 找到了 我们上面说的 led.so 最后的做的事情就是 dlopen

```c
static int load(const char *id,
        const char *path,
        const struct hw_module_t **pHmi)
{        
         ...
         void * handle = dlopen(path, RTLD_NOW);
         ...
         const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
         struct hw_module_t * hmi = 
                  (struct hw_module_t *)dlsym(handle, sym);
         *pHmi = hmi;
         ...
         return 0;       
}
```

看到此，恍然大悟，万事不过 _**dlopen ，**_接着 _**init**_ 函数调用了 _**open**_ 函数得到了设备的结构，接着就是把它存储在一个全局变量了。另外俩个开关的函数，只是利用这个全局变量来调用它的接口。

### Service 和 Manager

现在**忽略底层**，我们从 LedService 起步，做一个本本分分的 Java 编程人员。注意它继承了一个基类，也就是刚才说的篇幅不够解释的一个接口，

```java
public final class LedService extends ILedService.Stub {
    public boolean setOn(int led);
    public boolean setOff(int led);
}

咳咳，一个不专业的人员写的 Java 类 “声明”  方便大家看

```

其实我们 new 一个 LedService 就可以访问硬件了，这是没错的，实际上已经链接在一起了，hw\_device\_t 可以 downcast 为 led\_control\_device\_t 的一个结构里面就有 set\_on set\_off 指针，即真正的操作。

但是这不是规范，我也不知道怎么解释其中的缺点，我个人拙见，是因为不好管理，如果每一个使用 led 的人都 new 一个对象，其实不能保证它们的并发访问的安全？底层的加载代码保证的是不同的线程只 dlopen 一次，即不同的线程都有 刚才 JNI 层里面提到的那个**全局变量**，但是相同的线程其实都在访问一个地址，所以我认为有点隐患。

所以正确的方案？就是所有的访问都调用一个接口，一个可以管理的接口。

{% code-tabs %}
{% code-tabs-item title="com/mokoid/LedTest/LedSystemServer.java" %}
```java
public class LedSystemServer extends Service {

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    public void onStart(Intent intent, int startId) {
        Log.i("LedSystemServer", "Start LedService...");

	/* Please also see SystemServer.java for your interests. */
	LedService ls = new LedService();

        try {
            ServiceManager.addService("led", ls);
            // 添加了一个叫 led 的服务
        } catch (RuntimeException e) {
            Log.e("LedSystemServer", "Start LedService failed.");
        }
    }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

这里实例化了一个对象，并把它注册在了 ServiceManager 上，一个静态的全局对象，以后所有的访问都经过它就好了。添加服务这种说法，其实就是一个_**进程间通信**_的幌子，大家都在操作系统里注册一个位置，然后别人访问的时候，操作系统就负责通信。进程间通信的手段很多，不过这里不是我们的重点。

除了要有一个 服务，还需要有一个服务的管理者

{% code-tabs %}
{% code-tabs-item title="frameworks/base/core/java/mokoid/hardware/LedManager.java" %}
```java
public class LedManager
{
    private static final String TAG = "LedManager";
    private ILedService mLedService;

    public LedManager() {
        mLedService = ILedService.Stub.asInterface(
                             ServiceManager.getService("led"))
    }

    public boolean LedOn(int n) {
        boolean result = false;
        result = mLedService.setOn(n);
        return result;
    }

    public boolean LedOff(int n) {
        boolean result = false;
        result = mLedService.setOff(n);
        return result;
    }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

如此这个管理者，每次跟系统的服务管理者拿到一个服务，同时又把它 upcast 到当时它注册到的基类，借此来调用。

### 应用程序的调用

那么我们用户如何做呢，只需要 new 一个 LedManager

```java
public class LedTest extends Activity implements View.OnClickListener {
    private LedManager mLedManager = null;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // Start LedService in a seperated process.
        startService(new Intent("com.mokoid.systemserver"));

        Button btn = new Button(this);
        btn.setText("Click to turn LED 1 On");
	    btn.setOnClickListener(this);

        setContentView(btn);
    }

    public void onClick(View v) {
        // Get LedManager.
        if (mLedManager == null) {
	        mLedManager = new LedManager();

        /** Call methods in LedService via proxy object 
         * which is provided by LedManager. 
         */
        mLedManager.LedOn(1);

        TextView tv = new TextView(this);
        tv.setText("LED 1 is On.");
        setContentView(tv);
    }
}
```

看，底层做了这么多的事情，都是为了上层的便利呀 O\(∩\_∩\)O

## 总结

终于到了敲黑板环节，又到了结尾了。自己对着掌握与否打勾。

* [ ] 内核实现的驱动由 HAL 层来操作
* [ ] HAL 操作之后包装成动态库
* [ ] JNI 打开动态库，完成和Java类的接口，同时自己也包装成一个动态库
* [ ] Java类继承相关的stub，静态加载jni动态库，并注册一个Service
* [ ] 编写一个 ServiceManager，跟系统服务管理进程所要对应的 Service
* [ ] 用户调用 new 一个 Manager

注意 3，4，5 这一层我们称之为（Framework）框架层，一切为了用户的层，而 HAL 和 内核就很容易辨识了。



