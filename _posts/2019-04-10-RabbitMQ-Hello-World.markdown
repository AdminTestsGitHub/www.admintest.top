---
layout: post
title:  "“Hello World!”"
subtitle:   "RabbitMQ学习第一篇."
author: AdminTest
date:   2019-04-10
header-mask: 0.6
catalog:    true
header-img: "img/post-bg-rmq.png"
tags:
    - iOS
    - RabbitMQ
---

>第一篇开始


#### "Hello World"
（使用Objective-C客户端）


在本教程的这一部分中，我们将编写一个简单的iOS应用程序。它将发送单个消息，消费该消息并使用`NSLog`进行记录。

在下图中，**“P”**是我们的生产者，**“C”**是我们的消费者。中间的框是一个队列 - RabbitMQ保留代表消费者的消息缓冲区。
![python-one](https://www.rabbitmq.com/img/tutorials/python-one.png)


#### 创建Xcode项目
使用RabbitMQ客户端，按照以下说明创建一个新的Xcode项目。

1. 使用File - > New - > Project创建一个新的Xcode项目......
2. 选择iOS应用程序 - > 单一视图应用程序
3. 单击下一步
4. 为您的项目命名，例如“RabbitTutorial1”。
5. 根据需要填写组织详细信息。
6. 选择Objective-C作为语言。出于本教程的目的，您不需要单元测试。
7. 单击下一步
8. 选择创建项目的位置，然后单击“创建”

现在我们必须添加Objective-C客户端作为依赖项。这部分是从命令行完成的。有关详细说明，请访问[客户端的GitHub页面](https://github.com/rabbitmq/rabbitmq-objc-client)。

将客户端添加为依赖项后，使用Product - > Build构建项目以确保它已正确链接。

#### 发送
![sending](https://www.rabbitmq.com/img/tutorials/sending.png)

为了方便本教程，我们将发送和接收代码放在同一个视图控制器中。发送代码将连接到RabbitMQ并发送单个消息。
让我们编辑 `ViewController.m` 并开始添加代码。

导入框架
首先，我们将客户端框架作为模块导入：
```objc
@import RMQClient;
```
现在我们从viewDidLoad调用一些`send`和`receive`方法：
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [self send];
    [self receive];
}
```
`send`方法以与`RabbitMQ`代理的连接开始：
```objc
- (void)send {
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];
}
```
连接抽象套接字连接，并为我们负责协议版本协商和身份验证等。在这里，我们使用所有默认设置连接到本地计算机上的代理。使用了日志记录委托，因此我们可以在`Xcode`控制台中看到任何错误。

如果我们想连接到不同机器上的代理，我们只需使用`initWithUri：delegate：` 便利构造器 指定其名称或IP地址：
```objc
RMQConnection *conn = [[RMQConnection alloc] initWithUri:@"amqp://myrabbitserver.com:1234"
                                                delegate:[RMQConnectionDelegateLogger new]];
```
接下来，我们创建一个通道，这是完成任务的大部分API所在的位置：
```objc
id<RMQChannel> ch = [conn createChannel];
```
要发送，我们必须声明一个队列供我们发送; 然后我们可以向队列发布消息：
```objc
RMQQueue *q = [ch queue:@"hello"];
[ch.defaultExchange publish:[@"Hello World!" dataUsingEncoding:NSUTF8StringEncoding] routingKey:q.name];
```
声明队列是幂等的 - 只有在它不存在的情况下才会创建它。

最后，我们关闭连接：
```objc
[conn close];
```

#### 接收
这就是发送。我们的receive方法将启动将从RabbitMQ推送消息的消费者，因此与发布单个消息的send不同，它将等待消息，记录它然后退出。

![receiving](https://www.rabbitmq.com/img/tutorials/receiving.png)

设置与发送相同; 我们打开一个连接和一个通道，并声明我们将要消耗的队列。请注意，这与发送到的队列匹配。
```objc
- (void)receive {
    NSLog(@"Attempting to connect to local RabbitMQ broker");
    RMQConnection *conn = [[RMQConnection alloc] initWithDelegate:[RMQConnectionDelegateLogger new]];
    [conn start];

    id<RMQChannel> ch = [conn createChannel];

    RMQQueue *q = [ch queue:@"hello"];
}
```
请注意，我们也在这里声明了队列。因为我们可能在发送方之前启动接收方，所以我们希望在尝试使用消息之前确保队列存在。

我们即将告诉服务器从队列中传递消息。由于它会异步地向我们推送消息，因此我们提供了一个回调，当RabbitMQ将消息推送给我们的消费者时，将执行该回调。这就是RMQQueue订阅的内容：确实如此。
```objc
NSLog(@"Waiting for messages.");
[q subscribe:^(RMQMessage * _Nonnull message) {
    NSLog(@"Received %@", [[NSString alloc] initWithData:message.body encoding:NSUTF8StringEncoding]);
}];
```

全部代码详见[ViewController.m](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/objective-c/tutorial1/tutorial1/ViewController.m)。

#### 运行

现在我们可以运行该应用程序。点击播放按钮，或cmd-R。

receive将通过RabbitMQ 记录从发送中获取的消息。接收器将继续运行，等待消息（使用“停止”按钮停止它），因此您可以尝试使用其他客户端将消息发送到同一队列。

你好，世界！

是时候转到[第二部分](/2019/04/11/RabbitMQ-Work-Queues/)并构建一个简单的工作队列了。

#### 参考资料

[RabbitMQ官方文档：“Hello World！”](https://www.rabbitmq.com/tutorials/tutorial-one-objectivec.html)
