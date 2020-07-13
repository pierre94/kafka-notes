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

## jmxterm
https://github.com/jiaqi/jmxterm

```shell script
java -jar jmxterm-1.0.1-uber.jar -l ip:port
```