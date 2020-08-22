# Kafka术语
- Abort 中止
- expires到期\失效
> 请求超时等场景用到

- Thunk 一个回调以及传递给它的关联FutureRecordMetadata参数。
> A callback and the associated FutureRecordMetadata argument to pass to it.

- drain 排空
> RecordAccumulator中将缓存的ProducerBatch排空，并整理成按节点对应的列表 `Map<Integer, List<ProducerBatch>>`

- Mute 静音
> mute all the partitions drained 
> 如果需要保证消息的强顺序性(maxInflightRequests == 1)，则缓存对应 topic 分区对象，防止同一时间往同一个 topic 分区发送多条处于未完成状态的消息。
> 实际上就是将本批次消息所在的分区信息添加到一个集合中，不能再往这个分区里排空数据，以保障每个topic下的该分区只有一个批次发送

- collated 整理
> `Map<Integer, List<ProducerBatch>> collated`drain后生成 经过`整理`的数据集

- magic 消息格式
> A record batch is a container for records. In old versions of the record format (versions 0 and 1)

