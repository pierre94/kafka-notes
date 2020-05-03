# Flink使用kafka-connector消费数据时看不到consumer-id等信息

## 问题
### 复现
使用connecor消费数据的时候，我们```./bin/kafka-consumer-groups.sh```查看消费的情况时发现异常
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410211255846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMxMjgyNjI=,size_16,color_FFFFFF,t_70)
而使用kafka-client的时候，这些信息是能正常显示的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410211350247.png)
### 初步结论

https://issues.apache.org/jira/browse/FLINK-11325

connecter消费数据的时候，使用 ```./bin/kafka-consumer-groups.sh```就是无法获取   CONSUMER-ID     HOST            CLIENT-ID等值。因为Flink实现connecter的时候，就没有使用到kafka的这个feature。此时我们需要通过Flink的metric可以看到消费情况。


## 源码分析
通过源码才能看到本质。这里我们源码的版本:
- kafka:2.2.0
- flink kafka-connector:1.9

### KafkaConsumer实现
我们使用kafka-client的时候，一般使用KafkaConsumer构建我们的消费实例，使用poll来获取数据:
```java
KafkaConsumer<Integer, String> consumer = new KafkaConsumer<>(props);
……
ConsumerRecords<Integer, String> records = consumer.poll();
```
我们看下```org.apache.kafka.clients.consumer.KafkaConsumer```核心部分
```java
    private KafkaConsumer(ConsumerConfig config, Deserializer<K> keyDeserializer, Deserializer<V> valueDeserializer) {
        try {
            String clientId = config.getString(ConsumerConfig.CLIENT_ID_CONFIG);
            if (clientId.isEmpty()) // 
                clientId = "consumer-" + CONSUMER_CLIENT_ID_SEQUENCE.getAndIncrement();
            this.clientId = clientId;
            this.groupId = config.getString(ConsumerConfig.GROUP_ID_CONFIG);
            LogContext logContext = new LogContext("[Consumer clientId=" + clientId + ", groupId=" + groupId + "] ");
            this.log = logContext.logger(getClass());
            boolean enableAutoCommit = config.getBoolean(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG);
            if (groupId == null) { // 未指定groupId的情况下，不能设置”自动提交offset“
                if (!config.originals().containsKey(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG))
                    enableAutoCommit = false;
                else if (enableAutoCommit)
                    throw new InvalidConfigurationException(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG + " cannot be set to true when default group id (null) is used.");
            } else if (groupId.isEmpty())
                log.warn("Support for using the empty group id by consumers is deprecated and will be removed in the next major release.");

            log.debug("Initializing the Kafka consumer");
            this.requestTimeoutMs = config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG);
            this.defaultApiTimeoutMs = config.getInt(ConsumerConfig.DEFAULT_API_TIMEOUT_MS_CONFIG);
            this.time = Time.SYSTEM;

            Map<String, String> metricsTags = Collections.singletonMap("client-id", clientId);
            MetricConfig metricConfig = new MetricConfig().samples(config.getInt(ConsumerConfig.METRICS_NUM_SAMPLES_CONFIG))
                    .timeWindow(config.getLong(ConsumerConfig.METRICS_SAMPLE_WINDOW_MS_CONFIG), TimeUnit.MILLISECONDS)
                    .recordLevel(Sensor.RecordingLevel.forName(config.getString(ConsumerConfig.METRICS_RECORDING_LEVEL_CONFIG)))
                    .tags(metricsTags);
            List<MetricsReporter> reporters = config.getConfiguredInstances(ConsumerConfig.METRIC_REPORTER_CLASSES_CONFIG,
                    MetricsReporter.class, Collections.singletonMap(ConsumerConfig.CLIENT_ID_CONFIG, clientId));
            reporters.add(new JmxReporter(JMX_PREFIX));
            this.metrics = new Metrics(metricConfig, reporters, time);
            this.retryBackoffMs = config.getLong(ConsumerConfig.RETRY_BACKOFF_MS_CONFIG);

            // load interceptors and make sure they get clientId
            Map<String, Object> userProvidedConfigs = config.originals();
            userProvidedConfigs.put(ConsumerConfig.CLIENT_ID_CONFIG, clientId);
            List<ConsumerInterceptor<K, V>> interceptorList = (List) (new ConsumerConfig(userProvidedConfigs, false)).getConfiguredInstances(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG,
                    ConsumerInterceptor.class);
            this.interceptors = new ConsumerInterceptors<>(interceptorList);
            if (keyDeserializer == null) {
                this.keyDeserializer = config.getConfiguredInstance(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, Deserializer.class);
                this.keyDeserializer.configure(config.originals(), true);
            } else {
                config.ignore(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG);
                this.keyDeserializer = keyDeserializer;
            }
            if (valueDeserializer == null) {
                this.valueDeserializer = config.getConfiguredInstance(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, Deserializer.class);
                this.valueDeserializer.configure(config.originals(), false);
            } else {
                config.ignore(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG);
                this.valueDeserializer = valueDeserializer;
            }
            ClusterResourceListeners clusterResourceListeners = configureClusterResourceListeners(keyDeserializer, valueDeserializer, reporters, interceptorList);
            this.metadata = new Metadata(retryBackoffMs, config.getLong(ConsumerConfig.METADATA_MAX_AGE_CONFIG),
                    true, false, clusterResourceListeners); 
            List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(
                    config.getList(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG), config.getString(ConsumerConfig.CLIENT_DNS_LOOKUP_CONFIG));
            this.metadata.bootstrap(addresses, time.milliseconds());
            String metricGrpPrefix = "consumer";
            ConsumerMetrics metricsRegistry = new ConsumerMetrics(metricsTags.keySet(), "consumer");
            ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(config, time);
            IsolationLevel isolationLevel = IsolationLevel.valueOf(
                    config.getString(ConsumerConfig.ISOLATION_LEVEL_CONFIG).toUpperCase(Locale.ROOT));
            Sensor throttleTimeSensor = Fetcher.throttleTimeSensor(metrics, metricsRegistry.fetcherMetrics);
            int heartbeatIntervalMs = config.getInt(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG);

            NetworkClient netClient = new NetworkClient(
                    new Selector(config.getLong(ConsumerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), metrics, time, metricGrpPrefix, channelBuilder, logContext),
                    this.metadata,
                    clientId,
                    100, // a fixed large enough value will suffice for max in-flight requests
                    config.getLong(ConsumerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                    config.getLong(ConsumerConfig.RECONNECT_BACKOFF_MAX_MS_CONFIG),
                    config.getInt(ConsumerConfig.SEND_BUFFER_CONFIG),
                    config.getInt(ConsumerConfig.RECEIVE_BUFFER_CONFIG),
                    config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG),
                    ClientDnsLookup.forConfig(config.getString(ConsumerConfig.CLIENT_DNS_LOOKUP_CONFIG)),
                    time,
                    true,
                    new ApiVersions(),
                    throttleTimeSensor,
                    logContext);
            this.client = new ConsumerNetworkClient(
                    logContext,
                    netClient,
                    metadata,
                    time,
                    retryBackoffMs,
                    config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG),
                    heartbeatIntervalMs); //Will avoid blocking an extended period of time to prevent heartbeat thread starvation
            OffsetResetStrategy offsetResetStrategy = OffsetResetStrategy.valueOf(config.getString(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG).toUpperCase(Locale.ROOT));
            this.subscriptions = new SubscriptionState(offsetResetStrategy);
            this.assignors = config.getConfiguredInstances(
                    ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
                    PartitionAssignor.class);

            int maxPollIntervalMs = config.getInt(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG);
            int sessionTimeoutMs = config.getInt(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG);
            // 指定groupId才会构建coordinator。这个coordinator后面会协调消费情况
            this.coordinator = groupId == null ? null :
                new ConsumerCoordinator(logContext,
                        this.client,
                        groupId,
                        maxPollIntervalMs,
                        sessionTimeoutMs,
                        new Heartbeat(time, sessionTimeoutMs, heartbeatIntervalMs, maxPollIntervalMs, retryBackoffMs),
                        assignors,
                        this.metadata,
                        this.subscriptions,
                        metrics,
                        metricGrpPrefix,
                        this.time,
                        retryBackoffMs,
                        enableAutoCommit,
                        config.getInt(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG),
                        this.interceptors,
                        config.getBoolean(ConsumerConfig.EXCLUDE_INTERNAL_TOPICS_CONFIG),
                        config.getBoolean(ConsumerConfig.LEAVE_GROUP_ON_CLOSE_CONFIG));
            // 构建获取数据的fetcher
            this.fetcher = new Fetcher<>(
                    logContext,
                    this.client,
                    config.getInt(ConsumerConfig.FETCH_MIN_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.FETCH_MAX_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG),
                    config.getInt(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.MAX_POLL_RECORDS_CONFIG),
                    config.getBoolean(ConsumerConfig.CHECK_CRCS_CONFIG),
                    this.keyDeserializer,
                    this.valueDeserializer,
                    this.metadata,
                    this.subscriptions,
                    metrics,
                    metricsRegistry.fetcherMetrics,
                    this.time,
                    this.retryBackoffMs,
                    this.requestTimeoutMs,
                    isolationLevel);

            config.logUnused();
            AppInfoParser.registerAppInfo(JMX_PREFIX, clientId, metrics);
            log.debug("Kafka consumer initialized");
        } catch (Throwable t) {
            // call close methods if internal objects are already constructed; this is to prevent resource leak. see KAFKA-2121
            close(0, true);
            // now propagate the exception
            throw new KafkaException("Failed to construct kafka consumer", t);
        }
    }
```
从这里我们大致了解了KafkaConsumer的构建过程，知道了client-id是由当参数传递给KafkaConsumer，或者是有client按“consumer-序列号”规则生成。
而consumer-id是有服务端生成，其过程:

