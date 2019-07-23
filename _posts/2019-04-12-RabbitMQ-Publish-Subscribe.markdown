---
layout: post
title:  Publish/Subscribe
subtitle:   "RabbitMQ学习第三篇."
author: AdminTest
date:   2019-04-12
header-mask: 0.6
catalog:    true
header-img: "img/post-bg-rmq.png"
tags:
    - iOS
    - RabbitMQ
---

>第三篇开始

#### 发布和订阅
（使用Objective-C客户端）


在[之前的教程](/2019/04/11/RabbitMQ-Work-Queues/)，我们创建了一个工作队列。工作队列背后的假设是每个任务都交付给一个特定的工作者。在这一部分，我们将做一些完全不同的事情 - 我们将向多个消费者传递信息。此模式称为**“发布/订阅”**。

为了说明这种模式，我们将构建一个简单的日志记录系统。它将包含两个程序 - 第一个将发出日志消息，第二个将接收和打印它们。

在我们的日志系统中，接收器的每个运行副本都将获得消息。这样我们就可以运行一个接收器并将日志指向磁盘; 同时我们将能够运行另一个接收器并在屏幕上看到日志。

基本上，发布的日志消息将被广播给所有接收者。

#### 代理
在本教程的前几部分中，我们向队列发送消息和从队列接收消息。现在是时候在RabbitMQ中引入完整的消息传递模型了。

让我们快速浏览前面教程中介绍的内容：

* 生产者是发送消息的用户应用程序。
* 队列是存储消息的缓冲器。
* 消费者是接收消息的用户应用程序。

RabbitMQ中消息传递模型的核心思想是生产者永远不会将任何消息直接发送到队列。实际上，生产者通常甚至不知道消息是否会被传递到任一队列。

相反，生产者只能向**代理**发送消息。**代理**是一个非常简单的东西。一方面，它接收来自生产者的消息，另一方面将它们推送到队列。**代理**必须确切知道如何处理它收到的消息。它应该附加到特定队列吗？它应该附加到许多队列吗？或者它应该被丢弃？其规则由交换类型定义。

![exchanges](https://www.rabbitmq.com/img/tutorials/exchanges.png)

有几种交换类型可供选择：`direct`, `topic`, `headers` and `fanout`。我们将专注于最后一个 - `fanout`。让我们创建一个这种类型的交换，并将其称为日志：

```objc
[ch fanout:@"logs"];
```

fanout代理非常简单。正如您可能从名称中猜到的那样，它只是将收到的所有消息广播到它知道的所有队列中。而这正是我们记录器所需要的。

#### 默认代理
在本教程的前几部分中，我们对**代理**一无所知，但仍然可以向队列发送消息。这是可能的，因为我们使用的是默认代理，由空字符串（`“”`）标识。

回想一下我们之前如何发布消息：

```objc
[ch.defaultExchange publish:@"hello" routingKey:@"hello" persistent:YES];
```

这里我们使用默认或无名代理：消息被路由到具有`routingKey`指定名称的队列（如果存在）。

现在，我们可以发布到我们命名的`代理`：

```objc
RMQExchange *x = [ch fanout:@"logs"];
[x publish:[msg dataUsingEncoding:NSUTF8StringEncoding]];
```

#### 临时队列
您可能还记得以前我们使用过具有特定名称的队列（还记得`hello`和`task_queue`吗？）。能够命名队列对我们来说至关重要 - 我们需要将worker指向同一个队列。当您想要在生产者和消费者之间共享队列时，为队列命名非常重要。

但我们的日志系统并非如此。我们希望了解所有日志消息，而不仅仅是它们的一部分。我们也只对目前流动的消息感兴趣，而不是旧消息。要解决这个问题，我们需要两件事。

首先，每当我们连接到RabbitMQ时，我们都需要一个新的空队列。为此，我们可以创建一个具有随机名称的队列，或者更好 - 让SDK为我们选择一个随机队列名称（其他客户端将此工作留给服务器，但是因为`Objective-C`客户端旨在避免阻塞调用线程，它更喜欢生成自己的名字）。

其次，一旦我们断开消费者，就应该自动删除队列。

在[Objective-C](https://github.com/rabbitmq/rabbitmq-objc-client)客户端中，当我们将队列名称作为空字符串提供时，我们创建一个具有生成名称的非持久队列：

```objc
RMQQueue *q = [ch queue:@"" options:RMQQueueDeclareExclusive];
```

方法返回时，队列实例包含SDK生成的随机队列名称。例如，它可能看起来像  `rmq-objc-client.gen-049F8D0B-F330-4D65-9277-0F418F529A93-41604-000030FB39652E07`。

当声明它的连接关闭时，队列将被删除，因为它被声明为独占。您可以在队列指南中了解有关独占标志和其他队列属性的更多信息。

#### 绑定

![bindings](https://www.rabbitmq.com/img/tutorials/bindings.png)

我们已经创建了一个`fanout`代理和一个队列。现在我们需要告诉交换机将消息发送到我们的队列。交换和队列之间的关系称为绑定。

```objc
[q bind:x];
```

从现在开始，日志`代理`会将消息附加到我们的队列中。

把它们放在一起

![python-three-overall](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)

发出日志消息的producer方法与前一个教程没有太大的不同。最重要的变化是我们现在想要将消息发布到我们的日志代理而不是无名代理。这里是`emitLog`的代码 ：

```objc
RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
[conn start];

id<RMQChannel> ch = [conn createChannel];
RMQExchange *x = [ch fanout:@"logs"];

NSString *msg = @"Hello World!";

[x publish:[msg dataUsingEncoding:NSUTF8StringEncoding]];
NSLog(@"Sent %@", msg);

[conn close];
```

如您所见，在建立连接后，我们宣布了`代理`。此步骤是必要的，因为禁止发布到不存在的`代理`。

如果没有队列绑定到`代理`，消息将会丢失，但这对我们没有问题; 如果没有消费者在听，我们可以安全地丢弃该消息。

`receiveLogs`的代码：

```objc
RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
[conn start];

id<RMQChannel> ch = [conn createChannel];
RMQExchange *x = [ch fanout:@"logs"];
RMQQueue *q = [ch queue:@"" options:RMQQueueDeclareExclusive];

[q bind:x];

NSLog(@"Waiting for logs.");

[q subscribe:^(RMQMessage * _Nonnull message) {
    NSLog(@"Received %@", [[NSString alloc] initWithData:message.body encoding:NSUTF8StringEncoding]);
}];
```

要了解如何监听消息的子集，让我们继续学习 [第四部分](/2019/04/13/RabbitMQ-Routing/)



#### 参考资料

[RabbitMQ官方文档：Publish/Subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-objectivec.html)
