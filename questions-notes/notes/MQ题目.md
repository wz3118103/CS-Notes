# 1.ActiveMQ

Q.什么是 ActiveMQ?
* activeMQ 是一种开源的，实现了 JMS1.1 规范的，面向消息(MOM)的中间件，为应用程序提供高效的、可扩展的、稳定的和安全的企业级消息通信

Q. ActiveMQ 服务器宕机怎么办？
* 这得从 ActiveMQ 的储存机制说起。在通常的情况下，非持久化消息是存储在内存中的，持久化消息是存储在文件中的，它们的最大限制在配置文件的<systemUsage>节点中配置。但是，在非持久化消息堆积到一定程度，内存告急的时候，ActiveMQ 会将内存中的非持久化消息写入临时文件中，以腾出内存。虽然都保存到了文件里，但它和持久化消息的区别是，重启后持久化消息会从文件中恢复，非持久化的临时文件会直接删除。
* 那如果文件增大到达了配置中的最大限制的时候会发生什么？我做了以下实验：
设置 2G 左右的持久化文件限制，大量生产持久化消息直到文件达到最大限制，此时生产者阻塞，但消费者可正常连接并消费消息，等消息消费掉一部分，文件删除又腾出空间之后，生产者又可继续发送消息，服务自动恢复正常。
设置 2G 左右的临时文件限制，大量生产非持久化消息并写入临时文件，在达到最大限制时，生产者阻塞，消费者可正常连接但不能消费消息，或者原本慢速消费的消费者，消费突然停止。整个系统可连接，但是无法提供服务，就这样挂了。
* 具体原因不详，解决方案：尽量不要用非持久化消息，非要用的话，将临时文件限制尽可能的调大。

Q.丢消息怎么办？
* 这得从 java 的 java.net.SocketException 异常说起。简单点说就是当网络发送方发送一堆数据，然后调用 close 关闭连接之后。这些发送的数据都在接收者的缓存里，接收者如果调用 read 方法仍旧能从缓存中读取这些数据，尽管对方已经关闭了连接。但是当接收者尝试发送数据时，由于此时连接已关闭，所以会发生异常，这个很好理解。不过需要注意的是，当发生 SocketException 后，原本缓存区中数据也作废了，此时接收者再次调用 read 方法去读取缓存中的数据，就会报 Software caused connection abort: recv failed 错误。
* 通过抓包得知，ActiveMQ 会每隔 10 秒发送一个心跳包，这个心跳包是服务器发送给客户端的，用来判断客户端死没死。如果你看过上面第一条，就会知道非持久化消息堆积到一定程度会写到文件里，这个写的过程会阻塞所有动作，而且会持续 20 到 30 秒，并且随着内存的增大而增大。当客户端发完消息调用connection.close()时，会期待服务器对于关闭连接的回答，如果超过 15 秒没回答就直接调用 socket 层 的 close 关闭 tcp 连接了。这时客户端发出的消息其实还在服务器的缓存里等待处理，不过由于服务器心跳包的设置，导致发生了 java.net.SocketException 异常，把缓存里的数据作废了，没处理的消息全部丢失。
* 解决方案：用持久化消息，或者非持久化消息及时处理不要堆积，或者启动事务，启动事务后，commit()方法会负责任的等待服务器的返回，也就不会关闭连接导致消息丢失了。

Q.持久化消息非常慢。
* 默认的情况下，非持久化的消息是异步发送的，持久化的消息是同步发送的，遇到慢一点的硬盘，发送消息的速度是无法忍受的。但是在开启事务的情况下，消息都是异步发送的，效率会有 2 个数量级的提升。所以在发送持久化消息时，请务必开启事务模式。其实发送非持久化消息时也建议开启事务，因为根本不会影响性能。

