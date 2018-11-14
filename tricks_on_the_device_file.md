# open 设备文件全过程（以块设备为例）

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

_mknod_  创建的一类特殊的设备文件，因为一块设备当然是没有大小的说法，但是却有设备号，所以在 Linux 下的文件系统驱动必然就应该提供这个接口，我们来看看ext2 文件系统的相关函数。

```c
static int ext2_mknod (struct inode * dir, struct dentry *dentry, int mode, dev_t rdev)
{
	struct inode * inode;
	int err;

	if (!new_valid_dev(rdev))
		return -EINVAL;
	dquot_initialize(dir);
	inode = ext2_new_inode (dir, mode, &dentry->d_name);
// 这里分配一个新的 inode ， 我们不用关心
	err = PTR_ERR(inode);
	if (!IS_ERR(inode)) {
		init_special_inode(inode, inode->i_mode, rdev);
// 这句是核心，对特殊的 inode 做初始化，即设备文件的inode
#ifdef CONFIG_EXT2_FS_XATTR
		inode->i_op = &ext2_special_inode_operations;
#endif
		mark_inode_dirty(inode);
		err = ext2_add_nondir(dentry, inode);
	}
	return err;
}

```

读者类比 open 便可以猜想到内核是怎么来到 _**ext2\_mknod**_，我们关心的是 _**init\_special\_inode**_ 这个函数，因为传入了设备号，很显然 trick 一定是在这了。_**init\_special\_inode**_ 是 Linux 内核提供的函数，思考一下为什么是这样，因为 mknod 必须得创建 inode，而不同的文件系统创建的 inode 不一样\(  因为需要在硬盘相关位置读写 \)，所以需要文件系统提供 mknod 接口，创建了 inode 之后，文件系统又如何知道这个内核版本的设备文件如何处理呢？所以就使用了**内核**提供的接口，分工明确，各司其职。

下面来揭开这个 _**init\_special\_inode**_ 的神秘面纱

```c
// fs/inode.c:1703:2.6.39.4
void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev)
{
	inode->i_mode = mode;
	if (S_ISCHR(mode)) {
		inode->i_fop = &def_chr_fops;
		inode->i_rdev = rdev;
	} else if (S_ISBLK(mode)) {
		inode->i_fop = &def_blk_fops;
		inode->i_rdev = rdev;
	} else if (S_ISFIFO(mode))
		inode->i_fop = &def_fifo_fops;
	else if (S_ISSOCK(mode))
		inode->i_fop = &bad_sock_fops;
	else
		printk(KERN_DEBUG "init_special_inode: bogus i_mode (%o) for"
				  " inode %s:%lu\n", mode, inode->i_sb->s_id,
				  inode->i_ino);
}
```

读者肯定也不惊讶，最终是内核设置了 inode 的文件操作函数，以块设备为例，**块设备文件创建之后，文件系统完成了 inode 的创建，而内核则是负责赋值了它的操作函数**。这就是 magic 所在拉。

keke，到了敲黑板时间。总结一下，首先 mknod 这一命令最终来到了文件系统提供的接口，文件系统创建了 inode，接着对它的初始化交给了内核，于是乎，内核给它的 inode-&gt;i\_fop 赋值，这便是我们上一节所提及的关键。 

## 设备文件的面纱

下面的关键就是， _**init\_special\_inode**_  所设置的文件操作函数拉。还是以块设备为例子。

```c
// fs/block_dev.c:1592:2.6.39.4

const struct file_operations def_blk_fops = {
	.open		= blkdev_open,
	.release	= blkdev_close,
	.llseek		= block_llseek,
	.read		= do_sync_read,
	.write		= do_sync_write,
  	.aio_read	= generic_file_aio_read,
	.aio_write	= blkdev_aio_write,
	.mmap		= generic_file_mmap,
	.fsync		= blkdev_fsync,
	.unlocked_ioctl	= block_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= compat_blkdev_ioctl,
#endif
	.splice_read	= generic_file_splice_read,
	.splice_write	= generic_file_splice_write,
};
```

当然，我们只需要关注 open 函数，其它都是类似的，读者有兴趣应该自己了解，我们关注的是它的tricks，它的实现思想。

```c
// fs/block_dev.c, line 1402， v2.6.39.4

static int blkdev_open(struct inode * inode, struct file * filp)
{
	struct block_device *bdev;

	/*
	 * Preserve backwards compatibility and allow large file access
	 * even if userspace doesn't ask for it explicitly. Some mkfs
	 * binary needs it. We might want to drop this workaround
	 * during an unstable branch.
	 */
	filp->f_flags |= O_LARGEFILE;

	if (filp->f_flags & O_NDELAY)
		filp->f_mode |= FMODE_NDELAY;
	if (filp->f_flags & O_EXCL)
		filp->f_mode |= FMODE_EXCL;
	if ((filp->f_flags & O_ACCMODE) == 3)
		filp->f_mode |= FMODE_WRITE_IOCTL;

	bdev = bd_acquire(inode);
	if (bdev == NULL)
		return -ENOMEM;

	filp->f_mapping = bdev->bd_inode->i_mapping;

	return blkdev_get(bdev, filp->f_mode, filp);
}
```

需要提醒的一点是，此时的操作都是由内核块设备模块开发那一部分的人提供的，文件系统没有需求，没有必要管这一部分，它只需要提供 _**mknod**_ 接口函数，负责自己的 inode 创建，接着调用内核提供的接口 _**init\_special\_inode**_ 就OK，不同的文件系统在这部分处理都应该是相似的，因为设备结构是其它模块的事情，文件系统只需要调用接口。

