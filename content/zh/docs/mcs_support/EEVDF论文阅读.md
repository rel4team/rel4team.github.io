---
title: "EEVDF 论文阅读"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 简介
EEVDF是Linux里面的新晋调度器（虽然也不是很新了），具体论文可以看
[EEVDF论文](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=805acf7726282721504c8f00575d91ebfd750564)

本文主要做论文的阅读理解。

文中提到一些现有的调度方法
 - 比例份额方法（proportional share）
 - 基于事件驱动的RT的方法
 - ...
这些不同的方法不太能通用
比如RT对于batch的应用，对于多媒体的应用并不太兼容（难以界定需要的服务时间）
再比如RT一般要求严格的实时性，但是对于有些情况，用户愿意接受稍微慢一点，可是RT就没法继续往里面塞任务了。
过于动态的环境下就不太灵活了。

所以EEVDF这个算法，给出了统一的算法试图兼容Batch，media，以及interactive等任务
当然他也基于比如说fluid-flow算法

# 一些Assumption

 - 一些名词解释
    - client ，这里面称为client而不是采用一个进程的说法，是因为EEVDF的设计场景，还包括比如网络带宽的处理等等场景下的分配。不过，在只考虑进程调度问题的时候，请当成task或者thread或者process就可以了。
    - request，这里是为了兼容不同需求。具体来说，每个client都有一个请求request。从任务调度的角度来说，这可以当作，一个task请求CPU，即发起一个CPU request，即，他进入了就绪队列中，可以看作一个意思。这个请求，之所以能兼容不同的需求，因为他可以表示RT的事件驱动的模型，同时，可以表示其他情况下，正如上面所说的，进入就绪队列，其实就相当于发起一个CPU request。另外，这个请求还有一个duration r，也就是期望得到的service time。
    
    另外，request他可能来自于client，也可能来自于调度器（比如RT中一个周期结束之后，他会重填一个request进去，让周期性的实时任务开启下一个周期这样子）
    
    当然按照上面这个说法，一个client应当最多只有一个request。所以实际上一个request也跟一个client关联起来。
    - weight，权重，我觉得这是很好理解的，一个任务有一个权重，实际上指代的是他期望得到的服务时间的比例（份额/share），比如两个进程的环境下，进程A有weight = 1，进程B有weight =3，那么A实际上期望得到25%的CPU事件，B期望得到75%的CPU时间。
    - duration r, 从灵活性上而言，他可以有任意值，但是如果duration > 时间片大小，那么在这个时间片结束之后，就会被抢占，考虑到上下文开销，所以通常来说，时间片尽量短但是多，会更好一些。
    - virtual eligible time和virtual deadline，这是两个相关的概念。首先这里都是虚拟时间（具体后面会说）。而virtual eligible time可以被认为是一个start time，而virtual deadline是finish time（当然都是虚拟时间），每个任务的一个request都应该关联一个virtual eligible time和virtual deadline。
    - eligible，我们前面在virtual eligible time这里提到了这个词，一个request被称作eligible的，条件是，他的virtual eligible time小于等于当前的virtual time
 - EEVDF算法
    将下一个时间片分给这样一个任务
    - 他具有一个request，并且这个request是eligible的
    - 在所有满足第一条的前提下的任务中，这个任务的virtural time是最早的。
 - 系统的模型
    - 系统里面有一大堆的client去共享比如处理器或者网络带宽等
    - 时间片，client对于时间片的使用，要么完整的用完，然后开启一个新的时间片，要么提早用完它（类比一个任务，要么等到一个时间片用完，要么在时间片还没用完的时候，这个任务结束了）
    - 考虑进抢占之后，常见的调度器的模型变成这样：一个任务会一直占用CPU，直到
      - 时间片用完
      - 更高优先级的任务抢占掉它
      - 自己把自己阻塞住，等待一些比如IO的事件完成
      - 任务完成
# 开始推导模型
对于clinet i，让
{{< katex display=true >}}
A(t)
{{< /katex >}}
表示在时刻t所有的client的集合，每个client具有一个weight 
{{< katex display=true >}}
w_i
{{< /katex >}}
那么每个client期望获得的份额为
{{< katex display=true >}}
f_i(t)=\frac{w_i}{\sum_{j\epsilon{A(t)}}w_j}
{{< /katex >}}
如果对于一个连续的过程来说（就是说，并不像现实中的调度那样，一段时间全都分给某个进程，然后之后一段分给另一个进程），那么取一个足够短的时间片
{{< katex display=true >}}
(t,\delta{t})
{{< /katex >}}
我认为之所以取一段足够短的时间，是为了表明这个时间区间内client的集合以及每个clinet的权重weight并没有发生任何变动
那么client会执行
{{< katex display=true >}}
f_i(t)\delta{t}
{{< /katex >}}
的时间长度。