Q.消息的不均匀消费。
* 有时在发送一些消息之后，开启 2 个消费者去处理消息。会发现一个消费者处理了所有的消息，另一个消费者根本没收到消息。原因在于 ActiveMQ 的 prefetch 机制。当消费者去获取消息时，不会一条一条去获取，而是一次性获取一批，默认是 1000 条。这些预获取的消息，在还没确认消费之前，在管理控制台还是可以看见这些消息的，但是不会再分配给其他消费者，此时这些消息的状态应该算作“已分配未消费”，如果消息最后被消费，则会在服务器端被删除，如果消费者崩溃，则这些消息会被重新分配给新的消费者。
* 但是如果消费者既不消费确认，又不崩溃，那这些消息就永远躺在消费者的缓存区里无法处理。更通常的情况是，消费这些消息非常耗时，你开了 10 个消费者去处理，结果发现只有一台机器吭哧吭哧处理，另外 9 台啥事不干。
解决方案：将 prefetch 设为 1，每次处理 1 条消息，处理完再去取，这样也慢不了多少。

Q.死信队列。
* 如果你想在消息处理失败后，不被服务器删除，还能被其他消费者处理或重试，可以关闭AUTO_ACKNOWLEDGE，将 ack 交由程序自己处理。那如果使用了 AUTO_ACKNOWLEDGE，消息是什么时候被确认的，还有没有阻止消息确认的方法？有！
* 消费消息有 2 种方法，
  - 一种是调用 consumer.receive()方法，该方法将阻塞直到获得并返回一条消息。这种情况下，消息返回给方法调用者之后就自动被确认了。
  - 另一种方法是采用 listener 回调函数，在有消息到达时，会调用 listener 接口的 onMessage 方法。在这种情况下，在 onMessage 方法执行完毕后，消息才会被确认，此时只要在方法中抛出异常，该消息就不会被确认。那么问题来了，如果一条消息不能被处理，会被退回服务器重新分配，如果只有一个消费者，该消息又会重新被获取，重新抛异常。就算有多个消费者，往往在一个服务器上不能处理的消息，在另外的服务器上依然不能被处理。难道就这么退回--获取--报错死循环了吗？
  在重试 6 次后，ActiveMQ 认为这条消息是“有毒”的，将会把消息丢到死信队列里。如果你的消息不见了，去 ActiveMQ.DLQ 里找找，说不定就躺在那里。

Q.ActiveMQ 中的消息重发时间间隔和重发次数吗？
* ActiveMQ：是 Apache 出品，最流行的，能力强劲的开源消息总线。是一个完全支持 JMS1.1 和 J2EE 1.4规范的 JMS Provider 实现。JMS（Java 消息服务）：是一个 Java 平台中关于面向消息中间件（MOM） 的 API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。
* 首先，我们得大概了解下，在哪些情况下，ActiveMQ 服务器会将消息重发给消费者，这里为简单起见，假定采用的消息发送模式为队列（即消息发送者和消息接收者）。
  - 如果消息接收者在处理完一条消息的处理过程后没有对 MOM 进行应答，则该消息将由 MOM 重发. 
  - 如果我们队某个队列设置了预读参数（consumer.prefetchSize），如果消息接收者在处理第一条消息时（没向 MOM 发送消息接收确认）就宕机了，则预读数量的所有消息都将被重发!
  - 如果 Session 是事务的，则只要消息接收者有一条消息没有确认，或发送消息期间 MOM 或客户端某一方突然宕机了，则该事务范围中的所有消息 MOM 都将重发。
  - 说到这里，大家可能会有疑问，ActiveMQ 消息服务器怎么知道消费者客户端到底是消息正在处理中还没来得急对消息进行应答还是已经处理完成了没有应答或是宕机了根本没机会应答呢？其实在所有的客户端机器上，内存中都运行着一套客户端的 ActiveMQ 环境，该环境负责缓存发来的消息，负责维持着和ActiveMQ 服务器的消息通讯，负责失效转移（fail-over）等，所有的判断和处理都是由这套客户端环境来完成的。
