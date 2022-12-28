## 概述
这是libp2p使用的可扩展 pubsub 协议规范，它基于随机主题网格和八卦。它是一种通用的 pubsub 协议，具有适量放大系数和良好缩放特性。该协议设计成通过更专业路由器进行扩展，路由器可添加协议消息和八卦，以提供针对特定应用程序配置的优化行为。

## 动机和之前的工作

libp2p [pubsub 接口规范](pubsub-interface-spec)定义了节点间交换的 RPC 消息，但故意不定义路由语义、连接管理或节点间如何交互等其他细节。这留给特定的 pubsub 协议，允许在协议设计中有很大的灵活性，来支持不同的用例/场景。

在介绍 [gossipsub 本身](#gossipsub八卦网状路由)之前，我们先来看看`floodsub` (最简单的 pubsub 实现)的属性。

### 一开始是floodsub

pubsub 接口的初始实现是`floodsub`，它采用了一种非常简单的消息传递策略——它通过让节点向已知特定主题中的所有其他节点广播，来简单地“淹没”网络。

通过泛洪，路由几乎是微不足道的：对于特定主题的每个传入消息，转发到该主题的所有已知对等节点。有一点逻辑：路由器维护一定时间的已收消息缓存，已收的消息不会被进一步转发。它也从不将消息转发回源或转发消息的对等方。

floodsub 路由策略具有以下非常理想的属性：

-   实施起来很简单;
-   它最大限度地减少了延迟；只要覆盖层连接足够好，消息就会通过最小延迟路径传递;
-   它非常坚固: 几乎没有维护逻辑或状态需要管理。

然而，问题是消息不仅遵循最小延迟路径，而是沿着所有的边界传输，从而造成洪水。网络的出站度是不受限的，而我们希望它是有限的，以减少带宽需求，并增加(分布性/去中心化)及可扩展性。

不受限的出站度给密集连接的节点带来了问题：它们有大量连接的对等点，但它们的带宽可能无法负担转发所有pubsub消息。类似地，**放大系数只受限于覆盖层中所有节点的度数之和**，这会为密集连接的覆盖层带来伸缩上的问题。

## gossipsub：八卦网状路由

gossipsub 通过对每个对等方的出站程度施加上限，并控制全局放大系数，来解决 floodsub 的缺点。

为了做到这一点，gossipsub 节点形成一个覆盖网格，其中每个节点将消息转发到其节点的子集，而不是主题中的所有已知节点。网格由节点在加入主题时构建，随后通过交换[控制消息](#control-messages)来维护。

网格的初始构造是随机的。当一个节点加入一个新主题时，它将检查[本地状态](#router-state)来查找已知该主题的成员节点。然后它将选择主题成员的子集，最多为D个，这是一个配置参数，用于阐述网络的输出度；这些节点将被添加到该主题的网格中，新添加的对等点将收到[`GRAFT`控制消息](#graft)通知。

当节点离开主题后，将通过[`PRUNE`消息](#prune)通知网格成员，并将网格从其 [本地状态](#router-state) 中删除。
[后期维护](#mesh-maintenance)将通过周期 [心跳](#heartbeat) 来维护，在节点上下线时，使网格大小保持在可接受的范围内。

网格连接是双向的: 当一个节点收到`GRAFT`消息，通知他被加到另一节点的网格中时，他反过来会将节点添加到自己的网格中，并假设对方已订阅该主题。
在稳定状态下（在[消息处理](#message-processing)后），如果一个对等`A`点在对等点的网格中`B`，那么对等`B`点也在对等点的网格中`A`。

为了让节点能“超越”自身的主题网格视图，我们使用_八卦_在整个网络中传播有关消息流的 _元数据_ 。这个八卦被发到不在网格中的随机子集。我们可以将网格成员视为“完整消息”节点，我们将主题中收到的所有消息转发给它们。主题中已知的其他节点，被认为是“仅元数据”节点，我们通过[心跳](#heartbeat)周期向它们发送八卦。

元数据可以是任意的，但作为基准，我们发送[`IHAVE`message](#ihave)，其中包括我们在过去几秒内收到的消息ID。这些消息内容应被缓存，以便收到八卦的对方可使用[`IWANT`消息](#iwant)来请求它们。

路由器可以使用此元数据来改进网格，例如构建在 gossipsub 之上的[episub](episub.md)路由可以创建流行病的广播树，适用于相对较小的发布者集向更大的观众广播的用例。

八卦的其他可能用途包括: 在覆盖网络的不同地方,重启动消息传输,以纠正下游的消息丢失; 或通过随机跳过hops跃点，来加速到网格中远距离对等点的消息传输。

## 依赖

Pubsub设计成适用于libp2p“生态系统”的模块化互补组件。因此，会假设已存在一些关键功能，而不申明成 pubsub 的一部分。

### [](#ambient-peer-discovery)周围节点发现

在节点可交换 pubsub 消息前，它们必须首先意识到彼此的存在。有几种可用的对等发现机制，例如：用于局域网的 MulticastDNS、通过 libp2p-kad-dht 的随机游走、rendezvous/会合协议，以及任何其他符合 libp2p 的对等节点发现机制。

由于 peer discovery 用途广泛，且不限于 pubsub，[pubsub 接口规范][pubsub-interface-spec]和本文档均未规定特定的发现机制。相反，假定此功能由环境提供。启用 pubsub 的 libp2p 应用程序还必须配置一个对等发现机制，该机制将发送周围连接事件以通知其他 libp2p 子系统（例如 pubsub）新的连接节点。

每当连接新的对等点时，gossipsub 实现都会检查对等点是否实现了 floodsub 或 gossipsub，如是，它会向它发送一个 hello 数据包，告知它当前订阅的主题。

## parameters

本节列出了控制 gossipsub 行为的配置参数，以及简短的描述和合理的默认值。在本文档的其他地方介绍了每个参数以及完整的上下文。

| 范围 | 目的 | 合理违约 |
| --- | --- | --- |
| `D` | 网络的期望出站度 | 6个 |
| `D_low` | 出站度下限 | 4个 |
| `D_high` | 出站度上限 | 12 |
| `D_lazy` | （可选）八卦排放的出站度 | `D` |
| `heartbeat_interval` | 心跳间隔 | 1秒 |
| `fanout_ttl` | 主题的扇出状态的生存时间 | 60秒 |
| `mcache_len` | 消息缓存中的历史窗口数 | 5个 |
| `mcache_gossip` | 发出八卦时使用的历史窗口数 | 3个 |
| `seen_ttl` | 已收消息 id 缓存的过期时间 | 2分钟 |

请注意，这`D_lazy`被认为是可选的。它用于控制[发送八卦](#gossip-emission)时的出站度，基于消息传播的紧急程度，这参数可分开设置。默认情况下，都使用`D`。

## router-state

路由器跟踪一些必要的状态，以维持稳定的主题网格，并发出有用的八卦。

状态大致可以分为两类：[节点状态](#peering-state)和[消息缓存](#message-cache)。

### peering-state

节点状态是路由如何跟踪其已知的具备pubsub能力的对等点以及与每个对等点的关系。

对等状态主要分为三个部分：

-   `peers`是一组支持 gossipsub 或 floodsub 的所有已知对等点的 id。在本文中 `peers.gossipsub`中，将表示支持 gossipsub 的对等点，而`peers.floodsub`表示 floodsub 对等点。
    
-   `mesh`是主题到一组对等点的Map，记录了已知订阅特定主题的覆盖网格中的对等节点列表。
    
-   `fanout`，就像`mesh`，是一个主题到一组对等点的Map，但是，该`fanout`映射包含的是 _未订阅_ 的主题。
    

除了上面列出的特定于 gossipsub 的状态之外，libp2p pubsub 框架还维护着一些“与路由无关”的状态。这包括订阅的主题列表，以及我们每个Peer订阅的主题列表。在本文档的其他地方，我们用`peers.floodsub[topic]`和`peers.gossipsub[topic]`表示在特定主题中，具备floodsub或gossipsub能力的对等节点列表。

### message-cache

消息缓存（也称为`mcache`）是一种数据结构，用于存储消息 ID 及其对应的内容，分段为“历史窗口”。每个窗口对应一个心跳间隔，并且窗口在[八卦发射](#gossip-emission)后的[心跳过程](#heartbeat)中移动。要保留的历史窗口数由 mcache_len [参数](#parameters)确定，而发送八卦时要检查的窗口数由`mcache_gossip`确定

消息缓存支持以下操作：

-   `mcache.put(m)`: 添加消息到当前窗口和缓存；
-   `mcache.get(id)`：如果消息仍然存在，则通过其 ID 从缓存中检索消息；
-   `mcache.get_gossip_ids(topic)`：检索特定主题的最近历史窗口中的消息ID。要检查的窗口数由`mcache_gossip`参数控制。
-   `mcache.shift()`: 移动当前窗口，丢弃早于缓存历史长度的消息 ( `mcache_len`)。

我们还保留了一个`seen`缓存，它是我们最近观察到的消息 ID 的定时最近最少使用的缓存。“最近”的值由`seen_ttl`[参数](#parameter) 决定，合理的默认值为两分钟，应将此值设为覆盖网络中近似的传播延迟。

`seen`缓存有两个用途。在所有 pubsub 实现中，我们可以在转发消息之前先检查`seen`缓存，以避免多次重新发布相同的消息。而在 gossipsub 中，在处理另一个节点发送的[`IHAVE`消息](#ihave)时会使用`seen`缓存，因此我们只请求我们没见过的消息。

在 go 实现中，`seen`缓存由 pubsub 框架提供，并与分开`mcache`，但在其他实现可能会将它们合而为一。

## topic-membership

[pubsub 接口规范][pubsub-interface-spec]定义了所有 libp2p pubsub 路由使用的基本 RPC 消息格式。作为 RPC 消息的一部分，节点可以包含有关他们希望订阅或取消订阅的主题的公告。这些公告将发送给所有已知的具有 pubsub 能力的对等方，无论我们目前是否有任何共同主题。

对于本文档，我们假设由底层 pubsub 框架发送 RPC 消息，来通知订阅更改。不基于当前libp2p pubsub框架的 gossipsub 实现，将需要自己实现这些 RPC 控制消息。

除了 pubsub 框架发送的`SUBSCRIBE`/`UNSUBSCRIBE`事件之外，gossipsub 还必须做额外的工作来维护它正在加入或离开的主题的网格。我们将下面的两个主题成员资格操作称为`JOIN(topic)`和`LEAVE(topic)`。

当应用程序调用`JOIN(topic)`时，路由将从其本地的[节点状态](#peering-state)中选择D个对等点来形成主题网格，这会先检查fanout Map，如果`fanout[topic]`中有对等点，路由会将这些对等点从 `fanout` 移到 `mesh[topic]`。如果fanout没有该主题，或fanout[topic]中包含的节点少于`D`个，路由器将尝试以 `peers.gossipsub[topic]` 来填充 `mesh[topic]` 对等点，这些对等点是已知的具有gossipsub能力的对等点，并且是该主题的成员。

无论他们来自`fanout`或`peers.gossipsub`，路由都会通过向`mesh[topic]`中的新成员，发送[`GRAFT`控制消息](#graft)来通知他们已被添加到网格中。

应用程序可以调用`LEAVE(topic)`来取消订阅某个主题。路由将通过向`mesh[topic]`中的所有对等方发送[`PRUNE`控制消息](#prune)来通知他们，以便他们可以从自己的主题网格中删除链接。发送`PRUNE`消息后，路由将忘记`mesh[topic]`，并将其从本地状态中删除。

## control-messages

交换控制消息以维护主题网格并发出八卦。本节列出了核心 gossipsub 协议中的控制消息，尽管值得注意的是 gossipsub 的扩展（例如[episub](./episub)可能会根据自己的需要定义更多的控制消息。

有关 gossipsub 路由如何响应控制消息的详细信息，请参阅[消息处理](#message-processing)。

[Protobuf](https://developers.google.com/protocol-buffers)部分详细介绍了控制消息的模式。

### graft

该`GRAFT`消息在主题网格中嫁接了一个新连接。`GRAFT`通知对等点，它被添加到 包含主题ID 的本地路由的网格视图中。

### prune

该`PRUNE`消息从主题网格中删除网格链接。`PRUNE`通知对等点它已从包含主题ID 的本地路由的网格视图中删除。

### ihave

`IHAVE`消息以 [gossip](#gossip-emission)形式发出。它向远程节点提供本地路由最近看到的消息列表。然后，远程节点可通过[`IWANT`内容](#iwant)消息请求完整的消息。

### iwant

该`IWANT`消息请求一个或多个消息ID的完整内容，这些消息的 ID 由远程对等点在[`IHAVE`消息](#ihave)中公布。

## message-processing

收到消息后，路由将首先处理消息负载。负载处理将根据应用程序定义的规则验证消息，并检查`seen`[缓存](#message-cache)以确定消息之前是否已被处理。它还将确保它不是消息的来源；如果路由收到它自己发布的消息，它不会进一步转发它。

如果消息是有效的，不是由路由器自己发布的，并且之前没有收到过，路由将转发该消息。首先，它会将消息转发给`peers.floodsub[topic]`的每个节点，以实现向后兼容。然后，它会将消息转发到其本地 gossipsub 主题网格中的每个对等点，即`mesh[topic]`

处理完消息负载后，路由将处理控制消息：

-   收到[`GRAFT(topic)`消息](#graft)后，路由将检查它是否确实订阅了消息中标识的主题。如果是这样，路由会将发件人添加到`mesh[topic]`. 如果路由不再订阅该主题，它会回复一条[`PRUNE(topic)`消息](#prune)，通知发送者它应该删除其网状链接。
    
-   收到[`PRUNE(topic)`消息](#prune)后，路由会将发件人从 中删除`mesh[topic]`。
    
-   收到[`IHAVE(ids)`消息](#ihave)后，路由将检查其`seen`缓存。如果`IHAVE`消息中包含尚未看到的消息 ID，路由将通过`IWANT`消息请求它们。
    
-   收到[`IWANT(ids)`消息](#iwant)后，路由将检查其[`mcache`](#message-cache)并将存在于其中的任何请求消息转发`mcache`给发送消息的对等方`IWANT`。
    

除了转发收到的消息外，路由当然可以代表自己发布消息，这些消息源自应用层。这与转发收到的消息非常相似：

-   首先，将消息发送给 中的每个对等点`peers.floodsub[topic]`。
-   如果路由订阅了主题，它会将消息发送给 中的所有对等点`mesh[topic]`。
-   如果路由没有订阅主题，它将检查`fanout[topic]`中的对等点集。如果此集合为空，路由将从`peers.gossipsub[topic]`中选择最多`D`个对等点并将它们添加到`fanout[topic]`。假设现在`fanout[topic]`中有一些对等体，路由将向每个对等体发送消息。

## control-message-piggybacking

八卦和其他控制消息不必在它们自己的消息中传输。相反，对于任何主题，它们可以合并并附着在常规流中的任何其他消息上。每当主题之间存在一些相关流时，这可能会导致消息速率降低，这对于密集连接的对等点可能很重要。

有关 piggyback（搭便车）实现的详细信息，请参阅[Go 实现](https://github.com/libp2p/go-libp2p-pubsub/blob/master/gossipsub.go)。

## heartbeat

每个对等点定期运行称为“心跳程序”的周期性稳定过程。心跳的频率由`heartbeat_interval` [参数](#parameters)控制，合理的默认值为 1 秒。

心跳具有三个功能：[网格维护](#mesh-maintenance)、[扇出维护](#fanout-maintenance)和[八卦发射](#gossip-emission)。

### mesh-maintenance

使用以下稳定算法维护主题网格：

```
for each topic in mesh:
 if |mesh[topic]| < D_low:
   select D - |mesh[topic]| peers from peers.gossipsub[topic] - mesh[topic]
    ; i.e. not including those peers that are already in the topic mesh.
   for each new peer:
     add peer to mesh[topic]
     emit GRAFT(topic) control message to peer

 if |mesh[topic]| > D_high:
   select |mesh[topic]| - D peers from mesh[topic]
   for each new peer:
     remove peer from mesh[topic]
     emit PRUNE(topic) control message to peer
```

该算法的[参数](#parameters)是：

-   `D`: 网络期望的出站度
-   `D_low`：可接受的下限阈值`D`。如果给定主题网格中的对等点少于`D_low`，我们会尝试添加新的对等点。
-   `D_high`：可接受的上限阈值`D`。如果给定主题网格中大于`D_high`，我们将随机选择节点进行删除。

### [](#fanout-maintenance)扇出维护

通过`fanout`跟踪每个主题的最后发布时间来维护地图。如果我们不在可配置的TTL时间内向主题发布任何消息，则扇出状态的主题将被丢弃。

我们还尝试确保每个`fanout[topic]`集合至少有`D`成员。

扇出维护算法为：

```
for each topic in fanout:
  if time since last published > fanout_ttl
    remove topic from fanout
  else if |fanout[topic]| < D
    select D - |fanout[topic]| peers from peers.gossipsub[topic] - fanout[topic]
    add the peers to fanout[topic]
```

该算法的[参数](#parameters)是：

-   `D`：所需的网络出站度。
-   `fanout_ttl`：我们为每个主题保持扇出状态的时间。如果我们在`fanout_ttl`内没发布新消息，则该`fanout[topic]`集合将被丢弃。

### [](#gossip-emission)八卦发射

八卦被发送到每个主题的随机选择的对等点，这些点还不是主题网格的成员：

```
for each topic in mesh+fanout:
  let mids be mcache.get_gossip_ids(topic)
  if mids is not empty:
    select D peers from peers.gossipsub[topic]
    for each peer not in mesh[topic] or fanout[topic]
      emit IHAVE(mids)

shift the mcache
```

请注意，我们使用相同的参数`D`作为八卦和网格成员的目标度数，但这不是规范的。可以使用一个单独的参数`D_lazy`来显式控制八卦传播因子，这允许调整 急切和惰性消息传输之间的权衡。

## [](#protobuf)协议缓冲区

gossipsub 协议用一个新字段`control`扩展了[现有的`RPC`消息结构][pubsub-spec-rpc]. 这是一个`ControlMessage`可能包含一个或多个控制消息的实例。

四个控制消息分别`ControlIHave`是[`IHAVE`](#ihave)消息、`ControlIWant`消息[`IWANT`](#iwant)、`ControlGraft`消息[`GRAFT`](#graft)和消息`ControlPrune`。[`PRUNE`](#prune)

protobuf如下：

```proto
message RPC {
    // ... see definition in pubsub interface spec
optional ControlMessage control = 3;
}

message ControlMessage {
repeated ControlIHave ihave = 1;
repeated ControlIWant iwant = 2;
repeated ControlGraft graft = 3;
repeated ControlPrune prune = 4;
}

message ControlIHave {
optional string topicID = 1;
repeated bytes messageIDs = 2;
}

message ControlIWant {
repeated bytes messageIDs = 1;
}

message ControlGraft {
optional string topicID = 1;
}

message ControlPrune {
optional string topicID = 1;
}
```

[pubsub-interface-spec]: ../README.md
[pubsub-spec-rpc]: ../README.md#the-rpc