KafkaConsumer实例构建后，会向服务端发起JOIN_GROUP操作
```kafkaApis```
```java
ApiKeys.JOIN_GROUP;
```
```handleJoinGroupRequest```=> ```handleJoinGroup```
```java
case Some(group) =>
  group.inLock {
    if ((groupIsOverCapacity(group)
          && group.has(memberId) && !group.get(memberId).isAwaitingJoin) // oversized group, need to shed members that haven't joined yet
        || (isUnknownMember && group.size >= groupConfig.groupMaxSize)) {
      group.remove(memberId)
      responseCallback(joinError(JoinGroupRequest.UNKNOWN_MEMBER_ID, Errors.GROUP_MAX_SIZE_REACHED))
    } else if (isUnknownMember) {
      doUnknownJoinGroup(group, requireKnownMemberId, clientId, clientHost, rebalanceTimeoutMs, sessionTimeoutMs, protocolType, protocols, responseCallback)
    } else {
      doJoinGroup(group, memberId, clientId, clientHost, rebalanceTimeoutMs, sessionTimeoutMs, protocolType, protocols, responseCallback)
    }

    // attempt to complete JoinGroup
    if (group.is(PreparingRebalance)) {
      joinPurgatory.checkAndComplete(GroupKey(group.groupId))
    }
  }
```
第一次请求时服务端没有分配memberId(即consumerId),按```isUnknownMember```处理
```java
// doUnknownJoinGroup
val newMemberId = clientId + "-" + group.generateMemberIdSuffix
```
```java
  def generateMemberIdSuffix = UUID.randomUUID().toString
```