* 我们可以来对 ActiveMQ 的重发策略（Redelivery Policy）来进行自定义配置，其中的配置参数主要有以
下几个：
  - collisionAvoidanceFactor 默认值 0.15 , 设置防止冲突范围的正负百分比，只有启用useCollisionAvoidance 参数时才生效。
  - maximumRedeliveries 默认值 6 , 最大重传次数，达到最大重连次数后抛出异常。为-1 时不限制次数，为 0 时表示不进行重传。
  - maximumRedeliveryDelay 默认值-1, 最大传送延迟，只在 useExponentialBackOff 为 true 时有效（V5.5），假设首次重连间隔为 10ms，倍数为 2，那么第二次重连时间间隔为 20ms，第三次重连时间间隔为40ms，当重连时间间隔大的最大重连时间间隔时，以后每次重连时间间隔都为最大重连时间间隔。
  - initialRedeliveryDelay 默认值 1000L, 初始重发延迟时间
  - redeliveryDelay 默认值 1000L, 重发延迟时间，当 initialRedeliveryDelay=0 时生效（v5.4） l useCollisionAvoidance 默认值 false, 启用防止冲突功能，因为消息接收时是可以使用多线程并发处理的，应该是为了重发的安全性，避开所有并发线程都在同一个时间点进行消息接收处理。所有线程在同一个时间点处理时会发生什么问题呢？应该没有问题，只是为了平衡 broker 处理性能，不会有时很忙，有时很空闲。
  - useExponentialBackOff 默认值 false, 启用指数倍数递增的方式增加延迟时间。
  - backOffMultiplier 默认值 5, 重连时间间隔递增倍数，只有值大于 1 和启用 useExponentialBackOff参数时才生效。



Q.activemq的几种通信方式
* publish(发布)-subscribe(订阅)(发布-订阅方式)
发布/订阅方式用于多接收客户端的方式.作为发布订阅的方式，可能存在多个接收客户端，并且接收端客户端与发送客户端存在时间上的依赖。一个接收端只能接收他创建以后发送客户端发送的信息。作为subscriber,在接收消息时有两种方法，destination的receive方法，和实现messagelistener接口的onMessage方法。
* p2p(point-to-point)(点对点)
p2p的过程则理解起来比较简单。它好比是两个人打电话，这两个人是独享这一条通信链路的。一方发送消息，另外一方接收，就这么简单。在实际应用中因为有多个用户对使用p2p的链路。
在p2p的场景里，相互通信的双方是通过一个类似于队列的方式来进行交流。和前面pub-sub的区别在于一个topic有一个发送者和多个接收者，而在p2p里一个queue只有一个发送者和一个接收者。

Q.activemq如果数据提交不成功怎么办(消息丢失)
* publish(发布)-subscribe(订阅)方式的处理
发布订阅模式的通信方式，默认情况下只通知一次，如果接收不到此消息就没有了。这种场景只适用于对消息送达率要求不高的情况。如果要求消息必须送达不可以丢失的话，需要配置持久订阅。每个订阅端定义一个id，
<propertyname="clientId"在订阅是向activemq注册。发布消息<propertyname="subscriptionDurable"value="true"/>和接收消息时需要配置发送模式为持久化
template.setDeliveryMode(DeliveryMode.PERSISTENT);。此时如果客户端接收不到消息，消息会持久化到服务端(就是硬盘上)，直到客户端正常接收后为止。
* p-p(点对点)方式的处理
点对点模式的话，如果消息发送不成功此消息默认会保到activemq服务端直到有消费者将其消费，所以此时消息是不会丢失的。

Q.如何解决消息重复问题
所谓消息重复,就是消费者接收到了重复的消息,一般来说我们对于这个问题的处理要把握下面几点,
* 消息不丢失(上面已经处理了)
* 消息不重复执行一般来说我们可以在业务段加一张表,用来存放消息是否执行成功,每次业务事物commit之后,告知服务端,已经处理过该消息,这样即使你消息重发了,也不会导致重复处理
大致流程如下:
业务端的表记录已经处理消息的id,每次一个消息进来之前先判断该消息是否执行过,如果执行过就放弃,如果没有执行就开始执行消息,消息执行完成之后存入这个消息的id

Q.大量的消息每页被消费，能否发生oom异常？
* 可以控制每个消息队列中数据的大小，不允许无线填充数据，避免该队列多大，导致过度消耗系统资源问题；可以控制队列的内存大小；

Q.activeMQ发送消息的方式有哪些？
* 同步方式
两个通信应用服务之间必须要进行同步，两个服务之间必须都是正常运行的。发送程序和接收程序都必须一直处于运行状态，并且随时做好相互通信的准备。
发送程序首先向接收程序发起一个请求，称之为发送消息，发送程序紧接着就会堵塞当前自身的进程，不与其他应用进行任何的通信以及交互，等待接收程序的响应，待发送消息得到接收程序的返回消息之后会继续向下运行，进行下一步的业务处理。
* 异步方式
两个通信应用之间可以不用同时在线等待，任何一方只需各自处理自己的业务，比如发送方发送消息以后不用登录接收方的响应，可以接着处理其他的任务。也就是说发送方和接收方都是相互独立存在的，发送方只管方，接收方只
能接收，无须去等待对方的响应。
Java中JMS就是典型的异步消息处理机制，JMS消息有两种类型：点对点、发布/订阅。

