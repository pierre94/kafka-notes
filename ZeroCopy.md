# ZeroCopy

## 系列前言
kafka作为一个处理实时数据和日志的管道,每秒可以处理几十万条消息。其瓶颈自然也在I/O层面，所以其高吞吐背后离不开如下几个特性:
- NIO
- 磁盘顺序读写
- Queue数据结构的极致使用
- 分区提高并发
- 零拷贝提高效率
- 异步刷盘
- 压缩提高数据传输效率 

本次我将从kafka的源码分析其ZeroCopy模块的细节。

## ZeroCopy基础概念
### 传统IO
- JVM虚拟机发送一个read()操作系统级别的方法
- 产生一个上下文的切换，从程序所在的用户空间切换至系统的内核空间
- 内核空间向磁盘空间请求数据，通过DMA直接内存访问的方式将数据读取到内核空间缓冲区
- 用户空间是无法直接使用,需要将这份缓冲数据原封不动的拷贝到用户空间
- 在用户空间里read数据

有两次上下文的切换，和两次数据的拷贝。

### ZeroCopy是什么
零拷贝是指计算机操作的过程中，CPU不需要为数据在内存之间的拷贝消耗资源。而它通常是指计算机在网络上发送文件时，不需要将文件内容拷贝到用户空间（User Space）而直接在内核空间（Kernel Space）中传输到网络的方式。

零拷贝技术减少了用户态与内核态之间的切换，让拷贝次数降到最低，从而实现高性能。


### Java中的ZeroCopy
在Java中的零拷贝实现是在NIO的FileChannel中，其中有个方法
- transferTo
- transferFrom




## kafka实现
### 具体使用
在org.apache.kafka.common.record.FileRecords中具体使用到了ZeroCopy。
kafak版本:2.2.0 https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/common/record/FileRecords.java

```java
// org.apache.kafka.common.record.FileRecords
    @Override
    public long writeTo(GatheringByteChannel destChannel, long offset, int length) throws IOException {
        long newSize = Math.min(channel.size(), end) - start;
        int oldSize = sizeInBytes();
        if (newSize < oldSize)
            throw new KafkaException(String.format(
                    "Size of FileRecords %s has been truncated during write: old size %d, new size %d",
                    file.getAbsolutePath(), oldSize, newSize));

        long position = start + offset;
        int count = Math.min(length, oldSize);
        final long bytesTransferred;
        if (destChannel instanceof TransportLayer) {
            TransportLayer tl = (TransportLayer) destChannel;
            bytesTransferred = tl.transferFrom(channel, position, count);
        } else {
            bytesTransferred = channel.transferTo(position, count, destChannel);
        }
        return bytesTransferred;
    }
```

### 使用场景
- partition leader到follower的消息同步
- consumer拉取partition中的消息

后面讲具体分析这块的逻辑。

------
## 参考
- Kafka Zero-Copy 使用分析 https://www.jianshu.com/p/d47de3d6d8ac    