当然，这只是理想的情况下，也就是前面说的，client的集合和权重并没有发生变化，但是对于一段时间而言，还是会变化的，因此，使用简单的积分思想就可以得到，在一段client的集合和权重会发生变化的时间段
{{< katex display=true >}}
[t_0,t_1]
{{< /katex >}}
之内，令
{{< katex display=true >}}
S_i(t_0,t_1)=\int_{t_0}^{t_1}{f_i(t)}dt
{{< /katex >}}
这就是它在某个区间之内，根据权重weight得到的，期望的在这期间得到的service time。之后，我们都会使用Si(t0,t1)的形式（大S）来描述一个client i期望得到的服务时间。

## 和RT情况下的对比以及兼容

一个RT的任务（通常是说周期性的实时任务）

会有一个周期period T

一个该周期内的最大服务时间 r

以及一个截止时间d（即ddl，同样是下一个周期开始的时间）

则cpu占用率即为
{{< katex display=true >}}
f=\frac{r}{T}
{{< /katex >}}
（这里的r和f的含义其实和上面的duration r以及份额share f其实是一样）

而因为他是个周期任务，所以有在时刻t的时候
{{< katex display=true >}}
T=\frac{r}{f} (即给定了cpu利用率f和期望的服务时间r之后，得把周期设多少)
{{< /katex >}}
以及
{{< katex display=true >}}
d=t+T=t+\frac{r}{f}
{{< /katex >}}
{{< katex display=true >}}
r=S(t,d)
{{< /katex >}}
同样的，如果份额f一直不变（这在RT的情况下是完全符合的,一个实时性的任务在确定好周期以及最大的服务时间之后，通常不会试图改变它）

那么应当有
{{< katex display=true >}}
S(t,d)=f(d-t) (S原本是一个在t到d上对f的一个积分，因为f是不变的，所以直接f乘以积分区间即可)
{{< /katex >}}
即
{{< katex display=true >}}
r=f(d-t) => d=t+\frac{r}{f}
{{< /katex >}}
表明这个框架是能够兼容RT情况下的分析的

当然，也有一些区别，比如RT情况下，事件来自于外部，而EEVDF这里来自于外部事件或者内部事件


## 考虑离散

不过，上述的情况是连续的情况。

也就是说，实际上，在现实中，服务时间是离散的。

另外，调度算法本身，以及上下文切换具有开销。还有就是，某些过程中，比如加了锁或者一些硬件的原子指令，都是无法被打断的。

所以，实际上会有一些差别。

所以它定义了另一个时间，也就是实际得到的服务时间si（这里是小S）
另外，令
{{< katex display=true >}}
t_0^i
{{< /katex >}}
表示在某个时刻一个任务变成acitve（我的理解就是request到了）（下面说到这个变量也是一个含义）

那么，实际服务时间和期望的服务时间之间存在一个差距（也许这个差距很微小，但是累计之后，就很多了）

用以下公式来表示这种实际和应得之间的差距
{{< katex display=true >}}
lag_i(t)=S_i(t_0^i,t)-s_i(t_0^i,t)
{{< /katex >}}

## 虚拟时间
考虑到上面的公式
{{< katex display=true >}}
f_i(t)=\frac{w_i}{\sum_{j\epsilon{A(t)}}w_j}
{{< /katex >}}
{{< katex display=true >}}
S_i(t_0,t_1)=\int_{t_0}^{t_1}{f_i(t)}dt
{{< /katex >}}
所以
{{< katex display=true >}}
S_i(t_0,t_1)=\int_{t_0}^{t_1}{\frac{w_i}{\sum_{j\epsilon{A(t)}}w_j}}dt
{{< /katex >}}
{{< katex display=true >}}
={w_i}\int_{t_0}^{t_1}{\frac{1}{\sum_{j\epsilon{A(t)}}w_j}}dt
{{< /katex >}}
则，我们让虚拟时间V（t）等于
{{< katex display=true >}}
V(t)=\int_{0}^{t}{\frac{1}{\sum_{j\epsilon{A(t)}}w_j}}dt
{{< /katex >}}
从而有
{{< katex display=true >}}
S_i(t_0,t_1)=w_i(V(t_1)-V(t_0))
{{< /katex >}}
这里关于积分上下限的处理，是常见的，我就不多说了，这个式子的实际作用就是，让一个任务的期望运行时间，通过虚拟时间的变化来进行描述，而不再使用真实的时间来进行描述。