Q.activeMQ如何调优
* 1.使用非持久化消息;
* 2.需要确保消息发送成功时使用事务来将消息分批组合.


# 2.RabbitMQ



Q.RabbitMQ 中的 broker 是指什么？cluster 又是指什么？
* broker 是指一个或多个 erlang node 的逻辑分组，且 node 上运行着 RabbitMQ 应用程序。cluster 是在 broker 的基础之上，增加了 node 之间共享元数据的约束。

Q.什么是元数据？元数据分为哪些类型？包括哪些内容？与 cluster 相关的元数据有哪些？元数据是如何保存的？元数据在 cluster 中是如何分布的？
* 在非 cluster 模式下，元数据主要分为 Queue 元数据（queue 名字和属性等）、Exchange 元数据（exchange 名字、类型和属性等）、Binding 元数据（存放路由关系的查找表）、Vhost 元数据（vhost 范围内针对前三者的名字空间约束和安全属性设置）。在cluster 模式下，还包括 cluster 中 node 位置信息和 node 关系信息。元数据按照 erlang node 的类型确定是仅保存于 RAM 中，还是同时保存在 RAM 和 disk 上。元数据在cluster 中是全 node 分布的。
下图所示为 queue 的元数据在单 node 和 cluster 两种模式下的分布图。

Q.RAM node 和 disk node 的区别？
* RAM node 仅将 fabric（即 queue、exchange 和 binding 等 RabbitMQ 基础构件）相关元数据保存到内存中，但 disk node 会在内存和磁盘中均进行存储。RAM node 上唯一会存储到磁盘上的元数据是 cluster 中使用的 disk node 的地址。要求在 RabbitMQ cluster中至少存在一个 disk node 。

Q.RabbitMQ 上的一个 queue 中存放的 message 是否有数量限制？
* 可以认为是无限制，因为限制取决于机器的内存，但是消息过多会导致处理效率的下降。

Q.RabbitMQ 概念里的 channel、exchange 和 queue 这些东东是逻辑概念，还是对应着进程实体？这些东东分别起什么作用？
* queue 具有自己的 erlang 进程；exchange 内部实现为保存 binding 关系的查找表；channel 是实际进行路由工作的实体，即负责按照 routing_key 将 message 投递给queue 。由 AMQP 协议描述可知，channel 是真实 TCP 连接之上的虚拟连接，所有AMQP 命令都是通过 channel 发送的，且每一个 channel 有唯一的 ID。一个 channel 只能被单独一个操作系统线程使用，故投递到特定 channel 上的 message 是有顺序的。但
一个操作系统线程上允许使用多个 channel 。channel 号为 0 的 channel 用于处理所有对于当前 connection 全局有效的帧，而 1-65535 号 channel 用于处理和特定 channel 相关的帧。AMQP 协议给出的 channel 复用模型如下其中每一个 channel 运行在一个独立的线程上，多线程共享同一个 socket。

Q.vhost 是什么？起什么作用？
* vhost 可以理解为虚拟 broker ，即 mini-RabbitMQ server。其内部均含有独立的queue、exchange 和 binding 等，但最最重要的是，其拥有独立的权限系统，可以做到vhost 范围的用户控制。当然，从 RabbitMQ 的全局角度，vhost 可以作为不同权限隔离的手段（一个典型的例子就是不同的应用可以跑在不同的 vhost 中）。


Q.在单 node 系统和多 node 构成的 cluster 系统中声明 queue、exchange ，以及进行 binding 会有什么不同？
* 当你在单 node 上声明 queue 时，只要该 node 上相关元数据进行了变更，你就会得到 Queue.Declare-ok 回应；而在 cluster 上声明 queue ，则要求 cluster 上的全部node 都要进行元数据成功更新，才会得到 Queue.Declare-ok 回应。另外，若 node 类型为 RAM node 则变更的数据仅保存在内存中，若类型为 disk node 则还要变更保存在磁盘上的数据。

