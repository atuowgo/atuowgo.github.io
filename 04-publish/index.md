# Redis发布与订阅


&emsp;这节介绍Redis的发布与订阅

&emsp;Redis提供了发布与订阅的功能，客户端能够向服务器订阅某个频道，当其他客户端向服务器的该频道发布消息时，服务器会将消息推送到订阅该频道的客户端。提供的命令包括：

```
SUBSCRIBE  channel  [channel  …]
```

该命令可以向服务器订阅多个频道的消息。与之对应的有UNSUBSCRIBE命令。

```
PSUBSCRIBE  pattern	[pattern  …]
```

该命令可以向服务器订阅多个频道中满足对应模式的消息。与之对应的有UNPSUNSCRIBE命令。

```
PUBLISH channel	message
```

客户端向服务器对应的频道发布消息。

```
PUBSUB CHANNELS [pattern]
```

查看服务器当前被订阅的频道信息。

```
PUBSUB NUMSUB [pattern …]
```

可以查看对应频道的订阅者数量。

```
PUBSUB NUMPAT
```

可以查看服务器当前被订阅模式的数量。

&emsp;内部实现比较简单，服务器记录客户端与订阅频道的关系，以链表的方式存储，执行对应命令的时候通过遍历链表获得相应的数据执行后输出。

