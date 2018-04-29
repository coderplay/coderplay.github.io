---
layout: post
title: 系统动态跟踪(一)
key: 20180428
tags:
  - Dynamic Tracing
mathjax: true
mathjax_autoNumber: true
---

题外话：近日觉得记性越来越差，刚查过的一些案例过些日子就忘了。  想想好记性确实不如烂笔头。多年前我维护过一个博客，上面写的内容我只须再刷一遍就能记起很多。最有意思的是还能抛砖引玉，这些年内还陆续有朋友发email和我讨论一些相关的内容，我也从他们各不相同的案例上积累到不少经验。这个博客后面因为来了美国荒废了，主要原因还是因为懒，还有一些其它生活和所在公司不开放的原因。对于最后者，近期我感觉可以学学王垠的经验，写一些与工作内容不大相关的案例。

## 开始了
言归正传，这一系列探讨的是Linux x86_64系统动态跟踪。这里系统的定义比较泛，可以是Linux操作系统，Xen hypervisor(当然会有一些东西会跟踪不到), MySQL数据库系统或者是Java虚拟机系统。作为大多数上层应用开发者来说，仿佛系统跟踪离应用太远，没有多少实际用途。我们接下来举个简单的例子来消除这个疑问。 在这之前，请确认以下先决条件。
* 查看内核编译配置，核对这些参数是否为y: CONFIG_KPROBES=y ,  CONFIG_KPROBE_EVENTS=y， CONFIG_FRAME_POINTER=y. 一般情况下内核编译配置在`/boot/config-$(uname -r)`文件里。
* 已安装[perf](https://perf.wiki.kernel.org) (请自行查找你的linux分发版本怎么安装, 一般情况只需要yum install perf或apt-get install linux-tools)
* 已安装kernel debuginfo (非必要, 请自行查找你的linux分发版本怎么安装, Debian分发版一般叫dbgsym)

### 示例一
下面开始第一个例子: 计算机最普遍的应用就是读数据和写数据。一个最常见的问题是：我想知道某生产机器上所有的应用正在读什么文件? strace只能看到文件描述符而不是文件名(有时候可以再通过lsof拿到文件),  blktrace只能看到block级的，有时候读文件只在page cache里就完成了。这时候我们就可以从Linux内核里的函数来进行更为准确的跟踪。 通过[内核函数图](http://www.makelinux.net/kernel_map/), 我们可以了解Linux有一层虚拟文件系统抽象。每个应用层面的读都得经过`vfs_read`这个函数。于是翻阅[vfs_read代码](https://github.com/torvalds/linux/blob/v4.16/fs/read_write.c#L432), 我们看到可以从`vfs_read`的第一个参数`file`着手，顺着file的成员找下去，可以拿到文件名。文件名在`file->f_path.dentry->d_name.name`里。 
```c
ssize_t  vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
```
`perf`的`probe`子命令可以通过`debuginfo`的信息, 在每次本机任何程序调用`vfs_read`时对`file->f_path.dentry->d_name.name`的值进行跟踪。这种动态跟踪分为三步: 1) 注册要跟踪的事件及值 2) 记录一个时间段内的事件 3) 重放结果。

1. 注册对`vfs_read`调用事件及文件名的跟踪。
	```
	$ sudo perf probe 'vfs_read file->f_path.dentry->d_name.name:string'
	```
	没有安装debuginfo的朋友, 可以使用以下命令完成注册:
	```
	$ sudo perf probe 'vfs_read name=+0(+40(+24(%di))):string'
	```
	为什么以上怪异的串`+0(+40(+24(%di))):string`和`file->f_path.dentry->d_name.name`等价？看完这系列文章，你就会完全明白, 先透露这是对`rdi`寄丰器的dereference和成员偏移量。

2.  加好后就可以记录整个系统或某进程的读文件情况了
	```
	$ sudo perf record -e  probe:vfs_read -a
	```
	运行一段时间`Ctrl+C`结束记录。

3. 重放结果，我们就能够知道这台机器上有什么文件正在被读了。
	```
	$ sudo perf script
	            sshd 82220 [018] 13583551.811406: probe:vfs_read: (ffffffff81200c50) name="TCP"
	            bash 82386 [063] 13583551.811990: probe:vfs_read: (ffffffff81200c50) name="cat"
	            bash 82386 [063] 13583551.812025: probe:vfs_read: (ffffffff81200c50) name="ld-2.17.so"
	             cat 82386 [063] 13583551.812219: probe:vfs_read: (ffffffff81200c50) name="libc-2.17.so"
	             cat 82386 [063] 13583551.812480: probe:vfs_read: (ffffffff81200c50) name="readelf.py"
	```

