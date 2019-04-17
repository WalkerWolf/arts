# 嵌入式Linux驱动学习四：字符设备

由于Linux中`一切皆文件`的思想，字符设备(键盘/鼠标/串口)、块设备(存储器)、网络设备(套接字)作为三种主要设备类型，我们在用户层基本可以通过`open close read write`来完成对设备的绝大部分操作。其中字符设备是涵盖面最广的一种，而且其概念相对容易理解，因此以字符设备为切入口，掌握内核与用户层之间的调用机制是再合适不过了。

**本章主要学习**：

- 字符设备驱动的基本概念
- 如何向内核注册/注销一个字符设备
- 如何通过设备节点、文件API调用驱动的函数
- 驱动的内存管理
- udev自动创建设备节点

## 什么是字符设备？

所谓字符设备，泛指那些输入输出以串行方式、按顺序、一个字符一个字符的收发数据的设备。比如键盘，按键数据是一个一个发给系统的，哪怕是组合键；鼠标的坐标；LED灯；串口终端等。而这些数据如果不能及时被系统处理，要么被缓存，要么直接丢弃，但不论如何，当系统去处理这些数据的时候，也是按先来后到的顺序。

你可以设想以下，你的电脑突然卡顿了5秒钟，而这期间你疯狂打了10个字，毫无疑问这10个字的数据被缓存起来了，等系统空闲的时候开始处理它们，现在你是希望这10个字是按照你打的顺序出现在屏幕呢，还是希望它们毫无章法地出现在屏幕。

所以，字符设备，就是数据的IO总是以连续的形式处理的设备类型。

顺带提一句块设备，与字符设备概念做个对比。比如硬盘：你和你的右手出去约会并拍了很多照片回来，然后你把照片导入电脑，而硬盘的存储空间并不是像水杯一样从下往上按顺序填满，而是根据某种算法“随机”存，所以在真实的硬盘中，存放你1号照片的存储区域旁边，并不是2号照片，可能是某份日语学习资料；同理，3号照片可能与去年下载的种子呆在一起。


## 第一个字符设备 meme

在操作系统中，设备不一定要真实存在，虚拟设备也是设备，可以通过文件函数进行操作。因此为了掌握字符设备的知识，最简单的方法就是自己动手写一个虚拟字符设备。好了，现在给这个虚拟设备取个名字——meme。

meme的中文意思是米姆、莫因，表示文化DNA的意思，而作为我学习驱动的实例，它就是“记事本”，用户可以用多种方式读写它，而我给这个虚拟设备取这么个名字没别的意思——just for fun！

先来看看我打算要meme实现哪些内容：
1. 向内核注册/注销自己
2. 支持文件操作open/close/read/write
3. 支持内存映射mmap
4. 支持跳转和控制seek/ioctl

### 字符设备的注册与注销

要像内核注册一个设备，必须提供它的设备号作为唯一标示，字符设备的设备号由**主设备号**和**次设备号**共同组成
- 主设备号(major)：表示设备的分类编号，比如1代表u盘、2代表显示器、3代表打印机
- 次设备号(minor)：表示同一分类的实例编号，比如1号u盘、2号u盘、3号u盘

主设备号是不允许有重复的，也就是说如果123号设备类型已被某个驱动申请过了，如果你的驱动还拿这个号申请注册，肯定会失败。好在Linux内核可以**静态注册**和**动态分配**的方式注册一个设备号：

```c
// 动态申请，由内核自动扫描空闲设备号给它
// @dev         设备编号
// @firstminor  次设备号的起始编号
// @count       要申请多少个连续的设备号
// @name        设备名称，将出现在/proc/devices中
int alloc_chrdev_region(dev_t *dev, unsigned firstminor, unsigned count, const char *name);

// 静态注册，必须确保该设备号是空闲的
// @first   要注册的起始设备号，主+次
int register_chrdev_region(dev_t first, unsigned count, const char *name);
```

⚠️注意⚠️：在实际开发过程中，强烈建议仅通过**动态分配**的方式注册设备，因为说不准你自己规划好的设备编号会在未来被Linux内核给占用了。

