# GC

## 机制介绍

垃圾回收（[Garbage Collection](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))）是一种自动内存管理机制，定期从根对象（主要指全局变量、寄存器、栈）开始，遍历所有内存空间，找到所有不存在引用关系的对象，最终回收这些对象占用的内存空间。GC 主要是清理堆中的对象，栈中的变量因为栈本身后进先出的特点，其内存管理较为简单，不需要较为复杂的 GC 机制。

GC 能够帮助开发者自动的将无用的数据所占内存还给操作系统，减少了一定的开发成本，许多编程语言都采用了这一机制，例如 GO，Java 等。但是 GC 会引入额外的耗时问题，在扫描内存期间，需要暂停整个程序，即 STW（Stop the world），存在一定的性能问题。

除此以外，还有一种被称作引用计数（[Automatic Reference Counting](https://en.wikipedia.org/wiki/Automatic_Reference_Counting)）的自动内存管理机制，为所有创建的对象分配一个计数器，当该对象被其他对象引用时计数加一，解除引用时计数减一，并定期回收所有计数为零的对象所占用的内存空间。

ARC 不会导致明显的卡顿问题，Objective-C、Python 等语言使用这种方式进行内存管理。但是会由于代码导致循环引用，两个或多个对象的计数器永远不会归零。对于这种场景，编程语言会通过 weak 关键字来声明弱引用，破坏循环，但是仍会因为代码问题，引发内存泄漏

与自动内存管理机制相对应的，是手动内存管理机制，例如在 C/C++ 中，需要开发者手动申请和释放内存空间，在性能上有较大优势，但是手动管理也最容易出现内存泄漏问题。另外，这些编程语言中，也会有三方库提供了 GC 的实现。

## 回收算法

GC 根据其回收算法不同，也有一些差异：

- 标记 - 清除：暂停程序中的所有线程，由 GC 线程从根对象开始，找到并标记堆中所有存活的对象，之后遍历堆中所有对象，回收所有未被标记到的对象，释放所占用的内存，最后恢复其他线程的运行
- 标记 - 压缩：流程与“标记-清除”类似，但会将被保留的对象迁移至连续的内存空间中，避免内存的碎片化
- 复制：将内存空间分为两部分，程序运行中的内存分配仅在其中一个半区进行，触发 GC 时，将保留的对象迁移至另外一个半区，然后回收当前半区的数据，以此循环
- 增量式：将内存空间分为若干区，每次只对某一区进行 GC
- 分代：将内存空间划分为年轻代与老年代，内存分配在年轻代进行，每次 GC 后会将保留的对象的寿命加一，达到阈值后迁移至老年代。针对不同区块，执行不同的 GC 策略。年轻代的对象生命周期较短，GC 较为频繁，老年代的对象生命周期较长，GC 的频率较低。某些场景下还会有不进行 GC 的永久代，用于存储数据常量等。

Golang 中的 GC 采用了“标记 - 清除”的方案，并在此基础了做了较多的优化，来减少 STW 的时间。最初的一个优化方案，是提前将关闭 STW 的时间提前。在最初的“标记-清除”算法中，STW 覆盖了整体的操作，流程为：

- 开启 STW -> 标记 -> 清除 -> 关闭 STW
  
但是实际上在清除阶段，所要删除的元素已经不可达了，所以可以提前关闭 STW，减少 STW 覆盖的时间，优化后的流程为：

- 开启 STW -> 标记 -> 关闭 STW -> 清除

## 三色标记算法

三色标记算法对“标记 - 清除”算法中的标记流程，做了一定的优化，提供了并发的基础。

三色标记算法将程序中的对象分为白、黑、灰三类：

- 白色：所有元素初始均为白色，当扫描完成后，白色对象被认为是不可达的对象
- 黑色：完成扫描，确定存活的对象，黑色对象所引用的所有白色对象，在扫描期间均会被标记为灰色
- 灰色：被找到但是本身未完成扫描的对象，可能引用着白色对象

在具体执行时，相当于从根对象开始进行广度优先的遍历，具体流程如下：

- 在 GC 开始时，将所有对象标记为白色，并将根对象标记为灰色
- 从灰色对象集合中选择一个，将其标记为黑色
- 扫描该黑色对象所引用的对象，将其标记为灰色
- 重复上述流程，直至灰色对象集合为空，仅存在黑色和白色对象

在三色标记算法执行中，其正确性依赖于层次遍历的结果，如果用户在遍历期间，使某一个黑色对象引用了一个新白色对象，当回收器没有办法从当前灰色对象访问到新白色对象，最终就没有办法对其进行标记，从而当作垃圾错误回收。所以三色标记算法在执行标记时，仍然需要 STW，想要并发或者增量式地执行标记过程，需要使用内存屏障进行保障。

## 屏障技术

[内存屏障](https://en.wikipedia.org/wiki/Memory_barrier)是一类同步屏障类型的指令，它可以使得 CPU 或编译器在对内存进行操作时，确保其操作在屏障前和屏障后有着严格的顺序。

可以按照其功能划分为混合屏障、读屏障、写屏障，混合屏障会保障在程序并发执行时，所有早于屏障的对于内存读或写的操作完成后，再执行晚于屏障的读写操作，读屏障、写屏障同理。

通过屏障技术，使得我们在并发时，能够保障一部分流程，严格按照我们预想的顺序去执行，进而达成想要的效果

### 三色不变性

在上文我们举例了一个场景，当我们修改黑色对象，使其引用一个新增的白色对象时，最终这个新对象会被错误的回收。通过对这些特殊场景做归纳，最终可以总结出如下场景

- 对白色或灰色对象做修改，不会导致问题，因为此时收集器还未对他们进行扫描操作
- 对黑色对象做修改，增加或删除对于黑色或灰色对象的引用，也不会导致问题，因为收集器已经对他们做了标记，完成或将要完成扫描任务
- 对黑色对象做修改，增加对于白色对象的引用，则可能会导致问题，因为该白色对象实际是可遍历到的，但是收集器不会从黑色对象重新进行遍历，如果此时不存在其他从灰色对象到该白色对象的遍历路径，则该白色对象对于收集器来说就是不可遍历到的，从而被错误删除

值得注意的是，对于 GC 而言，我们需要保障的是从根对象可触达的对象不会被错误删除，某些不可触达的对象如果没有被删除（在标记完成后，变为不可触达）是可以接受的。

对如上出现问题的场景做分析，可以明确想要确保标记过程的正确性，需要满足两种三色不变式（[Tricolour invariant](https://dl.acm.org/doi/10.1145/301589.286863)）中的一种：

- 强三色不变式：黑色对象不能指向白色对象，只能指向黑色对象或灰色对象
- 弱三色不变式：黑色对象所指向的白色对象，必须存在一条从灰色对象到该白色对象的可达路径

在此基础上，我们额外引入一些写屏障，在恰当的时机，例如黑色对象引用白色对象时，做一些处理，使得其满足三色不变式，即可保障标记过程在并发或增量的环境下，仍然是有效的。

### 插入写屏障

Dijkstra 提出了一种插入写屏障，在引用新对象时，直接将新对象修改为灰色，以满足强三色不变式。

```go
func updatePoint[T any](fieldPtr *T, newValuePtr T) {
    greyMark(newValuePtr)
    *fieldPtr = newValuePtr
}
```

但是对于栈中的对象来说，改动较为频繁，引入插入写屏障，性能开销较大，一般不会引入插入写屏障。此时，则需要额外针对于栈中对象启动 STW，重新进行扫描。

### 删除写屏障

Yuasa 提出了一种删除写屏障，在删除对于旧元素的引用时，如果旧元素是白色，则将旧元素标记为灰色，确保被删除元素所引用的元素仍然可以被收集器访问到，以满足弱三色不变式。

```go
func updatePoint[T any](fieldPtr *T, newValuePtr T) {
    if isWhite(*fieldPtr) {
        greyMark(*fieldPtr)
    }
    *fieldPtr = newValuePtr
}
```

这种写屏障方案可以保证 GC 开始时的所有对象，在 GC 期间全部都会被扫描到，故也被称作快照垃圾回收（Snapshot GC）。但是对于在 GC 期间新创建的对象，可能会被遗漏标记。

### 混合写屏障

Golang 在某个版本中，基于以上两种屏障技术，实现了一种混合写屏障，能够结合两者的优点，满足了弱三色不变式，同时又避免了对于栈开启 STW 进行二次扫描，进一步减少了 GC 期间整体 STW 的时间。

混合写屏障具体方案如下：

- GC 开始时，将栈中所有可达对象全部标记为黑色
- GC 期间，栈中新创建的对象直接标记为黑色
- 被删除的对象标记为灰色
- 被添加的对象标记为灰色

可以看到，这种方案包含了插入写屏障与删除写屏障的所有内容，并且对于不启用内存屏障的栈元素来说，确保了在 GC 结束时，所有可达元素全部为黑色，避免了二次扫描，进一步减少 STW 的时间。其缺点就是写屏障的性能开销翻倍。

## GC 流程

写屏障的使用保障了 GC 的标记期间，即使与用户程序并发执行，最终也可满足对所有可触达对象的正常标记，此时，完整的 GC 流程如下所示：

- 清理终止阶段：
  - 开启 STW
  - 等待所有所有 Goroutine 进入安全点（程序执行时的一个安全点，可以执行抢占式调度）
- 标记阶段：
  - 切换状态，开启写屏障，将根对象入队
  - 关闭 STW，并发执行带有写屏障的三色标记流程
- 标记终止阶段：
  - 开启 STW，切换状态位
  - 清理线程缓存
- 内存清理阶段：
  - 修改状态位，初始化清理状态，关闭写屏障
  - 关闭 STW，清理待回收的内存