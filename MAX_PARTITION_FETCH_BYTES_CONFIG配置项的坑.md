# MAX_PARTITION_FETCH_BYTES_CONFIG的坑

## 0.9.0.1的缺陷
生产端配置参数
```java
        props.put("client.id", "DemoProducer");
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.BATCH_SIZE_CONFIG,"100000");
        props.put(ProducerConfig.LINGER_MS_CONFIG,"1000");
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG,"snappy");
```
```java
        props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, IntegerDeserializer.class.getName());
        props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");
//      0.9.0.1 不支持
//        props.setProperty(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "100");
        props.setProperty(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG, "100");
```

Exception in thread "main" org.apache.kafka.common.errors.RecordTooLargeException: There are some messages at [Partition=Offset]: {kafka_test-0=6720248} whose size is larger than the fetch size 100 and hence cannot be ever returned. Increase the fetch size, or decrease the maximum message size the broker will allow.

## 2.2.0