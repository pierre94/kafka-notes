# 深入理解RecordBatch

## 一个小问题
> 假如我生产客户端batch.size设置为512000，则生产客户端往服务端发送最大的数据包就是512000么？

答案显然不是。我们需要确认几个事实:

- batch.size是指生成一个RecordBatch的最大大小，对应0.8客户端的batch.num.messages是指发送时条数
- 客户端往服务端发送最大数据包是由max.request.size这个参数决定的