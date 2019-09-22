---
title: pika TCP连接断开
date: 2019-09-22 00:01:44
tags: python, pika, rabbitmq
password: 
---

趁着工作任务开发学习了一波[AMQP协议](http://rabbitmq.mr-ping.com/AMQP/AMQP_0-9-1_Model_Explained.html)，以及其实现rabbitMQ，其实概念不难，也容易懂

rabbitMQ官方文档传送门：[点这里](https://www.rabbitmq.com/getstarted.html)（P.S. 官方文档真的写得通俗易懂，没用什么深奥词汇，英语渣的福音） 

这里使用的是python下的pika来进行学习，按照教程走一遍基本上就知道怎么使用了。

本博文主要是关心一个情况，就是对于处理一些耗时的任务，TCP连接会断开的问题。

正常情况下，rabbitMQ会有一套检测TCP连接情况的机制，就是发送[心跳包](https://www.rabbitmq.com/heartbeats.html)了。

客户端和服务器会决定一个叫做`heartbeat timeout`的值，具体就是双方同时提出一个非零值，谁小就用谁；若果有一个是0，就用非零的，rabbitmq默认的值是`60`。若果超过了这个时间值都没有收到心跳包，就会认为这个TCP连接不可达的了。经过抓包分析，这个发送行为是`双向`进行的。

这里还有一个概念就是`heartbeat interval`，就是心跳包的发送周期，为`heartbeat timeout / 2`。若果错过2个心跳包，就会认为TCP连接不可达，刚好对应`heartbeat timeout`。

好了，有了这些概念后，我们就可以开始今天的主题了：处理一些耗时的任务，TCP连接会断开。

首先贴出代码。

生产者代码:
```python
import pika

if __name__ == '__main__':

    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))

    channel = connection.channel()

    channel.queue_declare(queue='test')
    channel.basic_publish(exchange='',
                          routing_key='test',
                          body='hello world')
    print(f'message sent')
    connection.close()
```

代码很简单，声明了一个`test`队列，然后发送了一个消息。注意一点，若果没有对队列进行绑定，就会自动绑定到默认交换机。

消费者代码。
```python
import pika

done = False

def callback(ch, method, properties, body):
    print(f'I received the message')
    print(f'the message: {body}')
    ch.basic_ack(delivery_tag=method.delivery_tag)

if __name__ == '__main__':

    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost', heartbeat=3))

    channel = connection.channel()

    channel.queue_declare(queue='test')
    channel.basic_consume(queue='test', on_message_callback=callback)
    channel.start_consuming()
    connection.close()
```

代码也很简单，就是声明了队列，然后绑定消息处理回调函数，就开始监听了。注意代码中的`heartbeat=3`，这里是定义的是`heartbeat timeout`的值，所以`heartbeat interval`就是`1.5`

运行一下。
```
# 生产者
message sent

# 消费者
I received the message
the message: b'hello world'
```

基础代码就是这样，但是我们得关注一下在没有消息的时候启动消费者，心跳包机制是怎么样的。
利用wireshark抓包如下

![图1](/img/python/TCP_1.png "图1")

    No.39 客户端发送心跳包
    No.40 服务端发送ACK包
    No.41 服务端发送心跳包
    No.42 客户端发送ACK包
    往下重复...

现在我们对代码修改一下，模拟处理耗时任务，修改如下。
```python
import pika
import time

done = False

def callback(ch, method, properties, body):
    print(f'I received the message')
    print(f'the message: {body}')
    print(f'now I will sleep 10 seconds')
    time.sleep(10)
    ch.basic_ack(delivery_tag=method.delivery_tag)

if __name__ == '__main__':

    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost',  heartbeat=3))

    channel = connection.channel()

    channel.queue_declare(queue='test')
    channel.basic_consume(queue='test', on_message_callback=callback)
    channel.start_consuming()
    connection.close()
```

在回调函数里面添加了睡眠10s的代码，运行一下，会报一下错误。

```
I received the message
the message: b'hello world'
now I will sleep 10 seconds
Traceback (most recent call last):
  File "F:/py_workplace/process/receiver.py", line 21, in <module>
    channel.start_consuming()
  ...

pika.exceptions.StreamLostError: Stream connection lost: ConnectionResetError(10054, '远程主机强迫关闭了一个现有的连接。', None, 10054, None)
```

程序报了`pika.exceptions.StreamLostError`错误，这是为什么呢？

这是因为程序设置了心跳超时是3s，但是回调中却要睡眠10s，这时候TCP连接就断开了。

wireshark的抓包如下。

![图2](/img/python/TCP_2.png "图2")

    No.37 服务器发送了一个消息到客户端
    No.38 客户端发送ACK包
    No.39 服务端发送心跳包
    No.40 客户端发送ACK包
    No.41 服务端发送心跳包
    No.42 客户端发送ACK包
    ...
    No.49 服务器发送了RST包

对比上面正常空闲时候的抓包，可以看出，这里面缺少了客户端发送的心跳包，因为这个时候客户端正在睡眠，线程被挂起了。所以后面服务器多次没有收到来自客户端的心跳包，就认为连接不可达，断开了连接。当客户端睡眠结束，发送ACK确认的时候，因为连接断开而报错。(这里有个问题，从抓包时间看，貌似断开连接不是在心跳超时的那个时间点，实际是往后了，这里我也想不懂，希望有大神能告知一下)。

这里要怎么解决？

1. 延长心跳超时的时间，不过这个在任务耗时未知的时候就只能靠经验设置了。

2. 关闭心跳检测，[方法](https://www.rabbitmq.com/heartbeats.html#disabling)是在客户端和服务端把`heartbeat interval`设为0。但是有一个问题就是TCP会永久连接，有可能导致系统资源的耗尽。

3. 增加一个心跳线程，专门用来处理心跳问题，这个方法没有验证过,貌似会有线程安全的问题。参考[这里](https://stackoverflow.com/questions/14572020/handling-long-running-tasks-in-pika-rabbitmq)

---

这里补充一下关于`channel.start_consuning()` 和`connention.process_data_events()`的差别。

首先，先了解一下`connention.process_data_events()`

[官方](https://pika.readthedocs.io/en/stable/modules/adapters/blocking.html)解释是

    Will make sure that data events are processed. Dispatches timer and channel callbacks if not called from the scope of 
    BlockingConnection or BlockingChannel callback. Your app can block on this method.

就是用来专门处理消息事件的，有一个参数`time_limit`,指定等待消息的时间，单位为秒。当为`None`的时候，就会一直等待，也就是说，会阻塞线程。

这个函数会处理服务器发过来的心跳包，而且也会向服务器发送心跳包。

然后到`channel.start_consuning()`，在源码里面有这样的调用

```python
    def start_consuming(self):
        """Processes I/O events and dispatches timers and `basic_consume`
        callbacks until all consumers are cancelled.

        NOTE: this blocking function may not be called from the scope of a
        pika callback, because dispatching `basic_consume` callbacks from this
        context would constitute recursion.

        :raises pika.exceptions.ReentrancyError: if called from the scope of a
            `BlockingConnection` or `BlockingChannel` callback
        :raises ChannelClosed: when this channel is closed by broker.
        """
        # Check if called from the scope of an event dispatch callback
        with self.connection._acquire_event_dispatch() as dispatch_allowed:
            if not dispatch_allowed:
                raise exceptions.ReentrancyError(
                    'start_consuming may not be called from the scope of '
                    'another BlockingConnection or BlockingChannel callback')

        self._impl._raise_if_not_open()

        # Process events as long as consumers exist on this channel
        while self._consumer_infos:
            # This will raise ChannelClosed if channel is closed by broker
            self._process_data_events(time_limit=None)
```

最后有一个`while`循环，就是执行`self._process_data_events(time_limit=None)`，

```python
    def _process_data_events(self, time_limit):
        """Wrapper for `BlockingConnection.process_data_events()` with common
        channel-specific logic that raises ChannelClosed if broker closed this
        channel.

        NOTE: We need to raise an exception in the context of user's call into
        our API to protect the integrity of the underlying implementation.
        BlockingConnection's underlying asynchronous connection adapter
        (SelectConnection) uses callbacks to communicate with us. If
        BlockingConnection leaks exceptions back into the I/O loop or the
        asynchronous connection adapter, we interrupt their normal workflow and
        introduce a high likelihood of state inconsistency.

        See `BlockingConnection.process_data_events()` for documentation of args
        and behavior.

        :param float time_limit:

        """
        self.connection.process_data_events(time_limit=time_limit)
        if self.is_closed and isinstance(self._closing_reason,
                                         exceptions.ChannelClosedByBroker):
            LOGGER.debug('Channel close by broker detected, raising %r; %r',
                         self._closing_reason, self)
            raise self._closing_reason  # pylint: disable=E0702
```

从代码可以看到，`channel.start_consuning()`会调用到`connection.process_data_events()`，而且参数是`None`，就是说会一直等待消息。

---

最后，我实在得吐槽一下，网上有一些博文感觉是真的不知道有没有验证过，我在查怎么处理耗时任务而导致连接断开的时候，很多博文都说用`connection.process_data_events()`就可以避免了，但是验证了一下，该报错的还是报错。还是自己动手，丰衣足食吧。