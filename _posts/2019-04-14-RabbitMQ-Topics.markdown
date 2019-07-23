---
layout: post
title:  Topics
subtitle:   "RabbitMQ学习第五篇."
author: AdminTest
date:   2019-04-14
header-mask: 0.6
catalog:    true
header-img: "img/post-bg-rmq.png"
tags:
    - iOS
    - RabbitMQ
---

>第五篇开始

#### Topics
（使用Objective-C客户端）

在[上一个教程中](/2019/04/13/RabbitMQ-Routing/)，我们改进了日志系统。我们使用的是`direct`**代理**，而不是使用只能进行虚拟广播的`fanout`**代理**，并且有可能选择性地接收日志。

虽然使用`direct`代理改进了我们的系统，但它仍然有局限性 - 它不能基于多个标准进行路由。

在我们的日志记录系统中，我们可能不仅要根据严重性订阅日志，还要根据发出日志的源来订阅日志。

要在我们的日志记录系统中实现这一点，我们需要了解更复杂的主题交换。

#### Topic代理
发送到`topic`代理的消息不能具有任意`routingKey` - 它必须是由`"."`分隔的单词列表。单词可以是任何内容，但通常它们指定与消息相关的一些功能。一些有效的路由键示例：`“stock.usd.nyse”`，`“nyse.vmw”`，`“quick.orange.rabbit ”`。路由密钥中可以包含任意数量的单词，最多可达255个字节。

绑定密钥也必须采用相同的形式。`topic`代理背后的逻辑类似于直接代理 - 使用特定路由密钥发送的消息将被传递到与匹配绑定密钥绑定的所有队列。但是绑定键有两个重要的特殊情况：

* **\***（星号）可以替代一个单词。
* **＃**（hash）可以替换零个或多个单词。
在一个例子中解释这个是最容易的：

![python-five](https://www.rabbitmq.com/img/tutorials/python-five.png)

在这个例子中，我们将发送所有描述动物的消息。消息将与包含三个单词（两个`"."`）的路由键一起发送。路由键中的第一个单词将描述速度，第二个是颜色，第三个是物种：`“ <speed>。<color>。<species> ”`。

我们创建了三个绑定：`Q1`绑定了绑定键`“ * .orange。* ”`，`Q2` 绑定了`“ *。*。rabbit ”`和`“ lazy。＃ ”`。

这些绑定可以概括为：

* **Q1**对所有橙色动物感兴趣。
* **Q2**希望听到关于兔子的一切，以及关于懒惰动物的一切。
路由密钥设置为`“ quick.orange.rabbit ”`的消息将传递到两个队列。消息`“ lazy.orange.elephant ”`也将同时发送给他们。另一方面，`“ quick.orange.fox ”`只会转到第一个队列，而`“ lazy.brown.fox ”`只会转到第二个队列。`“ lazy.pink.rabbit ”`将仅传递到第二个队列一次，即使它匹配两个绑定。`“ quick.brown.fox ”`与任何绑定都不匹配，因此它将被丢弃。

如果我们违反合同并发送带有一个或四个单词的消息，例如`“ orange ”或“ quick.orange.male.rabbit ”`，会发生什么？好吧，这些消息将不匹配任何绑定，并将丢失。

另一方面，`“ lazy.orange.male.rabbit ”`，即使它有四个单词，也会匹配最后一个绑定，并将被传递到第二个队列。

#### 把它们放在一起
我们将在我们的日志记录系统中使用主题交换。我们将首先假设日志的路由键有两个词：`“ <facility>。<severity> ”`。

代码与上一个教程中的代码几乎相同 。

`emitLogTopic`的代码：

```objc
- (void)emitLogTopic:(NSString *)msg routingKey:(NSString *)routingKey {
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id<RMQChannel> ch = [conn createChannel];
    RMQExchange *x    = [ch topic:@"topic_logs"];

    [x publish:[msg dataUsingEncoding:NSUTF8StringEncoding] routingKey:routingKey];
    NSLog(@"Sent '%@'", msg);

    [conn close];
}
```

`receiveLogsTopic`的代码：

```objc
- (void)receiveLogsTopic:(NSArray *)routingKeys {
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id<RMQChannel> ch = [conn createChannel];
    RMQExchange *x    = [ch topic:@"topic_logs"];
    RMQQueue *q       = [ch queue:@"" options:RMQQueueDeclareExclusive];

    for (NSString *routingKey in routingKeys) {
        [q bind:x routingKey:routingKey];
    }

    NSLog(@"Waiting for logs.");

    [q subscribe:^(RMQMessage * _Nonnull message) {
        NSLog(@"%@:%@", message.routingKey, [[NSString alloc] initWithData:message.body encoding:NSUTF8StringEncoding]);
    }];
}
```

要接收所有日志：

```objc
[self receiveLogsTopic:@[@"#"]];
```

要从设施`“ kern ” `接收所有日志：

```objc
[self receiveLogsTopic:@[@"kern.*"]];
```

或者，如果您只想听到`“ critical ”`日志：

```objc
[self receiveLogsTopic:@[@"*.critical"]];
```

您可以创建多个绑定：

```objc
[self receiveLogsTopic:@[@"kern.*", @"*.critical"]];
```

并使用路由键`“ kern.critical ”`类型发出日志：

```objc
[self emitLogTopic:@"A critical kernel error" routingKey:@"kern.critical"];
```

玩这些方法玩得开心。请注意，代码不会对路由或绑定密钥做出任何假设，您可能希望使用两个以上的路由密钥参数。



#### 参考资料

[RabbitMQ官方文档：Topics](https://www.rabbitmq.com/tutorials/tutorial-five-objectivec.html)
