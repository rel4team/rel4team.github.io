---
title: "论文阅读 3"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Preemptive Scheduling of Multi-Criticality Systems with Varying Degrees of Execution Time Assurance

这篇论文是非常老的一篇论文，但也因此比较经典

## 多临界系统的模型

首先来说明在vestal这篇论文中的mcs模型

他有n个period task。

Ti表示周期

Di表示deadline

有Di ≤Ti，也就是说，截止日期一定要小于最短的任务到达间隔

L = {A,B,C,D}，表示任务可能存在的critical level（其中A最高）

在不同的优先级l下，任务i具有不同的执行时间Cil

所以有

CiA ≥CiB ≥CiC ≥CiD（显然优先级越高，允许执行的时间越多，这也很好理解）

## RTA分析
这里首先使用了Finding Response Times in a Real-Time System这篇论文的结论，这篇论文是对单核的RTA分析的一个论文（多核另算）

非常抱歉我没能找到对应的论文（没有access权限），但是并不妨碍我们看懂在本篇论文中的一个描述和结论

他的结论如下：

$$
R_i=\sum_{j:ρ_j≤ρ_i} \lceil {\frac{R_i}{T_j}} \rceil C_{jL_i}
$$

在这里，Ri表示某个任务的响应时间，那么在这个任务响应之前，比他优先级更高的任务一定先被响应了，并且完成了他们的Ci运行时间。

所以Ri除以Tj并向上取整，是说，在Ri的时候，对于优先级更高的Tj为周期的任务j，应当有，Ri除以Tj向上取整这么多个周期，这个值再乘以j的每个周期的运行时间Ci，就是Ri到达之前，任务j到底运行了多久。

所以这个公式就比较好理解了。当然他扩展到了不同L情况下的不同Ci。（我也表示理解）

至于计算的方法，根据Finding Response Times in a Real-Time System这篇论文即可，使用迭代逼近法，从Ri=Ci开始，逐步逼近计算即可。

上述是不考虑多临界系统的模型

### 优化1
然后说起，周期转换的情况（这个其实在看的review中也是有的，只不过没那么重要我就没放）

简单来说，就是把任务拆分成几块，然后他就具有更短的周期，在某些系统中，更短的周期意味着更高的优先级，当然，主要是在上述的响应时间公式上进行修改

### 优化2
另一个需要提到的是Audsley的工作，也就是优先级assignment的（见另一篇文章的阅读）他的算法方案在此不再赘述

但是需要提到的是将audsley的工作应用到多个criticality上，vestal提出可以在此基础上选择可行解中，具有最高的critical scaling factor的task

如果没法区分，把高critical的任务放到high priorty

然后来说critial scaling factor，这个概念是说，所有任务的执行时间同时放大多少倍，仍然可调度的最大值。按照这个参数来规划静态优先级的priority

后面的都是这个evaluation的过程了。

## evaluation
在这篇文章中，对于evaluation的过程主要对比了几个

最有效的就是那个transform加上audsley的算法搞出来的。

相比之下，比起edf什么的更快。



# 评价

对于这个的评价，我觉得有他的创新性，毕竟那是2007年，但是说到底就是给audsley上面糊一个critial scaling factor。