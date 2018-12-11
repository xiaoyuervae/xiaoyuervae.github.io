---
title: RocketMq 学习之核心参数
tags:
  - rocketmq
abbrlink: 1044148734
date: 2018-12-10 22:53:23
---

> producerGroup

生产者组名，一个服务默认只允许配置唯一一个生产组名。

> createTopicKey

自动创建topic key。生产环境不建议使用，尽量是由公司的架构组提供统一对RocketMq的API进行二次封装，隐藏一些API。

> defaultTopicQueueNums

一个topic下默认的消息队列，默认是4个

> compressMsgBodyHowmuch

消息体自动压缩、默认为 4096 个字节、便于在网络中进行传输。

> retryTimesWhenSendFailed

消息发送失败后重试的次数、可配置，这是针对于同步发送消息的，异步发送消息对应有另外的配置 `retryTimesWhenSendAsyncFailed`。

> retryAnotherBrokerWhenNotStoreOK

如果在broker上存储失败可以对其他broker进行重试，默认为false

> maxMessageSize

最大消息大小，默认为128k

> producer的配置项:

![](https://ws1.sinaimg.cn/large/e1417e4bly1fy21g5h5gtj20ya0r6dq7)