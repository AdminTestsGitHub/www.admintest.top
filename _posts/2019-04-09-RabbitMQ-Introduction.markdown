---
layout: post
title:  RabbitMQ
subtitle:   "RabbitMQ介绍"
author: AdminTest
date:   2019-04-09
header-mask: 0.6
catalog:    true
header-img: "img/post-bg-rmq.png"
tags:
    - iOS
    - RabbitMQ
---

>开始


#### 介绍

<!-- RabbitMQ is a message broker: it accepts and forwards messages. You can think about it as a post office: when you put the mail that you want posting in a post box, you can be sure that Mr. or Ms. Mailperson will eventually deliver the mail to your recipient. In this analogy, RabbitMQ is a post box, a post office and a postman.

The major difference between RabbitMQ and the post office is that it doesn't deal with paper, instead it accepts, stores and forwards binary blobs of data ‒ messages.

RabbitMQ, and messaging in general, uses some jargon.
 -->

`RabbitMQ`是一个消息代理：它用来接收和转发消息。你可以把它想象成邮局：当你把要发送的邮件放到邮箱里时，你确定邮递员最终会把邮件发给收件人。和这个类似，`RabbitMQ`是一个邮政信箱，一个邮局，一个邮递员。

`RabbitMQ`和邮局的主要区别就是，它不处理邮件内容，取而代之的是接收，存储和转发二进制的大数据 - **信息**。

`RabbitMQ`里消息的传递，使用一些术语。


<!-- * Producing means nothing more than sending. A program that sends messages is a producer :

* A queue is the name for a post box which lives inside RabbitMQ. Although messages flow through RabbitMQ and your applications, they can only be stored inside a queue. A queue is only bound by the host's memory & disk limits, it's essentially a large message buffer. Many producers can send messages that go to one queue, and many consumers can try to receive data from one queue. This is how we represent a queue:

* Consuming has a similar meaning to receiving. A consumer is a program that mostly waits to receive messages:

Note that the producer, consumer, and broker do not have to reside on the same host; indeed in most applications they don't. An application can be both a producer and consumer, too.
 -->
* 生产其实就是发送。一个发送消息的程序就是生产者：
![producer](https://www.rabbitmq.com/img/tutorials/producer.png)

* `RabbitMQ`内部的队列就如同邮政信箱的名字。虽然消息流经RabbitMQ和您的应用程序，但它们只能存储在队列中。队列仅由主机的存储器＆磁盘限制约束，它本质上是一个大的消息缓冲器。许多生产者可以发送消息到一个队列，并且许多消费者可以尝试从一个队列接收数据。这就是我们代表队列的方式：
![queue](https://www.rabbitmq.com/img/tutorials/queue.png)

* 消费与接受有类似的意义。一个消费者是一个程序，主要是等待接收信息：
![consumer](https://www.rabbitmq.com/img/tutorials/consumer.png)

请注意，生产者，消费者和代理不必驻留在同一主机上; 实际上在大多数应用中他们没有。应用程序可以即是生产者也可以是消费者。

#### 正文

以下是把官网的资料翻译成了中文，方便的大家理解，如有错误，还请指正。

第一篇：[“Hello World！”](/2019/04/10/RabbitMQ-Hello-World/)

第二篇：[Work Queues](/2019/04/11/RabbitMQ-Work-Queues/)

第三篇：[Publish/Subscribe](/2019/04/12/RabbitMQ-Publish-Subscribe/)

第四篇：[Routing](/2019/04/13/RabbitMQ-Routing/)

第五篇：[Topics](/2019/04/14/RabbitMQ-Topics/)
