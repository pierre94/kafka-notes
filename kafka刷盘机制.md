# kafka刷盘机制

## 系列前言
kafka作为一个处理实时数据和日志的管道,每秒可以处理几十万条消息。其瓶颈自然也在I/O层面，所以其高吞吐背后离不开如下几个特性:
- NIO
- 磁盘顺序读写
- Queue数据结构的极致使用
- 分区提高并发
- 零拷贝提高效率
- 异步刷盘
- 压缩提高数据传输效率 

本次我将从kafka-2.2.0的源码分析其顺序写入与刷盘机制的细节。




## 一、日志写入

### 1.Log文件结构简介
kafka的日志文件

```kafka-logs/${topic}-${partition}/:```
```
|-- 00000000000000000000.index
|-- 00000000000000000000.log
|-- 00000000000000000000.timeindex
|-- 00000000000000002309.snapshot
|-- 00000000000000167080.snapshot
`-- leader-epoch-checkpoint
```

A segment of the log 由3部分组成:
- log: FileRecord,即实际的消息
- index: 索引
- timeindex: 时间索引

其中:
- 命名规则为 segment 文件最后一条消息的 offset 值。
- log.segment.bytes 日志切割(默认 1G)

这里不详细介绍Log文件结构以及message从接收到处理的过程.

(补充问题:在partition中如何通过offset查找message)

### 2.写入过程
```java
// org.apache.kafka.common.record.FileRecords
/**
 * Append a set of records to the file. This method is not thread-safe and must be
 * protected with a lock.
 *
 * @param records The records to append
 * @return the number of bytes written to the underlying file
 */
public int append(MemoryRecords records) throws IOException {
    if (records.sizeInBytes() > Integer.MAX_VALUE - size.get())
        throw new IllegalArgumentException("Append of size " + records.sizeInBytes() +
                " bytes is too large for segment with current file position at " + size.get());

    int written = records.writeFullyTo(channel);
    size.getAndAdd(written);
    return written;
}

/**
 * Write all records to the given channel (including partial records).
 * @param channel The channel to write to
 * @return The number of bytes written
 * @throws IOException For any IO errors writing to the channel
 */
public int writeFullyTo(GatheringByteChannel channel) throws IOException {
    buffer.mark();
    int written = 0;
    while (written < sizeInBytes())
        written += channel.write(buffer);
    buffer.reset();
    return written;
}
```

- 每个分区写入过程没有带offset,这种append-only的写法保证了顺序写入,一定程度降低磁盘负载(避免随机写操带来的频繁磁盘寻道问题)。

- 由于kafka的网络模型是1+N+M，也就意味着M个worker线程可能写同1个log文件，append显然不是线程安全的,上层调用时需要加锁。

- 此时仅仅写入文件系统的PageCache(内存)中,
    - 不做特殊操作的话,将由操作系统决定什么时候把 OS Cache 里的数据真的刷入磁盘文件中。
    - kafka本身提供强制刷盘机制来强制刷盘,下文将详细介绍


附加锁写入:

Log -> LogSegment -> FileRecords
```java
// kafka.log.Log

private def append() ={
    ……
    lock synchronized {
    ……
        segment.append(largestOffset = appendInfo.lastOffset,
          largestTimestamp = appendInfo.maxTimestamp,
          shallowOffsetOfMaxTimestamp = appendInfo.offsetOfMaxTimestamp,
          records = validRecords)
    ……
    }
}
```

## 二、刷盘分析
### 1.刷盘参数
kafka提供3个参数来优化刷盘机制
- log.flush.interval.messages   //多少条消息刷盘1次
- log.flush.interval.ms  //隔多长时间刷盘1次
- log.flush.scheduler.interval.ms //周期性的刷盘。

kafka配置获取入口
```java
// kafka.server.KafkaConfig

