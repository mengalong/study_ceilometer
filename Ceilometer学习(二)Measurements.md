# Measurements
　Telemetry 通过轮询架构中的元素以及消费通过其他服务通知的消息来收集各种meters。通过轮询和消息消费模式获取到的meters有多种类型，如下是原生支持的：

Comulative：随时间的累积值

Delta：变化值

Gauge：离散或者波动值

Telemetry支持存储抽样值。这些metadata可以被扩展以供Compute和对象存储使用
