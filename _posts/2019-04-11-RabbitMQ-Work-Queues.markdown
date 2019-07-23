---
layout: post
title:  Work Queues
subtitle:   "RabbitMQ学习第二篇."
author: AdminTest
date:   2019-04-11
header-mask: 0.6
catalog:    true
header-img: "img/post-bg-rmq.png"
tags:
    - iOS
    - RabbitMQ
---

>第二篇开始

#### 工作队列
（使用Objective-C客户端）

![python-two](https://www.rabbitmq.com/img/tutorials/python-two.png)

在第一篇教程中，我们编写了从已有名字的队列**发送**和**接收**消息的方法。在这篇中，我们将创建一个`工作队列(Work Queue)`，用于在多个工作人员（消费者）之间分配耗时的任务。

`工作队列`（又称：**任务队列**）背后的主要思想是避免立即执行资源密集型任务，并且必须等待它完成。相反，我们安排任务稍后完成。我们将任务封装为消息并将其发送到队列。在后台运行的工作进程将弹出任务并最终执行作业。当您运行许多工作程序时，它们之间将共享任务。

这个概念在Web应用程序中特别有用，因为在这些Web应用程序中，在短的HTTP请求窗口期间处理复杂的任务是不可能的。

#### 准备
在[之前的部分](/2019/04/10/RabbitMQ-Hello-World/)，我们发送了一条包含“Hello World！”的消息。现在我们将发送代表复杂任务的字符串。我们没有真实的任务，比如要调整图片大小或要渲染pdf文件，所以让我们假装我们很忙 - 通过使用`sleep`来模拟耗时任务。我们把字符串中的`“.”`点数作为其复杂性; 每个`“.”`点都会占据“工作”的一秒钟。例如：`Hello ...` 描述的假任务 将花费三秒钟。

我们将稍微修改之前示例中的send方法，以允许将任意字符串作为方法参数发送。此方法将任务安排到我们的工作队列，因此我们将其重命名为`newTask`。除了新参数之外，实现保持不变：

```objc
- (void)newTask:(NSString *)msg {
    NSLog(@"Attempting to connect to local RabbitMQ broker");
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id<RMQChannel> ch = [conn createChannel];

    RMQQueue *q = [ch queue:@"hello"];

    NSData *msgData = [msg dataUsingEncoding:NSUTF8StringEncoding];
    [ch.defaultExchange publish:msgData routingKey:q.name];
    NSLog(@"Sent %@", msg);

    [conn close];
}
```

我们的旧接收方法需要进行一些更大的更改：它需要为消息体中的每个`“.”`点伪造一秒钟的工作。如果每个worker都有一个名字，它将帮助我们理解正在发生什么，并且每个人都需要从队列中弹出消息并执行任务，所以我们称之为`workerNamed`:

```objc
[q subscribe:^(RMQMessage * _Nonnull message) {
    NSString *messageText = [[NSString alloc] initWithData:message.body encoding:NSUTF8StringEncoding];
    NSLog(@"%@: Received %@", name, messageText);
    // imitate some work
    unsigned int sleepTime = (unsigned int)[messageText componentsSeparatedByString:@"."].count - 1;
    NSLog(@"%@: Sleeping for %u seconds", name, sleepTime);
    sleep(sleepTime);
}];
```

请注意，我们的假任务模拟执行时间。

从`viewDidLoad`运行它们:

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [self newTask:@"Hello World..."];
    [self workerNamed:@"Flopsy"];
}
```

日志输出应指示Flopsy正在睡眠三秒钟。

#### 轮询调度
使用`任务队列`的一个优点是能够轻松地并行工作。如果我们准备做很多工作，我们可以添加更多工人，这样就可以轻松扩展。

让我们尝试同时运行两个`workerNamed：`方法。他们都会从队列中获取消息，但究竟如何呢？让我们来看看。

更改`viewDidLoad`以发送更多消息并启动两个worker：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [self workerNamed:@"Jack"];
    [self workerNamed:@"Jill"];
    [self newTask:@"Hello World..."];
    [self newTask:@"Just one this time."];
    [self newTask:@"Five....."];
    [self newTask:@"None"];
    [self newTask:@"Two..dots"];
}
```

让我们看看交给worker的是什么：

