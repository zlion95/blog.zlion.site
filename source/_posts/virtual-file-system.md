---
title: Virtual File System
date: 2020-11-10 14:30:54
toc: true
categories: Technology
tags:
- linux kernel
- file system
description: 本文主要用于记录自己对linux文件系统的学习进程和感悟，本文主要集中在vfs这个重要的机制，也许后面看心情会再写一些其他的文件系统.
---

# Linux Virtual File System

当我们再linux使用c的open，write，read，close函数去打开文件，读写文件时，c的库函数会经过最终调用sys_read, sys_write的系统调用，最终交由内核进行同步阻塞或非阻塞读写。文件系统在这个过程中充当了中介的角色，它会帮助我们管理文件，比如找到一个空闲的物理空间，分配给该文件使用，比如在目录中查找一个文件等都是由文件系统实现的。

那么vfs(Virtual File System)是什么？

## 为什么需要vfs层

假设现在我们有一个场景，需要从基于NTFS文件系统的U盘上，拷贝一个文件到基于EXT4文件系统的本地目录下，假如没有vfs层存在，每种文件系统都有自己的实现方式，暴露的接口也不统一，这个拷贝操作就可能发生意料之外的错误。而当有vfs层的存在，即使NTFS和EXT4两种文件系统的存储，管理，读写策略都不一样，也可以通过暴露接口，抽象数据结构的统一，使得相互传输称为可能。

另外，linux的设计理念就是一切皆文件，很大程度上就是依托于vfs层存在的，无论是实际的物理存储设备(块设备文件)，文件系统管理的文件，甚至是虚拟的文件(proc，sysfs)都可以抽象成文件，通过cat，管道重定向之类的操作实现于内核的交互。

## 通用文件模型(common file model)

vfs层的设计思想是通过引入一个通用的文件模型，以最小的额外开销，运行本地文件系统，并且能够表示所有支持的文件系统。(注意：如果对于文件操作，调用系统调用时，不需要涉及到具体的读写文件，比如open，close，lseek这类调用，只需要在vfs层的文件描述上即可完成，并不需要调用底层的文件系统)

通用文件模型由几部分构成：
1. super_block object：超级块，用于存放安装文件系统的相关信息
2. inode object：用于描述文件存放位置的索引结构，inode号唯一标志文件系统中的文件
3. file object：内核描述进程打开文件的数据结构，当文件的引用为0时会自动销毁
4. dentry object：目录项，用于表示文件路径（链接）关系，比如/home/zlion/test.txt这个文件就由/，home，zlion和test.txt几级目录项组成

值得一提的是，vfs除了能实现对所有文件系统的实现提供一个通用接口外，还能对系统性能起到重要作用。如通过将目录项对象缓存到目录项缓存(dentry cache)的磁盘高速缓存中，从而加速从文件路径到最后一个路径分量的索引节点的转换过程。

## vfs层的数据结构

要系统的理解vfs层的机制，就要理解相应的数据结构。

### super block

struct super_block是用于描述文件系统超级块的数据结构，其中比较重要的一些数据域如下：

```c
struct super_block {
	struct list_head s_list;				//文件系统的超级块会通过串联到一个双向链表上，通过sb_lock保护链表不被多处理器同时访问
	dev_t s_dev;							//设备号，其中前12位表示major，后20位表示minor
	loff_t s_maxbytes;						//文件的最大长度

	struct file_system_type *s_type;		//文件系统类型
	const struct super_operations *s_op;	//当前文件系统的超级块操作方法
	const struct dquot_operations *dq_op;	//磁盘限额的处理方法
	const struct quotactl_ops *s_qcop;		//磁盘限额的管理方法
	const struct export_operations *s_export_op;//网络文件系统的输出方法

	unsigned long s_flags;					//文件系统安装设置
	unsigned long s_magic;
	struct dentry *s_root;					//文件系统的根目录

	struct list_head s_inodes;				//该链存放文件系统所有inodes

	struct list_head s_dentry_lru;			//未被使用的dentry
	int s_nr_dentry_unused;					//在lru链上的dentry数目
	spinlock_t s_inode_lru_lock;			//保护s_dentry_lru和s_nr_dentry_unused

	struct list_head s_inode_lru;
	int s_nr_inodes_unused;

	struct quota_info   s_dquot;			//磁盘限额的描述结构

	void *s_fs_info;						//存放文件系统的私有信息
	...
}
```


