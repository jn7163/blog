---
title: Python 实现观察者模式/发布订阅模式
date: 2019-02-07 10:21:55
tags:
  - design pattern
  - python
---

春节回乡拜年莫得电脑，于是窝在阁楼上看了些文字 —— 其中就包括`发布订阅模式`的一些相关内容 。`发布订阅模式`是设计模式中的比较简单一种；它定义了消息在生产者/发布者与消费者/订阅者之间传递的方式 。实际开发中我们通常会使用 Redis / ZeroMQ 等中间件来实现消息分发，但这里我只用 Python 来描述其中的逻辑关系 —— 使用其他语言乃至中间件的逻辑关系基本都是相同的，只有一些细节性差别 。**值得一提的是**，尽管多数情况下我们将`观察者模式`与`发布订阅模式`混为一谈（或认为`观察者=发布+订阅`），但事实上它们之间存在一定的区别，这点我稍后也会提到 。

<!--more-->

## 实现发布订阅模式

先看一看具体的实现可能有助于更直观的理解（完整代码可以查看 [GitHub Gist](https://gist.github.com/queensferryme/ffabe20f58fd84ddba3e5d2f28a1ec74)）：

**Intermedia**

```python
class Intermedia:
    channels = defaultdict(dict)
    @classmethod
    def broadcast(cls, channel, msg):
        '''broadcast a message to specific subscribers'''
        for name, callback in cls.channels[channel].items():
            callback({
                'channel': channel,
                'msg': msg,
                'name': name
            })
    @classmethod
    def register(cls, channel, subscriber):
        '''register a subscriber into specific channel'''
        cls.channels[channel][subscriber.name] = subscriber.callback
    @classmethod
    def unregister(cls, channel, subscriber):
        '''unregister a subscriber from specific channel'''
        del cls.channels[channel][subscriber.name]
```

**Publisher**

```python
class Publisher:
    @classmethod
    def publish(self, channel, msg):
        '''publish a message to specific channel'''
        Intermedia.broadcast(channel, msg)
```

**Subscriber**

```python
class Subscriber:
    def __init__(self, name, callback=None):
        self.name = name
        self.callback = callback if callable(callback) else self.callback
    def callback(self, msg):
        '''default event handler for current subscriber'''
        print('Subscriber "{0[name]}" receive message "{0[msg]}" from channel "{0[channel]}"'.format(msg))
    def subscribe(self, channel):
        '''subscribe specific channel from publisher'''
        Intermedia.register(channel, self)
    def unsubscribe(self, channel):
        '''unsubscribe specific channel from publisher'''
        Intermedia.unregister(channel, self)
```

代码并不难理解 。可以看到，订阅者（Subscriber）能够从信息媒体（Intermedia）处订阅（subscribe）特定的频道（channel）或取消订阅（unsubscribe）；而发布者可以在信息媒体上特定的频道（channel）发布消息（message），消息发布后只有订阅了该频道的订阅者才会收到这条消息，并执行相应的回调函数（callback）来处理这条消息 。消息通过信息媒体在发布者/订阅者之间传递 。

注意这里 `Intermedia` 的 `channels` 属性 。它以字典的形式记录了每个频道的所有订阅者以及每个订阅者处理消息的回调函数 。事实上 `channels` 根本不必作为属性，它可以是一个全局字典变量，甚至可以是一个 Redis 中间件 —— 事实上它就是一个**作为消息代理的中间件**，无论是发布者发布特定消息还是订阅者订阅某个频道，都是在这个中间件上进行操作；发布者与订阅者无需知晓对方的存在 。而中间件负责了消息的过滤与分发的工作 。这一**中间件**是区别观察者模式与发布订阅模式的主要特征 。

维基百科是这样描述的：

> 在许多发布/订阅系统中，发布者发布消息到一个中间的消息代理，然后订阅者向该消息代理注册订阅，由消息代理来进行过滤。消息代理通常执行存储转发的功能将消息从发布者发送到订阅者。

## 观察者模式及其区别

![observer-pattern](https://i.loli.net/2019/02/08/5c5d031b976ad.jpg)

从上图不难看出，观察者模式与发布订阅模式的主要区别就是**中间件** 。发布订阅模式在生活中社交媒体上很常见，而观察者模式也有个常见的场景：JavaScript 中的事件模型。例如：

```javascript
document.querySelector('#submit').addEventListener('click', clickHandler)；
```

这里可以理解为 `#submit` 元素订阅了 `click` 频道；一旦该元素被点击，`click` 事件就会被触发（fire），然后相应的回调函数（此处为 `clickHandler`）就会被调用来处理消息（此处可能为 `event.target`）

------

**参考文献：**

- [Tutorial: The Observer Pattern in Python](https://www.protechtraining.com/blog/post/tutorial-the-observer-pattern-in-python-879)
- [掘金：观察者模式 vs 发布-订阅模式](https://juejin.im/post/5a14e9edf265da4312808d86)
- [SegmentFault：JavaScript 观察者模式](https://segmentfault.com/a/1190000013009772)