```objc
# => Jack: Waiting for messages
# => Jill: Waiting for messages
# => Sent Hello World...
# => Jack: Received Hello World...
# => Jack: Sleeping for 3 seconds
# => Sent Just one this time.
# => Jill: Received Just one this time.
# => Jill: Sleeping for 1 seconds
# => Sent Five.....
# => Sent None
# => Sent Two..dots
# => Jill: Received Five.....
# => Jill: Sleeping for 5 seconds
# => Jack: Received None
# => Jack: Sleeping for 0 seconds
# => Jack: Received Two..dots
# => Jack: Sleeping for 2 seconds
```

默认情况下，RabbitMQ将按顺序将每条消息发送给下一个消费者。平均而言，每个消费者将获得相同数量的消息。这种分发消息的方式称为轮询法。与三个或更多工人一起尝试。

#### 消息确认

执行任务可能需要几秒钟。你可能想知道如果其中一个消费者开始一项长期任务并且只是部分完成而死亡会发生什么。使用我们当前的代码，一旦RabbitMQ向消费者发送消息，它立即将其标记为删除。在这种情况下，如果你杀掉一个worker，我们将丢失它刚刚处理的消息。我们还将丢失已经分发的但尚未处理的所有消息。

但我们不想失去任何任务。如果worker死亡，我们希望将任务交付给另一个worker。

为了确保消息永不丢失，RabbitMQ支持[消息确认](https://www.rabbitmq.com/confirms.html)。消费者发回ack（nowledgement）告诉RabbitMQ已收到，处理了特定消息，RabbitMQ可以自由删除它。

如果消费者死亡（其通道关闭，连接关闭或TCP连接丢失）而不发送确认，RabbitMQ将理解消息未完全处理并将重新排队。如果同时有其他在线消费者，则会迅速将其重新发送给其他消费者。这样你就可以确保没有消息丢失，即使worker偶尔会死亡。

不会有任何消息超时; 当消费者死亡时，RabbitMQ将重新发送消息。即使处理消息需要非常长的时间，也没关系。

默认情况下，客户端会关闭消息确认，但**AMQ**协议中不会关闭消息确认（`AMQBasicConsumeNoAck`选项会自动发送通过`subscribe:`)。是时候通过把`AMQBasicConsumeNoOptions`打开并在完成任务后从工作人员发送适当的确认。

```objc
RMQBasicConsumeOptions manualAck = RMQBasicConsumeNoOptions;
[q subscribe:manualAck handler:^(RMQMessage * _Nonnull message) {
    NSString *messageText = [[NSString alloc] initWithData:message.body encoding:NSUTF8StringEncoding];
    NSLog(@"%@: Received %@", name, messageText);
    // imitate some work
    unsigned int sleepTime = (unsigned int)[messageText componentsSeparatedByString:@"."].count - 1;
    NSLog(@"%@: Sleeping for %u seconds", name, sleepTime);
    sleep(sleepTime);

    [ch ack:message.deliveryTag];
}];
```

使用此代码，我们可以确定即使worker在处理消息时死亡，也不会丢失任何内容。worker死后不久，所有未经确认的消息将被重新传递。

