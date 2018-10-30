# open 设备文件全过程

## Preface

不知不觉接触 Linux 已经两年了，一直很想总结一下读源代码的笔记，但是总是看见网上分享的博客比自己写的好了不知道多少，于是一直都没有动笔，直到自己真正开了博客，就希望写一下别人没有提及的部分，或者是提及的比较少但是比较关键的点。

现在想来，博客应该是记录自己的笔记的一个所在地，记录自己学习的过程，于是这篇文章就这样诞生了，去年我是重点在 VFS 和 设备驱动 这一部分下了功夫，那段时间学到的东西真的好多，今年想起来却发现有些细节已经不记得了，于是干脆把现在还能记得的部分都记录下来，当然我是回去再看了一遍的，所以现在的映像还是比较深刻的。

## open\(\) 的过程

写这篇博客的时候，我最开始的想法从底至上，就是从设备文件被创建那里开始，不过这样感觉不会太直观，所以还是从顶往下的顺序开始讲述把。因为的我们的标题就是指设备文件，所以我们就以下面这句代码开始把。

```c
open("/dev/sda" ...);  //这里省略了参数，一般是 READ_ONLY
```

这里不会涉及具体的函数的调用过程，因为其实调用的过程随着内核版本的变更，变化还是不小，但是实质通过函数指针实现的多态性，这一点是一点都没有改变呀。

简单介绍一下这个函数的调用过程把，首先这句代码会触发软中断，然后参数存入寄存器，来到了内核态，接着根据我们调用的参数，设置路径查询的参数，比如，我们在打开一般文件的时候可以指定，如果文件不存在就创建 等参数。当然，我们设备文件一般是内核自动为我们创建的哟，以前还得自己手动。

这个函数最关键就是根据路径名最终找到了属于它的 inode，这其实不难想到对把，最简单的办法就是连成一棵树，然后根据路径名搜索就是。inode 是我们在加载这个块设备驱动的时候为它创建的，inode 决定了 read write open 等等一系列操作的流程，因为它提供了一个函数指针。

```c
struct inode {
	...
	const struct inode_operations	*i_op;
	const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
	...
};
/* 
inode 结构其实真正决定了一个文件到底是怎么操作的，它提供的 i_fop 
最终是要赋值给 file 结构的，这里我假设读者熟悉这个结构，因为要面面俱到
那怕是都将不清楚，总之 这个地址赋值给 file->f_ops->open() 
那么就是真正的 open 函数了
*/

// 对于目录 还有 readdir 功能 总之 目录也是一个文件
struct file_operations {
	...
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	int (*readdir) (struct file *, void *, filldir_t);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	...
};

// 
struct inode_operations {
	...
	int (*create) (struct inode *,struct dentry *,int, struct nameidata *);
	struct dentry * (*lookup) (struct inode *,struct dentry *, struct nameidata *);
	int (*link) (struct dentry *,struct inode *,struct dentry *);
	int (*unlink) (struct inode *,struct dentry *);
	int (*symlink) (struct inode *,struct dentry *,const char *);
	int (*mkdir) (struct inode *,struct dentry *,int);
	int (*rmdir) (struct inode *,struct dentry *);
	int (*mknod) (struct inode *,struct dentry *,int,dev_t);
	int (*rename) (struct inode *, struct dentry *,
			struct inode *, struct dentry *);
	...
};
```

请读者认真看看上面的注释，_**file\_operations**_ 这个结构决定了一个文件具体操作的过程，而内核只需要拿到指针，负责调用，其实就是一种 OOP 的思想。好奇心强的读者想必会问，那么 inode 的 i\_fop 的值是如何得来的，那么又得知道 inode 是如何创建的了，决定 inode 如何创建的是文件系统的超级块，即 _**super\_block**_

```c
struct super_block {
    ...
    struct super_block_operations * sbo;
    ...
};

struct super_operations {
   	struct inode *(*alloc_inode)(struct super_block *sb);
};
```

是不是很相似，超级块决定了inode 如何创建，那么好奇的读者肯定又会问，那么超级块如何创建，答案就是文件系统驱动的编写者，调用内核创建函数，然后自己写一个 _call\_back_，让内核完成常规操作之后，调用 _call\_back_， _call\_back_ 函数最关键的一部必然就是设置 _**super\_operations**_ 的值为自己编写的结构了。

