---
description: 上节简单阐述了 UEFI 的动机以及优势，这次我们来关注它的引导过程
---

# UEFI 的引导过程

先来了解一下传统的 BIOS 引导过程，在设置页面我们会选择引导的介质，例如是哪个盘，选择之后，那个盘里的 MBR 在 BIOS 完成自检和初始化设备之后 就会被读到 0x7c00 的位置，接着一切都是操作系统的事情了，BIOS 只会提供几个中断，给操作系统一些设备的基本信息。

{% hint style="info" %}
所谓修改什么优先级，其实就是在一个数组里面的位置，而这个数组存储在断电也不会消失的内存上，所以我们修改之后关机之后依然有效
{% endhint %}

我们说了，这种方式并不好，尤其是对多操作系统的支持，其次当操作系统出错的时候，是更换一个文件来的简单，还是更换磁盘的前 512 字节容易呢？ 而且 MBR 式的引导往往是占用的不止是512字节，只是BIOS 只会读512字节，像最早期的 linux，全部都放在硬盘的最前面。而且 MBR 里面还有分区信息，更换的时候还得保证分区信息保留，也是一件麻烦的事情，而 UFEI 下的引导就没有这个问题。

首先， UEFI 摈弃了 MBR 的引导，并且把 MBR 的部分变成保留的内容（ 很明显，是为了向后兼容 ），换言之，现在硬盘的第一个扇区是被保留，是不使用的。

```bash
$ gdisk /dev/sda
    output: 
              MBR: protective  # MBR is not used in UEFI 
              BSD: not present
              APM: not present
              GPT: present

```

GPT 是一种分区表，也是 UEFI 引导的核心，我们先放着，这不影响我们从整体上理解 UEFI 的引导过程。首先，UEFI 的核心在于有一个分区，我们称之为 ESP（EFI  System Partition），可以把 EFI 理解成 UEFI，区分它们是没有意义的，也就是说有一个 ESP 分区，里面存放的就是可以引导的文件，我们操作系统的引导文件全部存放在这里。

```bash
$  ls /boot/efi  # 可以看见有一个 EFI 文件
    output: /EFI
$  cd /EFI
$  ls
    output: /BOOT /Microsoft  /centos
```

可以看出，/boot/efi 下挂载的就是 ESP 分区，里面的 EFI 是一个目录文件，所有的操作系统都会在里面建立一个属于自己操作系统目录，然后在这个目录里面存放自己的引导文件，然后增加全局变量 BOOT\#\#\#\# ，这个全局变量是保存在非易失存储上，也就是修改了在断电之后仍然生效。\# 代表16进制数。

```bash
$ efibootmgr -v  # linux 下 efibootmgr 命令可以修改这些变量，本质利用的还是EFI提供的接口 
    output(删除了部分冗余输出)
    
    BootCurrent: 0003
    Timeout: 2 seconds
    BootOrder: 0003,0000,2003,2001,2002    
    Boot0000* Windows Boot Manager	HD(1,GPT,6fca7b42-3edd-4559-b75b-b8ef45159130,0x800,0x64000)/File(\EFI\Microsoft\Boot\bootmgfw.efi)
    Boot0003* CentOS Linux	HD(1,GPT,6fca7b42-3edd-4559-b75b-b8ef45159130,0x800,0x64000)/File(\EFI\centos\shim.efi)
    Boot0004* EFI Network 0 for IPv4 (C8-5B-76-C9-4E-A5) 	PciRoot(0x0)/Pci(0x1c,0x0)/Pci(0x0,0x0)/MAC(c85b76c94ea5,0)/IPv4(0.0.0.0:0<->0.0.0.0:0,0,0)RC
    Boot0005* EFI Network 0 for IPv6 (C8-5B-76-C9-4E-A5) 	PciRoot(0x0)/Pci(0x1c,0x0)/Pci(0x0,0x0)/MAC(c85b76c94ea5,0)/IPv6([::]:<->[::]:,0,0)RC
    Boot2001* EFI USB Device	RC
    Boot2002* EFI DVD/CDROM	RC
    Boot2003* EFI Network	RC
```

输出还是挺清晰的，我们先来关注一些重点，首先是 BootCurrent ，表示的是现在正在使用的引导程序的序列号，也就是当前使用的操作系统，对比下面的 Boot 信息，可以知道我当前使用的操作系统是 CentOS，具体我们还是看 manual 所写的比较好，因为笔者的翻译水平实在有待提高。

