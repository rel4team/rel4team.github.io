---
title: "论文阅读 2"
weight: 1
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


## RTA分析
这里首先使用了Finding Response Times in a Real-Time System这篇论文的结论，这篇论文是对单核的RTA分析的一个论文（多核另算）

非常抱歉我没能找到对应的论文（没有access权限），但是并不妨碍我们看懂在本篇论文中的一个描述和结论

他的结论如下：

在这里，Ri表示某个任务的响应时间，那么在这个任务响应之前，比他优先级更高的任务一定先被响应了，并且完成了他们的Ci运行时间。

所以Ri除以Tj并向上取整，是说，在Ri的时候，对于优先级更高的Tj为周期的任务j，应当有，Ri除以Tj向上取整这么多个周期，这个值再乘以j的每个周期的运行时间Ci，就是Ri到达之前，任务j到底运行了多久。

所以这个公式就比较好理解了。当然他扩展到了不同L情况下的不同Ci。（我也表示理解）

至于计算的方法，根据Finding Response Times in a Real-Time System这篇论文即可，使用迭代逼近法，从Ri=Ci开始，逐步逼近计算即可。

上述是不考虑多临界系统的模型