# Linux input 子系统

## 序

早在几年前，我第一次翻开 Linux 0.11 源代码完全注释那本书的时候，我就折服于这神奇的计算机世界，其实这里面的知识不是有多难，而是沉淀了太久，让上面的人琢磨不透，这也是封装的弊端，总得有人知道轮子是怎么造出来的。

后来涉及到驱动的架构的时候，读了挺多的 PCI 架构的一些知识，当时是用打印机把相关代码打印出来，然后用笔写一些注释，但是看懂已经很吃力了，还得发到网上分享，其实是一件非常累的事情，所以我很佩服在网络上能分享知识的人，因为都是无偿的，和出书不同。

这几天又看了一些输入系统的相关的文章，特在此处写一下关键的部分，不然像之前阅读的代码那时，很多东西其实没有记下来，有点后悔。

## 驱动的初始化

### 设备驱动的特点

这里都是说一些源代码，还有自己的一些理解，其实不同的子系统（总线，USB），都有一个显著的特点，就是有俩个**全局变量**，分别是_**一条链把所有的设备连接，另外一条就连接所有的驱动**_，然后设备和驱动之间如果匹配成功（所谓匹配成功，以 PCI设备和驱动为例，PCI 设备在Rom区会标识自己的ID，及生产厂商等信息，驱动加载的时候也会提供这些信息，然后不管挂载新驱动还是设备，都会遍历所有的设备（如果是挂载设备，就遍历驱动），然后比对他们的ID是否相同，如果相同就会挂钩，挂钩就是指一些指针存储着对方的信息，这时候我们就叫匹配成功），就会有相关的指针存储对方的信息。

**重点**： 1. 程序有办法知道目前挂载的所有设备还有驱动 2. 驱动和设备都有一个相同的id用来辨识

{% hint style="info" %}
如果有读者看过文件系统和超级块（vfs 实现），其实也有全局变量把它们串联在一起，目的就是方便遍历
{% endhint %}

### input 子系统

了解上面这个特点，现在看 input，一个全局的 List 头，用来挂载所有的 输入设备，还有一个挂载的就是驱动，上面一个宏定义了内核能支持的最大输入设备数，另外还有一个表，用来快速获取对应的驱动，等下们可以看到它的用处。\( linux -2.6.39 \)

```c
#define INPUT_DEVICES    256

static LIST_HEAD(input_dev_list);
static LIST_HEAD(input_handler_list);

static struct input_handler *input_table[8];
```

下面来看input子系统的初始化

```c
static const struct file_operations input_fops = {
    .owner = THIS_MODULE,
    .open = input_open_file,
    .llseek = noop_llseek,
};

static int __init input_init(void)
{
    int err;

    err = class_register(&input_class);
    if (err) {
        pr_err("unable to register input_dev class\n");
        return err;
    }

    err = input_proc_init();
    if (err)
        goto fail1;

    err = register_chrdev(INPUT_MAJOR, "input", &input_fops);
    if (err) {
        pr_err("unable to register char major %d", INPUT_MAJOR);
        goto fail2;
    }

    return 0;

 fail2:    input_proc_exit();
 fail1:    class_unregister(&input_class);
    return err;
}
...
subsys_initcall(input_init);
```

关键在，注册一个字符设备，它的设备号就是 13（INPUT\_MAJOR），设备号分为主次设备号的原因就是**分类**，其实**本质就是一个数字**，多嘴一句，如何通过设备号找到对应的设备很简单，就是一个映射的表，最简单的办法就是创建一个数组大小就是 设备号的最大值，里面存放设备的地址，但是这样太浪费了，所以我们就用数组加链表，本质其实还是一样的。

这里还很多谜团没有解开，比如 open 函数的作用（参考块设备的open函数也不难猜到），还有input设备/驱动的注册

#### input设备的注册

