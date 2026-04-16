## What is your take-away message from this paper?

## What is the motivation for this work (both people problem and technical problem), and its distillation into a research question? Why doesn’t the people problem have a trivial solution? What are the previous solutions and why are they inadequate?

### motivation：

由于内存墙的存在，cache缺失成为了延时的重要原因之一。商用处理器的解决方法就是使用预取器来提前从memory中取出数据从而避免缺失。目前temporal prefetcher开始在真实的处理核心中被使用。

已有的一种处理方法是Correlated Miss Chaining temporal prefetcher，通过存储大量的在cache片段中必要的元数据。这种结构在之前的Triage中被使用过。

### research question

本篇文章要解决的两个问题是

* Triage的设计是否真实可行，需要进行复现
* 是否可以进行改进

###  previous solution and limatation

#### solution

Triage中存储了 $(x, y)$ 地址对，当 $x$ 在L2发生缺失时（或者是预取出来的数据第一次被使用）， $y$ 就会被预取出来， $y$ 是 $x$ 最后一次被取到cache后发生的第一个缺失的地址。

Traige有一个PC索引的训练表，记录了那些指令对于哪些地址发生了缺失，之后根据缺失的地址找到下一个要预取的地址。

这种临时预取器产生了一定的影响，有一定的应用。

#### limitation

本篇文章认为Triage的部分特性在物理上是无法实现的，部分则是受制于复杂性和芯片面积。并且在边缘情况下很多设计性能会大幅下降。

Triage的正确率不到50%

Triage实现马尔可夫表的过程是通过将lookup address的地址砍去6位offset，以及用来索引的11位，剩余的tag利用10位映射大lookup table中，而target除了11位存到了表中其余并无区别。这里用lookup address找target就必须要反向用tag找到马尔可夫表的entry，非常的不方便，需要对于lookup table进行遍历。

而Triage-ISR的方式则是使用hash的方式来存储多余的tag，但是会引起target最终对应多个目标地址的问题。

原方法的内存替换策略相比于简单的LRU没有实现好的性能提升，没有能很好的学习的prefetch的pattern


## What is the proposed solution (hypothesis, idea, design)? Why is it believed it will work? How does it represent an improvement? How is the solution achieved?

### solution

#### 问题一

对于第一个问题，由于之前研究细节上的缺失，本研究中使用了最现代的方式进行模拟

##### 马尔可夫元数据表

对于Pre_addr首先在马尔可夫表中定位，然后再通过lookuptable找到prefetch target

lookup table存在一个SRAM中，容量受到马尔可夫表中的索引位的限制。lookup_table可以视为一个按index的array

文章最终采用了lookup address tag部分hash而target直接存储的方式

马尔可夫表存储的最终实现使用16路组相联并采用了二级编码，再每一个cache line内部同样设计为多路的。同时也进行了动态设计，与分区大小的变化是对应的，在每次分区大小发生变化时，对于整个存储表要进行一次重排以保证寻址的正确性
#### 问题二

在Triage的基础上，本文章认为积极的预取器是可以提高性能的，并且发现通过提高前置的偏移量来存储马尔可夫表中的非相邻条目来提高及时性，使用元数据格式来解决物理地址的非连续性。

提出了Triagel的预取器，
* 使用了新的Samplers来观察长期的数据复用情况来发现规律
* 添加了Meta Reuse Buffer，在不增加L3Cache通信负担的情况下来实现更积极的数据预取
* 用Set-Dueller替代了Bloom-filter sizing mechanism，能够模拟缓存分区配置来得到预取与缺失的折中。


## What is the author’s evaluation of the solution? What logic, argument, evidence, artifacts (e.g., a proof-of-concept system), or experiments are presented in support of the idea?

### evidence

与单步跨度的预取器相比，实现了26.4%的加速，与Triage相比提升了9.3%

相比于Traige，内存信息流值提高了10%，在效率和表现上都有显著提升。