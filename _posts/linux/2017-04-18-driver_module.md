---
layout: post
title: "linux 字符设备驱动"
description:
category: linux
tags: [linux]
mathjax: 
chart:
comments: false
---

###1.字符设备概念
字符设备是指一个字节一个字节读写的设备，不能随机读取设备内存中的某一数据。字符设备是面向流的设备，常见的字符设备有鼠标键盘等


###2.重要的数据结构

文件操作file_operations

	struct file_operations {
		struct module *owner;
		loff_t (*llseek) (struct file *, loff_t, int);
		ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
		ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
		ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
		ssize_t (*aio_write) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
		ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
		ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
		int (*iterate) (struct file *, struct dir_context *);
		unsigned int (*poll) (struct file *, struct poll_table_struct *);
		long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
		long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
		int (*mmap) (struct file *, struct vm_area_struct *);
		int (*open) (struct inode *, struct file *);
		int (*flush) (struct file *, fl_owner_t id);
		int (*release) (struct inode *, struct file *);
		int (*fsync) (struct file *, loff_t, loff_t, int datasync);
		int (*aio_fsync) (struct kiocb *, int datasync);
		int (*fasync) (int, struct file *, int);
		int (*lock) (struct file *, int, struct file_lock *);
		ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
		unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
		int (*check_flags)(int);
		int (*flock) (struct file *, int, struct file_lock *);
		ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
		ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
		int (*setlease)(struct file *, long, struct file_lock **, void **);
		long (*fallocate)(struct file *file, int mode, loff_t offset,loff_t len);
		int (*show_fdinfo)(struct seq_file *m, struct file *f);
	};

在这些callback函数中，并不都是必须的，根据驱动需要实现即可

###3.一个简单的字符设备实现

	chr_num = register_chrdev(0,"helloworlddev",&hellowrld_dev_fops);//获得设备号
	if (chr_num < 0) {
		printk("helloworld Failed to register char device!\n");
	}

	helloworld_class = class_create(THIS_MODULE , "helloworldclass");//创建一个总线类型，会在/sys/class/下生成helloworldclass的目录
	if (IS_ERR(helloworld_class)) {
		unregister_chrdev(chr_num, "helloworlddev");
		printk("Failed to create class.\n");
		return PTR_ERR(helloworld_class);
	}
	device_create(helloworld_class, NULL, MKDEV(chr_num, 0),NULL,"helloworld0");//会在/dev/下生成helloworld0的设备节点

	static int helloworld_proc_read(struct seq_file *buf, void *v)
	{
		seq_printf(buf, "helloworldproc\n");
		return 0;
	}
	static int helloworld_proc_open(struct inode *inode, struct  file *file)
	{
			return single_open(file, helloworld_proc_read, NULL);
		
	}

	static struct file_operations helloworld_proc_ops = {
		.open = helloworld_proc_open,
		.read = seq_read,
		.release = single_release,
	};

	static const struct file_operations hellowrld_dev_fops = {
		.owner = THIS_MODULE,
		.unlocked_ioctl = helloworld_dev_ioctl,
		.open = helloworld_dev_open,
		.read	= helloworld_dev_read,
		.write	= helloworld_dev_write,

	};

[参考代码](https://github.com/jsno9/public/blob/master/andiord/driversample/helloworld/helloworld.c)