{% code title="/drivers/input/input.c" %}
```c
int input_register_handler(struct input_handler *handler)
{
    struct input_dev *dev;
    int retval;

    retval = mutex_lock_interruptible(&input_mutex);

    INIT_LIST_HEAD(&handler->h_list);

    if (handler->fops != NULL) {
        ... // minor/32得到它的位置，便于快速访问驱动
        input_table[handler->minor >> 5] = handler;   
    }
    // 添加到驱动的链表尾端
    list_add_tail(&handler->node, &input_handler_list);

    // 遍历所有的设备，匹配驱动和设备
    list_for_each_entry(dev, &input_dev_list, node)
        input_attach_handler(dev, handler);

    input_wakeup_procfs_readers();

 out:
    mutex_unlock(&input_mutex);
    return retval;
}
```
{% endcode %}

这个函数是内核暴露出来的驱动接口，供驱动模块调用，这里省略了错误的判断，为了更直观的显示它的作用。

有一点读者肯定很好奇，为什么 minor 要除以32，得到了它的位置，也就是设备号的低5位是被忽略了，_**也就是说 一个输入驱动（input handler），可以处理32个设备**_。这里注意的 input handler 和 input handle 是俩个结构，最开始我也弄错了

{% code title="/include/linux/input.h" %}
```c
struct input_handler {
    void *private;

    void (*event)(struct input_handle *handle, unsigned int type, unsigned int code, int value);
    bool (*filter)(struct input_handle *handle, unsigned int type, unsigned int code, int value);
    bool (*match)(struct input_handler *handler, struct input_dev *dev);
    int (*connect)(struct input_handler *handler, struct input_dev *dev, const struct input_device_id *id);
    void (*disconnect)(struct input_handle *handle);
    void (*start)(struct input_handle *handle);

    const struct file_operations *fops;
    int minor;
    const char *name;

    const struct input_device_id *id_table;

    struct list_head    h_list;  // handle_list 连接它的handle会挂载于这里
    struct list_head    node;    // 挂载到全局驱动表上
};

/**
 * struct input_handle - links input device with an input handler
 * @private: handler-specific data
 * @open: counter showing whether the handle is 'open', i.e. should deliver
 *    events from its device
 * @name: name given to the handle by handler that created it
 * @dev: input device the handle is attached to
 * @handler: handler that works with the device through this handle
 * @d_node: used to put the handle on device's list of attached handles
 * @h_node: used to put the handle on handler's list of handles from which
 *    it gets events
 */
struct input_handle {

    void *private;

    int open;
    const char *name;

    struct input_dev *dev;
    struct input_handler *handler;

    struct list_head    d_node;
    struct list_head    h_node;
};
```
{% endcode %}

所有的设计都是为了方便操作，驱动里面的这些函数大部分会被底层的设备调用，我们先以具体的一个输入设备来举例子。

#### evdev 事件设备

{% code title="drivers/input/evdev.c" %}
```c
static int __init evdev_init(void)
{
    return input_register_handler(&evdev_handler);
}

static struct input_handler evdev_handler = {
    .event        = evdev_event,
    .connect    = evdev_connect,
    .disconnect    = evdev_disconnect,
    .fops        = &evdev_fops,
    .minor        = EVDEV_MINOR_BASE,
    .name        = "evdev",
    .id_table    = evdev_ids,
};

static const struct input_device_id evdev_ids[] = {
    { .driver_info = 1 },    /* Matches all devices */
    { },            /* Terminating zero entry */
};
```
{% endcode %}

id\_table 就是我们上面所说的和设备匹配的一个表，注释里说明了一个事实，就是_**所有的输入设备都会和这个evdev的驱动匹配**_。

