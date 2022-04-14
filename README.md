# proj135-seamless-kernel-upgrade
基于双内核的内核热升级优化


### 项目描述与背景

现在数据中心或云服务器的Linux内核升级（假设服务器上面 使用的linux 4.19的内核，此时需要升级到linux 5.15）的传统做法是：
- 1、将服务器上面的业务迁走。
- 2、然后升级内核到linux 5.15。
- 3、然后重启服务器，加载和运行升级后的linux内核。
- 4、然后再将业务迁到已经升级好内核的机器上面来。
整个过程耗费的时间是比较长的。

传统做法的缺点和影响是：
- 1、虽然VM的迁移较为成熟，但是也有约束，比如要求是相同处理器、机型，对直通设备的迁移也有限制），而进程或长生命周期的普通容器的迁移技术还不成熟。不管是VM、进程还是普通容器、安全容器，整个过程需要管控的干预，需要相同机型的一批空闲的腾挪的机器（如果数据中心需要升级的机器种类越多，则需要更多用于腾挪的机器），耗费的时间是比较长的，还有很多迁移失败的情况，也有不少技术限制。
- 2、因此不管在数据中心还是云服务器集群，全面升级一次Linux内核是一项非常痛苦、非常浩大的工程，这就导致了很多情况下，一个内核通常使用5年甚至更长时间，这大大阻碍了新技术的演进，让业务无法定期（更无法最快）享受到Linux内核高速发展中不断推层出新的新特性带来的红利。

如果有一种机制让业务不迁移，直接原地快速地重启系统使用到新的内核，然后迅速让业务继续之前的RIP运行起来的话，整个过程就相对轻量、便捷。这就是原地的内核热升级技术。目前来看业界已经有一些技术探索，总的来说，原地快速内核热升级的核心是两点：数据持久化存储、内核快速启动。前者容易理解，毕竟内核热升级，需要不中断业务执行，比如内核重启前，数据需要保存起来，重启过程中继续复用这部分数据，这部分难度不是太高，不是本课题的重点。本课题重点是后者，即出于业务的downtime时间要求考虑，通常情况下，很多关键业务downtime的时间是要求50～100ms，假设OS服务、业务应用层的状态恢复需要50ms，那么旧内核的shutdown、新内核的加载初始化必须要满足50ms以内，才能满足最终业务进程downtime 50ms~100ms的要求。这个时间要求目前还没有任何一个技术/方案可以做到。

目前一个方案是如果第二个内核可以和第一个内核并行运行的话，第二个内核、系统服务初始化时，第一个内核继续运行，之后突然把第一个的多个进程状态、资源逐个切换到第二个内核，这样可以大大缩短downtime的时间。


双Linux内核并行执行：
```
          | ------------------------------------ |
          |       OS  A     |       OS B         |
          |--------------------------------------|
          |               hardware               |
          |--------------------------------------|
```
- 1 、通过某种方式（例如kexec）在运行着linux 4.19 内核操作系统上面隔离一些资源出来（CPU、内存等）运行一个linux 5.15内核的操作系统. 同时不要终止linux 4.19内核的执行（划分出来的资源4.19内核不再去访问）。
- 2、 运行在4.19内核上面的业务继续运行。直到linux5.15的操作的操作系统完全运行起来之后，将4.19内核上面跑的业务迁移到5.15内核上面继续运行。
- 3、 然后将linux 4.19的内核终止运行，并将其所占有的一切资源给到linux 5.15的操作系统。

通过双内核的方式就可以无感的切换内核，大大降低了升级内核对业务带来的影响。


### 功能描述

#### 基础功能：
- 1、第一个内核可以通过kexec分裂出第二个内核，第二个内核启动时，与第一个内核并存
- 2、当第二个内核或连同系统服务初始化到一定阶段，可以把所有资源都切换到第二个内核（下线第一个内核）后继续完成剩余部分的初始化工作（缩短downtime）。
- 3、本课题不需要考虑进程/VM状态等软件程序的状态备份恢复，只需要将硬件资源（CPU、内存等）从第一个内核快速切换到第二个内核, 并停止第一个内核。

课题的所有功能可在QEMU虚拟机里面完成：有如下提示：
- 1、首先使用QEMU启动一个虚拟机，运行着linux A。
- 2、然后通过kexec-tools加载运行linux B, 在加载运行linux B的时候不要停止linux A的运行。同时A需要在加载运行linux B之前为linux B提前预留好linux B所能够使用到的（cpu和内存等）资源，
避免由于资源的相互使用导致内核carsh.
- 3、QEMU 启动虚拟机的时候为虚拟机配置两个串口: 串口A 和串口B, linux A使用串口A, linux B使用串口B。

为了简化题目难度，可以做如下裁剪：
- 1、linux A和linux B的根文件系统可以采用initramfs， 这样就不需要考虑磁盘设备的共用问题。
- 2、QEMU 启动虚拟机的时候为虚拟机配置两个串口：串口A 和串口B, linux A使用串口A、 linux。 B使用串口B，这样就不涉及到串口资源的并发访问问题。


环境要求：为避免内核版本不同造成差异，在Linux 5.15进行开发

#### 扩展功能：

- 1、实现资源逐个切换（而不是所有资源一起切换）。先启动第二个内核（可以跳过硬件和设备的初始化），然后物理核、内存资源以及其它共享资源（比如ioapic等必备的资源，但是不要求磁盘、网卡、串口），按顺序地从第一个内核切换到第二个内核。并下线掉第一个内核。 
- 2、下线第一个内核前，最终可以分别通过串口A和串口B登录到linux A和linux B。 
- 3、使用带2张网卡的宿主机做上述实验。 
- 4、linux A和linux B两个内核共用一块网卡设备。 
- 5、linux A和linux B两个内核之间可以互相通信（例如核间中断方式）

### 所属赛道

2022全国大学生操作系统比赛的“OS功能挑战”赛道


### 参赛要求

- 以小组为单位参赛，最多三人一个小组，且小组成员是来自同一所高校的本科生或者研究生
- 如学生参加了多个项目，参赛学生选择一个自己参加的项目参与评奖
- 请遵循“2022全国大学生操作系统比赛”的章程和技术方案要求



### 项目导师
黄杰 huangjie.albert@bytedance.com

### 难度
困难

### 参考资料
[https://lwn.net/Articles/271991/](https://note.youdao.com/)  
[https://nan01ab.github.io/2017/12/Multi-Kernel.html](https://note.youdao.com/)  
[http://www.popcornlinux.org/](https://note.youdao.com/)  
[https://github.com/horms/kexec-tools](https://note.youdao.com/)  

### License
* [GPL-2.0](https://opensource.org/licenses/GPL-2.0)

## 预期目标

完成基础功能和扩展功能的开发，并输出说明文档一篇。

**注意：下面的内容是建议内容，不要求必须全部完成。选择本项目的同学也可与导师联系，提出自己的新想法，如导师认可，可加入预期目标**

参考任务描述部分。