Q.客户端连接到 cluster 中的任意 node 上是否都能正常工作？
* 是的。客户端感觉不到有何不同。

Q.若 cluster 中拥有某个 queue 的 owner node 失效了，且该 queue 被声明具有durable 属性，是否能够成功从其他 node 上重新声明该 queue ？
* 不能，在这种情况下，将得到 404 NOT_FOUND 错误。只能等 queue 所属的 node恢复后才能使用该 queue 。但若该 queue 本身不具有 durable 属性，则可在其他 node上重新声明。

Q.cluster 中 node 的失效会对 consumer 产生什么影响？若是在 cluster 中创建了mirrored queue ，这时 node 失效会对 consumer 产生什么影响？
* 若是 consumer 所连接的那个 node 失效（无论该 node 是否为 consumer 所订阅queue 的 owner node），则 consumer 会在发现 TCP 连接断开时，按标准行为执行重连逻辑，并根据“Assume Nothing”原则重建相应的 fabric 即可。若是失效的 node 为consumer 订阅 queue 的 owner node，则 consumer 只能通过 Consumer Cancellation Notification 机制来检测与该 queue 订阅关系的终止，否则会出现傻等却没有任何消息来到的问题。

Q.能够在地理上分开的不同数据中心使用 RabbitMQ cluster 么？
* 不能。
  - 你无法控制所创建的 queue 实际分布在 cluster 里的哪个 node 上（一般使用 HAProxy + cluster 模型时都是这样），这可能会导致各种跨地域访问时的常见问题；
  - Erlang 的 OTP 通信框架对延迟的容忍度有限，这可能会触发各种超时，导致业务疲于处理；
  - 在广域网上的连接失效问题将导致经典的“脑裂”问题，而RabbitMQ 目前无法处理（该问题主要是说 Mnesia）。


Q.为什么 heavy RPC 的使用场景下不建议采用 disk node ？
* heavy RPC 是指在业务逻辑中高频调用 RabbitMQ 提供的 RPC 机制，导致不断创建、销毁 reply queue ，进而造成 disk node 的性能问题（因为会针对元数据不断写盘）。所以在使用 RPC 机制时需要考虑自身的业务场景。

Q.向不存在的 exchange 发 publish 消息会发生什么？向不存在的 queue 执行consume 动作会发生什么？
* 都会收到 Channel.Close 信令告之不存在（内含原因 404 NOT_FOUND）。

Q.routing_key 和 binding_key 的最大长度是多少？
* 255 字节。

Q.RabbitMQ 允许发送的 message 最大可达多大？
* 根据 AMQP 协议规定，消息体的大小由 64-bit 的值来指定，所以你就可以知道到底能发多大的数据了。

Q.什么情况下 producer 不主动创建 queue 是安全的？
* 1.message 是允许丢失的；
* 2.实现了针对未处理消息的 republish 功能（例如采用Publisher Confirm 机制）。

Q.“dead letter”queue 的用途？
* 当消息被 RabbitMQ server 投递到 consumer 后，但 consumer 却通过 Basic.Reject进行了拒绝时（同时设置 requeue=false），那么该消息会被放入“dead letter”queue 中。
该 queue 可用于排查 message 被 reject 或 undeliver 的原因。

Q.为什么说保证 message 被可靠持久化的条件是 queue 和 exchange 具有durable 属性，同时 message 具有 persistent 属性才行？
* binding 关系可以表示为 exchange – binding – queue 。从文档中我们知道，若要求投递的 message 能够不丢失，要求 message 本身设置 persistent 属性，要求 exchange和 queue 都设置 durable 属性。其实这问题可以这么想，若 exchange 或 queue 未设置durable 属性，则在其 crash 之后就会无法恢复，那么即使 message 设置了 persistent 属性，仍然存在 message 虽然能恢复但却无处容身的问题；同理，若 message 本身未设置persistent 属性，则 message 的持久化更无从谈起。

