---
draft: true 
date: 2024-07-30
# authors:
categories:
  - 设计空间探索
  - 马恺声
---

# Inter-layer Scheduling Space Definition and Exploration for Tiled Accelerators

- 作者：马恺声， Cai Jingwei

研究加速器层间调度。提出了资源分配树Resource Allocation Tree (RA Tree),用来表示不同的空间、时间调度策略。
基于这个符号，分析不同调度策略对加速器性能、能效的影响。开发端到端的调度框架SET。与SOTA的开源 Tangram 框架
相比，实现1.78倍的性能提升和13.2%的能源成本降低。

<!-- more -->

## Introduction

![picture 1](../pictures/Inter-layer%20Scheduling%20Space%20Definition%20and%20Exploration%20for%20Tiled%20Accelerators/41da0ba6a7dbe84f678a15619870e5dcd4ada9ed6db11c33a51a429dc800ee7a.png){ width=70% }

### Tiled Accelerators 平铺加速器

加速器的硬件抽象，指支持以下行为的加速器：

- 不同层可以在不同的硬件HW-tile上并行执行，需要硬件支持对每个tile独立控制，以及支持对DRAM并发访问
- 支持不同tile之间，tile与DRAM之间通信，有灵活的NOC支持对缓存的并发访存

### intra-layer scheduling 层内调度

层内调度研究如何将某一层映射到并行计算单元上，将工作负载划分为小块来满足缓存、计算单元的限制，决定每个工作负
载的计算顺序。其中涉及到相当多的工作，例如OS(output stationary)、WS(weight stationary)、IS(input stationary)、
RS(row stationary)等等。这一类被作者称之为“启发式时期”(Heuristic Period)。随着研究的深入，层内调度进入了“自动化时期”
(Automation Period)，提出了各种表示，例如：计算中心表示(Computation-centric notation)、数据中心表示(Data-centric notation)、
关系中心表示(Relation-centric notation)等等。

### inter-layer scheduling 层间调度

层间调度主要有两种启发式模式：顺序模式LS(layer-sequential)和流水线模式LP(layer-pipeline)。LS使用所有计算资源计算一层，逐层计算，
而LP将不同层的负载分配到不同计算资源上，用流水线的模式进行并行计算。作者认为现有对层间调度的工作仍然处于启发式时期，文章对层间调度
的空间进行了符号化表示，并且进行了探索和分析。

## RA Tree

![picture 2](../pictures/Inter-layer%20Scheduling%20Space%20Definition%20and%20Exploration%20for%20Tiled%20Accelerators/98bfdee695c1ce89951ef216629a16c0b0aaea5df86dfee8eab4f60d514e7994.png)  

### 总体介绍

RA树用来表示资源的分配，资源有两类：空间资源（即计算资源）；时间资源（即计算资源的时间占用）。同时引入了时空图S-T Graph(Spatial-Temporal
Graph)，用来直观地显示两种资源的分配。

RA树的节点包括leaf node和cut node，cut又根据空间分割和时间分割的不同，划分为S Cut和T Cut。叶子节点表示处理单个层，叶子节点数量等于DNN的层数。

每个节点分配一组HW-tiles，成为HW-tile group，负责该组节点的工作负载。每个节点有自己的batch size，每个cut node会分为多个子节点，batch
也会被切分。Leaf node不会被切分。

构建RA tree时要考虑层与层之间的依赖关系。Leaf node会根据依赖顺序从左到右来构建RA tree。

### 空间、时间资源划分

![picture 3](../pictures/Inter-layer%20Scheduling%20Space%20Definition%20and%20Exploration%20for%20Tiled%20Accelerators/faf10c2672a6e4c07f10dffffcbc251d3d4829f4ac285ee79dcaac5f79010d04.png)  

**S Cut** 对空间计算资源进行划分。S Cut的子节点并行执行或者流水线执行，取决于是否存在依赖关系。如果存在依赖关系，则后继节点必须等待至少
一个批次的时间，等待前继节点处理完产生一个批次的输出特征图。如Figure 3 LP所示。L1执行完[0,1]的batch，L2才开始执行。

**T Cut** 对时间资源进行划分，对同一个HW-tile而言，一个T Cut表示的一个batch的工作负载被划分为多个subbatch，不同的subbatch在时间上从左
到右依次顺序处理。如Figure 3 LS所示。

对于一个n层的DNN，只考虑树的结构以及两种Cut的类型，划分的空间复杂度为 $O((5+2\sqrt{6})^n) \approx O({9.899}^n)$。如果要计算空间的总大小，
需要乘以DNN的拓扑阶数，以及每个Cut的subbatch划分数量。

### 数据流

对于feature map，如果是T cut的子节点，那么fmap会被送到DRAM中。如果是S cut的子节点，当左节点的数据会立即被右子节点消耗时，
那么数据会从一个HW-tile传输到下一个HW-tile。如果数据独立不会立即被消耗，那么会被存储在DRAM中。

(*难道f ram size 足够的时候，也要存回DRAM？*)

对于Weight，如果是T cut的子节点，每一层的权重会从DRAM中load，如果是S cut的子节点，权重会一直固定在芯片上。

(*难道w ram size 足够的时候，也要不停load？*)

### HW-tiles的分配<a id="HW-tiles的分配"></a>

一般是按照不同层的OPS等比例分配，但是实际情况很难这么理想。

1. 相同OPS的不同层在同一个硬件上执行效率不同
2. 不同层之间有数据依赖，不能同时开始计算，至少延迟一个subbatch的时间

考虑这两个因素，引入了归一化处理时间 Normalized Porcessing Time(NPT)，能计算相关的利用率。由于硬件资源划分是离散的，没法完全根据NPT来成比例
划分，因此会有bubble，导致利用率下降。

### LP和LS的表示

一般节点会被切分成多个segment。LP会分配给每个segment不同的HW-tile，LS会分配给每个segment全部的HW-tile。LP应该是S cut，LS应该是T cut。
这样调度空间的复杂度为 $O(2^n)$。

(*作者说LS处理有依赖的不同层时，数据会通过NOC传递而不需要经过DRAM。难道不是LP才是这样？*)

??? quote "原文"

    ``` plaintext
    In contrast, LS processes each layer in a segment with all the HW-tiles and switches the fmaps of the layers with dependencies 
    through the on-chip buffer without accessing the DRAM.
    ```

### 定性分析

![picture 4](../pictures/Inter-layer%20Scheduling%20Space%20Definition%20and%20Exploration%20for%20Tiled%20Accelerators/53093a036b060eebb84319dd7d72e40ca48bb3df4a4cc449788fdd022916c394.png)  

S Cut给每个工作负载分配到的HW-tile group比较小。小的HW-tile好处有：

1. 更少的数据重复。每个HW-tile都需要有一份input feature拷贝，HW-tile group小意味着拷贝的数据少。
2. 更大的tile内搜索空间。对于同样大小的subbatch，更少的算力意味着更大的intra-tile搜索空间，可能会带来更高的利用率。

也有缺点：

1. 带来建立/退出流水的延迟开销 filling and draining overheads。当处理有数据依赖的层时，需要等待前一层subbatch计算完成。
2. 气泡开销 bubble overheads。参考[HW-tiles的分配](#HW-tiles的分配)
3. 数据预取开销。如果HW-tile group太小没法缓存一层的所有数据，那么会带来额外的DRAM访存开销。

较小的subbatch有一些好处：

1. 更小的建立/退出流水的延迟开销 filling and draining overheads。因为计算更细粒度。
2. 需要的缓存容量更小。

缺点是更小的粒度，会导致探索空间更小，影响计算资源的利用率、能效。


