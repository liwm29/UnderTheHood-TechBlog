# TSAR

taobao system activity reporter

该工具本质是在读取linux系统/proc目录下的一些计数器文件,本片文章来介绍这些文件,及其内部包含的信息

## CPU

### coreInfo

使用此指令打印出一个逻辑核的相关信息,其他核是类似的信息,因为是SMP

cat cpuinfo | head -n 27

```sh
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 142
model name      : Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz
stepping        : 10
microcode       : 0xffffffff
cpu MHz         : 1799.999
cache size      : 6144 KB
physical id     : 0
siblings        : 8
core id         : 0
cpu cores       : 4
apicid          : 0
initial apicid  : 0
fpu             : yes
fpu_exception   : yes
cpuid level     : 21
wp              : yes
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 movbe popcnt aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single pti ssbd ibrs ibpb stibp fsgsbase bmi1 avx2 smep bmi2 erms invpcid rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 xsaves flush_l1d arch_capabilities
bugs            : cpu_meltdown spectre_v1 spectre_v2 spec_store_bypass l1tf mds swapgs itlb_multihit
bogomips        : 3599.99
clflush size    : 64
cache_alignment : 64
address sizes   : 39 bits physical, 48 bits virtual
power management:
```

解释:

> 摘自https://www.cnblogs.com/wxxjianchi/p/10522049.html

```sh
processor　：系统中逻辑处理核心数的编号，从0开始排序。
vendor_id　：CPU制造商
cpu family　：CPU产品系列代号
model　　　：CPU属于其系列中的哪一代的代号
model name：CPU属于的名字及其编号、标称主频
stepping　 ：CPU属于制作更新版本
cpu MHz　 ：CPU的实际使用主频
cache size ：CPU二级缓存大小
physical id ：单个物理CPU的标号
siblings ：单个物理CPU的逻辑CPU数。siblings=cpu cores [*2]。
core id ：当前物理核在其所处CPU中的编号，这个编号不一定连续。
cpu cores ：该逻辑核所处CPU的物理核数。比如此处cpu cores 是4个，那么对应core id 可能是 1、3、4、5。
apicid ：用来区分不同逻辑核的编号，系统中每个逻辑核的此编号必然不同，此编号不一定连续
fpu ：是否具有浮点运算单元（Floating Point Unit）
fpu_exception ：是否支持浮点计算异常
cpuid level ：执行cpuid指令前，eax寄存器中的值，根据不同的值cpuid指令会返回不同的内容
wp ：表明当前CPU是否在内核态支持对用户空间的写保护（Write Protection）
flags ：当前CPU支持的功能
bogomips：在系统内核启动时粗略测算的CPU速度（Million Instructions Per Second
clflush size ：每次刷新缓存的大小单位
cache_alignment ：缓存地址对齐单位
address sizes ：可访问地址空间位数
power management ：对能源管理的支持
```

#### 注解

上面已经讲的很清楚了,可以看出,我们的cpu的缓存是64字节为一个缓存行的,有专门的浮点数alu,每秒大概能执行3.6G条指令,虚拟地址空间是48位,物理地址空间是39位.cpu的实际主频是1.8GHZ,在flags里,我们看到了熟悉的avx,也就是是否支持向量化拓展指令集.

使用

cat cpuinfo | grep processor

可以看到输出是8个处理器,这是因为intel的单CPU四核八线程,这里的线程可以理解为就是处理器的意思

#### 超线程技术

我们知道一个核支持并行执行指令有几个级别,比如数据级并行,指令级并行和线程级并行

- 数据级并行
  - 多个ALU
- 指令级并行
  - 多个取址译码器,多个ALU,同时执行一个线程的多条指令
  - 也叫多发射
    - 动态多发射叫超标量,即运行时确定同时执行哪些指令
    - 与之相比的是静态多发射,由编译器确定同时执行哪些指令
  - 瓶颈很明显,由于各自依赖(比如数据依赖),单线程没有那么多指令可以并行
- 线程级并行
  - 多个取址译码器,多个ALU,多个Context(执行上下文/也就是寄存器组)
  - 同时执行不同线程的多条指令

超线程应该就是同时多线程的一种实现

![image-20210324171440077](static\image-20210324171440077.png)



## cpuTime

cat /proc/stat 此命令查看cpu的时间分配,典型的就是用户态运行时间,内核态运行时间,io阻塞时间,空闲空转时间,中断时间等

参考: http://gityuan.com/2017/08/12/proc_stat/

```sh
//CPU指标：user，nice, system, idle, iowait, irq, softirq
cpu  151 0 1822 3035462 65 0 66 0 0 0
cpu0 17 0 737 378763 6 0 44 0 0 0
cpu1 6 0 38 379698 8 0 11 0 0 0
cpu2 30 0 320 379258 28 0 11 0 0 0
cpu3 8 0 24 379718 1 0 0 0 0 0
cpu4 29 0 239 379427 6 0 0 0 0 0
cpu5 5 0 15 379698 0 0 0 0 0 0
cpu6 22 0 422 379221 8 0 0 0 0 0
cpu7 34 0 27 379674 4 0 0 0 0 0
intr 31441 0 0 0 0 0 0 0 0 0 18 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
ctxt 151746
btime 1616574037
processes 403
procs_running 1
procs_blocked 0
softirq 147819 0 41104 0 123 4812 0 20320 42642 0 38818
```

| cpu指标    | 含义                              |
| ---------- | --------------------------------- |
| user       | 用户态时间(一般/高优先级,nice<=0) |
| nice       | 用户态时间(低优先级，nice>0)      |
| system     | 内核态时间                        |
| idle       | 空闲时间                          |
| iowait     | I/O等待时间                       |
| irq        | 硬中断                            |
| softirq    | 软中断                            |
| steal      | 被盗时间                          |
| guest      | 来宾时间                          |
| guest_nice | nice来宾时间                      |

> 单位是jiffies , 1 jiffies = 0.01s = 10ms
>
> 统计cpu利用率: 总时间就是它们的和

如果想看某个进程的统计信息,一是strace -p [pid],二就是直接看统计文件

某个进程的统计文件在/proc/[pid]/stat



