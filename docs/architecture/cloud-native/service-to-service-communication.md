---
title: 服务到服务通信
description: 了解后端云和本地微服务与其他后端微服务通信的方式。
author: robvet
ms.date: 05/13/2020
ms.openlocfilehash: dec06cc28ac177381b882f9e441e19e5c51bd5ad
ms.sourcegitcommit: 27db07ffb26f76912feefba7b884313547410db5
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 05/19/2020
ms.locfileid: "83613702"
---
# <a name="service-to-service-communication"></a>服务到服务通信

从前端客户端开始，我们会解决后端微服务之间的通信。

构造云本机应用程序时，您需要对后端服务之间的通信方式保密。 理想情况下，服务间通信越少，性能越好。 不过，由于后端服务通常相互依赖于另一项来完成操作，因此不一定能避免这种情况。

有几种广泛接受的方法来实现跨服务通信。 *通信交互的类型*通常会确定最佳方法。

请考虑下列交互类型：

- *Query* –调用微服务需要来自称为微服务的响应，例如，"你好，为给定客户 Id 提供买方信息。"

- *Command* –调用微服务需要另一个微服务来执行操作，但不需要响应，如 "你好，只是发送此订单"。

- *事件*–当名为发布服务器的微服务引发状态已更改或操作已发生的事件时。 其他感兴趣的微服务（称为订户）可以相应地对事件做出反应。 发布服务器和订阅服务器彼此之间并不知道。

当执行需要跨服务交互的操作时，微服务系统通常会结合使用这些交互类型。 我们来看一看每个问题及其实现方式。

## <a name="queries"></a>查询

很多时候，一个微服务可能需要*查询*另一个，需要立即响应才能完成操作。 购物篮微服务可能需要产品信息和价格才能将物品添加到购物篮中。 有多种方法可实现查询操作。

### <a name="requestresponse-messaging"></a>请求/响应消息传送

实现此方案的一种方法是，调用后端微服务向其需要查询的微服务发出直接 HTTP 请求，如图4-8 所示。

![直接 HTTP 通信](./media/direct-http-communication.png)

**图 4-8**。 直接 HTTP 通信

尽管微服务之间的直接 HTTP 调用相对简单，但要执行此操作，但要尽量减少这种做法。 首先，这些调用始终*同步*，并将阻止操作，直到返回结果或请求超时。 如果是自包含的独立服务，可以独立地单独进行发展并经常进行部署，现在会相互耦合。 随着微服务的增长，它们的体系结构优势得以降低。

如果执行的是不频繁的请求，则会对某些系统允许单个直接 HTTP 调用另一个微服务。 但是，调用多个微服务的直接 HTTP 调用的高容量调用并不可取。 它们可能会增加延迟，并对系统的性能、可伸缩性和可用性产生负面影响。 更糟的是，长串直接 HTTP 通信可能导致同步微服务调用的深层和复杂链，如图4-9 所示：

![链接 HTTP 查询](./media/chaining-http-queries.png)

**图 4-9**. 链接 HTTP 查询

当然，您可以想像一下设计上图中所示的风险。 如果步骤3失败，会发生什么情况 \# ？ 或者步骤 \# 8 失败了？ 如何进行恢复？ 如果 \# 由于基础服务繁忙而导致步骤6慢，会发生什么情况？ 如何继续？ 即使全部工作正常，也可以考虑此调用会产生的延迟，这是每个步骤的延迟的总和。

上图中的大量耦合建议，服务未进行最佳建模。 这会使团队重新访问其设计来说。

### <a name="materialized-view-pattern"></a>具体化视图模式

