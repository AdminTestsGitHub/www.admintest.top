---
layout: post
title:  Routing
subtitle:   "RabbitMQ学习第四篇."
author: AdminTest
date:   2019-04-13
header-mask: 0.6
catalog:    true
header-img: "img/post-bg-rmq.png"
tags:
    - iOS
    - RabbitMQ
---

>第四篇开始

#### Routing
（使用Objective-C客户端）


在[上一个教程中](/2019/04/12/RabbitMQ-Publish-Subscribe/)，我们构建了一个简单的日志系统 我们能够向许多接收者广播日志消息。

在本教程中，我们将为其添加一个功能 - 我们将只能订阅一部分消息。例如，我们只能将关键错误消息定向到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息。


#### 绑定
在前面的例子中，我们已经创建了绑定。您可能会记得以下代码：

```objc
[q bind:exchange];
```

绑定是交换和队列之间的关系。这可以简单地理解为：队列对来自此**代理**的消息感兴趣。

绑定可以采用额外的`routingKey`参数。为了避免与`RMQExchange publish：`参数混淆，我们将其称为**绑定密钥**。这就是我们如何使用键值创建绑定：

```objc
[q bind:exchange routingKey:@"black"];
```

**绑定密钥**的含义取决于**代理**类型。我们之前使用的`fanout`**代理**只是忽略了它的价值。

#### direct代理
我们上一个教程中的日志记录系统向所有消费者广播所有消息。我们希望扩展它以允许根据消息的严重性过滤消息。例如，我们可能希望将日志消息写入磁盘的脚本仅接收严重错误，而不是在警告或日志消息上浪费磁盘空间。

我们使用的是`fanout`代理，它没有给我们太大的灵活性 - 它只能进行无意识的广播。

我们将使用`direct`代理。`direct`代理背后的路由算法很简单 - 消息进入队列，其  绑定密钥与消息的路由密钥完全匹配。

为了说明这一点，请考虑以下设置：
![direct-exchange](https://www.rabbitmq.com/img/tutorials/direct-exchange.png)


在此设置中，我们可以看到`direct`代理`X`与两个绑定到它的队列。第一个队列绑定橙色绑定，第二个绑定有两个绑定，一个绑定密钥为黑色，另一个绑定为绿色。

在这样的设置中，使用路由密钥`orange`发布到交换机的消息将被路由到队列Q1。路由键值为黑色 或绿色的消息将转到Q2。所有其他消息将被丢弃。

在第一篇教程中，我们编写了从命名队列发送和接收消息的方法。在这一篇中，我们将创建一个工作队列，用于在多个worker之间分配耗时的任务。

#### 多个绑定
![direct-exchange-multiple](https://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

使用相同的绑定密钥绑定多个队列是完全合法的。在我们的示例中，我们可以在`X`和`Q1`之间添加绑定键黑色的绑定。在这种情况下，`direct`代理将表现得像`fanout`一样，并将消息广播到所有匹配的队列。路由键值为黑色的消息将传送到`Q1`和`Q2`。

#### 发送日志
我们将此模型用于我们的日志系统。我们会将消息发送给`direct`代理，而不是`fanout`。我们将提供日志严重性作为路由密钥。这样接收方法将能够选择它想要接收的严重性。让我们首先关注发送日志。

一如既往，我们需要先创建一个代理：

```objc
[ch direct:@"logs"];
```

我们已准备好发送消息：

```objc
RMQExchange *x = [ch direct:@"logs"];
[x publish:[msg dataUsingEncoding:NSUTF8StringEncoding] routingKey:severity];
```
为简化起见，我们假设“严重性”可以是“信息”，“警告”，“错误”之一。

#### 订阅
接收消息将像上一个教程一样工作，但有一个例外 - 我们将为我们感兴趣的每个严重性创建一个新的绑定。

```objc
RMQQueue *q = [ch queue:@"" options:RMQQueueDeclareExclusive];

NSArray *severities = @[@"error", @"warning", @"info"];
for (NSString *severity in severities) {
    [q bind:x routingKey:severity];
}
```

#### 把它们放在一起
![python-four](https://www.rabbitmq.com/img/tutorials/python-four.png)
`emitLogDirect`方法的代码：

```objc
- (void)emitLogDirect:(NSString *)msg severity:(NSString *)severity {
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id<RMQChannel> ch = [conn createChannel];
    RMQExchange *x    = [ch direct:@"direct_logs"];

    [x publish:[msg dataUsingEncoding:NSUTF8StringEncoding] routingKey:severity];
    NSLog(@"Sent '%@'", msg);

    [conn close];
}
```

`receiveLogsDirect`的代码：

```objc
- (void)receiveLogsDirect {
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id<RMQChannel> ch = [conn createChannel];
    RMQExchange *x    = [ch direct:@"direct_logs"];
    RMQQueue *q       = [ch queue:@"" options:RMQQueueDeclareExclusive];

    NSArray *severities = @[@"error", @"warning", @"info"];
    for (NSString *severity in severities) {
        [q bind:x routingKey:severity];
    }

    NSLog(@"Waiting for logs.");

    [q subscribe:^(RMQMessage * _Nonnull message) {
        NSLog(@"%@:%@", message.routingKey, [[NSString alloc] initWithData:message.body encoding:NSUTF8StringEncoding]);
    }];
}
```

要发出错误日志消息，只需调用：

```objc
[self emitLogDirect:@"Hi there!" severity:@"error"];
```

转到[第五部分](/2019/04/14/RabbitMQ-Topics/)，了解如何根据模式监听消息。


#### 参考资料

[RabbitMQ官方文档：Routing](https://www.rabbitmq.com/tutorials/tutorial-four-objectivec.html)
