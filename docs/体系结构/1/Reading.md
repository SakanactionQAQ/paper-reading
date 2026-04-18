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

##### idea

对于生成的预取信息，采用一定的策略进行过滤，过滤掉重复与效果不好的预取。

在Triage的基础上，本文章认为积极的预取器是可以提高性能的，并且发现通过提高前置的偏移量来存储马尔可夫表中的非相邻条目来提高及时性，使用元数据格式来解决物理地址的非连续性。

提出了Triagel的预取器，
* 使用了新的Samplers来观察长期的数据复用情况来发现规律
* 添加了Meta Reuse Buffer，在不增加L3Cache通信负担的情况下来实现更积极的数据预取
* 用Set-Dueller替代了Bloom-filter sizing mechanism，能够模拟缓存分区配置来得到预取与缺失的折中。

### how achieved

#### training table

当L2发生缺失或者prefetch命中时进行更新。内容包括：
* PC-Tag：使用哈希进行压缩
* LastAddr $[0, 1]$ ：存储了该指令处发生过的缺失和预取，作为一个移位寄存器？当History Sampler将这条指令分为积极的，就会进入马尔可夫表。每当有新的值被填入时，$[0] \to [1]$  而Addr $[1]$ 作为马尔可夫表的索引
* Timestamp（时间戳）：用来计算同一个PC两次重复之间的间隔
* ReuseConfidence：是一个饱和计数器用来评估x重复出现的模式
* PatternConfidence：两个4比特的饱和计数器，用来评估预取的正确率。前一个饱和计数器的置信度达到66%时，就可以进行元数据的存储和预取；后一个饱和计数器在置信度达到83%时发起高强度预取
* SampleRate：控制采样率
* Lookahead：保存时使用LastAddr $[0]$ 还是 $[1]$ 作为马尔可夫表训练的索引
#### 马尔可夫表

仍然放再L3中，用10位hash来压缩lookup address，用31位存储完整的target，用一个confidence位来标记是否替换：替换置为0，当training table与markov表中的一致时置为1
#### History Sampler

用来决定是否要存储一个给定PC的历史信息。通过对于training table中的元数据进行采样来实现。包含了 addr-tag，train-idx，target，timestamp，accessed。

History决定了当前的PC信息是否要写入到training table中，每当要发生更新时，对应training table中的addr $[0]$ 就在history sampler中被查找，当发生命中，并且二者时间相隔较近就可以考虑来更新training table。当HS中的target与预取器当前训练的得到的target相同时，就可以对于PatternConf++。

##### Resue confidence

当PC命中所对应的training table中的Addrs0与HS命中时，说明我们在同一条指令访问了同一个memory，就说明了这个地址对中的x出现的模式可信度上升了

##### Pattern conf and Second-Chance Sampler

当我们生成了新的prefetch关系时，并不能直接进行替换，因为单独对于预取数据的重新访问并不代表了prefetch的有用（这个prefetch后续未必会被用到）。只有当一个patter（x，y）重复出现时，对于Patternconf++。

如果target 已经在cache中了，那么patternconf不变；但如果pattern 不在cache中

pattern 计数器通过向上取1向下取2与向上取1向下取5的方式来强化不同重复模式的置信度

对于 Second-Chance Sampling 当一个地址对再 History Sampler 中出现，但不在Training Table 中时，我们在这个缓冲区中存储y和时间戳，如果再512次训练访问内访问了y，我们就增加置信度，反之降低置信度。

##### 抽样方式

通过表中的 SampleRate 动态控制抽样率。SampleRate 是一个4位的饱和计数器，初始化为8。将对应表项插入 History Sampler 的概率为

$$
\frac{SamplerSize}{MaxSize}\cdot 2^{SamplerRate - 8}
$$

Maxsize 是马尔可夫表中的最大缓存区分配的表项数目，SamplerSize 是抽样器中的表项数目。

#### 激进策略优化

Traige 采用的每个 miss 产生更多的预取虽然提高了预取的及时性，但是却降低了正确率并提高了通信成本。

在 Triagel 中，对于 ReuseConf 和 PatternConf 都很高的PC来说，高程度的预取也可以保持较高的正确率。

除了高程度预取， Triangel 还增加了预取的前瞻距离，从与预取下一个缺失可能发生的地址变为预取下下一个缺失可能发生的地址从而来缓解由于读表带来的延迟。改变前瞻距离需要在马尔可夫表的存储方式上直接发生改变，因此这个前瞻的距离必须是在长时间里是稳定的。

当 HighPatternConf 达到最大值15时，前瞻距离置为2；当 BasePatternConf 回到8时置为1.

当 HighPatternConf 高于默认值8时，进行一个链式的预取，连续在马尔可夫表上去除四个prefetch

#### Metadata Reuse Buffer


## What is the author’s evaluation of the solution? What logic, argument, evidence, artifacts (e.g., a proof-of-concept system), or experiments are presented in support of the idea?

### evidence

与单步跨度的预取器相比，实现了26.4%的加速，与Triage相比提升了9.3%

相比于Traige，内存信息流值提高了10%，在效率和表现上都有显著提升。

## What is your analysis of the identified problem, idea and evaluation? Is this a good idea? What flaws do you  perceive in the work? What are the most interesting or controversial ideas? For work that has practical implications, ask whether this will work, who would want it, what it will take to give it to them, and when it might become reality?


## What are the paper's contribution?

## What are the future directions for this research?

## What questions are you left with?

