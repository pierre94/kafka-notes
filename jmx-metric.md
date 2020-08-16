# jmx-metric

## 简介
JMX的全称为Java Management Extensions. 顾名思义，是管理Java的一种扩展。这种机制可以方便的管理、监控正在运行中的Java程序。常用于管理线程，内存，日志Level，服务重启，系统环境等。


## 问题

> 调用远程主机上的 RMI 服务时抛出 java.rmi.ConnectException: Connection refused to host: 127.0.0.1 异常原因及解决方案

```java
        // ipa=`ip addr | grep -E 'state UP|state UNKNOWN' -A2|grep "link/ether" -A1|grep eth| tail -n1 | awk '{print $2}' | cut -f1 -d '/'`
        // sed -i "s/com.sun.management.jmxremote.ssl=false/& -Djava.rmi.server.hostname=$ipa/g" kafka-run-class.sh
        // java.rmi.server.hostname 上面配置后，就能客户端访问到，不然只能本地访问
```

## 工具集合
### jmxterm
https://github.com/jiaqi/jmxterm

```shell script
# uber是指with-dependence的意思
java -jar jmxterm-1.0.1-uber.jar -l ip:port
```

## 统一JMX监控

### 方案一:

jmx-exporter+client-conf.yml

Yet another example Kafka client configs
https://github.com/prometheus/jmx_exporter/pull/413

> jmx-exporter 貌似不赞同此方案,上面pr一直没有合入

### 方案二:
https://github.com/sysco-middleware/kafka-client-collector

嵌入到自己的程序里，统一以一个prometheus http server 对外。

进度:

- 测试通过

- 思考: 以prometheus push gateway方案 效果会不会更好一点

## 参考资料
http://cloudurable.com/blog/kafka-tutorial-kafka-producer-advanced-java-examples/index.html