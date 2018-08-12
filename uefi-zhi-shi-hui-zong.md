# UEFI 知识汇总

好的资料可以起到事半功倍的作用，我这里附上一些网址，也是我写作的时候参考的材料，因为在gitbook 的脚注我还没有摸清楚原理，所以只能全部汇总到一篇文章来。

[EFI 的基本概念](http://www.rodsbooks.com/efi-bootloaders/principles.html)， 这篇文章是我最早接触到UEFI的材料，Rod Smith 是 Linux 下几个引导程序的维护人员，对 EFI 有着深刻的认识，这是他所写的系列文章的第一篇，整个系列都值的阅读，我曾经多次想要翻译成中文，无奈翻译水平有限。

[UEFI 和 BIOS的 区别](https://www.howtogeek.com/56958/htg-explains-how-uefi-will-replace-the-bios/)，也是我第一篇文章的标题，觉得我写的不够清楚的读者可以阅读这一篇文章。

[UEFI 引导](https://www.happyassassin.net/2014/01/25/uefi-boot-how-does-that-actually-work-then/)，好文章，适合事先完全不了解引导的读者阅读，如果我早期能看见这篇文章，想必也会事半功倍。

[GPT Wiki 百科](https://en.wikipedia.org/wiki/GUID_Partition_Table)， [UEFI Wiki 百科](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)， 维基是真正的百科，概念准确并且做到了比大部分专业人员的人员都写的要好，这也是开源的好处，多人参与必然有好的成果。

[UEFI Spec 2.5](http://www.uefi.org/sites/default/files/resources/UEFI%202_5.pdf#page=536)，最重要的参考资料，当然这是学习 UEFI 编程的朋友必须要阅读的指南了，如果只想了解他的大概功能，可以选读一部分，另外，可能因为我比较笨的原因，无论我是读 RFC 还是这类 指南，总是觉得写的比较难懂，反倒是有些文章介绍的很不错。

接下来的文章是有关于 UEFI 在 Linux 下引导的过程。

[Redhat EL7 手册](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/installation_guide/s1-boot-init-shutdown-process)，红帽的手册一直都是好资料的来源，感觉都是程序员耐着性子，在编写代码之后还认真写的一部手册，这里简述了引导的过程，是理解 UEFI 引导的资料，并且跟 BIOS 有区分。

[EFI 引导过程](https://jdebp.eu/FGA/efi-boot-process.html)，中间提及了几个 NVRAM 变量的作用。