## EEVDF的核心思想
这里用e表示设置的virtual eligible time时间

他的核心思想就是，要让一个任务的新的request（原文其实挺绕的，我是真翻译不来）开始的时间满足如下的约束，这里t应该是一个新的request到达的时间
{{< katex display=true >}}
S_i(t_0^i,e)=s_i(t_0^i,t)
{{< /katex >}}
后者是表示在这个新的request开始之前，已经获得的实际的运行时间。

而前者是表示从他开始active到e的期望服务时间

在t时刻：
 - 如果完全没有lag，那么理论上在t时刻就刚好运行，即e=t
 - 但是，如果在t时刻，他获得了比理论值更多的si运行时间，那么相应的，需要把对应的虚拟开始时间向后推迟。从而给其他的进程更多的机会来运行。即e>t
 - 相反的，如果在t时刻，他获得了比理论值更少的运行时间，那么需要把e向前推，即e < t

这件事情其实也是有意义的，因为，EEVDF会选择虚拟截止时间更早的额那个，而虚拟开始时间更早了，虚拟截止时间肯定也向前推，所以，他能够获得更大的机会来运行。

根据这一思想，我们开始计算：
{{< katex display=true >}}
S_i(t_0^i,e)=s_i(t_0^i,t)
{{< /katex >}}
{{< katex display=true >}}
S_i(t_0^i,e)=w_i(V(e)-V(t_0^i))
{{< /katex >}}
所以可以有：
{{< katex display=true >}}
V(e)=V(t_i^0)+\frac{s_i(v_0^i,t)}{w_i}
{{< /katex >}}
我们上面说了，已经开始用虚拟时间来开始描述了，因此求出虚拟时间即可

同理，在这种情况下我们可以求出V（d）

在request给出了服务时间r的情况下

因为上面的公式
{{< katex display=true >}}
r=S_i(t,d)=w_i(V(d)-V(e))
{{< /katex >}}
从而有
{{< katex display=true >}}
V(d)=V(e)+\frac{r}{w_i}
{{< /katex >}}
如果，期望服务时间和实际运行时间相同（即不考虑lag的情况），令u表示实际运行时间

另外，如果一个requeset结束了，到ddl了，就立马开始下一个新的request（因为没有lag，所以按照上面的分析e=t），也就是
{{< katex display=true >}}
V_k(d)=V_{k+1}(e) (其中k表示请求的序号，也就是说第k个请求结束之后立马是k+1个请求的开始)
{{< /katex >}}
代入之后计算
{{< katex display=true >}}
V_{k+1}(e)=V_k(e)+\frac{r}{w_i}
{{< /katex >}}


## 虚拟时间在各种事件上的变化和lag的处理
这里面我们之所以说份额f是会发生变化的，从而导致S变成一个积分的形式，是因为可能有以下三种事件
 - 某个新的client加入竞争（即task加入就绪队列）
 - 某个client离开竞争（即某个task离开就绪队列）
 - 某个client改变了他的权重

而在上述的处理过程中，我们没有考虑lag的情况

但是如果某个client，带着一个不为0的lag，产生了上述的情况，该如何处理？

如果某个client，带着不为0的lag，先离开了竞争，再加入了竞争，又要如何处理lag

按照EEVDF的策略，比如某个client存在一个负的lag，说明实际用了更多的时间，而相应的其他的进程就需要做出补偿，存在一定的损失，那么EEVDF会按照weight来成比例的把这个损失分配给其他的进程。

为了说明这个问题，作者举了个例子

假设系统中有三个进程task1，task2，task3

都从t0时刻开始变成active，在t时刻，task1和task2都还在，但是task3离开了竞争。

但是task3带着一个非零的lag走的

在t0到t的时刻，因为没有人离开或者加入，并且没有改变权重，所以可以直接乘起来算S的那个积分，因此，每个进程的lag为
{{< katex display=true >}}
lag_i(t)=w_i\fra{t-t_0}{w_1+w_2+w_3}-s_i(t_0,t)
{{< /katex >}}
对于task1和task2，他实际运行的现实时间是
{{< katex display=true >}}
t-t_0-s_3(t_0,t)
{{< /katex >}}
而考虑一个离开之后的瞬间时刻t+，令t+无限接近t

