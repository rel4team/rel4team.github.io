---
title: "论文阅读 4"
weight: 7
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Schedulability analysis of sporadic tasks with multiple criticality specifications
对于这篇论文，其实我觉得阅读的价值，额可以有，具体来说，给出了一些例子说明，这种情况下的调度，和传统调度已经不一样了，在某些任务以特定的关键性水平运行下可以通过的例子，并不能通过传统的调度的分析。
同时，可以预见的是，如果随着时间变化，priority是可变的，那么分析会更加的复杂。

## 核心算法
对于这篇文章他提出的核心算法，我觉得还是直接搬运吧。

![](/mcs_support/vestal_2008_1.png)

对于这个算法的解读其实是对前述Audsley提出的priority assignment的一个同样的扩展。
这个伪代码有几个概念
{{< katex display=true >}}
τ 表示所有的任务的集合
{{< /katex >}}
{{< katex display=true >}}
τ_cur 表示放在当前优先级p的任务集合
{{< /katex >}}
{{< katex display=true >}}
τ_hi 表示放在更高优先级p+1（或者更高）的任务集合
{{< /katex >}}
{{< katex display=true >}}
π1 , π2 , . . . , πl 表示所有的关键级别（critiality level）
{{< /katex >}}
当然同样的，还有从低到高（从1到某个priority级别p的）的优先级分布

对于这个伪代码，最初始的情况是，从优先级最低的情况开始考虑。而此时的`τ_cur`指向即为全部的任务集合`τ`。

按照ausdley的优先级安排的算法，AUGMENTEDAUDSLEY算法入参数为当前优先级p，以及当前优先级的任务集合`τ_cur`

然后在当前的优先级别，他按照关键级别从高到低，分别对任务集合进行可调度性的计算。

这会存在两个结果

如果，在当前优先级给定的任务集合，以及所有的关键级别上，所有的任务的响应时间都满足ddl，那么整个系统都是可调度的。

如果，在当前优先级给定的任务集合上，在某个关键级别，存在一个任务，错过了他的ddl。

在这种情况下，会被认为这个任务必须安排到更高的优先级去。



