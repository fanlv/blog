---
title: 高并发服务器IO模型
tags:
  - Backend
  - IO
categories:
  - Backend
date: 2018.07.15 20:20:04
updated: 2018.07.15 20:20:04
---


服务端IO模型总结 草稿

## 网络框架视角

### 零、Nginx

![image](https://upload-images.jianshu.io/upload_images/12321605-ee3acec698831d00?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 一、Netty（主从Reactor）

```
MainReactor负责客户端的连接请求，并将请求转交给SubReactor
SubReactor负责相应通道的IO读写请求
非IO请求（具体逻辑处理）的任务则会直接写入队列，等待worker threads进行处理

```

![image](https://upload-images.jianshu.io/upload_images/12321605-9216883059354e4b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/12321605-58af79a163dc0002?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 二、GRPC-GO （Goroutine Per Connection）

```
net.Listen -> Serve() -> lis.Accept() net库的accept 
-> 一个连接开个一个goroutine -> s.handleRawConn(rawConn) 
-> newHTTP2Transport(conn, authInfo) ->  newHTTP2Server
-> 开个gorutine for 循环检查 是否有 数据没有发送=> t.loopy.run(); 
-> go t.keepalive()
 -> 设置读取鉴权信息、超时配置、Http2Transport、一堆有的没的配置 最后生成st对象 -> serveStreams(st)
-> 收到Request 、 HandleStreams (这个方法里面会for{} 不停读写消息内容)
-> ReadFrame() 读取数据 -> 判断是什么帧类型数据
http2.MetaHeadersFrame、http2.DataFrame、http2.RSTStreamFrame、http2.SettingsFrame
http2.PingFrame、http2.WindowUpdateFrame -> t.operateHeaders 处理数据 -> s.handleStream
-> 解析出service和method 找到对应的handle方法 -> processUnaryRPC ->  md.Handler
->  执行对应方法的handle _Greeter_Login_Handler -> s.getCodec 解码出req数据
-> srv.(GreeterServer).Login 执行对应的函数 -> sendResponse -> encode、compress、Write
-> writeHeaderLocked-> dataFrame -> writeQuota 剑去包大小 -> t.controlBuf.put(df)
->  executeAndPut -> c.list.enqueue(it) -> WriteStatus -> 另个有个gorutine 会调用 t.loopy.run()
-> l.processData() -> str.itl.peek().(*dataFrame)  ->  l.framer.fr.WriteData 

```

![image](https://upload-images.jianshu.io/upload_images/12321605-b95dc7af4b943476?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 三、Thrift-GO （ Goroutine Per Connection）

```
Serve() -> Listen() -> AcceptLoop() 
-> for 循环接受连接请求 {innerAccept()} 一个连接开一个goroutine
-> 拿到net.coon -> client = NewTSocketFromConnTimeout(coon,timeout)
-> processRequests(cleint) 
-> TProcessor(interface包含:Process、ProcessorMap、AddToProcessorMap )
->  调用 TransportFactory.GetTransport(client)拿到 inputTransport,outputTransport
-> ProtocolFactory.GetProtocol(inputTransport)拿到 inputProtocol和outputProtocol
-> for { ReadFrame processor.Process } 这里循环读取数据，读出请求，然后返回resp，然后再继续读
-> 调用到IDL生成的代码中对应方法的Process(inputProtocol和outputProtocol) ->
name, _, seqId, err := iprot.ReadMessageBegin() (seqId回复的数据包要回写回去)
-> 根据name 找到对应方法的 Process，调用对应的Process
-> args.Read(iprot) -> iprot.ReadMessageEnd() -> handler.xxxx 方法拿到结果 ->
oprot.WriteMessageBegin-> response字段.Write(oprot) -> oprot.WriteMessageEnd() -> oprot.Flush() -> 判断是否 ErrAbandonRequest 是的话关闭连接，不是的话继续读 processor.Process

```

![image](https://upload-images.jianshu.io/upload_images/12321605-d19e6b183ef7baac?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 四、Kite （Goroutine Per Connection）

```
kite跟Thrift Go Server 流程基本一样，只是在整理流程中加入了一些服务治理的东西，比如判断的连接是否过载，加一些中间件（打点、熔断、限流、ACL、压测、定时拉取ms配置）等等。

```

```
kite.Run() -> RPCServer.ListenAndServe -> CreateListener() -> Serve() 
-> for {Accept()} 一个连接开一个goroutine  -> processRequests
-> for 循环 { processor.Process} -> Process(in, out TProtocol)
-> name, _, seqId, err := iprot.ReadMessageBegin() (seqId回复的数据包要回写回去)
-> 根据name 找到对应方法的 Process，调用对应的Process
-> args.Read(iprot) -> iprot.ReadMessageEnd() -> handler.xxxx 方法拿到结果 ->
oprot.WriteMessageBegin-> response字段.Write(oprot) -> oprot.WriteMessageEnd() ->
oprot.Flush() -> 判断是否 ErrAbandonRequest 是的话关闭连接，不是的话继续读 processor.Process

```

![image](https://upload-images.jianshu.io/upload_images/12321605-3bd0241472e2566f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 五、Kitex-Netpoll（主从Reactor）

其实为了解决 [字节跳动在 Go 网络库上的实践](https://mp.weixin.qq.com/s/wSaJYg-HqnYY4SdLA2Zzaw)提到的“**Go 调度导致的延迟问题**” 最新的Netpoll已经改成了单Reactor模式。

![image](https://upload-images.jianshu.io/upload_images/12321605-902e074e38161b26?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

随便说下Go自身的Net库没有这个问题，是因为Golang的网络请求都是在GO自己的一个[Sysmon监控线程维护](https://github.com/golang/go/blob/master/src/runtime/proc.go#L5099)的，**Sysmon线程不需绑定P**，首次休眠20us，每次执行完后，增加一倍的休眠时间，但是最多休眠10ms。

Sysmon主要做以下几件事

```
1\. 释放闲置超过5分钟的span物理内存
2\. 如果超过两分钟没有执行垃圾回收，则强制执行
3\. 将长时间未处理的netpoll结果添加到任务队列
4\. 向长时间运行的g进行抢占
5\. 收回因为syscall而长时间阻塞的p

```

[Golang runtime的netpoll函数](https://github.com/golang/go/blob/master/src/runtime/netpoll_epoll.go#L106)主要做的是就是调用epollwait拿到活跃的FD，然后唤醒相关阻塞的gorotine。

![image](https://upload-images.jianshu.io/upload_images/12321605-b7f607bde828cf3a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/12321605-59a9682ef60f1a64?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Goroutine Pool ：** 减少gorotine 调度开销， 最大10000

**LinkBuffer** : 用来“分离网络IO”和“业务处理过程” ， 减少序列化和传递过程中字节的拷贝

**Codec**：Kitex 支持自定义Codec。默认支持Thrift和ProtoBuf 两种Codec。

```
/*
kitex server

-> 启动netpoll初始化
netpool init函数 -> 初始化 loops数量 -> SetNumLoops() ->  m.Run()
-> openPoll()这个里面调用syscall.EpollCreate1 ->  go poll.Wait() 这里调用 epollwait Reactor
-> subReactor 数量 = CPUNUM() / 20 + 1

-> 初始化Server
LarkSvcDemo.NewServer -> server.NewServer -> 初始化中间件 -> RegisterService -> Run()
-> richRemoteOption() -> 初始化opt.RemoteOpt 属性 -> addBoundHandlers
-> 添加入流量和出流量处理类 In/Out boundsHandler(ServerBaseHandler、ServerTTHeaderHandler)
-> Start() -> buildListener -> netpoll.CreateListener -> go s.transSvr.BootstrapServer()
-> netpoll server.Run() -> pollmanager.Pick() 找个一个 epoll 出来。
-> 注册 listen fd 的 onRead 事件里面主要是Accept Socket ，可以通过 OnRead != nil 判断这个fd是不是listen的FD

-> 等待接受链接
-> newConnectionWithPrepare 初始化链接相关 设置回调函数，添加链接到epoll、保存FD->connection关系
-> connection.init -> 设置 Fd 为 noblocking -> 初始化inputBuffer outputBuffer = NewLinkBuffer()
-> 设置 supportZeroCopy -> onEvent . onPrepare 设置熟悉 -> 这里onRequest 就是 transServer.onConnRead
-> onEvent .process = onRequest -> onPrepare 就是 transServer.onConnActive -> ts.connCount.Inc()
-> OnActive 新建立连接是触发 -> inboundHdrls 执行OnActive -> svrTransHandler OnActive 初始化RCPinfo
-> register -> pollmanager.Pick() -> 添加Fd到 epoll -> s.connections.Store(fd, connection)

-> 等待接受客户端数据
-> epollWait 返回活跃链接 -> operator.Inputs -> Book 这个主要是判断是否需要扩大buffer
-> syscall.SYS_READV -> inputAck -> MallocAck(n)  linkbuffer的malloc += n ,buf = [:n]

-> 读取完以后处理数据
-> onEvent.onRequest -> gopool.CtxGo(on.ctx, task) 新建一个task丢pool里面去让worker处理
-> 执行task -> handler -> transServer. onConnRead -> transMetaHandler.OnRead -> svrTransHandler.OnRead
-> NewMessageWithNewer -> SetPayloadCodec -> NewReadByteBuffer(ctx, conn, recvMsg)
-> Decode -> flagBuf = Peek(8) 先读8个字节出来 , 根据前8个字节判断是不是TTHeader或者MeshHeader编码的
-> IsTTHeader -> isMeshHeader -> checkPayload 这个是没有header的编码 -> isThriftBinary
-> 得到编码数据codecType是Thrift还是PB, transProto 是Framed还是transport.TTHeaderFramed
-> decodePayload -> GetPayloadCodec 拿解码器，可以自定义codec，默认支持thrift和PB
-> pCodec.Unmarshal -> thriftCodec.Unmarshal
-> methodName, msgType, seqID, err := tProt.ReadMessageBegin()
-> req.Read(tProt) 读取数据解码到 request -> tProt.ReadMessageEnd() -> tProt.Recycle()
-> sendMsg = remote.NewMessage -> transPipe.OnMessage -> TransPipeline.OnMessage
-> transMetaHandler.OnMessage -> serverBaseHandler.ReadMeta 这里主要是设置logID、caller、GetExtra
-> serverTTHeaderHandler.ReadMeta -> svrTransHandler.OnMessage -> NewServerBytedTraceMW
-> NewStatusCheckMW -> NewRPCConfigUpdateMW -> NewACLMiddleware -> invokeHandleEndpoint
-> getConfigDemoHandler 到业务方的代码handler -> 执行完业务代码拿到response

-> 把拿到的response回复出去
-> transPipe.Write -> outboundHdrls.Write -> transMetaHandler.Write -> serverTTHeaderHandler.WriteMeta
-> svrTransHandler.Write -> NewWriteByteBuffer -> codec.Encode -> defaultCodec.Encode
-> getPayloadBuffer -> encodePayload -> pCodec.Marshal -> thriftCodec.Marshal
-> tProt.WriteMessageBegin() BinaryProtocol -> BinaryProtocol 底层内存用的是LinkBuffer 减少一次Copy？
-> msg.Write(tProt) -> tProt.WriteMessageEnd() -> bufWriter.Flush() -> connection.flush
-> atomic.StoreInt32(&c.writing, 2) 加锁 -> sendmsg -> syscall SYS_SENDMSG
-> outputAck() -> 调整底层LinkBuffer的指针

*/

```

## IO模型视角

### **一、同步阻塞IO**

![image](https://upload-images.jianshu.io/upload_images/12321605-7dd0f91bbe4f0680?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### **二、同步非阻塞IO**

![image](https://upload-images.jianshu.io/upload_images/12321605-fca24d340f5ba47f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
func SetNonblock(fd int, nonblocking bool) (err error) {
   flag, err := fcntl(fd, F_GETFL, 0)
   if err != nil {
      return err
   }
   if nonblocking {
      flag |= O_NONBLOCK
   } else {
      flag &^= O_NONBLOCK
   }
   _, err = fcntl(fd, F_SETFL, flag)
   return err
}

```

### **三、IO多路复用 （epoll、kqueue、select）**

![image](https://upload-images.jianshu.io/upload_images/12321605-3cfef77b51bb63dc?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 四、信号驱动

![image](https://upload-images.jianshu.io/upload_images/12321605-a09ac98836a4daef?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 五**、异步IO**

![image](https://upload-images.jianshu.io/upload_images/12321605-356652fa26c0b0f1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/12321605-6983d49e972be5e1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 线程模型视角

### 一、**线程模型** Thread Per Connection

**生产环境基本没有使用这种模型的**

```
采用阻塞式 I/O 模型获取输入数据；
每个连接都需要独立的线程完成数据输入，业务处理，数据返回的完整操作。
缺点：
当并发数较大时，需要创建大量线程来处理连接，系统资源占用较大；
连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在 Read 操作上，造成线程资源浪费。

```

![image](https://upload-images.jianshu.io/upload_images/12321605-714c4431d96deb48?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 二、单Reactor单线程

```
优点：简单，没有多线程，没有进程通信
缺点：性能，无法发挥多核的极致，一个handler卡死，导致当前进程无法使用，IO和CPU不匹配
场景：客户端有限，业务处理快，比如redis

```

![image](https://upload-images.jianshu.io/upload_images/12321605-d410a6277eecae1a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 三、单Reactor多线程

```
优点：充分利用的CPU
缺点：进程通信，复杂，Reactor承放了太多业务，高并发下可能成为性能瓶颈

```

![image](https://upload-images.jianshu.io/upload_images/12321605-df0beafafd370a62?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 四、主从Reactor多线程

```
主Reactor负责建立连接，建立连接后的句柄丢给子Reactor，子Reactor负责监听所有事件进行处理
优点：职责明确，分摊压力
Nginx/netty/memcached都是使用的这

```

![image](https://upload-images.jianshu.io/upload_images/12321605-4e21da74445bf72e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 五、Proactor 模型（异步IO）

```
  编程复杂性，由于异步操作流程的事件的初始化和事件完成在时间和空间上都是相互分离的，因此开发异步应用程序更加复杂。应用程序还可能因为反向的流控而变得更加难以 Debug；
  内存使用，缓冲区在读或写操作的时间段内必须保持住，可能造成持续的不确定性，并且每个并发操作都要求有独立的缓存，相比 Reactor 模式，在 Socket 已经准备好读或写前，是不要求开辟缓存的；
  操作系统支持，Windows 下通过 IOCP 实现了真正的异步 I/O，而在 Linux 系统下，Linux 2.6 才引入，目前异步 I/O 还不完善。

```

![image](https://upload-images.jianshu.io/upload_images/12321605-940d12d2acc853d5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