### inode

```c
struct inode {
	umode_t         i_mode;					//文件类型和访问权限
	unsigned int        i_flags;			//文件系统安装标志
	const struct inode_operations   *i_op;	//索引节点的操作方法
	struct super_block  *i_sb;
	unsigned long i_ino;					//索引节点号
	enum {									//硬链接数
		const unsigned int i_nlink;
		unsigned int __i_nlink;
	}

	unsigned long       i_state;			//索引节点状态标志
	//注意： I_DIRTY_SYNC | I_DIRTY_DATASYNC | I_DIRTY_PAGES 这些标志表示该索引节点是“脏”的，需要同步
	//其他还有 I_CLEAR表示索引节点内容无意义，I_FREEING表示正在释放，I_NEW代表索引节点被创建，但还未从磁盘读取数据来填充
	struct hlist_node   i_hash;
	atomic_t        i_count;				//引用计数
	...
}
```

### file

文件对象描述的是进程与一个打开的文件是如何进行交互的。最重要的是文件指针，用于表明文件的当前位置，以及下次操作的位置。进程之间可以同时访问同一个文件，每个进程打开的文件对应一个struct file对象，可以引用同一个inode。

```c
struct file {
	union {//用于挂接到通用文件对象链表
		struct list_head fu_list;
		struct rcu_head fu_rcuhead;
	} f_u;
	struct path f_path;
	#define f_dentry f_path.dentry;
	struct inode        *f_inode;				//cache value?
	const struct file_operations    *f_op;
	fmode_t         f_mode;						//进程访问文件的模式
	loff_t          f_pos;						//当前文件位偏置(文件指针)
	struct file_ra_state    f_ra;				//文件预读状态
	u64         f_version;						//版本号，每次使用后自增

	struct list_head    f_ep_links;				//eventpoll事件轮询时等待者链表的头
	...
}
```

### file_operation

通过定义文件操作的统一接口描述，使得vfs层对支持的文件系统都可以采用同样的上层调用，主要的操作包括：
1. llseek 更新文件指针
2. read 同步阻塞/非阻塞读取数据
3. aio_read 启动一个异步读I/O操作(配合io_submit()方法使用)
4. write 同步阻塞/非阻塞写数据
5. aio_write 启动一个异步写I/O操作
6. ioctl, unlocked_ioctl, compat_ioctl 其中unlocked_ioctl的区别主要是不需要获取大内核锁，compat_ioctl是用于64位内核执行32位系统调用
7. mmap 执行文件内存映射，将文件cache直接对应到用户态地址空间
8. open 创建一个file对象打开文件，并将其链接到对应的inode对象
9. flush 当打开文件的引用被关闭时，会调用该方法，具体作用取决于文件系统
10. release 释放file对象
11. fsync 将文件缓存的数据全部回刷到磁盘上
12. aio_fsync 启动异步I/O回刷操作
13. lock 对文件申请一个锁
14. readv(file, vector, count, offset) 从文件读字节，并将结果放入vector缓冲区中，缓冲区个数由count指定
15. writev 类似于readv，将vector缓冲区的字节写入文件
16. sendfile(in_file, offset, count, file_send_actor, out_file) 将数据从in_file传送到out_file
17. sendpage(file, page, offset, size, pointer, fill) 将数据从文件传送到高速缓存页

### dentry 目录项

```c
struct dentry {
	unsigned int d_flags;					//dentry告诉缓存标志
	struct dentry *d_parent;				//父目录
	struct lockref d_lockref;				//lockref用于描述per-dentry lock 和 引用的次数
	struct inode *d_inode;
}
```