然后下面这个公式是我最难以理解的，问，在t+时刻的Si是多少
{{< katex display=true >}}
S_i(t_0,t^+)=(t-t_0-s_3(t_0,t))\frac{w_i}{w_1+w_2}
{{< /katex >}}
关于这个地方的理解，我的理解是这样的，这个东西之所以能这样算，跟虚拟时间是密切相关的，也就是说，挤进来的进程越多，虚拟时间相比于真实的时间流速会越慢（参考虚拟时间的计算公式嘛，wi求和越大，V(t)在某个时间段内越小（也就是流速越慢）

在三个进程竞争情况下，虚拟时间流的更慢，两个进程竞争之后他的流速变快了，而这个公式之所以能够转换到两个进程的时候成立，我的理解就是，他保证了对进程而言的一种虚拟时间的不变性。

当然即使这么说，我也只能给一个感觉上的能行，没法给一个确定的数学或者逻辑上的说明

后面的推导就比较简单了，由于前面说
{{< katex display=true >}}
lag_i(t)=w_i\frac{t-t_0}{w_1+w_2+w_3}-s_i(t_0,t)
{{< /katex >}}
从而有
{{< katex display=true >}}
s_i(t_0,t)=w_i\frac{t-t_0}{w_1+w_2+w_3}-lag_i(t)
{{< /katex >}}
所以可以令i=3代入上面的s3(t0,t)
{{< katex display=true >}}
S_i(t_0,t^+)=(t-t_0-(w_i\frac{t-t_0}{w_1+w_2+w_3}-lag_3(t)))\frac{w_i}{w_1+w_2}
{{< /katex >}}
{{< katex display=true >}}
=(\frac{w_1+w_2}{w_1+w_2+w_3}(t-t_0)+lag_3(t))\frac{w_i}{w_1+w_2}
{{< /katex >}}
{{< katex display=true >}}
=\frac{w_i}{w_1+w_2+w_3}(t-t_0)+w_i\frac{lag_3(t)}{w_1+w_2}
{{< /katex >}}
{{< katex display=true >}}
=w_i(V(t)-V(t_0))+w_i\frac{lag_3(t)}{w_1+w_2}
{{< /katex >}}
又由于
{{< katex display=true >}}
S_i(t_0,t^+)=w_i(V(t^+)-V(t_0))
{{< /katex >}}
可得到
{{< katex display=true >}}
V(t^+)=V(t)+\frac{lag_3(t)}{w_1+w_2}
{{< /katex >}}

同样的，据此可以求出对于t+时刻task1和task2的新的lag值（我觉得上面的结论成立之后，lag值求起来是很自然的
{{< katex display=true >}}
lag_i(t^+)=lag_i(t)+w_i\frac{lag_3(t)}{w_1+w_2}
{{< /katex >}}
这也说明了他的lag被分配到两个task上。

因此，类比上面的情况，我们可以得到某个task离开之后的V(t)的变化
{{< katex display=true >}}
V(t^+)=V(t)+\frac{lag_j(t)}{\sum_{i\epsilon{A(t^+)}}w_i}
{{< /katex >}}
task加入则是
{{< katex display=true >}}
V(t^+)=V(t)-\frac{lag_j(t)}{\sum_{i\epsilon{A(t^+)}}w_i}
{{< /katex >}}
更改权重，则可以视作某个时刻，某个task离开并瞬间以新的weight加入
{{< katex display=true >}}
V(t^+)=V(t)+\frac{lag_j(t)}{\sum_{w_i}-w_j}-\frac{lag_j(t)}{\sum_{w_i}-w_j+w_j^{new}}
{{< /katex >}}
上面这三个公式，就是在考虑了lag之后，对于V(t)的修正

更为复杂的情况是，如果离开之后，并不瞬时再加入，而且带着一个lag
这篇论文给出了三个策略
 - 可以随时离开或者加入，并按照lag修正，但是会有一些公平性上的问题
 - 让一个client离开之后就没有lag了。这个更接近实时
 - 只有lag=0才允许修改或者离开或者加入

# 关于实现
论文实际上是一个平衡树的操作，也许在1995年需要写入，但是2025年，对于树的操作应该算是计算机教育里面必备的内容，我觉得毫无介绍价值

为了维护request的顺序，他做了一棵树。包括三个接口
 - insert request
 - delete request
 - get request

在维护request的基础上
实现了client的几个函数接口
 - join
 - leave
 - change weight
对应上面的三个公式，也就是对lag进行一定的处理
另外，还有得到get_current_vt和update_lag的函数。
我觉得对于上面的分析过程看懂了的话，并没有任何的难点，直接看论文即可。
