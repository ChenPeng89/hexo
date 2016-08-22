---
title: Disruptor学习笔记:一、入门
date: 2016-06-06 10:41:46
tags: Disruptor
---
## 前言
Disruptor是LMAX公司开源的一个高性能的线程间通讯库，它关注的是并发，高性能和非阻塞算法，现在已经是LMAX交易系统的核心构件。

Disruptor是一个非阻塞队列，它的作用是在同一进程中的几个线程间移动数据。和普通队列比起来，它有以下特点：

- 通过Consumer依赖关系图将Events广播给Consumer。
- 为Events提前分配内存。
- 无锁（可选）。

## 核心概念：

- Ring Buffer: 用于对通过Disruptor移动的数据进行存储和更新，在一些比较复杂的场景中，可以由用户自定义实现。
- Sequence: 使用递增的序号来管理进行数据交换的Event，Disruptor对数据的处理过程总是按照序号逐个递增。因此，Sequence可以用来跟踪标识事件处理者（RingBuffer/Consumer）的处理进度。Sequence包含了AtomicLong的大部分并发特性，但是Sequence可以防止不同Sequence之间CPU缓存伪共享的问题。
- Sequencer: Sequencer是Disruptor的核心。它有两个实现，single producer, multi producer，定义了producer和consumer之间快速、正确的传递数据（Event）的并发算法。
- Sequence Barrier: 它由Sequencer创建，它包含了Sequencer发布的主要Sequence和所有相关消费者的Sequence的引用。它还能决定是否有供消费者来消费的Event的逻辑。
- Wait Strategy: Wait Strategy决定了consumer怎样等待producer放入Disruptor的数据。
- Event: 从producer到consumer的数据传递单元。它是由用户自己定义的。Disruptor中没有任何关于它的代码。
- EventProcessor:主要的事件循环，用于处理Disruptor中的Event，并且拥有消费者的Sequence。它有一个实现类是BatchEventProcessor，包含了event loop有效的实现，并且将回调到一个EventHandler接口的实现对象。
- EventHandler: Disruptor 定义的事件处理接口，由用户实现，用于处理事件，是 Consumer 的真正实现。
- Producer: 即生产者，只是泛指调用 Disruptor 发布事件的用户代码，Disruptor 没有定义特定接口或类型。

![](http://i.imgur.com/I5EnXdb.png)

## 广播事件
这个是Disruptor和队列的最大的不同。当有多个consumer监听同一个Disruptor，那么Event会发布给所有的consumer，而队列，则是一个单独的Event只能发给一个consumer。Disruptor一般在当你需要并行处理相同的数据的情况下使用。
在LMAX中的典型例子是，我们有三个操作，传输（将输入的数据写入持久化文件），复制（将输入的数据发送到远程机器上备份），业务逻辑处理。当使用Executor并行处理不同事件时，可以使用WorkerPool。

## Consumer依赖图
为了支持现实中的并发处理，consumer之间的协作是非常必要的。参考上图的例子，保证bussiness logic consumer在journalling和replication consumer执行完再执行是非常必要的。我们将这种概念成为gating。gating出现在两种情况。首先，我们要保证消费者不会发生过载情况。这可以通过为Disruptor添加相关的Consumer来实现（调用RingBuffer.addGatingConsumers()方法）。其次，前面提到的那种情况可以通过创建包含Sequence的SequenceBarrier来实现，Sequence需要在保证先完成它们处理逻辑的组件（比如类似bussiness logic consumer等）里面创建。

## Event内存预分配
Disruptor一个重要的目标就是低延迟。在低延迟系统中，减少和消除内存分配是很必要的。在基于java的系统中，目标就是减少由于gc引起的延迟。
为了解决这个问题，Disruptor可以为Events预分配内存。在用户提供构造器和EventFactory时，它们将会被RingBuffer中的每一个实体调用。当发布新数据到Disruptor时，API允许用户得到构造对象，以便于它们能够在那个存储对象上调用方法或更新域。Disruptor提供了保证，只要它们正确的实现，那么这些操作将是线程安全的。

## 无锁（可选）
另一个实现低延迟的关键细节就是大规模使用无锁算法来实现Disruptor。所有内存的可见性和正确性是通过使用 memory barriers（内存屏障，就是它让一个处理器内的内存状态对其他处理器可见。） 或 compare-and-swap（CAS,CAS有三个操作参数：内存地址，期望值，要修改的新值，当期望值和内存当中的值进行比较不相等的时候，表示内存中的值已经被别线程改动过，这时候失败返回，当相等的时候，将内存中的值改为新的值，并返回成功。） 来实现的。