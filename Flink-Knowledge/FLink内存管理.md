# Flink 内存管理

如今，大多数的大数据框架（Hadoop、Spark、Flink等）都是基于JVM的，基于JVM的大数据引擎需要将大量的数据加载至内存当中，这就不得不面对JVM存在的几个问题：

1. java对象的存储密度低。个只包含 boolean 属性的对象占用了16个字节内存：对象头占了8个，boolean 属性占了1个，对齐填充占了7个。
2. Full GC会极大的影响性能，尤其是为了处理大数据而开了很大内存空间的JVM来说，GC会达到秒级甚至分钟级别。
3.  OOM影响稳定性。OutOfMemoryError是分布式计算框架经常会遇到的问题，当JVM中所有对象大小超过分配给JVM的内存大小时，就会发生OutOfMemoryError错误，导致JVM崩溃，分布式框架的健壮性和性能都会受到影响。

所以目前，新生代的大数据框架都开始自己管理JVM内存了，比如Spark的Tungsten，为的就是获得像 C 一样的性能以及避免 OOM 的发生。

## 积极的内存管理

FLink并不是将大量的对象存在堆上，而是将对象都序列化到一个预分配的内存块上，即MemorySegment。它代表了一段固定长度的内存（默认大小为 32KB），也是 Flink 中最小的内存分配单元，并且提供了非常高效的读写方法。MemorySegment非常像java.nio.ByteBuffer。它的底层可以是一个普通的 Java 字节数组（`byte[]`），也可以是一个申请在堆外的 `ByteBuffer`。每条记录都会以序列化的形式存储在一个或多个`MemorySegment`中。

Flink 中的 Worker 名叫 TaskManager，是用来运行用户代码的 JVM 进程。TaskManager 的堆内存主要被分成了三个部分：

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\taskManager-merroy.md)

- **Network Buffers** ： 一定数量的32KB大小的 buffer，主要用于数据的网络传输。在 TaskManager 启动的时候就会分配。默认数量是 2048 个，可以通过 `taskmanager.network.numberOfBuffers` 来配置。
- **Memory Manager Pool**：这是一个由 `MemoryManager` 管理的，由众多`MemorySegment`组成的超大集合。Flink 中的算法（如 sort/shuffle/join）会向这个内存池申请 MemorySegment，将序列化后的数据存于其中，使用完后释放回内存池。默认情况下，池子占了堆内存的 70% 的大小。
- **Remaining (Free) Heap**： 这部分的内存是留给用户代码以及 TaskManager 的数据结构使用的。因为这些数据结构一般都很小，所以基本上这些内存都是给用户代码使用的。从GC的角度来看，可以把这里看成的新生代，也就是说这里主要都是由用户代码生成的短期对象。

***注意： Memory Manager Pool 主要在Batch模式下使用。在streaming模式下，该池子不会预分配内存，也不会向该池子请求内存块。也就是说该部分的内存都是可以给用户代码使用的。***

Flink 采用类似 DBMS 的 sort 和 join 算法，直接操作二进制数据，从而使序列化/反序列化带来的开销达到最小。所以 Flink 的内部实现更像 C/C++ 而非 Java。如果需要处理的数据超出了内存限制，则会将部分数据存储到硬盘上。如果要操作多块MemorySegment就像操作一块大的连续内存一样，Flink会使用逻辑视图（`AbstractPagedInputView`）来方便操作。下图描述了 Flink 如何存储序列化后的数据到内存块中，以及在需要的时候如何将数据存储到磁盘上。

![img](D:\workspace\workspace-github\Knowledge\big-data\resouces\memory-mgmt.png)

FLink这种积极的内存 管理和直接操作二进制数据的方式有以下几点好处：

- **内存运行安全**：由于所有的运行时数据结构和算法只能通过内存池申请内存，可以有效地监控剩余内存资源。在内存吃紧的情况下，算子（sort/join等）会高效的将一大批内存写到磁盘，之后再读回来，因此可以有效的避免`OutOfMemoryErrors` 。
- **减少GC压力：** 因为所有常驻型数据都以二进制的形式存在 Flink 的`MemoryManager`中，这些`MemorySegment`一直呆在老年代而不会被GC回收。其他的数据对象基本上是由用户代码生成的短生命周期对象，这部分对象可以被 Minor GC 快速回收。只要用户不去创建大量类似缓存的常驻型对象，那么老年代的大小是不会变的，Major GC也就永远不会发生。从而有效地降低了垃圾回收的压力。另外，这里的内存块还可以是堆外内存，这可以使得 JVM 内存更小，从而加速垃圾回收。