val LogFlushIntervalMessagesProp = "log.flush.interval.messages"
val LogFlushSchedulerIntervalMsProp = "log.flush.scheduler.interval.ms"
val LogFlushIntervalMsProp = "log.flush.interval.ms"
```

### 2.参数详解与刷盘源码解读
#### 2.1 log.flush.interval.messages参数
```log.flush.interval.messages```即多少条消息刷盘1次,这个参数在```Log```类中使用。


```Log```类是append-only的LogSegments的序列。在append的时候直接判断未刷新的message数是否达到阈值```log.flush.interval.messages```
```java
// kafka.log.Log
class Log(……){
    ……
    private def append(……): LogAppendInfo = {
        ……
        if (unflushedMessages >= config.flushInterval)
          flush()
        appendInfo
    }
    ……
}
```

#### 2.2 log.flush.interval.ms与log.flush.scheduler.interval.ms参数

```log.flush.interval.ms```与```log.flush.scheduler.interval.ms```2个参数需要相互配合,在```LogManager```使用。


```LogManager```类是kafka日志管理子系统的入口，负责日志的创建、检索和清理。```LogManager```启动一个调度线程，根据```flushCheckMs```周期执行```flushDirtyLogs```方法

注: ```flushCheckMs```即周期性刷盘参数```log.flush.scheduler.interval.ms```。
```java
// kafka.log.LogManager

def startup() {
/* Schedule the cleanup task to delete old logs */
if (scheduler != null) {
  info("Starting log cleanup with a period of %d ms.".format(retentionCheckMs))
  scheduler.schedule("kafka-log-retention",
                     cleanupLogs _,
                     delay = InitialTaskDelayMs,
                     period = retentionCheckMs,
                     TimeUnit.MILLISECONDS)
  info("Starting log flusher with a default period of %d ms.".format(flushCheckMs))
  scheduler.schedule("kafka-log-flusher",
                     flushDirtyLogs _,
                     delay = InitialTaskDelayMs,
                     period = flushCheckMs,
                     TimeUnit.MILLISECONDS)
  scheduler.schedule("kafka-recovery-point-checkpoint",
                     checkpointLogRecoveryOffsets _,
                     delay = InitialTaskDelayMs,
                     period = flushRecoveryOffsetCheckpointMs,
                     TimeUnit.MILLISECONDS)
  scheduler.schedule("kafka-log-start-offset-checkpoint",
                     checkpointLogStartOffsets _,
                     delay = InitialTaskDelayMs,
                     period = flushStartOffsetCheckpointMs,
                     TimeUnit.MILLISECONDS)
  scheduler.schedule("kafka-delete-logs", // will be rescheduled after each delete logs with a dynamic period
                     deleteLogs _,
                     delay = InitialTaskDelayMs,
                     unit = TimeUnit.MILLISECONDS)
}
if (cleanerConfig.enableCleaner)
  cleaner.startup()
}

```
flushDirtyLogs调用时会计算上次flush的间隔,对比```log.flush.interval.ms```配置对应的```flushMs```变量决定是否```flush```
```java
// kafka.log.LogManager

private def flushDirtyLogs(): Unit = {
debug("Checking for dirty logs to flush...")

for ((topicPartition, log) <- currentLogs.toList ++ futureLogs.toList) {
  try {
  // 求出上次flush的间隔
    val timeSinceLastFlush = time.milliseconds - log.lastFlushTime
    debug(s"Checking if flush is needed on ${topicPartition.topic} flush interval ${log.config.flushMs}" +
          s" last flushed ${log.lastFlushTime} time since last flush: $timeSinceLastFlush")
    if(timeSinceLastFlush >= log.config.flushMs)
      log.flush
  } catch {
    case e: Throwable =>
      error(s"Error flushing topic ${topicPartition.topic}", e)
  }
}
}
```
最终实现使用用```FileRecords```的```flush```方法,即调用JDK的```java.nio.channels.FileChannel```的```force```方法强制刷盘
```java
// org.apache.kafka.common.record.FileRecords

/**
 * Commit all written data to the physical disk
 */
public void flush() throws IOException {
    channel.force(true);
}

```
## 最后

实际上，官方不建议通过上述的刷盘3个参数来强制写盘。其认为数据的可靠性通过replica来保证，而强制flush数据到磁盘会对整体性能产生影响。