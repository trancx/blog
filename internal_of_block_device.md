---
description: def_blk_fops 到底如何实现从一个 inode 到 实实在在的块设备结构呢，这次我们就来揭开它的面纱
---

# 块设备文件

上一节说到了了，def\_blk\_fops 之后就没有继续讲述，因为严格来说，这里属于其他模块了，对于不想了解块设备的读者来说，知识覆盖已经足够了。而这里，则是它的另一延续，可以衍生到块设备和字符设备，当然，两者实现其实很相似。

```c

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

就从这个结构的 open 字段赋的值开始看起吧， blk\_open 必然要实现的转换，就是从内核 VFS 规定的接口，到自己的块设备结构，这里提一句，这些结构实现的就是 C++ 的继承，结构的定义实际上就是父类，或者说是接口，而后面的重写，赋值，其实就是 override，实际上 C++ 就是这么实现的，所谓 组合（Composition） 其实就是内嵌，而多态性（Polymorphism） 就是函数指针。C 语言同样是面向对象的，OOP 只是一种思想。

```c
// v2.6.39.4 fs/block_dev.c:1380
 
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

_**struct block\_device**_ 就是内核对块设备的一个抽象，bd\_acquire\(\) 顾名思义，就是从 inode 得到了块设备，这就是我们说的 _trick_ 所在。首先， open\(\) 函数原型是内核规定的，它的参数不能改变，但是我们块设备又必须得到块设备，**而且是从现有的参数里面得到**，sturct file 是根据 inode 创建的，而 inode 是根据我们 mknod 创建的，所以必须在 inode 结构里面存放一些参数，让内核可以通过它来找到跟这个 inode 对应的块设备。

```c
// 最简单想到的办法 直接 inode 里面存放一个指针
struct inode {
    ...
    struct block_device * i_bdev; 
}
//  实际上，内核就是这么做的！ 是不是很惊讶 
```

但是有个问题，我们在创建一个块设备的节点的时候，好像没有看见对这个字段赋值，原因是什么呢？原来，大部分初始化都是在打开的时候才做的，这就是典型的**懒惰策略，**就是不到最后一刻不做初始化，因为资源很珍贵，如果你一直不用，那我又自作多情呢。block\_device 结构只有当你打开一个块设备文件的才会初始化，也就是说在我们 mknod 的时候这一指针还没有被赋值。

```c
static struct block_device *bd_acquire(struct inode *inode)
{
	struct block_device *bdev;

	// 如果 bdev 字段非空，我们增加计数就可以退出
	spin_lock(&bdev_lock);
	bdev = inode->i_bdev;
	if (bdev) {
		ihold(bdev->bd_inode);
		spin_unlock(&bdev_lock);
		return bdev;
	}
	spin_unlock(&bdev_lock);

	// 这是块设备第一次打开的情况，还没有初始化 bdev 
	bdev = bdget(inode->i_rdev);
	if (bdev) {
		spin_lock(&bdev_lock);
		if (!inode->i_bdev) {
			/*
			 * We take an additional reference to bd_inode,
			 * and it's released in clear_inode() of inode.
			 * So, we can access it via ->i_mapping always
			 * without igrab().
			 */
			ihold(bdev->bd_inode);
			inode->i_bdev = bdev;
			inode->i_mapping = bdev->bd_inode->i_mapping;
			list_add(&inode->i_devices, &bdev->bd_inodes);
		}
		spin_unlock(&bdev_lock);
	}
	return bdev;
}
```

bget\(\)  算是核心了，如何从一个 inode 构建一个块设备结构，想想都觉得刺激，难吗？其实仔细想想，inode 里面储存了设备号，这个设备号在当前操作系统内核内核是唯一的，也是设备驱动在挂载的时候给设备注册的，最简单的方法就是建立一个数组，设备号作为索引，马上就能得到这个结构。

```c
struct block_device *bdget(dev_t dev)
{
	struct block_device *bdev;
	struct inode *inode;

	inode = iget5_locked(blockdev_superblock, hash(dev),
			bdev_test, bdev_set, &dev);

	if (!inode)
		return NULL;

	bdev = &BDEV_I(inode)->bdev;

	if (inode->i_state & I_NEW) {
		bdev->bd_contains = NULL;
		bdev->bd_inode = inode;
		bdev->bd_block_size = (1 << inode->i_blkbits);
		bdev->bd_part_count = 0;
		bdev->bd_invalidated = 0;
		inode->i_mode = S_IFBLK;
		inode->i_rdev = dev;
		inode->i_bdev = bdev;
		inode->i_data.a_ops = &def_blk_aops;
		mapping_set_gfp_mask(&inode->i_data, GFP_USER);
		inode->i_data.backing_dev_info = &default_backing_dev_info;
		spin_lock(&bdev_lock);
		list_add(&bdev->bd_list, &all_bdevs);
		spin_unlock(&bdev_lock);
		unlock_new_inode(inode);
	}
	return bdev;
}
```

很多内核代码都是这样，你想到了问题是什么，就会觉得解决的方法是非常容易得到的，只是可能有出于其他的考虑，会和你想的有点出入，但是思想不会改变。内核代码并不是有什么神奇的地方，只是有时候实现的非常隐蔽，让人望而生畏。

看上面的代码，利用设备号生成了一个哈希值，得到了一个 inode？ 这里读者肯定是觉得奇怪了，为什么要从一个 inode 得到另外一个 inode，内核把块设备结构都抽象成了一个文件，然后利用 bdevfs 来管理它们。简单点理解就是，一个 block\_device 同时也是 bdevfs 的 inode。

好了，回到正题，回忆一下 init\_special\_inode 都做了什么事，当然我会贴出代码，翻过去实在是太累了。

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

发现了吗，最特别的复制了 rdev 这个值，在 Linux 下设备都有设备号，分为 major 和 minor ，其实我们可以认为就是一串数字，它只是用于区分每一个设备，现在又要考你了，如何从一串数字到一个数据结构呢？我们学的哈希，学的数组，是不是都可以，只要想办法把数字作为索引是不是就可以了？最笨的办法就是建立一个设备号为下标的全局数组，那么直接用设备号就得到目标数据结构了。

内核实现的机制就是一样的， bd\_acquire\(\) 就是一个映射的过程，如果是块设备实现的就是 这个设备号到块设备的结构的过程啦，当然后面读者会发现有点不一样，稍后我们就知道了，现在来揭开这个关键函数的面纱。



