# VFS 序

这篇导读是后面才回来写的，是因为有些问题觉得提前说出来会让读者更容易理解，自己以后回来复习的时候更好理解。我先提出新人最开始好奇的问题（以自己为例，以前这些问题都是最开始自己非常好奇的 ），然后简单（让人听懂）的回答，之后在后的文章补充。

## FAQ

### 1. 我们读写普通文件（path/to/file），跟读写设备节点（/dev/sda）有区别吗？

```c
没区别，首先明确一点，在文件系统的眼里，这些都是文件，只是不同的类型，我们设备的节点
在关机之后如果文件系统不会删除，那就跟普通文件，那么区别到底在哪里？
处理方式不一样。
open：
    if( file is normal file ) {
        file->ops = &normal_ops;
    } else if( device file ) {
        file->ops = &device_ops;
    }
    
ops 就是一系列的函数指针，当我们打开一个文件的时候
文件系统的open函数初始化的

/*
 * NOTE:
 * read, write, poll, fsync, readv, writev, unlocked_ioctl and compat_ioctl
 * can be called without the big kernel lock held in all filesystems.
 */
struct file_operations {
	...
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	int (*readdir) (struct file *, void *, filldir_t);
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
	...
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	...
};
```

对于普通文件，read/write 最终就是从硬盘中读取数据，而对于设备文件，则是从“设备”读取文件，其实读者发现了，硬盘不就是一个设备，所以两者没有任何区别，只是数据的获取方式不一样。

###  2. 文件操作函数怎么注册的？

冤有头，债有主，文件系统的模块提供的，也可以说是驱动提供的，思考一问题，这不就是一个函数指针的数组吗，只要操作系统可以找到不就好了，最简单的办法，在操作系统里面增加一个map

```c

static MAP map[] {
    {"ext2", ext2_file_operations},
    {"minix", minix_file_operations},
    ....
}
```

当然，Linux 有一个相对复杂的注册机制，但是本质就是获得了文件操作函数，所以可以使得一个文件可以正确被打开。

### 3. 操作系统如何知道一个文件的文件系统

任意一个目录下，都可能挂载一个不同文件系统，那么到底操作系统如何知道的呢？其实也就是说，如何为一个文件获取一个正确的 _**file\_operations**_。

还是一样的，冤有头债有主，考虑一个所谓的目录结构，是怎么出现的。

```c
 mount('/anywhere/we/want/0.0/:=)/', name_of_a_filesystem, dev);
 可能不是根目录，但无论如何处理，都一定有几个参数，也就是
 挂载的目录，还有 挂载的文件系统 以及 存储了文件的一个设备
```

{% hint style="info" %}
很多时候，我们不指定文件系统类型，实际上内核帮我们检测了，一个一个去试直到得到一个正确的。
{% endhint %}

举个例子，假设我们在 /mnt/ 目录下，mount 了一个 ext2 文件系统，来自一个全新的硬盘，里面什么信息都没有，刚刚被我们格式化为 ext2  。

```text
        "/"     -  内核根文件系统
        /
     "mnt"     -  内核根文件系统
       /
     "/"        -  ext2 文件系统           
        
```

这里是一个树结构，当我们在一个目录下挂载之后，就会出现这个情况。现在我们设计一个简单的结构来表示它。

```c
struct file {
    struct file * parent;
    
    struct file * children;
    struct file * sibling;
    
    int mount_point;   /* 1 - mountpoint 说明有文件系统挂载在这 */
    struct file * root;
    void * private;    /* */ 
}

struct filesystem_type {
    char * name;
    struct file_operations * f_ops;
    struct file_operations * dir_ops;
}

struct filesystem_type ext2 {
    "ext2",
    ext2_f_ops,
    ext2_dir_ops,
}
```

当我们 mount 的时候就会这样，首先先搜索找到 "mnt" 的 file 结构

```c

void demo_mount(struct file * parent, struct filesystem_type * type ) {
       
         struct file * new_root = malloc( sizeof(*new_root) );
         
         new_root->parent = parent;   // 设置父母
         set_name(new_root, "/");     // 名字
         new_root->private = type;    // 储存文件系统信息
         new_root->parent = new_root; // 根目录指向自己
         
         attach_child(parent, new_root);  // 添加到链表
         parent->mountpoint = 1;        // 这个目录下挂载了新的文件系统
}
```

如此一来，就实现了挂载，当然这是简化版，关键我们理解，我们其实通过了一个新建了子根目录，并且利用了一下结构，来让操作系统明白，这里有新的文件系统。

{% hint style="info" %}
一般来说，一个文件系统的加载必然要涉及到读取相关的块设备，也就是涉及到块设备的知识，我们这里直接介绍到 VFS，不涉及具体的文件系统~
{% endhint %}

```bash
cd /mnt/ && mkdir test
```

考虑前一条的切换当前目录的，我们写一个简单的demo来处理。

