## 模块重构

### seata-core-common模块

1. 重构说明：本质原因是grpc采用proto文件来定义和序列化传输消息实体，而业务处理过程所需要的实体是proto文件未定义的model类实体，也就是从Rpc—>Handler之间需要有一层转换，将传输的Message消息类型转换为业务可处理的model类型，而这层转换在`seata-serializer-protobuf`模块中已经实现，为了可以复用这块的相关代码，需要引入该序列化模块（从proto文件的使用考虑上也是需要引用，rpc相关的传输消息实体的protobuf定义都在该模块下已有）。但是`seata-serializer-protobuf`模块中需要使用到传输消息实体的相关类来做这层转换，而这些传输消息实体类的定义在`seata-core`模块中，两者相互引用会出现循环依赖的问题。为了解决这个问题，需要将protocol包下的相关传输消息实体类定义单独抽取作为一个模块（`seata-core-common`），由`seata-core`和`seata-serializer-protobuf`各自引入

2. 具体实施：

   - 考虑到类与类之间的依赖问题，虽然proto包下的大部分都是requset/response的传输消息实体类，但还是不乏依赖与其他包的类，比如`ProtocolConstants`类就需要`compressor`包下的CompressorType来表示压缩类型，因此就没有将整个包都抽取出来，造成抽取出来的模块的功能性区分有一点不是很清晰。最终的`seata-common`的模块文件结构如下：

     ![](https://img.goodboycoder.com/images/2022/09/05/521c9540315e00bc317097a3db5ce2fc.png)

   - 在分离resquest/response时，存在的一个问题时原本的消息处理链路需要通过message本身来做转发，大致图如下：

     ![](https://img.goodboycoder.com/images/2022/09/05/d75fd2a63b5b7d24e77214ad4e3c442d.png)

     导致message本身需要强依赖于RpcContext，而RpcContext作为Rpc的核心，基本无法从`seata-core`中单独分离。此外，原本的处理方式也无形中放大了Message类的类职责。于是对于该处理链路做了调整，整体方案如下：

     ![](https://img.goodboycoder.com/images/2022/09/05/0727918b73a89a2a21d15e419d4dd383.png)

     在MessageHandler中维护一个functionMap，负责管理从消息类型（MessageType）到function接口实现处理方法的的映射关系，具体代码表现为：

     ```java
     //DefaultCoordinator.java
     
     private final Map<Short, BiFunction<AbstractMessage, RpcContext, ? extends AbstractResultMessage>> functionMap = new ConcurrentHashMap<>();
     
     //初始化举例
     functionMap.put(MessageType.TYPE_GLOBAL_BEGIN, (request, context) -> {
         if (!(request instanceof GlobalBeginRequest)) {
             throw new IllegalArgumentException("GlobalBeginRequest is required, but is actually " + request.getClass());
         }
         return handle((GlobalBeginRequest) request, context);
     });
     
     //消息处理过程
     public AbstractResultMessage onRequest(AbstractMessage request, RpcContext context) {
         if (!(request instanceof AbstractTransactionRequestToTC)) {
             throw new IllegalArgumentException();
         }
         BiFunction<AbstractMessage, RpcContext, ? extends AbstractResultMessage> function = functionMap.get(request.getTypeCode());
         if (null == function) {
             throw new IllegalArgumentException("no available function for message type: " + request.getTypeCode());
         }
     
         return function.apply(request, context);
     }
     ```

     

### protobuf-lib模块

1. 模块说明：此举是为了可以统一管理proto文件及其生成的接口文件，官方也建议应该有一个interface项目，包含原始 protobuf 文件并生成 java model 和 service 类，可以供不同模块共享



## Grpc主要类结构说明

1. 主体类继承结构

   ![](https://img.goodboycoder.com/images/2022/09/06/3223625b42e49857748adb939daee309.png)

2. 由于grpc客户端与服务端所使用的channel类型不同，需要分别进行包装

   ![](https://img.goodboycoder.com/images/2022/09/06/a7f6a38155b8a149ea9e86cc468849a3.png)

3. 配置类统一继承BaseRpcConfig，客户端与服务端的配置各不相同

   ![](https://img.goodboycoder.com/images/2022/09/06/f298271eb5724db1b5c97c99325199ac.png)

4. 

## 执行流程说明

### 服务端实现

1. 服务端接收请求

   服务端整体分为三个服务提供来接收客户端的请求，分为`TransactionManagerService`——接收TM相关请求、`ResourceManager`——接收RM相关请求以及处理心跳健康检测的`HeathCheckService`，均为继承通过proto文件生成接口的静态实现（这方面的一个考虑：动态的服务生成相比于静态在接口定义的约束要小，因为不直接依赖于proto文件，这就可能导致最后的服务与实际的proto接口定义文件有出入，这个不太利于之后的多语言对接）。整体处理流程如下：

   ![](https://img.goodboycoder.com/images/2022/09/06/04696fd530d3d4ac647ebe3c31472fcb.png)

2. TC二阶段下发

   Server端对于事务二阶段指令下发请求（分支事务回滚与提交）主要是通过实现RemotingServer接口对外提供请求下发功能，分为同步与异步请求，均使用注册的双向流进行通信。整体通信流程如下：

   ![](https://img.goodboycoder.com/images/2022/09/07/978029b27eb0a4ea4bfbd2740f6adc18.png)

3. 

### 客户端实现

1. 客户端(RM/TM)发送请求

   由于是静态服务生成，需要为所有可用服务的存根维护一个MessageType—>GrpcStubFunction的映射表，GrpcStubFunction是自定义的处理客户端stub发送请求的函数式接口，返回请求结果的future。映射表的初始化代码如下：

   ```java
   //AbstractGrpcRemotingClient.java
   private static final Map<Short, GrpcStubFunction> STUB_FUNCTION_MAP = new ConcurrentHashMap<>();
   
   //registerDefaultClientStub()
   STUB_FUNCTION_MAP.put(MessageType.TYPE_GLOBAL_BEGIN, (GrpcStubFunction<GlobalBeginRequestProto, GlobalBeginResponseProto>) (channel, req) -> {
   TransactionManagerServiceGrpc.TransactionManagerServiceFutureStub futureStub = 		TransactionManagerServiceGrpc.newFutureStub((io.grpc.Channel)channel.originChannel());
   return futureStub.globalBegin(req);
   });
   ```

   整体的请求流程如下：

   ![](https://img.goodboycoder.com/images/2022/09/07/e53049f87426d72dec1d42d173f9b86c.png)

2. 客户端(RM/TM)处理响应

   - 一元请求的同步请求响应，直接在请求时阻塞等待返回
   - 一元请求的异步请求响应，目前没有实现合并批量请求，而其他的请求均为同步请求，这方面依旧走原本的异步处理流程，由`ClientOnResponseProcessor`处理
   - 双向流的请求响应（实际为TC的二阶段下发），`(RM/TM)GrpcRemotingClient#bindBiStream`中处理，主要工作为对BiStreamMessage进行解包，并做Proto到model的类型转换，最后交由各自的`RemotingProcessor`处理



## 配置梳理

### 原有配置基本说明

#### NettyBaseConfig

1. transport.threadFactory.bossThreadPrefix（废弃）
2. transport.threadFactory.workerThreadPrefix（废弃）
3. transport.threadFactory.shareBossWorker（废弃）
4. **transport.threadFactory.workerThreadSize**
   - 说明：全局工作线程大小，一般为线程池核心线程大小的默认值，有两种配置方式：
     - 配置为数字
     - 配置为模式，填写模式字符串
       - auto——核心线程*2+1
       - pin——核心线程数
       - busyPin——核心线程数+1
       - default——核心线程数*2
   - 默认值：default模式，即核心线程数*2
5. transport.server
   - 说明：Netty传输服务类型，可配置为NIO或者Native类型（会与transport.type一起关联影响BaseConfig中的SERVER_CHANNEL_CLAZZ和CLIENT_CHANNEL_CLAZZ）
   - 默认值：NIO
6. transport.type
   - 说明：Netty传输协议类型，可选配置为tcp或者unix-domain-socket
   - 默认值：tcp
7. **transport.heartbeat**
   - 说明：client和server通信心跳检测开关
   - 默认值：true
8. **非可配置项（心跳健康检测相关）**
   1. DEFAULT_WRITE_IDLE_SECONDS = 5
   2. READIDLE_BASE_WRITEIDLE = 3
   3. MAX_WRITE_IDLE_SECONDS、MAX_READ_IDLE_SECONDS、MAX_ALL_IDLE_SECONDS——分别对应Netty心跳检测（IdleStateHandler）的触发时间，为0时表示不触发



#### NettyServerConfig

1. transport.serverSelectorThreads（废弃）
2. transport.serverSocketSendBufSize
   - 说明：TCP socket的发送缓冲区大小，对应Netty的ChannelOption.SO_SNDBUF配置
   - 默认值：153600
3. transport.serverSocketResvBufSize
   - 说明：TCP socket的接收缓冲区大小，对应Netty的ChannelOption.SO_RCVBUF配置
   - 默认值：152600
4. transport.serverWorkerThreads
   - 说明：Netty服务端工作线程池的核心线程大小
   - 默认值：默认为transport.threadFactory.workerThreadSize配置项大小
5. transport.soBackLogSize
   - 说明：tcp/ip协议中listen函数的backlog参数，用于初始化服务端可连接队列，对应Netty的ChannelOption.SO_BACKLOG配置项
   - 默认值：1024
6. transport.writeBufferHighWaterMark与transport.writeBufferLowWaterMark
   - 说明：Netty服务端写缓冲区的高低水位，对应Netty的ChannelOption.WRITE_BUFFER_WATER_MARK配置项
   - 默认值：67108864与1048576
7. transport.rpcTcRequestTimeout
   - 说明：RPC同步请求时的最大等待时间
   - 默认值：30s
8. transport.serverChannelMaxIdleTimeSeconds（废弃）
9. transport.minServerPoolSize
   - 说明：Seata服务端的工作线程池的核心线程数
   - 默认值：50
10. transport.maxServerPoolSize
    - 说明：Seata服务端的工作线程池的最大线程数
    - 默认值：500
11. transport.maxTaskQueueSize
    - 说明：Seata服务端的工作线程池的最大任务等待队列长度，同时也是RPC批量结果响应线程池的最大任务等待队列长度
    - 默认值：20000
12. transport.keepAliveTime
    - 说明：Seata服务端的工作线程池的线程保活时间（最大空闲时间），同时也是RPC批量结果响应线程池的线程保活时间
    - 默认值：500
13. transport.minBranchResultPoolSize、transport.maxBranchResultPoolSize
    - 说明：RPC批量结果响应线程池的核心线程数和最大线程数
    - 默认值：均为transport.threadFactory.workerThreadSize配置项大小
14. transport.enableTcServerBatchSendResponse
    - 说明：是否允许TC批量发送响应的开关
    - 默认值：false
15. server.servicePort（似乎在NettyServerConfig中但是没有启用，而是其他地方手动获取）
    - 说明：Netty服务端的监听端口
    - 默认值：8091
16. transport.threadFactory.bossThreadPrefix
    - 说明：Netty服务端boss线程的命名前缀
    - 默认值：NettyBoss
17. transport.threadFactory.workerThreadPrefix
    - 说明：Netty服务端work线程的命名前缀
    - 默认值：根据是否开启epoll决定为NettyServerEPollWorker，否则为NettyServerNIOWorker
18. transport.threadFactory.serverExecutorThreadPrefix（废弃）
19. transport.threadFactory.bossThreadSize
    - 说明：Netty服务端boss线程工厂的总大小
    - 默认值：1
20. transport.shutdown.wait
    - 说明：Netty服务端的停止等待时间，确保没有未完成的数据传输
    - 默认值：3s





#### NettyClientConfig

1. 不可配置项

   ```
   # Netty超时连接毫秒数，对应配置项ChannelOption.CONNECT_TIMEOUT_MILLIS
   connectTimeoutMillis = 10000
   
   # Netty客户端的TCP发送和接收缓冲区的大小，分别对应ChannelOption.SO_SNDBUF和ChannelOption.SO_RCVBUF
   clientSocketSndBufSize = 153600
   clientSocketRcvBufSize = 153600
   
   # 控制客户端的连接数配置（已废弃）
   perHostMaxConn = 2
   PER_HOST_MIN_CONN = 2
   pendingConnSize = Integer.MAX_VALUE
   
   # （已废弃）
   vgroup、clientAppName、clientType
   
   # the max inactive channel check（已废弃）
   maxInactiveChannelCheck = 10
   
   # Netty客户端判断channel是否可写的最大重试次数
   MAX_NOT_WRITEABLE_RETRY = 2000
   
   # Netty客户端判断获取可用的channel的最大等待重试次数
   MAX_CHECK_ALIVE_RETRY = 300
   
   # Netty客户端判断获取可用的channel的每次等待时间
   CHECK_ALIVE_INTERVAL = 10
   
   # socket地址分隔符
   SOCKET_ADDRESS_START_CHAR = "/"
   
   # GenericKeyedObjectPool的相关配置
   MAX_ACQUIRE_CONN_MILLS = 60 * 1000L
   DEFAULT_MAX_POOL_ACTIVE = 1
   DEFAULT_MIN_POOL_IDLE = 0
   DEFAULT_POOL_TEST_BORROW = true
   DEFAULT_POOL_TEST_RETURN = true
   DEFAULT_POOL_LIFO = true
   
   ```

2. transport.rpcRmRequestTimeout

   - 说明：RM的同步Rpc请求的最大等待时间
   - 默认值：30s

3. transport.rpcTmRequestTimeout

   - 说明：TM的同步Rpc请求的最大等待时间
   - 默认值：30s

4. transport.enableClientBatchSendRequest（废弃）

5. transport.enableRmClientBatchSendRequest

   - 说明：是否允许RM客户端批量发送请求的开关
   - 默认配置：transport.enableClientBatchSendRequest，再者若无配置则为true

6. transport.enableTmClientBatchSendRequest

   - 说明：是否允许TM客户端批量发送请求的开关
   - 默认配置：false（似乎与RM的配置项不统一）

7. transport.threadFactory.clientSelectorThreadSize

   - 说明：Netty的selector thread size
   - 默认配置：1

8. transport.threadFactory.clientSelectorThreadPrefix、transport.threadFactory.clientWorkerThreadPrefix

   - 说明：对应线程的名字前缀
   - 默认配置：NettyClientSelector、NettyClientWorkerThread

9. 合并请求发送的相关配置



### Grpc配置

除了Grpc特有配置以外，其余相关配置均可独立配置，与原有的Netty配置做了隔离

1. 客户端新增配置项
   - client.rpc.type：客户端的rpc类型配置，可选配置为netty、grpc，默认配置为Netty