Q.什么情况下会出现 blackholed 问题？
* blackholed 问题是指，向 exchange 投递了 message ，而由于各种原因导致该message 丢失，但发送者却不知道。可导致 blackholed 的情况：1.向未绑定 queue 的exchange 发送 message；2.exchange 以 binding_key key_A 绑定了 queue queue_A，但向该 exchange 发送 message 使用的 routing_key 却是 key_B。

Q.如何防止出现 blackholed 问题？
* 没有特别好的办法，只能在具体实践中通过各种方式保证相关 fabric 的存在。另外，如果在执行 Basic.Publish 时设置 mandatory=true ，则在遇到可能出现 blackholed 情况时，服务器会通过返回 Basic.Return 告之当前 message 无法被正确投递（内含原因 312NO_ROUTE）。

Q.Consumer Cancellation Notification 机制用于什么场景？
* 用于保证当镜像 queue 中 master 挂掉时，连接到 slave 上的 consumer 可以收到自身 consume 被取消的通知，进而可以重新执行 consume 动作从新选出的 master 出获得消息。若不采用该机制，连接到 slave 上的 consumer 将不会感知 master 挂掉这个事情，导致后续无法再收到新 master 广播出来的 message 。另外，因为在镜像 queue 模式下，存在将 message 进行 requeue 的可能，所以实现 consumer 的逻辑时需要能够正确处理出现重复 message 的情况。

Q.Basic.Reject 的用法是什么？
* 该信令可用于 consumer 对收到的 message 进行 reject 。若在该信令中设置requeue=true，则当 RabbitMQ server 收到该拒绝信令后，会将该 message 重新发送到下一个处于 consume 状态的 consumer 处（理论上仍可能将该消息发送给当前consumer）。若设置 requeue=false ，则 RabbitMQ server 在收到拒绝信令后，将直接将该message 从 queue 中移除。
另外一种移除 queue 中 message 的小技巧是，consumer 回复 Basic.Ack 但不对获取到的message 做任何处理。而 Basic.Nack 是对 Basic.Reject 的扩展，以支持一次拒绝多条 message 的能力。

Q.为什么不应该对所有的 message 都使用持久化机制？
* 首先，必然导致性能的下降，因为写磁盘比写 RAM 慢的多，message 的吞吐量可能有 10 倍的差距。
* 其次，message 的持久化机制用在 RabbitMQ 的内置 cluster 方案时会出现“坑爹”问题。矛盾点在于，若 message 设置了 persistent 属性，但 queue 未设置durable 属性，那么当该 queue 的 owner node 出现异常后，在未重建该 queue 前，发往该 queue 的 message 将被 blackholed ；若 message 设置了 persistent 属性，同时queue 也设置了 durable 属性，那么当 queue 的 owner node 异常且无法重启的情况下，则该 queue 无法在其他 node 上重建，只能等待其 owner node 重启后，才能恢复该 queue 的使用，而在这段时间内发送给该 queue 的 message 将被 blackholed 。
* 所以，是否要对 message 进行持久化，需要综合考虑性能需要，以及可能遇到的问题。若想达到 100,000 条/秒以上的消息吞吐量（单 RabbitMQ 服务器），则要么使用其他的方式来确保 message 的可靠 delivery ，要么使用非常快速的存储系统以支持全持久化（例如使用 SSD）。
* 另外一种处理原则是：仅对关键消息作持久化处理（根据业务重要程度），且应该保证关键消息的量不会导致性能瓶颈。

Q.RabbitMQ 中的 cluster、mirrored queue，以及 warrens 机制分别用于解决什么问题？存在哪些问题？
* cluster 是为了解决当 cluster 中的任意 node 失效后，producer 和 consumer 均可以通过其他 node 继续工作，即提高了可用性；另外可以通过增加 node 数量增加 cluster的消息吞吐量的目的。cluster 本身不负责 message 的可靠性问题（该问题由 producer 通过各种机制自行解决）；cluster 无法解决跨数据中心的问题（即脑裂问题）。另外，在cluster 前使用 HAProxy 可以解决 node 的选择问题，即业务无需知道 cluster 中多个node 的 ip 地址。可以利用 HAProxy 进行失效 node 的探测，可以作负载均衡。下图为
HAProxy + cluster 的模型。
* Mirrored queue 是为了解决使用 cluster 时所创建的 queue 的完整信息仅存在于单一node 上的问题，从另一个角度增加可用性。若想正确使用该功能，需要保证：
  - 1.consumer需要支持 Consumer Cancellation Notification 机制；
  - 2.consumer 必须能够正确处理重复message 。
