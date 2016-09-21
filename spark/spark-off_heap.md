在文章的开头，安利一下我自己的github上的一个项目：[AlluxioBlockManager](https://github.com/chengqiangboy/spark-alluxio-blockstore)，同时还有我的github上的博客：[blog](https://github.com/chengqiangboy/blog)
这个项目的作用是替代Spark2.0以前默认的`TachyonBlockManager`，稍后解释为什么要重新开发AlluxioBlockManager，以及Spark2.0的off_heap。

#OFF_HEAP
Spark中RDD提供了几种存储级别，不同的存储级别可以带来不同的容错性能，例如 `MEMORY_ONLY`,`MEMORY_ONLY_SER_2`...其中，有一种特别的是`OFF_HEAP`
`off_heap`的优势在于，在内存有限的条件下，减少不必要的内存消耗，以及频繁的GC问题，提升程序性能。
Spark2.0以前，默认的off_heap是Tachyon，当然，你可以通过继承`ExternalBlockManager` 来实现你自己想要的任何off_heap。
这里说Tachyon，是因为Spark默认的TachyonBlockManager开发完成之后，就再也没有更新过，以至于Tachyon升级为Alluxio之后移除不使用的API，导致Spark默认off_heap不可用，这个问题Spark社区和Alluxio社区都有反馈：
https://alluxio.atlassian.net/browse/ALLUXIO-1881

#Spark2.0的off_heap
从spark2.0开始，社区已经移除默认的TachyonBlockManager以及ExternalBlockManager相关的API：[SPARK-12667](https://issues.apache.org/jira/browse/SPARK-12667)。
那么，问题来了，在Spark2.0中，OFF_HEAP是怎么处理的呢？数据存在哪里？
上代码：
首先，在StorageLevel里面，不同的存储级别解析成不同的构造函数，从OFF_HEAP的构造函数可以看出来，OFF_HEAP依旧存在。
```
Object StorageLevel {
val NONE = new StorageLevel(false, false, false, false)
val DISK_ONLY = new StorageLevel(true, false, false, false)
val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
val MEMORY_ONLY = new StorageLevel(false, true, false, true)
val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
val OFF_HEAP = new StorageLevel(false, false, true, false)
              ......  ........
}
```
在`org.apache.spark.memory`中，有一个`MemoryMode`，`MemoryMode`标记了使用`ON_HEAP`还是`OFF_HEAP`,在`org.apache.spark.storage.memory.MemoryStore`中，根据`MemoryMode`类型来调用不同的存储
```
def putBytes[T: ClassTag](    
  blockId: BlockId,    
  size: Long,    
  memoryMode: MemoryMode,
   _bytes: () => ChunkedByteBuffer): Boolean = {
      .............
      val entry = new SerializedMemoryEntry[T](bytes, memoryMode, implicitly[ClassTag[T]])
      entries.synchronized { 
         entries.put(blockId, entry)
      }
      .............
  }
```
再看`MemoryStore`中存数据的方法：`putIteratorAsBytes`
```
val allocator = memoryMode match {  
  case MemoryMode.ON_HEAP => ByteBuffer.allocate _ 
  case MemoryMode.OFF_HEAP => Platform.allocateDirectBuffer _
}
```
终于找到Spark2.0中off_heap的底层存储了：`Platform`是利用java unsafe API实现的一个访问off_heap的类。

#总结
spark2.0 off_heap就是利用java unsafe API实现的内存管理。
优点：依然可以减少内存的使用，减少频繁的GC，提高程序性能。
缺点：从代码中看到，使用OFF_HEAP并没有备份数据，也不能像alluxio那样保证数据高可用，丢失数据则需要重新计算。