如果有非常细心的读者，会发现，上述过程是在设备文件创建的时候才赋值，那么我们关机之后，创建的设备文件节点实际上还在文件系统中，那我们利用的实质其实就变成了文件系统的 open 函数，其实，不难想到的一点，在文件系统打开的函数的时候，必然要实现的一步，就是对 inode-&gt;i\_fops 的赋值，只要检测它是否属于设备节点，如果是，就赋值为 _**init\_special\_inode**_，又回到了上面这一过程。

```c
// /fs/ext2/inode.c:1291
struct inode *ext2_iget (struct super_block *sb, unsigned long ino)
{
	struct ext2_inode_info *ei;
	struct buffer_head * bh;
	struct ext2_inode *raw_inode;
	struct inode *inode;
	long ret = -EIO;
	int n;

	inode = iget_locked(sb, ino);
	if (!inode)
		return ERR_PTR(-ENOMEM);
	if (!(inode->i_state & I_NEW))
		return inode;

	ei = EXT2_I(inode);
	ei->i_block_alloc_info = NULL;

	raw_inode = ext2_get_inode(inode->i_sb, ino, &bh);
	if (IS_ERR(raw_inode)) {
		ret = PTR_ERR(raw_inode);
 		goto bad_inode;
	}
	.....
	.....
	if (S_ISREG(inode->i_mode)) {
		inode->i_op = &ext2_file_inode_operations;
		if (ext2_use_xip(inode->i_sb)) {
			inode->i_mapping->a_ops = &ext2_aops_xip;
			inode->i_fop = &ext2_xip_file_operations;
		} else if (test_opt(inode->i_sb, NOBH)) {
			inode->i_mapping->a_ops = &ext2_nobh_aops;
			inode->i_fop = &ext2_file_operations;
		} else {
			inode->i_mapping->a_ops = &ext2_aops;
			inode->i_fop = &ext2_file_operations;
		}
	} else if (S_ISDIR(inode->i_mode)) {
		inode->i_op = &ext2_dir_inode_operations;
		inode->i_fop = &ext2_dir_operations;
		if (test_opt(inode->i_sb, NOBH))
			inode->i_mapping->a_ops = &ext2_nobh_aops;
		else
			inode->i_mapping->a_ops = &ext2_aops;
	} else if (S_ISLNK(inode->i_mode)) {
		if (ext2_inode_is_fast_symlink(inode)) {
			inode->i_op = &ext2_fast_symlink_inode_operations;
			nd_terminate_link(ei->i_data, inode->i_size,
				sizeof(ei->i_data) - 1);
		} else {
			inode->i_op = &ext2_symlink_inode_operations;
			if (test_opt(inode->i_sb, NOBH))
				inode->i_mapping->a_ops = &ext2_nobh_aops;
			else
				inode->i_mapping->a_ops = &ext2_aops;
		}
	} else {
		inode->i_op = &ext2_special_inode_operations;
		if (raw_inode->i_block[0])
			init_special_inode(inode, inode->i_mode,
			   old_decode_dev(le32_to_cpu(raw_inode->i_block[0])));
		else 
			init_special_inode(inode, inode->i_mode,
			   new_decode_dev(le32_to_cpu(raw_inode->i_block[1])));
	}
	brelse (bh);
	ext2_set_inode_flags(inode);
	unlock_new_inode(inode);
	return inode;
	
bad_inode:
	iget_failed(inode);
	return ERR_PTR(ret);
}
```

这篇文章到这里就快结束了，上面的代码最关键的函数就是  22行，涉及了具体的块设备相关函数，这篇文章的强调的是以下几点。

* 文件是这是一个沟通的媒介，用户打开文件，内核实际上做的事情很简单，就是从文件对应的 inode 那里拿到处理函数（ file\_operations ）, 接着初始化。
* 文件系统对于设备文件的支持也很容易实现，常规文件的需要文件系统提供处理函数，为什么？因为一般文本文件内容的分布是和特定的文件系统息息相关的，而设备文件不需要，因为设备文件的内容不由文件系统的决定。
* 对于设备文件的处理最终是由内核的相关设备模块提供，因为不同的 块/字符 设备文件处理方法都不一样，所以需要相关的来提供。最终，def\_blk/char\_fops 应运而生。

{% hint style="info" %}
对于第二点，我们举个例子，当我们读 /dev/sda 的时候，读出来的内容一般有 分区表，文件系统超级块等结构，还有文件系统里面的文件，当然，一般的文本文件内容是分散的，所以我们没办法知道 /dev/sda 有哪些文本文件。

文件系统在初始化的时候，直接读写的就是 /dev/sdax ，也就是这个块设备的分区，之后它能识别上面的文件，为什么？因为是它写进去的，它对于每一个文件的分布都有记录，当然这些记录也在硬盘上，请参考下面那张图。

我们读 /dev/sda 的时候，也可以读出这些记录，但是我们不了解，只有特定的文件系统了解。

所以得出结论，/dev/sda 的内容和特定的文件系统无关（不由文件系统决定），但是一般文本文件和文件系统息息相关。
{% endhint %}

![&#x5757;&#x8BBE;&#x5907;&#x6587;&#x4EF6;&#x548C;&#x6587;&#x4EF6;&#x7CFB;&#x7EDF;&#x7684;&#x5305;&#x542B;&#x5173;&#x7CFB;](.gitbook/assets/image%20%288%29.png)

最后以一张图结束本文。

![](.gitbook/assets/image%20%282%29.png)