* Warrens 是为了解决 cluster 中 message 可能被 blackholed 的问题，即不能接受producer 不停 republish message 但 RabbitMQ server 无回应的情况。Warrens 有两种构成方式，一种模型是两台独立的 RabbitMQ server + HAProxy ，其中两个 server 的状态分别为 active 和 hot-standby 。该模型的特点为：两台 server 之间无任何数据共享和协议交互，两台 server 可以基于不同的 RabbitMQ 版本。如下图所示
另一种模型为两台共享存储的 RabbitMQ server + keepalived ，其中两个 server 的状态分别为 active 和 cold-standby 。该模型的特点为：两台 server 基于共享存储可以做到完全恢复，要求必须基于完全相同的 RabbitMQ 版本。如下图所示
* Warrens 模型存在的问题：对于第一种模型，虽然理论上讲不会丢失消息，但若在该模型上使用持久化机制，就会出现这样一种情况，即若作为 active 的 server 异常后，持久化在该 server 上的消息将暂时无法被 consume ，因为此时该 queue 将无法在作为 hot standby 的 server 上被重建，所以，只能等到异常的active server 恢复后，才能从其上的queue 中获取相应的 message 进行处理。而对于业务来说，需要具有：
  - a.感知 AMQP 连接断开后重建各种 fabric 的能力；
  - b.感知 active server 恢复的能力；
  - c.切换回 active server 的时机控制，以及切回后，针对 message 先后顺序产生的变化进行处理的能力。
* 对于第二种模型，因为是基于共享存储的模式，所以导致 active server 异常的条件，可能同样会导致 cold-standby server 异常；另外，在该模型下，要求 active 和 cold-standby的 server 必须具有相同的 node 名和 UID ，否则将产生访问权限问题；最后，由于该模型是冷备方案，故无法保证 cold-standby server 能在你要求的时限内成功启动。


Q.Basic.Reject的用法是什么？
* 该信令可用于consumer对收到的message进行reject。若在该信令中设置requeue=true，则当RabbitMQserver收到该拒绝信令后，会将该message重新发送到下一个处于consume状态的consumer处（理论上仍可能将该消息发送给当前consumer）。若设置requeue=false，则RabbitMQserver在收到拒绝信令后，将直接将该message从queue中移除。
* 另外一种移除queue中message的小技巧是，consumer回复Basic.Ack但不对获取到的message做任何处理。而Basic.Nack是对Basic.Reject的扩展，以支持一次拒绝多条message的能力。


Q.为什么heavyRPC的使用场景下不建议采用disknode？
* heavyRPC是指在业务逻辑中高频调用RabbitMQ提供的RPC机制，导致不断创建、销毁replyqueue，进而造成disknode的性能问题（因为会针对元数据不断写盘）。所以在使用RPC机制时需要考虑自身的业务场景。

Q.向不存在的exchange发publish消息会发生什么？向不存在的queue执行consume动作会发生什么？
* 都会收到Channel.Close信令告之不存在（内含原因404NOT_FOUND）。

Q.什么情况下producer不主动创建queue是安全的？
* 1.message是允许丢失的；
* 2.实现了针对未处理消息的republish功能（例如采用PublisherConfirm机制）。

Q.“deadletter”queue的用途？
* 当消息被RabbitMQserver投递到consumer后，但consumer却通过Basic.Reject进行了拒绝时（同时设置requeue=false），那么该消息会被放入“deadletter”queue中。该queue可用于排查message被reject或undeliver的原因。

Q.为什么说保证message被可靠持久化的条件是queue和exchange具有durable属性，同时message具persistent属性才行？
* binding关系可以表示为exchange–binding–queue。从文档中我们知道，若要求投递的message能够不丢失，要求message本身设置persistent属性，要求exchange和queue都设置durable属性。其实这问题可以这么想，若exchange或queue未设置durable属性，则在其crash之后就会无法恢复，那么即使message设置了persistent属性，仍然存在message虽然能恢复但却无处容身的问题；同理，若message本身未设置persistent属性，则message的持久化更无从谈起。


# 3.RocketMQ






































