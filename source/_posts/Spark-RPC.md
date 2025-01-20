---
layout: title
title: Spark RPC
date: 2021-03-29 18:06:01
tags: Spark
categories: [分布式系统, 分布式计算, Spark]
---

# 概述

在分布式系统中，通信是很重要的部分。集群成员很少共享硬件资源，通信的单一解决方案是客户端-服务器模型（C/S）中的消息交换。RPC 是 Remote Procedure Call 的缩写，当客户端执行请求时，它被发送到存根（stub）。当请求最终到达对应的服务器时，它还会到达服务器的存根，捕获的请求会转换为服务器端可执行过程。在物理执行后，将结果发送回客户端，示意图如下。

{% note warning %}
为屏蔽客户调用远程主机上的对象，必须提供某种方式来模拟本地对象，这种本地对象称为存根（stub），负责接收本地方法调用，并将它们委派给各自的具体实现对象。
{% endnote %}

![](https://cdn.jsdelivr.net/gh/gleonSun/images@main/image/rpc_schema-1737348115557.png)

在 Spark 0.x.x 和 1.x.x 版本中，组件间的消息通信主要借助于 Akka，但是 Spark 2.0.0 版本已经移除了该部分的依赖，基于 Netty 实现了 RPC 功能。

{% note warning %}
用户 Spark Application 中 Akka 版本和 Spark 内置的 Akka 版本可能会冲突，而 Akka 不同版本之间无法互相通信。Spark 用的 Akka 特性比较少，这部分特性很容易自己实现，基于以上种种考量最终 Spark 废弃了 Akka，详见 [JIRA](https://issues.apache.org/jira/plugins/servlet/mobile#issue/SPARK-5293)。
{% endnote %}

# 参考模型

Spark RPC 主要参考了 Actor 模型和 Reactor 模型。

## Actor

用于解决多线程并发条件下锁等一系列线程问题，以异步非阻塞方式完成消息的传递。Actor 由状态（state）、行为（behavior）、邮箱（mailbox）三者组成。

Actor 遵循以下规则:
- 创建其他的 Actor
- 发送消息给其他的 Actor
- 接受并处理消息，修改自己的状态

上面的规则还隐含了以下意思：

1. 每个 Actor 都是独立的，能与其他 Actor 互不干扰的并发运行，同时每个 Actor 有自身的邮箱，任意 Actor 可以向自己地址发送的信息都会放置在这个邮箱里，邮箱里消息的处理遵循 FIFO 顺序。
2. 消息的投递和读取是两个过程，这样 Actor 之间的交互就解耦了。
3. Actor 之间的通信是异步的，发送方只管发送，不关心超时和错误，这些都交给框架或者独立的错误处理机制。
4. Actor 的通信兼顾了本地和远程调用，因此本地处理不过来的时候可以在远程节点上启动 Actor 再把消息转发过去进行处理，拥有了扩展的特性。

这里贴出 Actor 模型的一个经典图片，另附上 B 站一段视频中关于 [Actor](https://www.bilibili.com/video/BV12y4y1a7e4?from=search&seid=8241130096895464139) 模型的解释。

![](https://cdn.jsdelivr.net/gh/gleonSun/images@main/image/actor-1737348115557.png)

## Reactor

Reactor 模型是一种典型的事件驱动的编程模型。

模型定义了三种角色：
1. Reactor - 将 I/O 事件分派给对应的 Handler
2. Acceptor - 处理客户端新连接，并分派请求到处理器链中
3. Handlers - 执行非阻塞读/写任务

为什么使用 Reactor 模型？我们来看一下传统的阻塞 I/O 模型：
- 每个线程都需要独立的线程处理，并发足够大时，会占用很多资源
- 采用阻塞 I/O 模型，连接建立后，即便没有数据读，线程的阻塞操作也会浪费资源

针对以上问题可以采用以下方案：
- 创建一个线程池，避免为每个连接创建线程池，连接完成就把逻辑交给线程池处理
- 基于 I/O 复用模型，多个连接共用同一个阻塞对象。有新数据时，线程不再阻塞，跳出状态进行处理。

而 Reactor 模型就是基于 I/O 复用和线程池的结合，根据 Reactor 数量和处理资源的线程数量不同，分为三类：

1. 单 Reactor 单线程模型（一般不用，对多核机器资源有些浪费）
2. 单 Reactor 多线程模型（高并发场景下存在性能问题）
3. 多 Reactor 多线程模型

此处附上大神 Doug Lea 在 Scalable IO in Java 中给出的阐述，其中 Netty NIO 默认模式沿用的是多 Reactor 多线程模型变种，对应 pdf 中 26 页框架图，另感兴趣可自行阅读 [Reactor](http://www.laputan.org/pub/sag/reactor.pdf) 架构设计。

{% pdf ./nio.pdf %}

# 源码分析

此处只提及核心部分，大部分的源码都在 `org.apache.spark.rpc` 包中，负责将消息发送到客户端存根的对象由 Dispatcher 类表示，通过内部的 post* 方法之一（postToAll、postRemoteMessage 等），准备消息实例（RpcMessage）并将其发送到预期端点（endpoint），具体实现类为 NettyRpcEndpointRef。

RPC endpoints 主要由两个类表示
1. RpcEndpoint
    - 每个节点都可以称为一个 RpcEndpoint（Client、Worker 等）
    - 主要方法为 onStart、receive、receiveAndReply、onStop

2. RpcEndpointRef
    - 作用是发送请求，本质是对 RpcEndpoint 的一个引用
    - 主要方法为 ask（异步请求-响应）、askSync（同步请求-响应）

{% note warning %}
onStart 和 onStop 在端点启动和停止时调用，receive 发送请求或响应，对应`RpcEndpointRef.send` 或 `RpcCallContext.reply`，而 receiveAndReply 则处理回应请求，对应`RpcEndpointRef.ask`。RpcEndpointRef 发送的请求允许用户传入超时时间。
{% endnote %}

其中 Dispatcher 也可称为消息收发器，将需要发送的消息和远程 RPC 端点接收到的消息，分发至对应的收件箱/发件箱，当轮询消息的时候进行处理。

- 分发
    ```java
    private[netty] def send(message: RequestMessage): Unit = {
        val remoteAddr = message.receiver.address
        if (remoteAddr == address) {
          // 将消息发送到本地 RPC 端点（收件箱），存入当前 RpcEndpoint 对应的 Inbox
          try {
            dispatcher.postOneWayMessage(message)
          } catch {
            case e: RpcEnvStoppedException => logDebug(e.getMessage)
          }
        } else {
          // 将消息发送到远程 RPC 端点（发件箱），最终通过 TransportClient 将消息发送出去
          postToOutbox(message.receiver, OneWayOutboxMessage(message.serialize(this)))
        }
      }
    ```

- 处理
    ```java
    /** Message loop used for dispatching messages. */
      private class MessageLoop extends Runnable {
        override def run(): Unit = {
          try {
            while (true) {
              try {
                // 从 receivers 中获得 EndpointData，receivers 是 LinkBlockingQueue，没有元素时会阻塞
                val data = receivers.take()
                if (data == PoisonPill) {
                  // Put PoisonPill back so that other MessageLoops can see it.
                  receivers.offer(PoisonPill)
                  return
                }
                //调用 process 方法对 RpcEndpointData 中 Inbox 的 message 进行处理
                data.inbox.process(Dispatcher.this)
              } catch {
                case NonFatal(e) => logError(e.getMessage, e)
              }
            }
          } catch {
            case _: InterruptedException => // exit
            case t: Throwable =>
              try {
                // Re-submit a MessageLoop so that Dispatcher will still work if
                // UncaughtExceptionHandler decides to not kill JVM.
                threadpool.execute(new MessageLoop)
              } finally {
                throw t
              }
          }
        }
      }  
    ```

Spark RPC 的源码抽象图大致如下图所示，RpcAddress 是 RpcEndpointRef 的地址（Host + Port），而 RpcEnv 则为 RpcEndpoint 提供处理消息的环境及管理其生命周期等。

![](https://cdn.jsdelivr.net/gh/gleonSun/images@main/image/rpc_structure-1737348115557.png)

{% note warning %}
其中 MessageEncoder 和 MessageDecoder 是用于解决可能出现的半包、粘包问题。在基于流的传输（如TCP/IP）中，数据会先存储到一个 socket 缓冲里，但这个传输不是一个数据包队列，而是一个字节队列。因此就可能出现这种情况，我们想发送 3 个数据包：ABC、DEF、GHI，但是由于传输协议，应用程序在接收时可能会变成这种情况：AB、CDEF、GH、I。所以需要对传输的数据流进行特殊处理，常见的比如：以特殊字符作为数据的末尾；或者发送固定长度的数据包（接收方也只接收固定长度的数据），不过这种情况不太适合频繁的请求。Spark 采用的是在协议上封装一层数据请求协议，即数据包=数据包长度+数据包内容，这样接收方就可以根据长度进行接收。
{% endnote %}

# 代码实例

通过自定义代码实例可以更好地了解 Spark RPC 是如何运作的，GitHub 上有个 [kraps-rpc](https://github.com/neoremind/kraps-rpc) 项目，该项目是从 Spark 中将 RPC 框架剥离出来的一部分，由于 GitHub 经常被墙，此处贴出我 fork 后在 Gitee 上更新过的 [kraps-rpc](https://gitee.com/gleonSun/kraps-rpc)。

## 服务端

注册自身引用、相应请求及定义消息体等。

```java
object FaceToFaceServer {

  val SERVER_HOST = "localhost"
  val SERVER_PORT = 4399
  val SERVER_NAME = "FaceServer"

  def main(args: Array[String]): Unit = {
    val config = RpcEnvServerConfig(new RpcConf, "FaceService", SERVER_HOST, SERVER_PORT)
    val rpcEnv: RpcEnv = NettyRpcEnvFactory.create(config)
    val faceEndpoint = new FaceEndpoint(rpcEnv)
    rpcEnv.setupEndpoint(SERVER_NAME, faceEndpoint)
    rpcEnv.awaitTermination()
  }

}

class FaceEndpoint(override val rpcEnv: RpcEnv) extends RpcEndpoint {

  override def onStart(): Unit = {
    println("Start FaceEndpoint.")
  }

  override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case SayMeeting(name) =>
      println(s"Hi $name, nice to meet you.")
      context.reply(name)
    case SayGoodBye(name) =>
      println(s"Hi $name, good bye.")
      context.reply(name)
    case _ =>
      println(s"Receiver unknown message.")
  }

  override def onStop(): Unit = {
    println("Stop FaceEndpoint.")
  }
}

case class SayMeeting(name: String)

case class SayGoodBye(name: String)
```

## 客户端

注册自身引用、寻找服务端引用和两种向服务端不同的请求方式（同步/异步）。

```java
object FaceToFaceClient {

  val CLIENT_NAME = "FaceClient"

  def main(args: Array[String]): Unit = {
    val rpcAddress = RpcAddress(SERVER_HOST, SERVER_PORT)
    faceAsync(rpcAddress)
//    faceSync(rpcAddress)
  }

  def faceAsync(rpcAddress: RpcAddress): Unit = {
    val config = RpcEnvClientConfig(new RpcConf, CLIENT_NAME)
    val rpcEnv: RpcEnv = NettyRpcEnvFactory.create(config)
    val serverEndpointRef = rpcEnv.setupEndpointRef(rpcAddress, SERVER_NAME)
    val future = serverEndpointRef.ask[String](SayMeeting("GLeon"))
    future.onComplete {
      case Success(value) => println(s"Get value: $value")
      case Failure(exception) => println(s"Get error: $exception")
    }
    // 等待 Future 完成或超时
    Await.result(future, Duration.apply("30s"))
  }

  def faceSync(rpcAddress: RpcAddress): Unit = {
    val config = RpcEnvClientConfig(new RpcConf, CLIENT_NAME)
    val rpcEnv: RpcEnv = NettyRpcEnvFactory.create(config)
    val serverEndpointRef = rpcEnv.setupEndpointRef(rpcAddress, SERVER_NAME)
    val result = serverEndpointRef.askWithRetry[String](SayMeeting("GLeon"))
    println(s"Send name: $result")
  }
}
```