消息确认必须在传输信息的同一频道上发送。尝试使用不同的通道进行消息确认将导致通道级协议异常。参阅[消息确认的指导文档](https://www.rabbitmq.com/confirms.html)，了解更多信息。

#### 消息持久性

我们已经学会了如何确保即使消费者死亡，任务也不会丢失。但是如果RabbitMQ服务器停止，我们的任务仍然会丢失。

当RabbitMQ退出或崩溃时，它将忘记队列和消息，除非你告诉它不要。确保消息不会丢失需要做两件事：我们需要将队列和消息都标记为持久。

首先，我们需要确保RabbitMQ永远不会丢失我们的队列。为此，我们需要声明它是持久的：

```objc
RMQQueue *q = [ch queue:@"hello" options:AMQQueueDeclareDurable];
```

虽然此命令本身是正确的，但它在我们当前的设置中不起作用。那是因为我们已经定义了一个名为`hello`的队列 ，这个队列不是持久化的。RabbitMQ不允许您使用不同的参数重新定义现有队列，并且会向尝试执行此操作的任何程序返回错误。但是有一个快速的解决方法 - 让我们声明一个具有不同名称的队列，例如`task_queue：`

```objc
RMQQueue *q = [ch queue:@"task_queue" options:AMQQueueDeclareDurable];
```

此选项：`AMQQueueDeclareDurable`需要在生产者和消费者的代码里都设置上。

此时我们确信即使RabbitMQ重新启动，`task_queue`队列也不会丢失。现在我们需要将消息标记为持久的 - 通过使用`persistent`选项。

```objc
[ch.defaultExchange publish:msgData routingKey:q.name persistent:YES];
```

**有关消息持久性的注释**
将消息标记为持久性并不能完全保证消息不会丢失。虽然它告诉RabbitMQ将消息保存到磁盘，但是当RabbitMQ接收消息并且尚未保存消息时，仍然有一个短时间窗口。此外，RabbitMQ不会为每条消息执行`fsync（2）` - 它可能只是保存到缓存而不是真正写入磁盘。持久性保证不强，但对于我们简单的任务队列来说已经足够了。如果您需要更强的保证，那么您可以使用 [发布者确认](https://www.rabbitmq.com/confirms.html)。

#### 公平派遣

您可能已经注意到调度仍然无法完全按照我们的意愿运行。例如，在有两个工人的情况下，当一种消息很多，甚至消息很少时，一个worker将经常忙，而另一个worker几乎不会做任何工作。那么，RabbitMQ对此一无所知，仍然会均匀地发送消息。

发生这种情况是因为RabbitMQ只是在消息进入队列时调度消息。它不会查看消费者未确认消息的数量。它只是盲目地向第n个消费者发送每个第n个消息。

![prefetch-count](https://www.rabbitmq.com/img/tutorials/prefetch-count.png)

为了打败我们，我们可以使用预取值为`@1`的`basicQos：global：`方法。这告诉RabbitMQ不要向一个worker发送多于一条消息。或者，换句话说，在处理并确认前一个消息之前，不要向worker发送新消息。相反，它会将它发送给下一个不是很忙的worker。

```objc
[ch basicQos:@1 global:NO];
```

`关于队列大小的说明`  
如果所有工作人员都很忙，您的队列就会填满。您将需要关注这一点，并可能添加更多工作人员，或者采取其他策略。

#### 把它们放在一起

`newTask`的最终代码：

```objc
- (void)newTask:(NSString *)msg {
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id<RMQChannel> ch = [conn createChannel];

    RMQQueue *q = [ch queue:@"task_queue" options:RMQQueueDeclareDurable];

    NSData *msgData = [msg dataUsingEncoding:NSUTF8StringEncoding];
    [ch.defaultExchange publish:msgData routingKey:q.name persistent:YES];
    NSLog(@"Sent %@", msg);

    [conn close];
}
```

`workerNamed:`

```objc
- (void)workerNamed:(NSString *)name {
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id<RMQChannel> ch = [conn createChannel];

    RMQQueue *q = [ch queue:@"task_queue" options:RMQQueueDeclareDurable];

    [ch basicQos:@1 global:NO];
    NSLog(@"%@: Waiting for messages", name);

    RMQBasicConsumeOptions manualAck = RMQBasicConsumeNoOptions;
    [q subscribe:manualAck handler:^(RMQMessage * _Nonnull message) {
        NSString *messageText = [[NSString alloc] initWithData:message.body encoding:NSUTF8StringEncoding];
        NSLog(@"%@: Received %@", name, messageText);
        // imitate some work
        unsigned int sleepTime = (unsigned int)[messageText componentsSeparatedByString:@"."].count - 1;
        NSLog(@"%@: Sleeping for %u seconds", name, sleepTime);
        sleep(sleepTime);

        [ch ack:message.deliveryTag];
    }];

```

[获取源码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/objective-c/tutorial2/tutorial2/ViewController.m)

使用消息确认和预取，您可以设置工作队列。即使RabbitMQ重新启动，持久性选项也可以使任务生效。

现在我们可以继续学习[第三部分](/2019/04/12/RabbitMQ-Publish-Subscribe/)并学习如何向许多消费者传递相同的消息。


#### 参考资料

[RabbitMQ官方文档：Work Queues](https://www.rabbitmq.com/tutorials/tutorial-two-objectivec.html)