```c


PCB 是当前任务的一个结构，这里简化处理

const struct file * namei(char * path  ) {
    struct file * curr, * child;  
    char * child_name;
    
    curr = PCB->current; 
    
    // 如果路径是根目录
    if( *path++ == '\\' ) {
        curr = PCB->root;
    }  
    while( *path ) {
        child_name = get_childname(&path);
       // 简单的处理函数，这里的例子 返回的就是 "mnt"
       
        child = get_file(curr, child_name);
      //  从当前的父目录，得到子目录的一个file结构
      // 同时 path 也更新了它的位置
      
        if( child->mount_point == 1 ) {
         //   说明这个子目录下也挂载了一个新的文件系统
            
            child = get_file(child, "//");
         //   及获取新的文件系统的根目录的file结构
        }
        curr = child;
    }
    
    return curr;
}

```

其实上面的函数非常的简单，当然忽略了非常多的错误处理，搜索file结构的过程其实是数据结构的知识，取决于我们如何设计它们的结构，关键在于，判断新的文件系统挂载于一个新目录的时候，我们的处理就是得到那个挂载文件系统的 **根目录** 的相关结构，这里就真正实现了跨越了文件系统，但是对上层则是**透明**的。

```bash

cd  /mnt / 
        ^ 此处跨越了一个文件系统 
        
        如今  curr 得到是  ext2 文件系统的 ·根目录· 的  file 结构
```

现在来考虑下一条指令

```c

void demo_vfs_mkdir( const char * name ) {
    struct file * parent, * child;
    struct filesystem_type *  fs;
    
    parent = PCB->current;
    经过刚才的 cd，已经指向 ext2 的根目录了
    fs =  file->root->private;
    在挂载的时候，这个值赋值的就是相关的文件结构
    
    /**
     struct filesystem_type {
           char * name;
           struct file_operations * f_ops;
           struct file_operations * dir_ops;
     }
     */
    fs->dir_ops->mkdir(parent, name);          
    
     child = malloc( sizeof(*child) );      
     child->parent = parent;   // 设置父母
     set_name(child, name);     // 名字
 
     attach_child(parent, child);  // 添加到链表
     
}
```

关键是什么，是调用了 ext2 提供的文件操作函数的指针，这也是为什么一定要切换到 ext2 的根目录的原因所在，就是因为之后对当前目录的所有操作，都是和**具体的文件系统挂钩**的，只有正确切换了，才能实现 VFS

{% hint style="info" %}
VFS  的实现关键就在于，1. 对于路径的搜索需要有机制来处理跨越文件系统的情况  2. 对不同的文件系统提供一个统一的抽象和接口，内核只负责调用接口，这样就可以屏蔽文件系统的存在
{% endhint %}

文件系统的跨越，本质就是为了隔离不同文件系统的区别，抽象出一个通用的接口，因为不同的文件系统有不同的处理方式，就比如上面的 **mkdir**，最终可能是，写一个具体的数据到了硬盘，具体怎么写，都是 file system-specific 的，内核不可能知晓，所以得文件系统本身挂载的时候，就要求文件系统的 操作函数指针 必须得注册在内核，提供一个通用的接口。

实现跨越文件系统，就是在路径搜索的过程，比如上面设置一个 mount point 来判断，虽然实现起来可能会很复杂，比如权限的判断，但是本质绝对一样。

实际上就是面向对象\( OOP \)的思想，内核的 VFS 就是一个 caller ，至于到底怎么操作，那是取决于文件系统本身。

这里仅仅介绍了 **mkdir**，实际上，对于 open，write，read...等操作，都是一个通用的接口。

```c

file_operations->read(x,x,x)/write(x,x,x)


比如我自己写的 read 函数  

ssize_t demo_read(struct file * f, char __user * user_buf, size_t, loff_t * off_t) {
        
        char * s = "wa~ 0.0 ~";
        sprinf(user_buf, s);
        
        return strlen(s);  
}

struct file_oprations demo_fops {
        ...
        .read = demo_read,
        ...
}
```

现在，我们只需要把文件系统给内核，然后挂载到相关的目录下，如此，这个目录下创建的文件，当我们 read 它的时候，必然是返回这句话。

{% hint style="info" %}
 ramfs 意味着在内存上的文件系统，其实就意味着它的 write 并不会真正的写到磁盘上，而只会在内存上，其实我们上面的这个实例函数，也算是，因为我们也没有从什么硬盘读出相关数据。
{% endhint %}

相信到此，读者已经有点理解，其实所谓写一个新的文件系统，就是自己写一个配合内核规范\( VFS调用规范 \)的函数，然后注册到内核，这样内核就可以成功调用。

如果有读者，写过一些 callback 函数的实现，其实很类似，这些思想都是一样的。

## END

具体到了真正 linux 下的实现，非常的复杂，因为经过长时间的演变，有时候为了适应具体的要求，衍生出了非常复杂的数据结构，总体来说有   inode dentry address\_space vfs\_mount等，但是它们的思想和早期没有区别，都是为了隔离，是不同文件系统对于上层的用户是透明的。

我们 read 一个设备文件和一个 proc 文件系统的文件，或者一个真正意义上的文件，处理都是一样的，根本的原因，就是挂载文件系统的时候，注册了与其对应的函数指针，VFS 最终落实到了这些函数的调用。

所以在其他的文章，我也说过了，文件其实只是沟通的媒介，真正的实现是落实在一个具体的函数的。是否要与 块设备沟通，又或者是只是在内存中，都由这个**注册**的函数来决定。

{% hint style="info" %}
建议读者再去看看一个简单的字符设备的驱动实现，那里其实就体现了一个内核封装等面向对象的思想。
{% endhint %}