用于删除微服务耦合的常用选项为[具体化视图模式](https://docs.microsoft.com/azure/architecture/patterns/materialized-view)。 使用此模式，微服务存储其自己的本地非规范化数据副本，这些数据由其他服务拥有。 它不会微服务查询产品目录和定价微服务，而是保留其自己的本地数据副本。 此模式消除了不必要的耦合，并改进了可靠性和响应时间。 整个操作在一个进程内执行。 我们将在第5章探讨此模式和其他数据问题。

### <a name="service-aggregator-pattern"></a>服务聚合器模式

消除微服务与微服务耦合的另一个选项是聚合器[微服务](https://devblogs.microsoft.com/cesardelatorre/designing-and-implementing-api-gateways-with-ocelot-in-a-microservices-and-container-based-architecture/)，如图4-10 中所示。

![聚合器服务](./media/aggregator-service.png)

**图 4-10**。 聚合微服务

此模式隔离了对多个后端微服务进行调用的操作，从而将其逻辑集中到专门的微服务中。  上图中的紫色结帐聚合器微服务协调结帐操作的工作流。 它包括按顺序顺序调用多个后端微服务。 工作流中的数据将聚合并返回给调用方。 尽管它仍实现直接 HTTP 调用，但聚合器微服务减少了后端微服务之间的直接依赖关系。

### <a name="requestreply-pattern"></a>请求/答复模式

分离同步 HTTP 消息的另一种方法是[请求-答复模式](https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReply.html)，这种模式使用队列通信。 使用队列的通信始终是单向通道，生成方发送消息，使用者接收消息。 使用此模式时，将实现请求队列和响应队列，如图4-11 所示。

![请求-答复模式](./media/request-reply-pattern.png)

**图 4-11**。 请求-答复模式

此处，消息创建者创建包含唯一相关 ID 的基于查询的消息，并将其放入请求队列。 使用服务取消排队消息，处理消息，并将响应放入具有相同相关 ID 的响应队列。 制造者服务取消排队该消息，并将其与相关 ID 匹配并继续处理。 我们将在下一节中详细介绍队列。

## <a name="commands"></a>命令

另一种类型的通信交互是*命令*。 微服务可能需要另一个微服务才能执行操作。 订单微服务可能需要发货微服务才能创建已批准订单的发货。 在图4-12 中，一个名为制造者的微服务将一条消息发送给另一个微服务。

![与队列的命令交互](./media/command-interaction-with-queue.png)

图 4-12  。 与队列的命令交互

大多数情况下，制造者不需要响应，可以*激发消息并*将其遗忘。 如果需要答复，则使用者将单独的消息发送回另一个通道上的制造者。 命令消息最好通过消息队列异步发送。 由轻型消息代理支持。 在前面的关系图中，请注意队列如何分隔并分离这两个服务。

消息队列是一个中介构造，使用者和使用者传递一条消息。 队列实现了异步的点到点消息传递模式。 制造者知道需要将命令发送到何处并正确路由。 队列保证消息仅由从通道读取的一个使用者实例处理。 在此方案中，制造者或使用者服务可以横向扩展，而不会影响另一个。 而且，每一方都可以完全不同的技术，这意味着我们可能会有一个 Java 微服务调用[Golang](https://golang.org)微服务。

第1章介绍了有关*后备服务*的信息。 支持服务是云本机系统所依赖的辅助资源。 消息队列是支持服务。 Azure 云支持通过两种类型的消息队列来实现命令消息传递： Azure 存储队列和 Azure 服务总线队列。

### <a name="azure-storage-queues"></a>Azure 存储队列

Azure 存储队列提供了一个简单的排队基础结构，该基础结构快速、经济实惠，并由 Azure 存储帐户提供支持。

[Azure 存储队列](https://docs.microsoft.com/azure/storage/queues/storage-queues-introduction)使用基于 REST 的排队机制，提供可靠且持久的消息传递机制。 它们提供了最小的功能集，但价格较低，并存储数百万条消息。 其容量范围最大为 500 TB。 单个消息的大小最大可为 64 KB。

可以通过使用 HTTP 或 HTTPS 的经过身份验证的调用，从世界上的任何地方访问消息。 存储队列可横向扩展到大量并发客户端，以处理流量高峰。

也就是说，该服务有一些限制：

- 不保证消息顺序。

- 消息在被自动删除前只能保留7天。

- 支持状态管理、重复检测或事务。

图4-13 显示了 Azure 存储队列的层次结构。

![存储队列层次结构](./media/storage-queue-hierarchy.png)

图 4-13  。 存储队列层次结构

在上图中，请注意存储队列如何将其消息存储在底层 Azure 存储帐户中。

对于开发人员，Microsoft 提供了多个用于存储队列处理的客户端和服务器端库。 支持大多数主要平台，包括 .NET、Java、JavaScript、Ruby、Python 和中转。 开发人员决不会直接与这些库通信。 这样做会将微服务代码紧密地用于 Azure 存储队列服务。 更好的做法是，隔离 API 的实现细节。 引入公开泛型操作并封装具体库的 intermediation 层或中间 API。 这种松散耦合使你能够将一个队列服务换出，而无需更改主线服务代码。

Azure 存储队列是在你的云本机应用程序中实现命令消息传递的经济实惠的选项。 尤其是当队列大小超过 80 GB 时，或简单的功能集是可接受的。 只需为消息存储付费;没有固定的每小时费用。

### <a name="azure-service-bus-queues"></a>Azure 服务总线队列

对于更复杂的消息传送要求，请考虑 Azure 服务总线队列。

在强大的消息基础结构的顶部， [Azure 服务总线](https://docs.microsoft.com/azure/service-bus-messaging/service-bus-messaging-overview)支持*中转消息传送模型*。 消息可靠地存储在代理（队列）中，直到使用者接收。 队列保证先进先出（FIFO）消息传递，与将消息添加到队列中的顺序相同。

消息的大小可以大得多，最高可达 256 KB。 消息在队列中保持无限长的时间。 Service Bus 不仅支持基于 HTTP 的调用，而且还支持[AMQP 协议](https://docs.microsoft.com/azure/service-bus-messaging/service-bus-amqp-overview)的完全支持。 AMQP 是一种跨供应商的开放标准，支持二进制协议和更高的可靠性。

服务总线提供了一组丰富的功能，包括[事务支持](https://docs.microsoft.com/azure/service-bus-messaging/service-bus-transactions)和[重复检测功能](https://docs.microsoft.com/azure/service-bus-messaging/duplicate-detection)。 队列保证每条消息 "最多传递一次"。 它会自动丢弃已经发送的消息。 如果制造者不确定，它可以重新发送相同的消息，服务总线保证只处理一个副本。 重复检测使你无需构建其他基础结构管道。

还有两个企业功能，即分区和会话。 常规服务总线队列由单个消息代理处理，并存储在单个消息存储区中。 但是，[服务总线分区](https://docs.microsoft.com/azure/service-bus-messaging/service-bus-partitioning)跨多个消息代理和消息存储分散队列。 总体吞吐量不再受单个消息代理或消息存储的性能限制。 消息存储的临时中断不会导致分区的队列不可用。

[服务总线会话](https://codingcanvas.com/azure-service-bus-sessions/)提供了一种对相关消息进行分组的方法。 假设工作流方案中，消息必须一起处理并且操作在结束时完成。 若要充分利用会话，必须为队列显式启用会话，并且每个相关消息必须包含相同的会话 ID。

但是，有一些重要的注意事项：服务总线队列大小限制为 80 GB，这比存储队列中的可用值小得多。 此外，服务总线队列在每个操作中产生基本成本和费用。

图4-14 概述了服务总线队列的高级体系结构。

![服务总线队列](./media/service-bus-queue.png)

图 4-14  。 服务总线队列

在上图中，请注意点到点关系。 同一个提供程序的两个实例将消息排队到单个服务总线队列。 每条消息仅由右侧三个使用者实例中的一个使用。 接下来，我们将讨论如何实现消息传递，其中不同的使用者可能会对同一消息感兴趣。

## <a name="events"></a>事件

消息队列是实现通信的有效方法，在此方法中，制造者可以异步发送使用者 a 消息。 但是，当*多个不同的使用者*对同一消息感兴趣时，会发生什么情况呢？ 每个使用者的专用消息队列不能很好地进行缩放，因此很难管理。

为了应对这种情况，我们将转到第三种类型的消息交互（即*事件*）。 一个微服务宣布发生了某个操作。 其他微服务，如有兴趣，对操作或事件做出反应。

事件是一个两步过程。 对于给定状态更改，微服务向消息代理发布事件，使其可供任何其他感兴趣的微服务使用。 相关的微服务会通过订阅消息代理中的事件获得通知。 使用[发布/订阅](https://docs.microsoft.com/azure/architecture/patterns/publisher-subscriber)模式来实现[基于事件的通信](https://docs.microsoft.com/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/integration-event-based-microservice-communications)。

图4-15 显示了一个购物篮微服务，其中包含两个其他微服务订阅的事件。

![事件驱动的消息传递](./media/event-driven-messaging.png)

**图 4-15**。 事件驱动的消息传递

请注意位于信道中间的*事件总线*组件。 它是一个自定义类，它封装消息代理并将其与基础应用程序分离。 订购和库存微服务独立运营事件，无任何其他知识，也不会微服务购物篮。 当注册的事件发布到事件总线时，它们会对其执行操作。

对于事件，我们将从队列技术转向*主题*。 [主题](https://docs.microsoft.com/azure/service-bus-messaging/service-bus-dotnet-how-to-use-topics-subscriptions)类似于队列，但支持一对多消息模式。 一个微服务发布一条消息。 多个订阅微服务可以选择接收并处理该消息。 图4-16 显示了一个主题体系结构。

![主题体系结构](./media/topic-architecture.png)

**图 4-16**。 主题体系结构

在上图中，发布服务器将消息发送到主题。 最终，订阅者从订阅接收消息。 在中间，主题将基于一组*规则*将消息转发到订阅，如深蓝框中所示。 规则作为将特定消息转发到订阅的筛选器。 此处，"CreateOrder" 事件将发送到订阅 \# 1 和订阅 \# 3，但不发送到订阅 \# 2。 "OrderCompleted" 事件将发送到订阅 \# 2 和订阅 \# 3。

Azure 云支持两个不同的主题服务： Azure 服务总线主题和 Azure EventGrid。

### <a name="azure-service-bus-topics"></a>Azure 服务总线主题

在 Azure 服务总线队列的相同可靠的中转消息模型的基础上，是[Azure 服务总线主题](https://docs.microsoft.com/azure/service-bus-messaging/service-bus-dotnet-how-to-use-topics-subscriptions)。 主题可以接收来自多个独立发布服务器的消息，并将消息发送到最多2000个订阅服务器。 订阅可以在运行时动态添加或删除，而无需停止系统或重新创建主题。

Azure 服务总线队列中的许多高级功能也可用于主题，包括[重复检测](https://docs.microsoft.com/azure/service-bus-messaging/duplicate-detection)和[事务支持](https://docs.microsoft.com/azure/service-bus-messaging/service-bus-transactions)。 默认情况下，服务总线主题由单个消息代理处理，并存储在单个消息存储区中。 但是，[服务总线分区](https://docs.microsoft.com/azure/service-bus-messaging/service-bus-partitioning)会通过将其分布到多个消息代理和消息存储区来缩放主题。

[计划的消息传递](https://docs.microsoft.com/azure/service-bus-messaging/message-sequencing)将标记有特定时间进行处理的消息。 消息不会在该时间之前出现在主题中。 通过[消息延迟](https://docs.microsoft.com/azure/service-bus-messaging/message-deferral)，可以将消息的检索推迟到以后的某个时间进行。 这两者通常用于工作流处理方案，在这些方案中，按特定顺序处理操作。 您可以推迟收到的消息的处理，直到上一工作完成。

Service Bus 主题是一项强大的、经过验证的技术，可用于在云本机系统中启用发布/订阅通信。

### <a name="azure-event-grid"></a>Azure 事件网格

尽管 Azure 服务总线是已经过测试的、具有一整套企业功能的消息传递代理，但[Azure 事件网格](https://docs.microsoft.com/azure/event-grid/overview)是块上的新孩子。

乍一看，事件网格可能只是另一个基于主题的消息系统。 但是，它在许多方面都是不同的。 它侧重于事件驱动的工作负荷，实现实时事件处理、深层 Azure 集成和开放平台的所有功能。 它适用于当代云本机和无服务器应用程序

作为集中式*事件底板*（或管道），事件网格可对 Azure 资源内的事件和来自你自己的服务的事件做出反应。

事件通知发布到事件网格主题，后者又将每个事件路由到订阅。 订阅服务器映射到订阅并使用事件。 与 Service Bus 类似，事件网格还支持*经过筛选的订户模式*，其中的订阅会针对其想要接收的事件设置规则。 事件网格提供快速的吞吐量，每秒保证10000000事件，从而实现近乎实时的交付-远远超过了 Azure 服务总线可以生成的内容。

事件网格的一个非常适合的位置是它与 Azure 基础结构的结构的深度集成。 Azure 资源（如 Cosmos DB）可以将内置事件直接发布到其他感兴趣的 Azure 资源，而无需自定义代码。 事件网格可以从 Azure 订阅、资源组或服务发布事件，使开发人员可以精细控制云资源的生命周期。 不过，事件网格并不局限于 Azure。 它是一个开放平台，可以使用发布自应用程序或第三方服务的自定义 HTTP 事件，并将事件路由到外部订阅服务器。

在从 Azure 资源发布和订阅本机事件时，无需进行任何编码。 通过简单的配置，可以将事件从一个 Azure 资源集成到另一个 Azure 资源，从而利用了主题和订阅的内置管道。 图4-17 显示事件网格分析。

![事件网格解析](./media/event-grid-anatomy.png)

**图 4-17**。 事件网格解析

EventGrid 和服务总线之间的主要区别是基础*消息交换模式*。

服务总线实现了一个较旧的样式*请求模型*，其中下游订阅服务器积极轮询主题订阅中的新消息。 在优势上，此方法使订阅者能够完全控制其处理消息的速度。 它控制在任意给定时间要处理的消息的时间和数量。 在处理之前，未读邮件保留在订阅中。 严重缺点是生成事件的时间与将该消息提取到订阅服务器进行处理之间的延迟时间。 此外，对于下一个事件，持续轮询的开销会消耗资源和资金。

不过，EventGrid 是不同的。 它实现了一种*推送模型*，在该模型中，事件会按接收发送到 EventHandlers，从而提供近乎实时的事件传递。 它还可降低成本，因为服务仅在需要使用事件时触发，而不是在轮询期间持续。 也就是说，事件处理程序必须处理传入的负载并提供阻止机制，以防止其成为不堪。 许多使用这些事件的 Azure 服务（例如 Azure Functions 和逻辑应用）提供自动自动缩放功能来处理增加的负载。  

事件网格是一种完全托管的无服务器云服务。 它可根据流量动态缩放，并仅对实际使用情况收费，而不是预先购买的容量。 每个月的前100000个操作是免费的–定义为事件入口（传入事件通知）、订阅传递尝试、管理调用和按主题筛选的操作。 使用99.99% 的可用性，EventGrid 保证在24小时内交付事件，并使用内置重试功能实现不成功的传递。 未传递的消息可以移动到 "死信" 队列以解决问题。  不同于 Azure 服务总线，事件网格进行优化以实现快速性能，并且不支持排序消息、事务和会话等功能。

### <a name="streaming-messages-in-the-azure-cloud"></a>在 Azure 云中传输消息

Azure 服务总线和事件网格为公开单一独立事件（例如新 Cosmos DB 文档）的应用程序提供了很大的支持。 但是，如果您的云本机系统需要处理*相关事件流*，该怎么办？ [事件流](https://docs.microsoft.com/archive/msdn-magazine/2015/february/microsoft-azure-the-rise-of-event-stream-oriented-systems)更复杂。 它们通常是按时间顺序排列的、相互关联的，并且必须作为一个组进行处理。

[Azure 事件中心](https://azure.microsoft.com/services/event-hubs/)是一种数据流式处理平台和事件引入服务，可收集、转换和存储事件。 它经过优化，可捕获流式处理数据，如从遥测上下文发出的连续事件通知。 此服务可高度缩放，每秒可以存储和[处理数百万事件](https://docs.microsoft.com/azure/event-hubs/event-hubs-about)。 图4-18 所示，它通常是事件管道的前门，它将插入流与事件使用分离。

![Azure 事件中心](./media/azure-event-hub.png)

**图 4-18**. Azure 事件中心

事件中心支持低延迟和可配置的时间保留。 与队列和主题不同，事件中心会在被使用者读取后保留事件数据。 此功能使其他数据分析服务（内部和外部）能够重播数据以供进一步分析。 存储在事件中心中的事件只会在保留期到期时删除，默认情况下为一天，但可配置。

事件中心支持常见的事件发布协议，包括 HTTPS 和 AMQP。 它还支持 Kafka 1.0。 [现有的 Kafka 应用程序可使用 Kafka 协议与事件中心进行通信](https://docs.microsoft.com/azure/event-hubs/event-hubs-for-kafka-ecosystem-overview)，该协议提供了管理大型 Kafka 群集的替代方法。 许多开源云本机系统都接受 Kafka。

事件中心通过[分区使用者模型](https://docs.microsoft.com/azure/event-hubs/event-hubs-features)实现消息流式处理，其中每个使用者只读取消息流的特定子集或分区。 此模式允许以极大的水平缩放规模进行事件处理，并提供队列和主题所不能提供的其他面向流的功能。 分区是事件中心内保留的有序事件。 当较新的事件到达时，它们将被添加到此序列的末尾。图4-19 显示事件中心中的分区。

![事件中心分区](./media/event-hub-partitioning.png)

**图 4-19**. 事件中心分区

每个使用者组读取消息流的一个子集或分区，而不是从同一资源中进行读取。

对于必须流式处理大量事件的云本机应用程序，Azure 事件中心可以是一个功能强大且经济实惠的解决方案。

>[!div class="step-by-step"]
>[上一页](front-end-communication.md)
>[下一页](grpc.md)
