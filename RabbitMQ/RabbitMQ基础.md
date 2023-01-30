# AMQP 0-9-1 模型说明

AMQP 0-9-1 (Advanced Message Queuing Protocol) 是一种消息传递协议，使得符合协议的客户端应用程序和消息传递中间件代理进行通信。

并且由于这是一个网络协议，所以生产者、消费者和代理都可以运行在不同的机器上。

## AMQP 0-9-1 模型简介

队列(queue)、交换器(exchange)和绑定(binding) 统称为 AMQP 实体

AMQP 0-9-1模型：消息发布到 exchange (通常将其余邮局或邮箱进行对比) 中，exchange 通过其绑定的规则将消息副本分发到队列。之后代理要么将消息传递给订阅该队列的消费者，要么消费者按需从队列中获取/拉取消息。

![hello-world-example-routing](E:\Typora\hello-world-example-routing.png)

发布消息时，生产者可以指定各种消息属性(消息元数据)，这些元数据某些可能会被代理使用，但某些对代理是不可见的，只允许消费者使用。

AMQP 0-9-1 模型有一个消息确认(分为 Consumer Acknowledgements and Publisher Confirms)的概念：当消息传递给消费者时，消费者会自动或主动选择后通知到代理。当使用消息确认时，代理只用收到消息(或消息组)的通知才会从队列中完全删除队列。

消息确认这一概念是保证即使在网络抖动或者某个消费者无法处理消息时，消息也能够被处理。

此外，消息可能会出现无法路由的情况，消息可能会返回到生产者或被丢弃。但如果代理实现了扩展，消息会被放回"死信队列(dead letter queue)"中。生产者通过特定的参数来选择如何处理这种情况。

## Exchanges and Exchange Types

交换器是消息发送的AMQP 0-9-1 实体。交换器接收消息并将其路由到零个或多个队列中，使用的路由算法取决于其类型和绑定的规则。AMQP 0-9-1 提供四种交换类型：

| Exchange type    | Default pre-declared names              |
| :--------------- | :-------------------------------------- |
| Direct exchange  | (Empty string) and amq.direct           |
| Fanout exchange  | amq.fanout                              |
| Topic exchange   | amq.topic                               |
| Headers exchange | amq.match (and amq.headers in RabbitMQ) |

除了交换类型之外，交换还声明了很多属性，其中最重要的：

- Name
- Durability (exchanges survive broker restart)
- Auto-delete (exchange is deleted when last queue is unbound from it)
- Arguments (optional, used by plugins and broker-specific features)

持久的交换在代理重启后仍会存在，而临时交换不会。

### Default Exchange

默认交换是没有代理声明名称(空字符串)的直接队列。其有一个特殊的属性：创建的每一个队列会自动绑定到一个与队列名称相同的 routing key。通俗来讲，默认交换使得它似乎可以直接将消息传递到队列

举个例子：当你声明一个名为"search-indexing-online" 队列时，AMQP 0-9-1代理会使用"search-indexing-online"作为路由键(routing key 又称为绑定键 binding key)将其绑定到默认交换中。然后，使用路由键"search-indexing-online" 发布到默认交换器上的消息将会被路由到队列"search-indexing-online"。

### Direct exchange

直接交换是基于消息路由键将消息传递到队列中，其很适合消息的单播路由(尽管也可以用于多播路由)。其工作流程：

- A queue binds to the exchange with a routing key K
- When a new message with routing key R arrives at the direct exchange, the exchange routes it to the queue if K = R

直接交换通常以循环方式在多个消费者(同一应用程序的实例)之间分配任务。这样做时，需要了解，在AMQP 0-9-1 中，消息在消费者之间而不是队列之间进行负载平衡。

![exchange-direct](E:\Typora\exchange-direct.png)

### Fanout exchange

扇出交换器会忽略路由键，将消息路由到绑定它的所有队列。举例：N个队列绑定到一个扇出交换器，当一个新消息发布到该交换器时，该消息的副本将传递到所有N个队列。扇出交换很适合广播消息，它的使用场景很相似：

- 大型多人在线 (MMO) 游戏可以将其用于排行榜更新或其他全球事件
- 体育新闻网站可以使用扇出交换近乎实时地向移动客户端分发比分更新
- 分布式系统可以广播各种状态和配置更新
- 群聊可以使用扇出交换在参与者之间分发消息（尽管 AMQP 没有内置的存在概念，因此 XMPP 可能是更好的选择）

![exchange-fanout](E:\Typora\exchange-fanout.png)

### Topic exchange

主题交换器根据路由键和用于将队列绑定到交换器模式之间的匹配，将消息路由到一个或多个队列。主题交换类型通常用于实现各种发布/订阅模式变体，适合消息的多播路由。

每当一个问题涉及多个消费者/应用程序，并且有选择的选择其想要接收的消息类型时，就需要考虑使用主题交换类型了。

主题交换示例场景：

- 分发与特定地理位置相关的数据，例如销售点
- 由多个工作人员完成的后台任务处理，每个工作人员都能够处理特定的任务集
- 股票价格更新（以及其他类型的财务数据更新）
- 涉及分类或标记的新闻更新（例如，仅针对特定运动或团队）
- 在云中编排不同种类的服务
- 分布式架构/特定于操作系统的软件构建或打包，其中每个构建器只能处理一种架构或操作系统

### Headers exchange

标头交换设计用于在多个属性上进行路由，这些属性比路由键更容易表示为消息标头。标头交换忽略路由键属性，用路由的属性取自 headers 属性。如果标头的值等于绑定时指定的值，则消息匹配。

在使用多个标头将队列绑定到标头交换的情况下，代理需要开发人员提供对应的信息，即明确交换器是考虑具有任何标头匹配的消息，还是所有匹配的消息。这也就是 "x-match" 绑定参数的用途，设置为 "any" 时，只有一个匹配的标头值；设置为 "all" 时，要求匹配所有值。而不管是 "any" 还是 "all"，以字符串`x-` 开头的标头都不会参与评估匹配项。设置为 "any-with-x" 或 "all-with-x" 才会将以 字符串 `x-` 开头的标头来评估匹配。

标头交换也可以被称为"direct exchanges on steroids"。因为其基于标头值进行路由，所有可以用作路由键不必是字符串的直接交换；如：它可以是整数或散列(字典)。



## 参考

[1] [AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts.html)