dentry对象有四种状态：
1. free：空闲状态，该状态下的目录项对象不包含有效的信息，且dentry还没有被VFS使用。(或者说该内存区域正被slab管理)
2. unused：未使用状态，该状态下的目录项对象当前还没有被内核使用。该对象的引用计数为0，但d_inode指向了关联的inode。虽然该dentry对象包含了有效信息，但是由于未被使用，在必要时，有可能会被舍弃以回收内存。
3. in use：正在使用状态，该状态的目录项对象当前正在被内核使用，该对象的引用计数大于0，其d_inode指向了关联的inode。该dentry对象包含了有效信息，正在被引用，因此不能舍弃。
4. negative：负状态，与目录项关联的索引节点不存在(因为相应的磁盘inode已被删除，或者是dentry是通过解析一个不存在的文件路径名创建的)，该状态下d_inode为NULL，而但该dentry仍然被保存在目录项高速缓存中，以便后续查找操作能快速完成。

### 目录项高速缓存

dcache机制可以保证删除，查找文件的快速完成

## 文件系统的类型

vfs除了能对网络和磁盘文件系统抽象，还能将一些特殊的功能通过特殊文件系统的方式导出为文件

### 特殊文件系统

1. bdev 块设备
2. devpts	/dev/pts	伪终端	
3. eventpollfs			事件轮询机制
4. futexfs				由futex(快速用户空间加锁)使用
5. proc		/proc		proc文件，提供内核数据结构的访问机制(一般作为读使用，写操作可用于修改内核参数)
6. rootfs				启动阶段的空根目录
7. shm					IPC共享线性区
8. mqueue				posix消息队列使用
9. sockfs				套接字使用
10. sysfs	/sys		对系统数据的常规访问
11. tmpfs				临时文件
12. usbfs	/proc/bus/usb	USB设备

## 文件系统注册

好了，说了这么多vfs层涉及的数据结构，最关键的文件系统是怎么被vfs使用到的？如何同时兼容多种文件系统呢？

内核模块有两种使用方式：
1. 通过编译内核时将模块配置进去，使得linux启动即可使用
2. 通过模块被动态装入

而vfs需要兼容多种不同文件系统并对其进行跟踪，而且用户也可以通过系统运行过程中挂载某种文件系统的目录，因此文件系统最好通过模块动态装入方式实现最方便。文件系统类型需要注册。

### 文件系统类型描述

文件系统在注册时，会用到file_system_type的对象来描述:

```c
struct file_system_type {
	const char *name;
	int fs_flags;
	/* fs_flags标志有:
	 * FS_REQUIRES_DEV		该类型的任何文件系统必须位于物理磁盘设备上
	 * FS_BINARY_MOUNTDATA	文件系统使用的二进制安装数据
	 */
	void (*kill_sb) (struct super_block *);		//删除超级块的方法
	struct module *owner;						//指向实现文件系统模块
	struct file_system_type * next;				//用于指向下一个文件系统类型
	struct hlist_head fs_supers;				//同一种文件系统类型的超级块对象会放入该hlist上
}
//list_head file_systems;
//				|
//				v
//			file_system_type obj1.next -> file_system_type obj2 ...
//								 .fs_supers
//									 |
//									 v
//								super_block obj1.next -> super_block obj2 ...
```

系统初始化时，会调用`register_filesystem()`函数来注册编译时指定的每个文件系统，并将对应的file_system_type对象插入到file_system链表上。

## Filesystem Handling

### 文件系统安装

传统的类Unix系统只会运行文件系统被安装一次，即只允许挂载到一个路径下；但是linux允许同一个块设备(比如格式化为ext4)被挂载到多个路径下。当然super_block还是使用同一个，即文件系统安装访问点可以有n个，但是使用的时同一个文件系统对象，super_block有且仅有一个。


### vfsmount & mount

文件系统层次可以堆叠，一个文件系统下的路径可以安装另一个文件系统的目录，而第二个文件系统的路径下也可以安装第三个文件系统。
另外，把多个安装堆叠在一个安装点上也是可以的，新安装的文件系统将隐藏前一个安装的文件系统。例如:

```bash
$ mount /dev/sda /mnt
$ mount /dev/sdb /mnt
$ mount /dev/sdc /mnt
```

虽然`/dev/sdc`会隐藏该路径下的其他设备文件系统的挂载，但确实是允许的，只有当覆盖的文件系统卸载后，才能重新暴露之前的挂载文件系统。
而要实现这样的挂载，就需要结构来描述
