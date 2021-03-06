# Linux

## 启动与停止软件

### CentOS 删除软件包

```bash
# 以删除 mongodb-org 为例
sudo yum erase $(rpm -qa | grep mongodb-org)
```

### CentOS 删除 systemctl 服务

> 来自：[https://superuser.com/a/936976](https://superuser.com/a/936976)

```bash
systemctl stop [servicename]
systemctl disable [servicename]
rm /etc/systemd/system/[servicename]
rm /etc/systemd/system/[servicename] # and symlinks that might be related
rm /usr/lib/systemd/system/[servicename] 
rm /usr/lib/systemd/system/[servicename] # and symlinks that might be related
systemctl daemon-reload
systemctl reset-failed
```

## Linux 性能优化实践

{% hint style="info" %}
资料来源：[Linux 性能优化实践 -- 倪朋飞](https://time.geekbang.org/column/article/68728)
{% endhint %}

### 如何排查 CPU 问题

#### 平均负载

（1）区分平均负载与 CPU 使用率

平均负载：单位时间内，活跃的进程数（处于可运行状态和不可中断状态的平均进程数）。

CPU 使用率：单位时间内，CPU 繁忙情况的统计。[简单的计算方式](https://www.eukhost.com/forums/forum/general/technology-forum/22321-what-is-cpu-utilization-and-how-can-it-be-calculated)是：

```text
U = R/C
U= Utilization
R= Requirements which in simple terms is the BUSY TIME
C= Capacity which is simple terms is BUSY TIME + IDLE TIME
```

 两者关系：

* CPU 密集型进程：CPU 使用率越高，平均负载越高。
* I/O 密集型进程：等待 I/O 会导致平均负载升高，但 CPU 使用率不一定高。
* 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高。

#### CPU 上下文切换

（1）CPU 上下文（CPU Context）：CPU 寄存器（存储指令）和程序计数器（Program Counter，PC，指向下一条指令）。

（2）CPU 上下文切换过程：保存上一任务，载入下一任务

* 系统调用：保存用户态，切换到内核态，执行内核态代码，之后恢复用户态，切换到用户空间，共发生两次 CPU 上下文切换（称之为特权模式切换），但未发生进程切换。
* 进程上下文切换：进程由内核调度，其切换过程发生在内核态，其上下文包括虚拟内存、栈、全局变量等用户态，还包括内核堆栈、寄存器等内核态。
* 线程上下文切换：其上下文只包括线程的私有数据、寄存器，不包括进程内所有线程共享的虚拟内存和全局变量。 
* 中断上下文切换：其上下文只包括内存堆栈、寄存器、硬件中断参数等内核态。

（3）CPU 上下文切换的时间：每次切换几十纳秒到数微秒的 CPU 时间，查看https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html

（4）进程上下文切换的场景：

* 时间片耗尽
* 系统资源不足
* sleep 函数被调用
* 高优先级进程插入
* 硬件中断

### CPU 中断

Linux 中的中断处理程序分为上半部和下半部：

* 上半部对应硬件中断，用来快速处理中断。
* 下半部对应软中断，用来异步处理上半部未完成的工作。

Linux 中的软中断包括网络收发、定时、调度、RCU 锁等各种类型，可以通过查看 /proc/softirqs 来观察软中断的运行情况。

```bash
$ watch -d cat /proc/softirqs
                    CPU0       CPU1
          HI:          0          0
       TIMER:    1083906    2368646
      NET_TX:         53          9
      NET_RX:    1550643    1916776
       BLOCK:          0          0
    IRQ_POLL:          0          0
     TASKLET:     333637       3930
       SCHED:     963675    2293171
     HRTIMER:          0          0
         RCU:    1542111    1590625
```

> TIMER（定时中断）、NET\_RX（网络接收）、SCHED（内核调度）、RCU（RCU 锁）。

在 Linux 中，每个 CPU 都对应一个软中断内核线程，名字是 ksoftirqd/CPU 编号。当软中断事件的频率过高时，内核线程也会因为 CPU 使用率过高而导致软中断处理不及时，进而引发网络收发延迟、调度缓慢等性能问题。

## 系统分析工具

![](../.gitbook/assets/image%20%2821%29.png)

![](../.gitbook/assets/image%20%2828%29.png)

![](../.gitbook/assets/image%20%2832%29.png)

### vmstat

vmstat （Virtual Memory Statistics，虚拟内存统计），报告关于进程、内存、I/O等系统整体运行状态。参考：https://man.linuxde.net/vmstat

（1）示例

```text
[root@xxx ~]# vmstat 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 382356 168420 2172340    0    0     1    11   14   18  0  0 100  0  0
 0  0      0 382232 168420 2172340    0    0     0    26  285  464  0  0 100  0  0
 0  0      0 382232 168420 2172340    0    0     0     0  271  455  0  0 100  0  0
 0  0      0 382232 168420 2172340    0    0     0    23  291  470  0  0 100  0  0
```

（2）指标

procs（进程）

*  r（running）：运行队列的长度，等待运行时的进程数量。运行队列是一个可运行状态进程的双向循环链表。运行队列中永远存在两个特殊的进程：当前进程（current\_task）和空进程（idle\_task）。运行队列的长度即是处于可运行状态的进程数目，用全局整型变量nr\_running表示，若nr\_running=0 表示队列中只有空进程。参见：https://book.aikaiyuan.com/kernel/4.5.3.htm nr\_running长期超 1 时需考虑增加 CPU
* b（blocking）：等待 IO 的进程数量。

memory（内存）

* swpd：使用的虚拟内存总量
* free：空闲内存总量
* buff：用作缓冲的内存总量（写入文件使用 buffer 以减少磁盘写 IO）
* cache：用作缓存的内存总量（频繁访问的文件会被 cache 以减少磁盘读 IO）

swap（磁盘中的交换分区）

* si：每秒从交换分区写入内存的总量
* so：每秒写入交换分区的内存总量

io（现在的 Linux 版本块大小为 1kb）

* bi（block in）：每秒读取的块数。
* bo（block out）：每秒写入的块数。

system

* in（interrupt）：每秒中端的次数，包含时钟中断。
* cs（context switch）：每秒上下文切花的次数。

cpu

* us：用户进程执行时间百分比。长期超 50% 时需考虑优化算法或提升 CPU 性能
* sy：内核系统进程执行时间百分比。
* wa：IO 等待时间百分比。
* id：空闲时间百分比。

### sysstat

（1）安装

```text
yum install sysstat
```

（2）pidstat

查看每个进程的 CPU 使用率

```text
[root@xxx ~]# pidstat -wt 1
Linux 3.10.0-1062.1.1.el7.x86_64 (xxx) 	2020年04月22日 	_x86_64_	(2 CPU)

17时21分18秒   UID      TGID       TID   cswch/s nvcswch/s  Command
…
17时21分19秒   996       657         -      9.90      0.00  redis-server
17时21分19秒   996         -       657      9.90      0.00  |__redis-server
17时21分19秒     0         -       889      0.99      0.00  |__in:imjournal
```

指标：

* cswch：自愿上下文切换，I/O、内存等系统资源不足时发生的上下文切换。
* nvcswch：非自愿上下文切换，时间片已到等原因被系统强制调度而发生的上下文切换。

（3）mpstat

Multiprocessor Statistics 多处理器统计，查看 CPU 使用率

```text

# -P ALL 表示监控所有CPU，后面数字5表示间隔5秒后输出一组数据
$ mpstat -P ALL 5
Linux 4.15.0 (ubuntu) 09/22/18 _x86_64_ (2 CPU)
13:30:06     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
13:30:11     all   50.05    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   49.95
13:30:11       0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
13:30:11       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
```

### uptime

```text

# -d 参数表示高亮显示变化的区域
$ watch -d uptime
...,  load average: 1.00, 0.75, 0.39
```

### sar

sar 是一个系统活动报告工具，既可以实时查看系统的当前活动，又可以配置保存和报告历史统计数据。

```bash
# -n DEV 表示显示网络收发的报告，间隔1秒输出一组数据
$ sar -n DEV 1
15:03:46        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
15:03:47         eth0  12607.00   6304.00    664.86    358.11      0.00      0.00      0.00      0.01
15:03:47      docker0   6302.00  12604.00    270.79    664.66      0.00      0.00      0.00      0.00
15:03:47           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
15:03:47    veth9f6bbcd   6302.00  12604.00    356.95    664.66      0.00      0.00      0.00      0.05
```

第一列：表示报告的时间。  
第二列：IFACE 表示网卡。  
第三、四列：rxpck/s 和 txpck/s 分别表示每秒接收、发送的网络帧数，也就是 PPS。第五、六列：rxkB/s 和 txkB/s 分别表示每秒接收、发送的千字节数，也就是 BPS。

eth0 ：接收的 PPS 比较大，达到 12607，而接收的 BPS 却很小，只有 664 KB。直观来看网络帧应该都是比较小的，我们稍微计算一下，664\*1024/12607 = 54 字节，说明平均每个网络帧只有 54 字节，这显然是很小的网络帧，也就是我们通常所说的小包问题。

### hping3

hping3 是一个可以构造 TCP/IP 协议数据包的工具，可以对系统进行安全审计、防火墙测试等。

```bash
# -S参数表示设置TCP协议的SYN（同步序列号），-p表示目的端口为80
# -i u100表示每隔100微秒发送一个网络帧
# 注：如果你在实践过程中现象不明显，可以尝试把100调小，比如调成10甚至1
$ hping3 -S -p 80 -i u100 192.168.0.30
```

### tcpdump

tcpdump 是一个常用的网络抓包工具，常用来分析各种网络问题。

```bash
# -i eth0 只抓取eth0网卡，-n不解析协议名和主机名
# tcp port 80表示只抓取tcp协议并且端口号为80的网络帧
$ tcpdump -i eth0 -n tcp port 80
15:11:32.678966 IP 192.168.0.2.18238 > 192.168.0.30.80: Flags [S], seq 458303614, win 512, length 0
...
```