当然，赋值给 _alloc\_inode_  函数指针的函数最关键的一步，就是在创建 inode 的时候，将 i\_fop 设置为自己编写的函数，这样是不是全部能联系在一起了。这里关键思想就是，_“写函数，然后把函数的地址给内核，接着内核调用”。_ 

总结一下，以打开 "/dev/sda" 为例子，那么 open\(\)，然后经过路径搜索，找到了 /dev/sda 的 inode，它的字段 i\_fops，决定了open 的具体操作， 而 i\_fops 是文件系统编写者写，然后注册到内核当中，当我们新建一个inode的时候就会被赋值，也就是创建一个新文件的时候，那个 inode 就会被创建，接着赋值。这里其实涉及了文件系统的编写了，总之我们得留下一个映像，**内核不知道文件怎么打开，它只是调用文件系统提供的功能，这就是 VFS 的实质**。

也就是说， open 函数，其实就是调用了驱动编写者提供的 open 函数？那么问题来了，这里是文件系统提供的函数对把，那么块设备文件可指不定创建在哪个文件系统上\( 可能 ext2 xfs ext4 minix ...\)，而且软盘，硬盘，cd 都是块设备，一个文件系统驱动又怎么可能自带块设备节点的处理呢？ 所以这些功能，是不是应该由块设备的编写者来提供呢？ 那么一个文件系统又怎么帮助一个设备文件找到真正属于它的 _**file\_operations**_ ，那就看看一个设备文件到底怎么被创建的把。

## 从设备文件被创建开始

这篇博客一直都建立在一个前提上，就是读者对于 VFS 有一些基础的认识，比如 open\(\) 这一系统调用，_**file**_ 结构，然后最终因为它们 inode 的 f\_ops 指针不一样，所以导致执行的操作不一样等等，我们重点关注的是这些指针是如何被赋值到 file 结构上的，以及为什么根据设备名能最终找到不同的设备，这些 tricks 是真正的重点。

冤有头，债有主。想知道设备文件读写的全过程，我们必须得知道这个文件是**如何被创建的**。这里先提一点，可能新手对于设备能被抽象为文件很是不解，我当初也是如此，但是你得知道所谓文件，就是你的读写最终是和磁盘沟通，所以可不可以理解成我们之前理解的文件，其实就是硬盘的存储空间呢？所以是不是也是设备文件，所以啊，其实所有的文件都可以理解为设备文件（先不考虑管道这些通信文件），随着学习的深入，我们会发现其实文件充当的作用只有一个，那就是作为一个**沟通的媒介**。

那么设备文件到底怎么创建呢。Linux 终端下我们可以 _**mkdir touch mknod ln**_ 等等指令，其实归根到底都是 _**inode\_operations**_ 提供的。

```c
struct inode_operations {
	...
	int (*create) (struct inode *,struct dentry *,int, struct nameidata *);
	struct dentry * (*lookup) (struct inode *,struct dentry *, struct nameidata *);
	int (*link) (struct dentry *,struct inode *,struct dentry *);
	int (*unlink) (struct inode *,struct dentry *);
	int (*symlink) (struct inode *,struct dentry *,const char *);
	int (*mkdir) (struct inode *,struct dentry *,int);
	int (*rmdir) (struct inode *,struct dentry *);
	int (*mknod) (struct inode *,struct dentry *,int,dev_t);
	int (*rename) (struct inode *, struct dentry *,
			struct inode *, struct dentry *);
	...
};
```

设备文件就是调用 mknod 创建，跟之前 open 类比，我们知道其实也是文件系统提供的函数，看看这个函数的参数，其中 dev\_t 就是设备号，这里不详细介绍，我们在查看设备文件有一字段就是设备号，是在 mknod 的时候指定的。

```text
[trance@centos ~]$ ls -l /dev/sda
brw-rw----. 1 root disk 8, 0 Oct 30 17:25 /dev/sda

```

以 ext2 文件系统为例子，我们来看看它的 _mknod_ 的函数，关键得知道，究竟文件系统给设备文件的 inode 的 i\_fops 施加了什么魔法，首先我们得确定一点，就是的确是 ext2 文件系统给 inode 赋值了文件操作的函数，因为用户在/dev 目录下 调用的 mknod 必然就是由这个文件系统来指定操作。