通过perf, 我们还可以知道是kernel上的哪些调用栈调用了`vfs_read`。 调用栈可以帮助我们从下至上理解`vfs_read`最终是被哪个函数调用的，同时也可以增强我们对程序的运行情况的理解。跟踪调用栈方法如下：

1.   在上次记录命令的基础上加`-g`, 如果安装了kernel debuginfo, 则可以再加`--call-graph dwarf`。
	
		```
		$ sudo perf record -e  probe:vfs_read -a
		```
		运行一段时间`Ctrl+C`结束记录。
1.  使用perf的report子命令打印出各调用栈的分布，由于结果太长，以下省略了部分结果。
	```
	$ sudo perf report --stdio
	...
	    25.90%    25.90%  (ffffffff811f9f40) name="cmdline"
	            |
	            |--15.23%--0x3c65
	            |          __libc_start_main
	            |          ...
	            |          event_base_loop
	            |          0x2176f
	            |          0x3f62c
	            |          _IO_getc
	            |          _IO_default_uflow
	            |          _IO_file_underflow@@GLIBC_2.2.5
	            |          __GI___libc_read
	            |          entry_SYSCALL_64_fastpath
	            |          vfs_read
	            |
	             --10.66%--entry_SYSCALL_64_fastpath
	                       vfs_read

	    16.21%    16.21%  (ffffffff811f9f40) name="moduli"
	            |
	            ---0xfad1
	               __libc_start_main
	               ...
	               |
	               |--15.67%--0x61a70
	               |          _IO_fgets
	               |          _IO_getline_info
	               |          _IO_default_uflow
	               |          _IO_file_underflow@@GLIBC_2.2.5
	               |          __GI___libc_read
	               |          entry_SYSCALL_64_fastpath
	               |          vfs_read
	               |
	```

### 示例二
再看一个用户态的例子：我想知道在当前机器上所有人运行的bash命令。假设我们知道bash的源代码有一个叫`readline`的函数，其每次调用的返回值就是一条bash命令。这时我们只需要跟踪这个函数的返回值就可以了。此例不需要bash的debuginfo，同样三步:   
1. 注册对`readline`的调用事件以及它的返回值. 注意此例不是跟踪linux kernel, 所以需要用`-x`指定包含`readline`的可运行程序或so文件路径。如果是跟踪kernel，则可以省略。函数名后加上`%return`表示在函数返回时跟踪。`$retval`表示返回值的地址, `+0($retval)`表示对返回值地址的dereference, 也就是返回值了。`string`表示它的类型。这些设置的语法将在以后的篇幅具体描述。
	```
	$ sudo perf probe  -x /bin/bash 'readline%return +0($retval):string'
	```
2. 现在我们注册好了，就可以记录bash命令的运行情况了。
	```
	$ sudo perf record -e probe_bash:readline -a
	```
3. 隔段时间，查看结果：
	```
	$ sudo perf script
            bash 117163 [063] 19006740.329346: probe_bash:readline: (488ee0 <- 41e62a) arg1="^X"
            bash 116984 [019] 19006741.731002: probe_bash:readline: (488ee0 <- 41e62a) arg1="sudo su "
            bash 117235 [043] 19006744.619300: probe_bash:readline: (488ee0 <- 41e62a) arg1="cat /sys/kernel/tracing/uprobe_events "
            bash 117235 [043] 19006748.073042: probe_bash:readline: (488ee0 <- 41e62a) arg1="ls"
            bash 117235 [043] 19006751.698744: probe_bash:readline: (488ee0 <- 41e62a) arg1="tree "
            bash 117235 [043] 19006761.841761: probe_bash:readline: (488ee0 <- 41e62a) arg1="cd /var/log/messages"
            bash 117235 [043] 19006762.241879: probe_bash:readline: (488ee0 <- 41e62a) arg1="ls"
            bash 117235 [043] 19006766.701663: probe_bash:readline: (488ee0 <- 41e62a) arg1="cd /var/log/"
            bash 117235 [043] 19006768.266072: probe_bash:readline: (488ee0 <- 41e62a) arg1="cat messages"
	```

类似的问题还很多，再举几个系统动态跟踪技术能回答的问题:

* 为什么网卡中断都在同一个CPU核上处理?
* 某网络应用经过TCP层传输包的平均大小是多少?
* 为什么应用iowait很高，通过iostat显示每秒读写block io合并率不高?
* page cache什么时候将数据刷到持久介质上的?
* 某条MySQL查询经过哪几轮查询优化?
* Java 进行G1 GC时每个Region有哪些对象?