{% code title="drivers/input/input.c" %}
```c
static const struct input_device_id *input_match_device(struct input_handler *handler,
                            struct input_dev *dev)
{
    const struct input_device_id *id;
    int i;
    // 注意到，evdev的id_table->driver_info = 1
    // 这个循环可以一直进行下去
    for (id = handler->id_table; id->flags || id->driver_info; id++) {

        ...    ....

        if (!handler->match || handler->match(handler, dev))
            return id;
    }

    return NULL;
}

static int input_attach_handler(struct input_dev *dev, struct input_handler *handler)
{
    const struct input_device_id *id;
    int error;

    id = input_match_device(handler, dev);
    if (!id)
        return -ENODEV;

    error = handler->connect(handler, dev, id);
    if (error && error != -ENODEV)
        pr_err("failed to attach handler %s to device %s, error: %d\n",
               handler->name, kobject_name(&dev->dev.kobj), error);

    return error;
}
```
{% endcode %}

所以最后会调用，evdev的 connect 函数

{% code title="drivers/input/evdev.c" %}
```c
/*
 * Create new evdev device. Note that input core serializes calls
 * to connect and disconnect so we don't need to lock evdev_table here.
 */
static int evdev_connect(struct input_handler *handler, struct input_dev *dev,
             const struct input_device_id *id)
{
    struct evdev *evdev;
    int minor;
    int error;

    for (minor = 0; minor < EVDEV_MINORS; minor++)
        if (!evdev_table[minor])
            break;

    evdev = kzalloc(sizeof(struct evdev), GFP_KERNEL);
    ...
    dev_set_name(&evdev->dev, "event%d", minor);
    evdev->exist = true;
    evdev->minor = minor;

    evdev->handle.dev = input_get_device(dev);
    evdev->handle.name = dev_name(&evdev->dev);
    evdev->handle.handler = handler;
    evdev->handle.private = evdev;

    evdev->dev.devt = MKDEV(INPUT_MAJOR, EVDEV_MINOR_BASE + minor);
    evdev->dev.class = &input_class;
    evdev->dev.parent = &dev->dev;
    evdev->dev.release = evdev_free;
    device_initialize(&evdev->dev);

    error = input_register_handle(&evdev->handle);
    error = evdev_install_chrdev(evdev);
    error = device_add(&evdev->dev);
    ...
    return 0;
}
```
{% endcode %}

也是省略了部分出错的判断，我们来看看关键的几个地方

* 首先，自己计算一个minor，在自己管理的32个minor的基础上
* 分配一个新的，evdev结构，初始化（包括了设备初始化）
* 注册一个新的 _**handle**_ **注意**不是handler
* 存储 evdev，就是放在一个数组中，方便索引

```c
int input_register_handle(struct input_handle *handle)
{
    struct input_handler *handler = handle->handler;
    struct input_dev *dev = handle->dev;
    int error;

    /*
     * We take dev->mutex here to prevent race with
     * input_release_device().
     */
    error = mutex_lock_interruptible(&dev->mutex);

    /*
     * Filters go to the head of the list, normal handlers
     * to the tail.
     */
    if (handler->filter)
        list_add_rcu(&handle->d_node, &dev->h_list);
    else
        list_add_tail_rcu(&handle->d_node, &dev->h_list);

    mutex_unlock(&dev->mutex);
    /*
     * Since we are supposed to be called from ->connect()
     * which is mutually exclusive with ->disconnect()
     * we can't be racing with input_unregister_handle()
     * and so separate lock is not needed here.
     */
    list_add_tail_rcu(&handle->h_node, &handler->h_list);

    if (handler->start)
        handler->start(handle);

    return 0;
}
```

注释很清楚啦， input\_handle 就是连接 handler 和 input device 一个媒介，这里就是添加其到相应的设备链表上。