### FlinkKafkaConsumer实现
Flink的通用kafka-connector部分源码:

FlinkKafkaConsumer
```java
private FlinkKafkaConsumer(
	List<String> topics,
	Pattern subscriptionPattern,
	KafkaDeserializationSchema<T> deserializer,
	Properties props) {

	super(
		topics,
		subscriptionPattern,
		deserializer,
		getLong(
			checkNotNull(props, "props"),
			KEY_PARTITION_DISCOVERY_INTERVAL_MILLIS, PARTITION_DISCOVERY_DISABLED),
		!getBoolean(props, KEY_DISABLE_METRICS, false));

	this.properties = props;
	setDeserializer(this.properties);

	// configure the polling timeout
	try {
		if (properties.containsKey(KEY_POLL_TIMEOUT)) {
			this.pollTimeout = Long.parseLong(properties.getProperty(KEY_POLL_TIMEOUT));
		} else {
			this.pollTimeout = DEFAULT_POLL_TIMEOUT;
		}
	}
	catch (Exception e) {
		throw new IllegalArgumentException("Cannot parse poll timeout for '" + KEY_POLL_TIMEOUT + '\'', e);
	}
}
```


connector拉取数据的逻辑见```org.apache.flink.streaming.connectors.kafka.internal.KafkaFetcher```

```java
@Override
public void runFetchLoop() throws Exception {
	try {
		final Handover handover = this.handover;

		// kick off the actual Kafka consumer
		consumerThread.start();

		while (running) {
			// this blocks until we get the next records
			// it automatically re-throws exceptions encountered in the consumer thread
			final ConsumerRecords<byte[], byte[]> records = handover.pollNext();

			// get the records for each topic partition
			for (KafkaTopicPartitionState<TopicPartition> partition : subscribedPartitionStates()) {
                                // 这里拉数据
				List<ConsumerRecord<byte[], byte[]>> partitionRecords =
					records.records(partition.getKafkaPartitionHandle());

				for (ConsumerRecord<byte[], byte[]> record : partitionRecords) {
					final T value = deserializer.deserialize(record);

					if (deserializer.isEndOfStream(value)) {
						// end of stream signaled
						running = false;
						break;
					}

					// emit the actual record. this also updates offset state atomically
					// and deals with timestamps and watermark generation
					emitRecord(value, partition, record.offset(), record);
				}
			}
		}
	}
	finally {
		// this signals the consumer thread that no more work is to be done
		consumerThread.shutdown();
	}

	// on a clean exit, wait for the runner thread
	try {
		consumerThread.join();
	}
	catch (InterruptedException e) {
		// may be the result of a wake-up interruption after an exception.
		// we ignore this here and only restore the interruption state
		Thread.currentThread().interrupt();
	}
}
```
```org.apache.kafka.clients.consumer.ConsumerRecords```从指定partition获取数据
```java
/**
 * Get just the records for the given partition
 * 从指定partition获取数据
 * @param partition The partition to get records for
 */
public List<ConsumerRecord<K, V>> records(TopicPartition partition) {
    List<ConsumerRecord<K, V>> recs = this.records.get(partition);
    if (recs == null)
        return Collections.emptyList();
    else
        return Collections.unmodifiableList(recs);
}
    
```


## 一句话总结
connector自己实现了FlinkKafkaConsumer,且没有按照kafka的feature实现coordinator以及JOIN_GROUOP的逻辑，导致我们传统认为理所当的值没有正常显示。