## 其它实用功能

### 函数列表
列出java中 G1GC相关的可跟踪的函数
```
$ sudo perf probe -v  -x /usr/java/jdk1.8.0_144/jre/lib/amd64/server/libjvm.so --funcs | grep G1GC
G1GCParPhasePrinter::print_multi_length(G1GCPhaseTimes::GCParPhases, WorkerDataArray<double>*)
G1GCParPhasePrinter::print_single_length(G1GCPhaseTimes::GCParPhases, WorkerDataArray<double>*)
G1GCParPhaseTimesTracker::G1GCParPhaseTimesTracker(G1GCPhaseTimes*, G1GCPhaseTimes::GCParPhases, unsigned int)
...
```
### 注册事件列表
列出已经注册的事件
```
$ sudo perf probe -l
  probe:slowpath       (on __alloc_pages_slowpath@mm/page_alloc.c)
  probe:tcp_sendmsg    (on tcp_sendmsg@net/ipv4/tcp.c with size dest)
  probe:vfs_read       (on vfs_read@fs/read_write.c with name)
  probe_bash:readline  (on readline%return in /bin/bash with arg1)
```

### 函数入参列表

此功能需要debuginfo, 以下是列出kernel中`tcp_sendmsg`函数的所有入参
```
$ sudo perf probe -V tcp_sendmsg
Available variables at tcp_sendmsg
        @<tcp_sendmsg+0>
                size_t  size
                struct msghdr*  msg
                struct sock*    sk
                struct sockcm_cookie    sockc
```

### 条件过滤
跟踪消息大于100字节且目标地址是`172.31.28.139`的tcp发送调用栈
* 已安装kernel debuginfo
	```
	$ sudo perf probe  --add 'tcp_sendmsg size dest=sk->__sk_common.skc_daddr:u32'
	```
* 未安装kernel debuginfo
	```
	$ sudo perf probe  --add 'tcp_sendmsg size=%dx:x64 dest=+0(%di):u32'
	```
记录时进行过滤
```
	$ sudo perf record -e probe:tcp_sendmsg --filter 'size > 100 && dest == 2333876140' -a -g
	$ sudo perf report --stdio
```
注： 上面的`2333876140`是`139.28.31.172`的无符号整数形式，它的字节顺序与`172.31.28.139`相反。

## 小结
系统动态跟踪可以使我们深入到正在运行的系统其内部几乎所有函数，取出函数被调用时的传入、传出值、调用栈以及对调用事件的条件过滤。 这种技术的强大之处是给予你一种持续过滤、跟踪的能力， 使用它排查问题就像剥洋葱一样一层一层地揭开。掌握这种技术后，在排查问题时其底层的系统再也不像黑盒一样地存在，而且对底层不再会有畏惧感，会使人有一种任督二脉被打通的感觉。这种技术相较于用gdb等调试技术最重要的一个优势是它的消耗非常小，不会像调试工具那样hang住应用，它在生产环境是安全的。

一般情况下使用系统动态跟踪技术排查问题有以下几步：
* 从日志或metrics发现问题
* 使用日志或metrics里的关键词在源码中找到发生问题的函数
* 跟踪函数的调用栈找出函数调用情况
* 根据调用栈查看源代码， 找出下一个可以分析的点
* 继续跟踪相关函数调用栈或传出传出值直至找到问题所在

总体来说这是一个自下而上的跟踪过程。此过程的关键是找到需要跟踪的函数定义，找函数的过程则需要阅读一些相关的源码。

说到动态跟踪，就不得不提起一个人: Brendan Gregg。他是最著名的DTrace, systemtap, perf, eBPF布道师，目前在netflix工作。从他的[个人网站](http://www.brendangregg.com/) 可以挖到很多相关的宝藏。对于perf的其它技巧可以看他[这篇](http://www.brendangregg.com/perf.html)。Brendan的文章涉及到动态跟踪的很多方面。我要完成的这一系列与他的文章有一些不同：首先，这系列文章主要目标受众是程序员，主要用于排查问题而不是用于监控；其次，这一系列会涉及到Brendan的文章里没有涉及的一些技巧，例如如何跟踪C++程序里this指针内的成员。接下来的几篇将会包含以下几个方面：

* 符号表、dwarf格式、Linux x86_64 calling convention等基础知识
* 在没有debuginfo的情况下如何找到函数的地址以及传入传出参数的寄存器并加以跟踪
* 如果跟踪的参数是struct/union怎么办？
* 如何跟踪C++, rust, go程序
* 实际案例