![&#x5BF9;&#x5E94;&#x5173;&#x7CFB;&#xFF0C;&#x6765;&#x6E90;&#x7F51;&#x7EDC;](../.gitbook/assets/tu-pian%20%283%29.png)

换言之，只要是输入设备，一定会与 evdev 的驱动匹配，从而创建一个 evdev设备（名字 event%d（0~31）），其父设备就是 Input device，然后 evdev 的 minor 是计算出来的，只要数组里的元素没有赋值，那就是可用的

```c
struct evdev {
    int open;
    int minor;
    struct input_handle handle;
    wait_queue_head_t wait;
    struct evdev_client __rcu *grab;
    struct list_head client_list;
    spinlock_t client_lock; /* protects client_list */
    struct mutex mutex;
    struct device dev;
    bool exist;
};
```

#### 具体的 input device 例子 - usb kbd

USB 键盘下接 USB总线，这里我先忽略，上接 input 子系统，我们关注它的这一部分

{% code title="drivers/hid/usbhid/usbkbd.c" %}
```c
struct usb_kbd {
    struct input_dev *dev;
    struct usb_device *usbdev;
    unsigned char old[8];
    struct urb *irq, *led;
    unsigned char newleds;
    char name[128];
    char phys[64];

    unsigned char *new;
    struct usb_ctrlrequest *cr;
    unsigned char *leds;
    dma_addr_t new_dma;
    dma_addr_t leds_dma;
};

......
static int usb_kbd_probe(struct usb_interface *iface,
             const struct usb_device_id *id)
{
    struct usb_device *dev = interface_to_usbdev(iface);
    struct usb_host_interface *interface;
    struct usb_endpoint_descriptor *endpoint;
    struct usb_kbd *kbd;
    struct input_dev *input_dev;
    int i, pipe, maxp;
    int error = -ENOMEM;

    interface = iface->cur_altsetting;


    kbd = kzalloc(sizeof(struct usb_kbd), GFP_KERNEL);
    input_dev = input_allocate_device();
    if (!kbd || !input_dev)
        goto fail1;

    if (usb_kbd_alloc_mem(dev, kbd))
        goto fail2;

    kbd->usbdev = dev;
    kbd->dev = input_dev;
    ...
    input_dev->name = kbd->name;
    input_dev->phys = kbd->phys;
    usb_to_input_id(dev, &input_dev->id);
    input_dev->dev.parent = &iface->dev;

    input_set_drvdata(input_dev, kbd);

    input_dev->evbit[0] = BIT_MASK(EV_KEY) | BIT_MASK(EV_LED) |
        BIT_MASK(EV_REP);
    input_dev->ledbit[0] = BIT_MASK(LED_NUML) | BIT_MASK(LED_CAPSL) |
        BIT_MASK(LED_SCROLLL) | BIT_MASK(LED_COMPOSE) |
        BIT_MASK(LED_KANA);

    for (i = 0; i < 255; i++)
        set_bit(usb_kbd_keycode[i], input_dev->keybit);
    clear_bit(0, input_dev->keybit);

    input_dev->event = usb_kbd_event;
    input_dev->open = usb_kbd_open;
    input_dev->close = usb_kbd_close;
    ...
    error = input_register_device(kbd->dev);
    ...
    return 0;
}
```
{% endcode %}

usb 特有的部分都可以忽略，关键在于**分配了一个 input device 并注册了它**，这就是关键了，联系上面，这个设备肯定会和 evdev 驱动匹配，（**intput\_register\_device 同样会遍历驱动的链表，逐项比对id，最终连接双方**），然后建立一个新的 evdev 设备，分配一个 Input handle 结构，连接驱动和现在分配的这个设备。

然后，当我们按下按键的时候，就会触发中断，最终来到这里。这里我们还可以发现，input device 也有自己open close 它们会在某个地方被调用，先卖个关子。

```c
static void usb_kbd_irq(struct urb *urb)
{
    struct usb_kbd *kbd = urb->context;
    int i;

    switch (urb->status) {
    case 0:            /* success */
        break;
    case -ECONNRESET:    /* unlink */
    case -ENOENT:
    case -ESHUTDOWN:
        return;
    /* -EPIPE:  should clear the halt */
    default:        /* error */
        goto resubmit;
    }

    for (i = 0; i < 8; i++)
        input_report_key(kbd->dev, usb_kbd_keycode[i + 224], (kbd->new[0] >> i) & 1);
    ..
    ..
    ..
    input_sync(kbd->dev);    
    ....
}
```

有一些按键的继续按下，这个我们可以忽略，关键在于调用了一个 input\_report\_key

```c
include/linux/input.h
static inline void input_report_key(struct input_dev *dev, unsigned int code, int value)
{
    input_event(dev, EV_KEY, code, !!value);
}

drivers/input/input.c
void input_event(struct input_dev *dev,
         unsigned int type, unsigned int code, int value)
{
    unsigned long flags;

    if (is_event_supported(type, dev->evbit, EV_MAX)) {

        spin_lock_irqsave(&dev->event_lock, flags);
        add_input_randomness(type, code, value);
        input_handle_event(dev, type, code, value);
        spin_unlock_irqrestore(&dev->event_lock, flags);
    }
}

drivers/input/input.c
static void input_handle_event(struct input_dev *dev,
                   unsigned int type, unsigned int code, int value)
{
    int disposition = INPUT_IGNORE_EVENT;

    switch (type) {
    ....
    case EV_KEY:
        if (is_event_supported(code, dev->keybit, KEY_MAX) &&
            !!test_bit(code, dev->key) != value) {

            if (value != 2) {
                __change_bit(code, dev->key);
                if (value)
                    input_start_autorepeat(dev, code);
                else
                    input_stop_autorepeat(dev);
            }

            disposition = INPUT_PASS_TO_HANDLERS;
        }
        break;
    ....
    }
    if ((disposition & INPUT_PASS_TO_DEVICE) && dev->event)
        dev->event(dev, type, code, value);

    if (disposition & INPUT_PASS_TO_HANDLERS)
        input_pass_event(dev, type, code, value);
}
对于 ENV_KEY 是可以传递给其他驱动的，就会来到 input_pass_event 了
```

这个函数读者肯定都可以猜到，就是遍历设备里的那个监听链表，我们在注册handle的时候放在了设备和驱动那儿，而且对于有 filter 的设备还是放在头部的，就是为了快速处理

```c
/*
 * Pass event first through all filters and then, if event has not been
 * filtered out, through all open handles. This function is called with
 * dev->event_lock held and interrupts disabled.
 */
static void input_pass_event(struct input_dev *dev,
                 unsigned int type, unsigned int code, int value)
{
    struct input_handler *handler;
    struct input_handle *handle;

    rcu_read_lock();

    handle = rcu_dereference(dev->grab);
    if (handle)
        handle->handler->event(handle, type, code, value);
    else {
        bool filtered = false;

        list_for_each_entry_rcu(handle, &dev->h_list, d_node) {
            if (!handle->open)
                continue;
            handler = handle->handler;
            if (!handler->filter) {
                if (filtered)
                    break;
                handler->event(handle, type, code, value);
            } else if (handler->filter(handle, type, code, value))
                filtered = true;
        }
    }
    rcu_read_unlock();
}
```

调用 handler的 event 函数，最终就会来到， evdev\_event\(\)

```c
static struct input_handler evdev_handler = {
    .event        = evdev_event,
    ....
}    

/*
 * Pass incoming event to all connected clients.
 */
static void evdev_event(struct input_handle *handle,
            unsigned int type, unsigned int code, int value)
{
    struct evdev *evdev = handle->private;
    struct evdev_client *client;
    struct input_event event;

    do_gettimeofday(&event.time);
    event.type = type;
    event.code = code;
    event.value = value;

    rcu_read_lock();

    client = rcu_dereference(evdev->grab);
    if (client)
        evdev_pass_event(client, &event);
    else
        list_for_each_entry_rcu(client, &evdev->client_list, node)
            evdev_pass_event(client, &event);

    rcu_read_unlock();

    wake_up_interruptible(&evdev->wait);
}
```

evdev 又会和监听它的上层挂钩，这里可以看出， evdev的存在就是监听所有的 Input device，每个设备会有自己的驱动，但是 evdev 就像是网卡的混淆模式，可以监听到所有的包，它可以监听到所有input device可传递的事件。

### 为什么要注册一个字符设备

最开始在子系统初始化的过程中，我们看到注册了一个主设备号是 INPUT\_MAJOR 的一个字符设备，并给它提供了一个文件操作函数，到目前为止我们都没有解释它存在的意义，注意，我们之前说的注册输入设备，本质就是挂载到了那个全局的链表，如果存在和它匹配的驱动，当然 evdev 驱动必然匹配，然后还会创建一个新的 evdev 设备，但是注意一点，**input device 设备并没有被打开**。即在那时，它提供的 open 函数并没有被调用。

```c
事实上，当驱动想要注册一个设备，不管是 Input，还是 evdev 设备，都一定调用一个接口
这个接口就是 device_add 它会根据设备的父设备在 sysfs 创建接口，根据设备的 class
和名字创建设备接口 

register_input_handler 
                                -> device_add
register_input_device

这里说的情况都是有 devtmpfs 机制内核，即每次注册一个设备都会
自动创建一个节点，关键是这个函数

int devtmpfs_create_node(struct device *dev)
{
    ....
    if (mode == 0)
        mode = 0600;
    if (is_blockdev(dev))
        mode |= S_IFBLK;
    else
        mode |= S_IFCHR; // 重点就在于此，这个设备是字符设备

    curr_cred = override_creds(&init_cred);

    err = vfs_path_lookup(dev_mnt->mnt_root, dev_mnt,
                  nodename, LOOKUP_PARENT, &nd);
    if (err == -ENOENT) {
        create_path(nodename);
        err = vfs_path_lookup(dev_mnt->mnt_root, dev_mnt,
                      nodename, LOOKUP_PARENT, &nd);
    }
    if (err)
        goto out;

    dentry = lookup_create(&nd, 0);
    if (!IS_ERR(dentry)) {
        err = vfs_mknod(nd.path.dentry->d_inode,
                dentry, mode, dev->devt);
        if (!err) {
            struct iattr newattrs;

            /* fixup possibly umasked mode */
            newattrs.ia_mode = mode;
            newattrs.ia_valid = ATTR_MODE;
            mutex_lock(&dentry->d_inode->i_mutex);
            notify_change(dentry, &newattrs);
            mutex_unlock(&dentry->d_inode->i_mutex);

            /* mark as kernel-created inode */
            dentry->d_inode->i_private = &dev_mnt;
        }
        dput(dentry);
    }
}
```

注意到，这些设备都被认为是字符设备，但是我们并没有调用 register\_chardev ，读者估计已经猜到了，肯定就是为了和input初始化那里对接上，试想想，我们给设备认为是字符设备，最终内核就会根据它的 MAJOR 和 MINOR找到最合适的一个 fops即文件操作函数，这个关键就是 kobj\_map 函数，其实就是一个映射表。总之，这些 input device，包括 evdev device，都是拥有相同的major，所以当用户打开这个设备的时候，就会调用到初始化的那个open函数。

```c
static const struct file_operations input_fops = {
    .owner = THIS_MODULE,
    .open = input_open_file,
    .llseek = noop_llseek,
};

static int input_open_file(struct inode *inode, struct file *file)
{
    struct input_handler *handler;
    const struct file_operations *old_fops, *new_fops = NULL;
    int err;

    err = mutex_lock_interruptible(&input_mutex);
    if (err)
        return err;

    /* No load-on-demand here? */
    handler = input_table[iminor(inode) >> 5];
    if (handler)
        new_fops = fops_get(handler->fops);

    mutex_unlock(&input_mutex);

    /*
     * That's _really_ odd. Usually NULL ->open means "nothing special",
     * not "no device". Oh, well...
     */
    if (!new_fops || !new_fops->open) {
        fops_put(new_fops);
        err = -ENODEV;
        goto out;
    }

    old_fops = file->f_op;
    file->f_op = new_fops;

    err = new_fops->open(inode, file);
    if (err) {
        fops_put(file->f_op);
        file->f_op = fops_get(old_fops);
    }
    fops_put(old_fops);
out:
    return err;
}
```

当用户第一次调用 open\("event0", RW\) 的时候，因为**它是字符设备，同时 major 又跟 最开始注册 input设备一样**，所以内核认为input设备注册是 fops 就是能适用，所以调用。

上面的代码不难理解，就是调用了handler的open函数，对于 evdev，它的open即为

```c
static const struct file_operations evdev_fops = {
    .owner        = THIS_MODULE,
    .read        = evdev_read,
    .write        = evdev_write,
    .poll        = evdev_poll,
    .open        = evdev_open,
    .release    = evdev_release,
    .unlocked_ioctl    = evdev_ioctl,
#ifdef CONFIG_COMPAT
    .compat_ioctl    = evdev_ioctl_compat,
#endif
    .fasync        = evdev_fasync,
    .flush        = evdev_flush,
    .llseek        = no_llseek,
};

这个函数由input设备的注册那个 open调用，紧接着它调用了自己的open
static int evdev_open(struct inode *inode, struct file *file)
{
    ...

    error = evdev_open_device(evdev);
    ...
}

这里最关键的一点，又调用了 input架构提供的open函数
static int evdev_open_device(struct evdev *evdev)
{
    int retval;

    retval = mutex_lock_interruptible(&evdev->mutex);
    if (retval)
        return retval;

    if (!evdev->exist)
        retval = -ENODEV;
    else if (!evdev->open++) {
        retval = input_open_device(&evdev->handle);
        if (retval)
            evdev->open--;
    }

    mutex_unlock(&evdev->mutex);
    return retval;
}
```

这里读者知道，刚才再说 usb 键盘驱动那里我们卖了一关子，usb 设备对 input 设备注册了自己 open 函数，但是没说哪里调用。

```c
/**
 * input_open_device - open input device
 * @handle: handle through which device is being accessed
 *
 * This function should be called by input handlers when they
 * want to start receive events from given input device.
 */
int input_open_device(struct input_handle *handle)
{
    struct input_dev *dev = handle->dev;
    int retval;
   ...
    handle->open++;

    if (!dev->users++ && dev->open)
        retval = dev->open(dev);
     // 就在这里调用了
   ....
 out:
    mutex_unlock(&dev->mutex);
    return retval;
}
```

这里设备的 open 函数，以 刚才 usb 键盘为例，就会来到 usb\_kbd\_open 了，这里与 USB 子系统有关，就不继续分析了。

## Summary

最后我们总结一下，最开始 input 设备注册的函数，只是一个分发器，为的就是调用它们的handler-&gt;open ，因为每个handler可能有自己自带的设备，注意，以 evdev 为例，_**它注册了一个新的设备号拥有不同的 minor， 名称为 event%d，但是它的父设备，又是与这个 handler 匹配成功的 input device**_，这里有点绕，但是必须理解清楚。

evdev 的handler 最终调用了 input device 的 open 函数，其实内核的代码注释里面也说了，这里直接引用吧。

> "
>
> ```text
> /**
>  * input_open_device - open input device
>  * @handle: handle through which device is being accessed
>  *
>  * This function should be called by input handlers when they
>  * want to start receive events from given input device.
>  */
> ```
>
> "

这是一种懒惰的思想，只有当驱动想要使用设备，也就是上层需要读取设备信息的时候，我们才打开设备，目的就是为了节省系统资源。

这篇文章得理解一下几点

* 为什么要注册那个字符设备
* 为什么字符设备的open函数可以最终到达目标输入设备的open
* 驱动的特点，即俩个链表，还有匹配的特点
* 设备信息如何传递到监听它的 handler（利用设备的 h\_list）
* evdev 可以自由分配自己的 32 个minor

