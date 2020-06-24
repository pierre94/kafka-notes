# kafka nio相关

`SocketServer.scala`模块

## nio基础
https://tech.meituan.com/2016/11/04/nio.html

## kafka网络模型
```java
/**
 * An NIO socket server. The threading model is
 *   1 Acceptor thread that handles new connections
 *   Acceptor has N Processor threads that each have their own selector and read requests from sockets
 *   M Handler threads that handle requests and produce responses back to the processor threads for writing.
 */
```
```java
//selector.select(300)会阻塞到0.3s后然后接着执行，相当于0.3s轮询检查
```

详见[网络模型](./网络模型.md)

## 0.9.0.1的一个bug复现
> https://www.jianshu.com/p/d2cbaae38014

