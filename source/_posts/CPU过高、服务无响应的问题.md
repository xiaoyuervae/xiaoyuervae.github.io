---
title: 一次排查MongoDB CPU过高、服务无响应的问题
date: 2018-12-02 14:59:38
tags: [mongo, cpu]
---

## 问题的出现

IM内的聊天记录等数据都存在Mongo DB中、最近手机短信一直收到持续不断的Apdex值比较低的告警，所以集中花时间看了一下到底是什么问题造成的。

![IMG_3045](https://ws1.sinaimg.cn/large/e1417e4bly1fxsavmvgnlj20u01szwsl)

在我们的`Marvin`监控系统中看到出现的慢日志和相关错误日志中看到基本上都来自于同一个接口:

![](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsenscdk3j21qw08c0uw)

查看具体代码日志发现都是在执行查询M`mongo`的时候造成的超时。

## 排查具体的问题

### 现象

在观察这个接口一段时间之后发现一个非常奇特的现象:

- 该接口正常情况下的`rt`还是很正常的, 然后突然`rpm`值会激增,同时`rt`也会激增, 从正常情况下的 `5ms` 左右突然激增到 `2s` 的时间左右, 这对于一个正常的接口来说是不可接受的。

    ![](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsenseslhj21i80riagf)


- 观察了一下mongo机器的 `zabbix` 监控, 查看了一下mongo服务器在当时的状态, 发现除了机器的 `cpu load` 有稍微的波动之外, 磁盘IO、内存等数值都没有明显的上涨:

    ![](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsensdw8ij223m0i27cc)

- 观察了线上mongo本身的日志, 发现在rt上涨的过程中 `mongo` 服务器的连接数不断在上涨, 从平稳的连接数95开始上涨, 1s钟之内上涨到了1500左右的连接,同时在这段期间貌似是没有任何返回,从日志中可以看到一个连接的响应时间的开始到结束有些甚至达到了**80s**之久！(什么sql能查这么久?!)

    ![从运维同学那边拿到的图](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsensij3wj212t0kie1m)


### 排查

联系了运维的同学帮忙看一下到底是什么问题造成的, 因为运维同学最近比较忙, 恰巧碰到他负责双十二全链路压测, 而且他主要也是运维MySQL的经验比较多, 帮我们粗略看了一下认为是并发太高的问题造成的(突然一时间并发连接数暴涨)。一时之间也看不出来有什么大的问题。

和运维同学一起排查的过程中我猜想是不是由于部分SQL没有走索引, 导致全表扫描或者是查询效率特别慢造成的阻塞, 运维同学回答我说有可能, 为了验证这个想法我们查看了在那段时间内的服务端日志中打印出来的慢查询的sql, 发现sql实在太多, 随便拉了几条出来看:


``` 
command historymessage.$cmd command: count { count: "of_history_message", query: { session: "86690631_97553434", $and: [ { msgTime: { $lte: 1543226394744 } } ] } } plan
Summary: COUNT_SCAN { session: 1.0, msgTime: -1.0 } keyUpdates:0 writeConflicts:0 numYields:0 reslen:44 locks:{ Global: { acquireCount: { r: 1 } }, MMAPV1Journal: { acquireCount: { r: 1 } }, Database: { acquireCount: { r: 1
 } }, Collection: { acquireCount: { R: 1 } } } 71270ms
```


```
{ count: "of_history_message", query: { session: "86492947_95561724", $and: [ { msgTime: { $lte: 1543235835000 } } ] } } plan
Summary: COUNT_SCAN { session: 1.0, msgTime: -1.0 } keyUpdates:0 writeConflicts:0 numYields:0 reslen:44 locks:{ Global: { acquireCount: { r: 1 } }, MMAPV1Journal: { acquireCount: { r: 1 } }, Database: { acquireCount: { r: 1
 } }, Collection: { acquireCount: { R: 1 } } } 151ms
```

看了一下这些sql 并在mongo 上打印了一下执行计划, 都是很正常的sql, 并且都走了索引, 同时放到mongo 上执行速度也很快，这些sql应该都是因为前面的执行sql发生了阻塞之后造成响应慢的。

为了能够更好的进行排查我们从运维的同学那边要到了mongo服务器的机器权限, 方便进行更好的排查, 势必是要解决当前的这个问题。拿到服务器的权限之后发现mongo 的服务器配置还是相当强悍的：

> 3台物理机, 逻辑CPU有48个, 内存有250G!

![CPU信息](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsense2isj217m0oagrn)

![内存信息](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsensa5orj213q04gjs6)


线上以Mongo副本集的方式部署, 这样的配置可谓是相当强悍, 所有mongo数据库中数据基本上都是加载到内存, 查询起来按理来说不会很慢的啊。

### 发现问题

线上mongo默认的日志好像是好几个月的日志都打在了同一个文件中，日志量实在太大，连grep一下都很难，只能看近期的一些日志，所有要做排查实在是非常困难。

既然在mongo服务器上看不出问题那就从客户端出发吧，既然是在某一个时间点突然爆发出现速度慢的情况我是不是只要找出那个第一条响应慢的日志就可以知道到底是什么问题了。

于是我在客户端查询的接口上加上了具体的调用日志，打印出每条查询mongo的sql的开始时间和结束时间，并且把对应的sql打印出来，上线之后观察一段时间，我找到了那第一条执行速度慢的日志:

![](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsentcwybj21h70u0wvi)

我发现这条查询语句非常奇怪，竟然是查询小于当前时间的数量, 奇怪归奇怪, 但是讲道理也不应该会那么慢的, 只是查询一个数量而已, 于是我把这条语句放到了mongo上查看了一下执行计划:

![](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsensr0u7j20xp0glk7g)

不看不知道, 执行了一下发现竟然没有返回,并且出现了报错, 于是我立马意识到这条sql可能压根都没有索引, 查询小于当前时间所有的数据进行了一次全表扫描:
![](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsensuy9gj20d501lglo)

8000多万数据进行了一次无索引的count, 查看了一下collection上的索引:
![](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsenswtr2j20fz0wnn0f)

发现果然是没有这个索引！

## 解决问题

发现了这个慢sql之后我非常好奇为什么会有这个sql出现, 同客户端的同学确认了一下之后发现这个sql的存在很有可能是由于参数的误传导致的, 因为这个接口中mongo查询的参数是由前端的同学传过来，sql经过参数的拼接而形成的。

为了验证我的想法我在业务的代码中增加了对这种sql形成的过滤(当然可能不止存在这一种没有走索引的慢sql)。

代码上线之后得到了惊人的效果, 慢sql的情况居然一去不复返了。接口的rt再也没有上过30ms, 说明暂时还没有其他的慢sql出现:

![](https://ws1.sinaimg.cn/large/e1417e4bgy1fxsensxo8jj21gq0pkjwo)

同时接口的rpm值也随之降下来了, 这时我意识到rpm的上涨应该是由于客户端超时之后进行了重试, 导致rpm值一下子暴涨, 所以mongo服务器的连接数才会一下子暴涨。

至此，问题终于得到了解决，这几天终于没有再收到告警，舒坦了。。。

## 总结

这次mongo服务器问题的排查经历了大概三天左右的时间才最终把问题找到解决, 也是我经历的最久的一次问题排查了，整个过程是比较艰辛的，虽然过程是很简单，但是中间是绕了不少的弯路，也查阅了不少mongo相关的资料，发现mongo这块的资料真的是少，不过好在最后找到了问题所在。以前排查问题很少有这样的坚持力，基本上排查了一段时间后如果说真的发现不了问题的话就放弃了，所以对待这种排查起来比较困难的问题还是要懂得坚持，理清自己的思路。

通过这次排查我也发现我们之前在使用mongo的时候也有很多不规范的地方，mongo是一个性能很强悍的NoSQL数据库, 我们可能还没有真正的发挥出Mongo自身的性能。同时也有几个不规范的点:

- 在业务中拼条件组成sql，可能大家为了代码的复用方便不愿去写多个SQL分别去做查询，但是这样造成了有些参数的缺失可能形成了一条性能极差的sql。
- 客户端的超时次数没有设置好，这次的问题也主要是由于客户端的超时次数太多了，其实一两条这种sql根本并不会造成严重的性能问题的，主要是因为超时重试造成了同一时间内多条这种低性能SQL阻塞了mongo的执行甚至拖垮整个mongo集群。
- mongo服务的监控做的不够到位，这个问题也不好说，mongo在公司的使用毕竟还不够主流，相关工具的支持也很不完善，出现了问题没有能够进行及时的反馈，导致这个问题在线上出现了很久。