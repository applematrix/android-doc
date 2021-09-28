# Android S RPC能力分析

## 整体关系图



## FdTrigger类

![image-20210925103832837](C:\Users\huangdezhi\Documents\GitHub\android-doc\images\rpcbinder\image-20210925103832837.png)

FdTrigge目前在Android版本上是一个pipe管道的管理器，其中保留了管道的read端和write端两个句柄。

FdTrigger将文件句柄包装后，提供了轮训的接口，检测对应的fd上是否有数据。

## RpcTransport类

RpcTransport类是一个抽象的接口类，用于代表一个socket连接，一个socket连接就是一个RpcTransport类。

![image-20210925105201215](images\rpcbinder\image-20210925105201215.png)



RpcTransport定义了一套标准接口，实际最终在系统中是哟功能的是其两个子类：RpcTranspotTls和RpcTransportRaw，其中一个是通过ssl支持安全通信的，另一个则是使用原始的通信机制。

Transport主要提供了两个接口，一个是写接口，一个是读接口。另外一个peek接口则是用来测试是否存在数据的。

## RpcConnection类

RpcConnection类只是一个简单的封装，封装了实际的RpcTransport对象和对应的线程id：

![image-20210925110851799](images\rpcbinder\image-20210925110851799.png)



## RpcSession类

## RpcState类

RpcState目前分析是RpcBinder的核心类，该类承载了Rpc调用过程中的数据发送，接收等。需要详细分析

## RpcState传输的数据

### **CommandData**

在RpcState中传输的数据中，通过commandData类来传输，commandData的数据如下所示：

![image-20210925135541969](images\rpcbinder\image-20210925135541969.png)

主要就是一块内存的数据指针和数据的大小，传递的内存数据中包括两类：

1. RpcWireHeader，数据包的包头
2. RpcWireTransaction，数据包的传输信息
3. data，数据的实际大小

### RpcWireHeader

![image-20210925140057780](images\rpcbinder\image-20210925140057780.png)



RpcWireHeader的传输头如上所示，在传输过程中，command默认为RPC_COMMAND_TRANSACT，bodySize为包体的大小，包括后面的RpcWireTransaction的数据和data数据的大小，预留了8个字节的保留位

### RpcWireTransaction

![image-20210925140755534](images\rpcbinder\image-20210925140755534.png)



tansaction中，包含了一个RpcWireAddress的地址信息，code和flags的含义与本地binder中的code和flags的含义类似，其中还包含一个asyncNumber序列号，最后一个data时可变的data的起始。

## ServiceDispatcher服务端启动过程

ServiceDispatcher是系统中的一个服务分发，是一个独立的进程。在系统中可以通过命令行：

servicedispatcher manager

启动ServiceDispatcher

![](images\rpcbinder\rpcsession分析-ServiceDispatcher启动过程.png)

ServiceDispatcher的启动功能过程如上所示

## Rpc会话启动过程

![](images\rpcbinder\rpcsession分析-Rpc会话启动过程.png)

在系统中目前发现的启动RpcBinder的客户端为ServiceManagerHost。以该类启动RpcSession的过程举例如上。理论任何应用都可以通过类似以上的流程连接到已有的Rpc服务。需要指定服务端IP和端口。

## 带Rpc会话的Parcel

Rpc通信过程中的Parcel对象不同于普通的parcel对象，从S版本开始，Android对Parcel加入一个额外的特性，用于识别该Parcel将会通过Rpc的方式进行序列化。

![image-20210925211534104](images\rpcbinder\image-20210925211534104.png)

Rpc的Parcel具备一个RpcSession用于标识该Parcel会用于哪个Rpc会话。



## BpBinder的创建过程

![image-20210925210034393](images\rpcbinder\image-20210925210034393.png)

Parcel在反序列化Binder对象的时候，如果发现其是在RpcBinder会话中。

## Rpc数据传出的过程

当拿到BpBinder对象的客户端，调用BpBinder的transact函数时，对外部，BpBinder与普通的binder调用方式没有差异，但是在前面BpBinder的创建时，如果该BpBinder是通过Rpc创建的，则该BpBinder的handle将关联到一个RpcSession，从而将通过如下的流程把数据通过传递到远端：

![image-20210925213441759](images\rpcbinder\image-20210925213441759.png)



