理解设备号的注册规则后，就实际动手完善meme驱动代码，向内核动态注册设备号，并把自己作为字符设备添加到内核当中：

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/cdev.h>

// 字符设备结构体，该字符设备的核心数据表示
static struct cdev cdev;

// 驱动的主设备编号，每种类型的设备都有唯一编号
static dev_t devno = 0;

// 模块加载函数：注册字符设备
static int __init meme_init(void)
{
    // 1. 向内核申请分配一个主设备号，此设备号从0开始，分配1个，名称为meme
    if (alloc_chrdev_region(&devno, 0, 1, "meme") < 0 ) {
        printk(KERN_ERR"init fail\n");
        return (-1);
    }

    // 2. 初始化cdev结构体，并与fops文件操作之间建立联系
    cdev_init(&cdev, &fops);

    // 3. 正式向内核注册一个字符设备
    cdev_add(&cdev, devno, 1);

    // 注册成功后打印出该设备的主、次设备号
    printk(KERN_ALERT"meme init: %d:%d\n", MAJOR(devno), MINOR(devno));
    return 0;
}
module_init(meme_init);


// 模块卸载函数：注销字符设备
static void __exit meme_exit(void)
{
    // 1. 向内核注销该字符设备
    cdev_del(&cdev);

    // 2. 向内核申请释放该设备号
    unregister_chrdev_region(devno, 1);

    printk("meme free\n");
}
module_exit(meme_exit);

// 杂七杂八的东西
MODULE_LICENSE("GPL v2");
MODULE_AUTHOR("Philon");
MODULE_DESCRIPTION("A character device driver");
```

上边的代码演示了完整的字符设备从注册到注销的过程，通常情况下字符设备在模块加载时注册，在模块卸载时注销。在代码中主要围绕两个数据结构进行操作`cdev`和`dev_t`，前者是字符设备的“对象”，后者就是个整型数字(设备号)。而注册/注销的过程也非常简单：

申请设备号 --> 初始化字符设备 --> 内核添加字符设备 --> 内核删除字符设备 --> 释放设备号

关于`cdev_init cdev_add cdev_del`等函数没有展开说明是因为后续有更好的方法，不要拘泥于此。不过注意`cdev_init(&cdev, &fops)`这行，fops是接下来要学习的一个非常重要的数据结构——**文件操作**。

### 字符设备的文件操作

因为Linux的一切皆文件，理论上在用户层任何设备都可以通过文件相关的API函数调用设备，而作为设备驱动本身，很重要的一部分工作就是把用户层的各种文件操作关联到各种驱动函数当中，而这离不开一个结构体`struct file_operations`，先来看看内核对其定义：

```c
struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
        ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
        ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
        int (*iterate) (struct file *, struct dir_context *);
        int (*iterate_shared) (struct file *, struct dir_context *);
        __poll_t (*poll) (struct file *, struct poll_table_struct *);
        long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
        long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
        int (*mmap) (struct file *, struct vm_area_struct *);
        unsigned long mmap_supported_flags;
        int (*open) (struct inode *, struct file *);
        int (*flush) (struct file *, fl_owner_t id);
        int (*release) (struct inode *, struct file *);
        int (*fsync) (struct file *, loff_t, loff_t, int datasync);
        int (*fasync) (int, struct file *, int);
        int (*lock) (struct file *, int, struct file_lock *);
        ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
        unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
        int (*check_flags)(int);
        int (*flock) (struct file *, int, struct file_lock *);
        ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
        ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
        int (*setlease)(struct file *, long, struct file_lock **, void **);
        long (*fallocate)(struct file *file, int mode, loff_t offset,
                          loff_t len);
        void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
        unsigned (*mmap_capabilities)(struct file *);
#endif
        ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
                        loff_t, size_t, unsigned int);
        loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
                                   struct file *file_out, loff_t pos_out,
                                   loff_t len, unsigned int remap_flags);
        int (*fadvise)(struct file *, loff_t, loff_t, int);
} __randomize_layout;
```

乍一看，很复杂！