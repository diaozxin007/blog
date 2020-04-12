---
title: Vertx入门到实战—实现钉钉机器人内网穿透代理
date: 2020-04-12 16:39:16
tags: [java,vertx,dingding]
---

最近研究 Vetrx 简直爱不释手。迫不及待的想给大家介绍一下。

![carbon](https://img.xilidou.com/img/carbon.png)

## Vertx 是什么

- Vertx 是一个运行在 JVM 上，用来构建响应式应用的工具集。
- 基于 netty 的高性能的，异步的网络库。
- 对 netty 进行了封装，提供更加友好的 API。
- 同时实现了一些基于异步调用的库，包括database connection, monitoring, authentication, logging, service discovery, clustering support, etc。

<!--more-->

## 为什么我推荐学习

其实随着技术的发展。异步调用其实越来越普及了。

1、现在随着 RPC 的普及。类似 Dubbo 这样的框架都是基于 NIO 的概念带来的，了解异步编程有助于学习理解框架。

2、响应式编程逐渐由客户端，前端向后端渗透。

3、更容易的编写出高性能的异步服务。

## Vertx 的几个重要概念

### Event Loop

Event Loop 顾名思义，就是事件循环的。在 Vertx 的生命周期内，会不断的轮询查询事件。

传统的多线程编程模型，每个请求就 fork 一个新的线程对请求进行处理。这样的编程模型有实现起来比较简单，一个连接对应一个线程，如果有大量的请求需要处理，就需要 fork 出大量的线程进行处理，对于操作系统来说调度大量线程造成系统 load 升高。

所以为了能够处理大量请求，就需要过渡到基于 Roactor 模型的 Event Loop上。

![https://img.xilidou.com/img//event-loop.png](https://img.xilidou.com/img/event-loop.png)

官网的这个图就很形象了。Eventloop 不断的轮训，获取事件然后安排上不同的 Handler 处理对应的Event。

这里要注意的是为了保证程序的正常运行，event 必须是非阻塞的。否则就会造成 eventloop 的阻塞，影响Vertx 的表现。但是现实中的程序肯定不能保证都是非阻塞的，Vertx 也提供了相应的处理阻塞的方法的机制。我们在下面会继续介绍。

### Verticle

在 Vertx 中我们经常可以看见 Vertical 组件。

![verticle](https://img.xilidou.com/img/verticle-threading-config.png)

Verticle 是由 Vert.x 部署和运行的代码块。默认情况一个 Vert.x 实例维护了N（默认情况下N = CPU核数 x 2）个 Event Loop 线程。Verticle 实例可使用任意 Vert.x 支持的编程语言编写，而且一个简单的应用程序也可以包含多种语言编写的 Verticle。

您可以将 Verticle 想成 [Actor Model](https://en.wikipedia.org/wiki/Actor_model) 中的 Actor。

一个应用程序通常是由在同一个 Vert.x 实例中同时运行的许多 Verticle 实例组合而成。不同的 Verticle 实例通过向 Event Bus 收发送消息来相互通信。

### Event bus

Vertx 中的 Event bus 如果类比后端常用的 MQ 就更加容易理解了。实际上 Event Bus 就是 Verticle 之间传递 信息的桥梁。

换句话说，就是 Java 通用设计模式中的监听模式，或者是我们常说的 基于 MQ 消息开发模式。

![Event bus](https://img.xilidou.com/img/event-bus.png)

## 回到 Vertx

上文我们讨论了 vertx 的模型和机制，现在人们就看看怎么使用 vertx 开发一个程序。

我会结合之前写的 暴打钉三多的来进行讲解,一切从 Vertx 开始。

```kotlin
    val vertx = Vertx.vertx()
```

vertx 是整个 vert.x 框架的核心。通常来说 Vertx 所有的行为就是从 vertx 这个类中产生的。

### Don’t call us, we’ll call you

Vert.x 是一个事件驱动框架。所谓事件驱动是指当某件事情发生以后，就做这个动作。

我们再回到标题， “Don’t call us, we’ll call you” 这个原则，其实就是当我们 发现你能完成这项工的时候，我们会找你的。你不需要主动来联系我。

我们通过代码来理解一下 Vertx 是怎么实现这个原则的 ：

```kotlin
    server.requestHandler(request -> {
      request.response().end("hello world!");
    });
```

这个代码块的意思是，每当 server 的 request 被调用的时候，就返回一个 `hello world` 。

所以 Vertx 中的 'you' j就是各种各样的 Handler 。大多数时候我们编写 Vertx 的程序，实际上就是在编写Handler 的行为。然后再告诉 Vertx ，每当 XXX 事件触发以后，你就调用 XXX Handler。

### Don’t block me

Vertx 是基于事件的，上文我们提到了 Event Loop ，在 Vertx 中，EventLoop 就是一个勤劳的小蜜蜂，不断的去寻找，到底有哪些事件被触发了。然后再执行对应的 Handler。假如执行 Hanlder 的线程，就是 Event Loop 线程。如过 Handler 执行的时间过长。就会阻塞 Event Loop 。造成别的事件触发的时候。Event Loop 还在处理时间花费较长的 Handler。Event loop就不及时的响应其他的事件。 

但是现实中，不可能所有的事件 都是非阻塞的。比如查询数据库，调用远程接口等等，那怎么办呢？

在事件驱动模型中，大概有两种套路解决，这个问题，比如在 Redis 中，Redis 会十分小心的维护一个时间分片。当某个人物执行事件过长的话，就保存当前事件的状态，然后暂停当前事件，重新由 Event loop 进行调度。防止 Event Loop 被事件阻塞。

还有一种套路，就是把阻塞的事件，交给别的线程来来执行。Event Loop 就可以继续进行事件的循环，防止被阻塞。事实上 Vertx 就是这么操作的。

```kotlin
    vertx.executeBlocking(promise -> {
      // Call some blocking API that takes a significant amount of time to return
      String result = someAPI.blockingMethod("hello");
      promise.complete(result);
    }, res -> {
      System.out.println("The result is: " + res.result());
    });
```

如果我们开发的时候意识到这个 Handler 是一个阻塞的，就需要告诉 vertx 这是是一个 Blocking 的需要交给别的线程来处理。

## 协调异步处理

上文提到. Vertx 是通过 Handler 来处理事件的，但是，很多时候，某个操作，通常需要不止一个 Handler 来对数据进行处理。如果一直使用 callback 的写法，就会形成箭头代码。产生地狱回调的问题。

作为一个异步框架，Vertx 一般使用 Future 来解决回调地狱的问题。理解 Vertx 中的 Future 是编写好的代码的核心。

通常我们理解 Future 只是一个占位符，代表某个操作未来某个时候的结果。不太清楚的可以看我以前写文章。

这里需要特别指出的是 Vertx 的 Future 和 Jdk 里面的 `CompletableFuture` 原理和理念类似，但是使用起来有很大的区别的。

Jdk 里面的 `CompletableFuture` 是可以直接使用 `result()` 阻塞的等待结果，但是 Vertx 中的 Future 如果直接使用 `result()` ，就会立刻从 Future 中取出结果，而不是阻塞的等待结果，就很容易收获一个 Null。

明确这个区别以后，写起代码就不会出错了。

## Event Bus

如果在日常开发中使用过消息系统，就很容易理解 Vertx 中的 Event bus 了。官方文档把 Event bus 比作 Vertx 的神经系统，其实我们就认为，Event bus是 Vertx 的消息系统，就好了。

## 钉钉内网穿透代理的的开发

这个小 Demo 麻雀虽小但是包含了 Vertx 几个关键组件的使用。写这个 Demo 的时候，正好在学习 Kotlin 所以顺手就用 kotlin 写了。如果写过 Java 或者 Typescript 那你也能很容易的看懂。

项目包含了

- Http Service 用于接收钉钉的回调
- WebSocket Service 用于向 Client 推送收到的回调，达到内网穿透的目的。
- Vertx Config 用于配置项目相关参数，便于使用
- Event Bus 的使用，用于 Http Service 和 WebSocket 之间传递消息。

### 先来一个 Verticle

Gradle 配置文件如下先引入包：

```groovy
    implementation ("io.vertx:vertx-core:3.8.5")
    implementation ("io.vertx:vertx-web:3.8.5")
    implementation ("io.vertx:vertx-lang-kotlin:3.8.5")
```

上文我我们已经介绍了 Verticle 是什么了，为了方便开发，Vertx 给我们提供了一个  AbstractVerticle 抽象类。直接继承：

```kotlin
    class DingVerticle : AbstractVerticle() {
    }
```

`AbstractVerticle` 中包含了 Vericle 常用的一些方法。

我们可以重写 `start()` 方法,来初始化我们 Verticle 的行为。

### HttpService 的创建

```kotlin
    override fun start() {
        val httpServer = vertx.createHttpServer()
        val router = Router.router(vertx)
        router.post("/ding/api").handler{event ->
            val request = event.request()
            request.bodyHandler { t ->
                println(t)
            }
            event.response().end();
        }
        httpServer.requestHandler(router);
        httpServer.listen(8080);
    }
```

代码比较简单:

1. 创建一个 httpService
2. 设置一个 Router，如果写过 Spring Mvc 相关的代码。这里的 Router 就类似 Controller 里面的 RequestMapping 。用于指定一个 Http 请求 URI 和 Method 对应的 Handler。这里的 Handler 是一个 lambda 表达式。只是简单的把请求的 body 打印出来。
3. 将 Router 加入到 httpService 中，并监听 8080 端口。

### WebSocketService

webSocket协议是这个 proxy 的关键，因为 WebSocket 不同于 Http，是双向通通信的。依赖这个特性我们可以把消息“推到”内网。达到内网“穿透”的目的。

```kotlin
    httpServer.webSocketHandler { webSocket: ServerWebSocket ->
        val binaryHandlerID = webSocket.binaryHandlerID()
        webSocket.endHandler() {
            log.info("end", binaryHandlerID)
        }
        webSocket.writeTextMessage("欢迎使用 xilidou 钉钉 代理")
        webSocket.writeTextMessage("连接成功")
    }
```

代码也比较简单，就是向 Vertx 注册一个处理 WebSocket 的 Handler。

### Event Bus 的使用

作为代理最核心的功能就是转发钉钉的回调消息，前面我说到，Event Bus 在 Vertx 中起到了“神经系统的作用”实际上 ，换句话说，就是http 服务收到回调的时候，可以通过 Event Bus 发出消息。WebSocket 在收到 Event Bus 发来的消息的时候，推送给客户端。如下图看图：

为了方便理解，我们就使用 MQ 里面通常的概念生产者和消费者。

所以我们使用在  HttpService 中注册一个生产者，收到钉钉的回调以后，把消息转发出来。

为了便于编写，我们可以单独写一个 HttpHandler

```kotlin
    //1
    class HttpHandler(private val eventBus: EventBus) : Handler<RoutingContext> {

        private val log = LoggerFactory.getLogger(this.javaClass);

        override fun handle(event: RoutingContext) {
            val request = event.request()
            request.bodyHandler { t->
                    val jsonObject = JsonObject(t)
                    val toString = jsonObject.toString()
                    log.info("request is {}",toString);
                    // 2
                    eventBus.publish("callback", toString)
            }
            event.response().end("ok")
        }
    }
```

这里需要注意几个问题:

1. 我们需要使用 Event Bus 发送消息，所以需要在构造函数里面传入一个 Event Bus
2. 我们在收到消息以后，可以先将数据转换为 Json 字符串，然后发送消息，注意这里使用的是 `publish()` 是广播的意思，这样所有订阅的客户端都能收到新消息。

有了生产者，并发出了数据，我们就可以，在 WebSocket 里面消费这个消息，然后推送给客户端了

再来写一个 WebSocket 的 Handler

```kotlin
    //1
    class WebSocketHandler(private val eventBus: EventBus) : Handler<ServerWebSocket> {

        private val log = LoggerFactory.getLogger(this.javaClass)

        override fun handle(webSocket: ServerWebSocket) {
            val binaryHandlerID = webSocket.binaryHandlerID()

            //2
            val consumer = eventBus.consumer<String>("callback") { message ->
                val body = message.body()
                log.info("send message {}", body)
                //3
                webSocket.writeTextMessage(body)
            }
            webSocket.endHandler() {
                log.info("end", binaryHandlerID)
                //4
                consumer.unregister();
            }
            webSocket.writeTextMessage("欢迎使用 xilidou 钉钉 代理")
            webSocket.writeTextMessage("连接成功")
        }
    }
```

这里需要注意几个问题:

1. 初始化的时候需要注入 eventBus
2. 写一个 `consumer()` 消费 HttpHandler 发来的消息
3. 将消息写入到 webSocket 中，发送给 Client
4. WebSocket 断开后需要回收 consumer

### 初始化 Vertx

做了那么多准备终于可以初始化我们的 Vertx 了

```kotlin
    class DingVerticleV2: AbstractVerticle(){
        override fun start() {

            //2
            val eventBus = vertx.eventBus()
            val httpServer = vertx.createHttpServer()

            val router = Router.router(vertx);
            //3
            router.post("/api/ding").handler(HttpHandler(eventBus));
            httpServer.requestHandler(router);

            //4
            httpServer.webSocketHandler(WebSocketHandler(eventBus));
            httpServer.listen(8080);
        }
    }

    //1
    fun main() {
        val vertx = Vertx.vertx()
        vertx.deployVerticle(DingVerticleV2())
    }
```

这里需要注意几个问题:

1. 初始化 Vertx 并部署他
2. 初始化 eventBus
3. 注册 HttpHandler
4. 注册 WebSocketHandler

## 总结

- Vertx 是一个工具，不是框架，所以可以很方便的与其他框架组合。
- Vertx 是一个基于 Netty 的异步框架。我们可以向编写同步代码一样，编写异步代码。
- vertx 在代码中主要有两个作用，一个是初始化组件，比如 ：
  
```kotlin
    val eventBus = vertx.eventBus()
    val httpServer = vertx.createHttpServer()
```

还有一个是注册 `Handler`:

```kotlin
     httpServer.webSocketHandler(WebSocketHandler(eventBus));
```

- Event Bus 是一个消息系统。用于不同的 Handler 直接传递数据，简化开发。

## 相关连接

1. 使用教程 [钉钉机器人回调内网穿透代理--使用篇](https://xilidou.com/2020/03/25/dingsanduo/)
2. Github 地址: [Github](https://github.com/diaozxin007/DingTalkProxy)
3. 官网教程：[A gentle guide to asynchronous programming with Eclipse Vert.x for Java developers](https://vertx.io/docs/guide-for-java-devs/);
4. [Vert.x Core Manual](https://vertx.io/docs/vertx-core/kotlin/)

欢迎关注我的微信公众号:

![二维码](https://img.xilidou.com/img/2019-04-25-022202.jpg)