> ```text
>           · BootCurrent - the boot entry used to start the currently running system
>           
>           · BootOrder - the boot order as would appear in the  boot  manager.
>             The  boot  manager  tries  to boot the first active entry in this
>             list.  If unsuccessful, it tries the next entry, and so on.
>
>           · BootNext - the boot entry which is scheduled to be  run  on  next
>             boot.   This  supercedes  BootOrder  for  one  boot  only, and is
>             deleted by the boot manager after first use.  This allows you  to
>             change the next boot behavior without changing BootOrder.
>
>           · Timeout  -  the  time  in  seconds  between when the boot manager
>             appears on the screen until when  it  automatically  chooses  the
>             startup value from BootNext or BootOrder.
>
>           · Five  boot  entries (0000 - 0004), along with the active/inactive
>             flag (* means active) and the name displayed on the screen.
> ```

也就是说，我们在 UEFI 的设置页面调整的引导顺序，也就是修改了 bootorder 的顺序，那里展示的名字，也就是在 BOOT\#\#\#\# 里面记录的信息，其实，BOOT\#\#\#\# 是一个结构体，其中有段字符数组，存放的就是它的名字，想要一探究竟的朋友可以看 [SPEC](http://www.uefi.org/sites/default/files/resources/UEFI%202_5.pdf#page=536) 。 注意到，这个变量还存放了文件路径，这就是UEFI 要运行的程序，当其根据先后顺序选择一个引导项之后，就打开这个路径的文件然后运行。

这个过程简述如下，首先 UEFI 先查看 BootOrder ，选择第一个 0003 ，然后查看 BOOT0003，根据它提供的文件路径名，运行 ESP 里面的 /EFI/centos/shim.efi ，这是一个运行文件，然后这个程序永远都不会返回，因为它是操作系统，如果它失败返回了，那么从 BootOrder 里面选择下一个，也就是我的 Win 系统来运行了，这也解释了我的操作系统引导出错的时候为什么会跳到 Windows 运行。当然，实际的过程很复杂，变量还有很多含义，比如要运行的驱动等等，但是这就是一个整体的框架。

```bash
$ ls /boot/efi/EFI/centos/ 
   output:
     BOOT.CSV     fonts     grub.cfg.bak  grubx64.efi  shim.efi            shimx64.efi
     BOOTX64.CSV  grub.cfg  grubenv       mmx64.efi    shimx64-centos.efi
```

是不是没有想象中的那么复杂？ 没错， UEFI 总体的设计是很干净的，现在我们来思考一个问题，我们这些文件都是放在硬盘里面，然后向 UEFI 注册 \( 也就是修改相关变量 \)，现在我把我的硬盘拔了，换上别人的硬盘，或者我插上U盘，准备用它来引导，问题就出现了，我的计算机没有可能事先知道我的U盘，然后还给它注册一个变量把。

我们注意到，在 EFI/ 目录下还有一个 BOOT 文件，不属于任何操作系统，它就是我们称为 backup 的引导，当所有在之前的引导都失败，比如文件不存在，或者失败返回，UEFI 就会引导 /EFI/BOOT/BOOTx86.EFI 文件，这相当于是一个固定路径，我们 U 盘如果是作为 UEFI 引导盘，打开 U 盘必然能发现 EFI/ 目录，打开之后也必然能发现 BOOT/ 目录，这就是 U 盘作为引导盘的神秘之处。

{% hint style="info" %}
部分读者用 Windows 系统可能发现找不到，这可能是因为你们用了引导制作软件给你分区，ESP 分区在 Windows 下默认是隐藏的，只要在 Linux 的环境下，就能看见全部分区
{% endhint %}

也就是说，当我们选择的别的引导媒介的时候，UEFI 直接读它 ESP 分区下 /EFI/BOOT/BOOTx86.EFI 文件运行就好了，是不是非常简单的设计，平时我们的硬盘的这个文件也是当其他文件都失败的时候作为备用来使用的。

总结以下，由于 UEFI 可以识别文件系统，操作系统直接把引导文件放在 ESP 分区下，然后创建全新的 BOOT\#\#\#\# 变量里面存放了文件路径，接着修改 BootOrder ，这样，UEFI 就可以识别到这个操作系统，进而选择它来引导。双系统只需要简单的创建两个变量，两个目录就好了，彼此之间互不干扰。

读者可能会提问，UEFI 到底是怎么识别到哪一个分区是 ESP 分区的，在多分区的硬盘，比如我的硬盘有6个分区，那么 UEFI 是如何知道的呢，想要知道这个就需要了解 GPT  分区，我会在下一节讲解。

