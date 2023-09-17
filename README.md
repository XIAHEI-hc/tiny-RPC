# tiny-RPC

> RPC 是 "Remote Procedure Call" 的缩写，中文翻译为 "远程过程调用"。它是一种通讯协议，使得程序能够在一个地址空间（例如一个程序或计算机）请求另一个地址空间（例如另一个程序或计算机）执行某个程序或函数，仿佛是本地过程调用。简而言之，RPC 允许你执行远程计算机上的函数，就像它们是本地函数一样。
>
> RPC是一种思想，其实Http也是RPC的一种思想的实现

## 编译成库调用方法

整个项目的产出，是一个库文件 `librocket.a` 和一系列头文件`rocket/*.h`. 库文件不是可执行程序，他不包含 main 函数，不能直接运行，需要我们写具体的 main 函数并且链接这个库。

首先创立对应的文件夹

1. **bin**: 这是“binary”的缩写。通常用于存放编译后的可执行文件。
2. **log**: 这个目录通常用于存放运行时的日志文件。这对于诊断问题、审计和监控应用程序的运行状态是很有用的。
3. **obj**: 这是“object”的缩写。它通常用于存放编译过程中生成的对象文件，这些文件是源代码文件编译后的中间产物。
4. **lib**: 这个目录通常用于存放库文件。在 C 和 C++ 项目中，这可能包括静态库（`.a` 或 `.lib` 文件）和动态链接库（如 `.so` 或 `.dll` 文件）。

```
function create_dir() {
  if [ ! -d $1 ]; then
    mkdir $1
  fi
}

create_dir 'bin'
create_dir 'log'
create_dir 'obj'
create_dir 'lib'

make clean && make -j4
```

然后，makefile文件make就行（需要提前安装好xml模块和protobuf模块）。

###  tinyxml

以tinyxml 为例，tinyxml 分为库文件 `libtinyxml.a` 和头文件 `tinyxml/*.h`

其中库文件一定需要安装在 `/usr/lib` 目录下，即绝对路径为 `/usr/lib/libtinyxml.a` ，如果不一致，请拷贝过去

而头文件，所有 `*.h` 的头文件，必须位于 `tinyxml/` 目录下，而整个 `tinyxml` 目录需要放在 `usr/include` 下，即绝对路径为 `/usr/include/tinyxml`, `tinyxml` 下包含所有的 `.h` 结尾的头文件

###  protobuf

同 tinyxml，库文件在 `/usr/lib/libprotobuf.a`, 所有头文件 `*.h` 在 `/usr/include/google/protobuf/` 下

# 设计思路

1. client进行**connect**连接

2. client**序列化**req

3. client根据约定的协议编码，向server发送**编码**后的数据

4. server**接收**到数据，**解码**得到方法名和序列化过后的req二进制流

5. server根据方法名找到req的类型，**反序列化**得到req对象

6. server调用本地方法得到res

7. server**序列化**res

8. server根据约定的协议**编码**并发送数据

9. client接收到数据并**解码**

10. clien**t反序列化**得到res

    ```
    Client:
    1. connection = connect_to_server()
    2. serialized_req = serialize(request)
    3. encoded_data = encode(serialized_req)
    4. send_to_server(connection, encoded_data)
    
    Server:
    5. received_data = listen_for_data()
    6. decoded_data = decode(received_data)
    7. method_name, serialized_req = extract(decoded_data)
    8. request_object = deserialize(serialized_req)
    9. response = local_method(request_object)
    10. serialized_res = serialize(response)
    11. encoded_response = encode(serialized_res)
    12. send_to_client(encoded_response)
    
    Client:
    13. received_data = listen_for_data()
    14. decoded_response = decode(received_data)
    15. response_object = deserialize(decoded_response)
    
    ```

    > 综合而言，需要设计
    >
    > **1.一个优秀的网络I/o,来面对高并发的情况（I/o复用）**
    >
    > **2.序列化以及网络传输协议定义（protobuf/自定义通讯协议）**
    >
    > **3.读取网络参数模块，定义网络传输参数（XML）**
    >
    > 4.**一个日志模块（参考自他人设计，一方面最好是一个异步日志！，另一方面需要解决多线程以及顺序的问题）**
    >
    > ---
    >
    > 其余小设计
    >
    > 5.定时器模块（防止一直在epoll_wait的过程中，导致连接队列里里面的任务一直没有执行）
    >
    > 6.编码解码模块-事件分发器模块（在RPC调用的过程中，必不可少的就是远程调用的远程函数或者方法名！要根据不同的业务名，使用事件分发器分发给不同的业务模块）

## 高性能网络部分设计

从本质上来看，事实上一个网络服务器要做的事无非就只有三步：

1. 建立连接

2. 读数据

3. 处理事件

4. 写数据

   其中的关键
   
   1.事件驱动
   
      对于accept、connect、read、write等系统调用，实际上都属于慢系统调用，他可能会永远阻塞直到套接字上发生 可读\可写 事件。事实上，通常不希望一直阻塞直到IO就绪，而应该等待IO就绪之后再通知我们过来处理。
   
   IO复用可以通过 select、poll、epoll来实现。这里只简单说下epoll。
   
   epoll就是实现这个功能的，使用epoll只需要三步：
   
   1. 调用epoll_create创建套接字epfd。
   2. 调用epoll_ctl 增加需要关心的套接字的事件，如 listenfd的可读事件。
   3. 调用epoll_wait沉睡，直到有关心的事件发生后epoll_wait会主动返回,此时我们去处理发生了IO事件的相应套接字即可。
   
   
   
   具体的epoll用法这里不再细说。在这里把epoll看成一个黑匣子即可，暂时不关心原理。我们只需要将关心得套接字事件注册到epoll上，epoll就会在这些事件发生时通知我们。
   
   2.将连接I/O事件和读写I/O事件分开（主从Reactor）

我所实现的其实是一个主从Reactor模式，其中单Reactor模式+线程池能够很大程度上支持并发了，但还可以优化。注意到单Reactor模式只有一个Reactor线程，所有的关心套接字都需要注册到这个Reactor上。可以看到，这个主线程的Reactor既要负责客户端连接事件的处理(即关心listenfd的事件)，又要关心已连接套接字的事件（即关心clientfd的io事件）。

   能不能用多个Reactor，其中一个Reactor只监听listenfd，获取到新连接后就把clientfd甩给其他Reactor来监听呢？这就是主从Reactor模式：

```
void loop() {
  while(!stop) {
      foreach (task in tasks) {
        task();
      }

      // 1.取得下次定时任务的时间，与设定time_out去较大值，即若下次定时任务时间超过1s就取下次定时任务时间为超时时间，否则取1s
      int time_out = Max(1000, getNextTimerCallback());
      // 2.调用Epoll等待事件发生，超时时间为上述的time_out
      int rt = epoll_wait(epfd, fds, ...., time_out); 
      if(rt < 0) {
          // epoll调用失败。。
      } else {
          if (rt > 0 ) {
            foreach (fd in fds) {
              // 添加待执行任务到执行队列
              tasks.push(fd);
            }
          }
      }
      
      
  }
}
```

mainReactor由主线程运行，他作用如下：通过epoll监听listenfd的可读事件，当可读事件发生后，调用accept函数获取clientfd，然后随机取出一个subReactor，将cliednfd的读写事件注册到这个subReactor的epoll上即可。也就是说，mainReactor只负责建立连接事件，不进行业务处理，也不关心已连接套接字的IO事件。



subReactor通常有多个，每个subReactor由一个线程来运行。subReactor的epoll中注册了clientfd的读写事件，当发生IO事件后，需要进行业务处理。

## 序列化以及自定义网络协议传输

### 序列化

序列化有很多方法，序列化是将数据结构或对象转换为可以存储或传输的格式的过程。常用的序列化格式包括 JSON、XML、YAML 以及 Protocol Buffers（protobuf）等。那么，为什么选择 protobuf 而不是其他格式呢？

首先，要明确 Protocol Buffers（protobuf）是由 Google 设计的，并且其设计目的是为了满足高效的序列化和反序列化操作。与此相反，JSON 和 XML 主要是为了数据交换的可读性设计的，而不是效率。

#### 压缩空间

- **Field Tag**: protobuf 使用数字标签（field tag）来标识字段，而 JSON 和 XML 使用字符串。数字标签占用的空间要小于字符串名。
- **数据格式**: protobuf 是二进制格式，而 JSON 和 XML 是文本格式。二进制格式通常更紧凑，特别是对于大型数据。
- **不存储字段名**: JSON 和 XML 都存储字段名，而 protobuf 不会。例如，JSON 中的 `{ "name": "Alice" }` 在 protobuf 中可能只是一个二进制值，不包含字段名 "name"。
- **数据类型**: protobuf 根据字段的数据类型采用了不同的编码方法，例如 varints。这种方法可以在许多情况下显著压缩数据。

#### 高效的编解码

protobuf 的编解码速度通常比 JSON 和 XML 快得多。这是因为 protobuf 的解码器可以直接将二进制数据转换为内部数据结构，而无需进行文本解析、数值转换或复杂的内存分配操作。

## 自定义传输协议

> **首先，为什么要自定义协议？**
>
> **其实，这不是必须的，可以使用http，或者其余别的什么别的协议！但是一方面http协议比较厚重，另一方面，自定义协议可以随便加功能，就像上面解决了一个优化大小的问题，打算用protobuf了，那这一堆字节流怎么实现功能表达？(加service/method)。怎么实现数据分包？(加开始位长度位结束位)。怎么实现请求-回应的映射？(加id)。怎么实现校验？(加校验码)。怎么让客户端知道执行成功与否？(加错误段).....**

当然，理想的情况是每次rpc调用我只发送一次，一次就是一个完整的数据包。另一方可以得到这个完整的包并且成功解析。然而事实肯定没这么容易，由于**网络阻塞**、或者**TCP粘包**等原因，服务端当时收到的包可能不是一个完整的包，或者说无法区别每个数据包的边界。

因此，**不能说直接把QueryReq 对象序列化之后就直接丢给网络传了**，而不管其他的。这让服务端怎么解包，万一收到了半个包？那边是多了一个字节，那么反序列化的结果也可能千奇百怪。另外有一点，有没有发现**方法名**好像也没有传输过去？那服务端怎么知道调哪个方法？

基于此，通常我们需要自定义协议，来完成数据包的制造和拆分。这个过程又可以称为**编码和解码。**

参考**陈硕**的文章，这里设计一个简单地传输协议格式如下：

```cpp
char* start;    // 0x02
int32_t pk_len;
int32_t service_full_name_len;
std::string service_full_name;
pb binary data;
int32_t checksum;
char* end;      // 0x03
```

- start 和 end表示一个数据包的开始和结束。
- pk_len为整个数据包的长度。
- service_full_name_len 为方法名的长度
- service_full_name 为方法名，如 "query_name"
- data就是请求对象的序列化后的二进制数据
- checksum是校验和，用于接收端校验是否数据包数据有变化

这里不需要考虑**字节对齐**，直接按顺序组合以上结构就行了。

综上，**发送端在发送数据时需要按照这个格式编码，而服务端接受到数据时需要按这个格式解码。**

但是实际中，可以多一些功能位，如下：

```
class TinyPbStruct : public AbstractData {
 public:

  // char start;                      // 标识 TinyPb 协议数据的开始
  int32_t pk_len {0};                 // 整个包的长度（包括开始字符和结束字符）
  int32_t msg_req_len {0};            // msg_req 的长度
  std::string msg_req;                // msg_req，用于标识一个请求
  int32_t service_name_len {0};       // 服务全名的长度
  std::string service_full_name;      // 服务的完整名称，例如 QueryService.query_name
  int32_t err_code {0};               // 错误代码，0 -- RPC 调用成功，否则 -- RPC 调用失败。只由 RpcController 设置
  int32_t err_info_len {0};           // err_info 的长度
  std::string err_info;               // 错误信息，空 -- RPC调用成功，否则 -- RPC调用失败，它会显示调用失败的具体原因。只由 RpcController 设置
  std::string pb_data;                // 业务 pb 数据
  int32_t check_num {-1};             // 整个包的校验数。用于检查数据的合法性
  // char end;                        // 标识 TinyPb 协议数据的结束

};

```



## 网络参数入口模块(.xml)



当然，一个基本的配置文件是必须的。TinyRPC 通过读取 xml 配置文件获取一些必要的配置信息，这些信息包括服务ip、端口、协议类型、以及其他配置等。

xml 文件模板已经定好了，只需要按需填写自己的数据即可，示例如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<root>
  <log>
    <log_level>DEBUG</log_level>
    <log_file_name>test_rpc_server</log_file_name>
    <log_file_path>../log/</log_file_path>
    <log_max_file_size>1000000000</log_max_file_size>
    <log_sync_interval>500</log_sync_interval>
  </log>

  <server>
    <port>12345</port>
    <io_threads>4</io_threads>
  </server>

  <stubs>
    <rpc_server>
      <!-- 默认配置 -->
      <!-- 调用下游RPC 服务时，需要将其 ip port 配置在此处 -->
      <name>default</name>
      <ip>0.0.0.0</ip>
      <port>12345</port>
      <timeout>1000</timeout>
    </rpc_server> 
  </stubs>


</root>

```

最重要的是 **ip** 和 **port** 以及 **protocal** 参数，其他配置按照默认值即可，一般不需要特殊修改。这里我绑定的地址是：192.168.245.7:19999

## 日志模块

首先需要创建项目：

日志模块：
```
1. 日志级别
2. 打印到文件，支持日期命名，以及日志的滚动。
3. c 格式化风控
4. 线程安全
```

LogLevel:
```
Debug
Info
Error
```

LogEvent:
```
文件名、行号
MsgNo
进程号
Thread id
日期，以及时间。精确到 ms
自定义消息
```

日志格式
```
[Level][%y-%m-%d %H:%M:%s.%ms]\t[pid:thread_id]\t[file_name:line][%msg]
```

Logger 日志器
1.提供打印日志的方法
2.设置日志输出的路径

## 小根堆实现定时器思路

> 定时器作用不再多数，网络程序中经常需要一些定时任务、周期任务，就需要靠 tiemr 实现。
>
> 定时器原理是 timerfd，建议 google 搜索一下 timerfd 了解其原理。
>
> 本质上他就是一个 fd，给他设置一个时间戳，到达这个时间戳后 fd 变得可读，epoll_wait 监听到可读后马上返回，即可执行其上面的定时任务。

伪代码

1. 使用小根堆维护所有的定时任务，堆顶的任务是最早到期的任务。

2. 每当有一个新的任务被加入或现有任务被删除/修改，检查堆顶的任务，确保其与 `timerfd` 的时间戳相匹配。

3. 使用 `epoll`（或其他 I/O 多路复用机制）监听 `timerfd` 的文件描述符。

4. 当 `timerfd` 变为可读时，执行堆顶的任务，然后从堆中移除该任务。再次检查堆顶任务并更新 `timerfd` 的时间戳。

   ```
   # 定义一个定时任务类，包含任务的到期时间和回调函数
   class TimerTask:
       expiration_time: datetime  # 到期时间
       callback: function         # 当到期时要执行的回调函数
   
   # 定义定时器管理类
   class TimerManager:
       min_heap: List[TimerTask]  # 使用小根堆存储所有的定时任务，堆顶是最早到期的任务
       timerfd: int               # timerfd 文件描述符，用于通知定时事件
       epoll: int                 # epoll 文件描述符，用于监听多个文件描述符
   
       def __init__(self):
           # 创建 timerfd 和 epoll，并将 timerfd 加入到 epoll 中进行监听
           self.timerfd = create_timerfd()
           self.epoll = epoll_create()
           epoll_add(self.epoll, self.timerfd)
   
       # 添加一个定时任务到小根堆
       def add_task(self, task: TimerTask):
           insert_into_heap(self.min_heap, task)
           # 如果新添加的任务是最早到期的任务，则更新 timerfd 的时间戳
           if task is at top of heap:
               set_timerfd(self.timerfd, task.expiration_time)
   
       # 从小根堆中移除一个定时任务
       def remove_task(self, task: TimerTask):
           remove_from_heap(self.min_heap, task)
           # 如果被移除的任务是最早到期的任务，则更新 timerfd 的时间戳为下一个最早到期的任务
           if task was at top of heap:
               set_timerfd(self.timerfd, top_of_heap(self.min_heap).expiration_time)
   
       # 运行定时器，循环监听 timerfd
       def run(self):
           while True:
               events = epoll_wait(self.epoll)
               for event in events:
                   # 当 timerfd 变为可读时，表示有定时任务到期
                   if event.fd == self.timerfd:
                       # 获取并移除最早到期的任务，并执行其回调函数
                       task = pop_from_heap(self.min_heap)
                       task.callback()
                       # 更新 timerfd 的时间戳为下一个最早到期的任务
                       set_timerfd(self.timerfd, top_of_heap(self.min_heap).expiration_time)
   
   # 使用示例
   timer_manager = TimerManager()
   task = TimerTask(expiration_time=in_5_minutes, callback=my_function)
   timer_manager.add_task(task)
   timer_manager.run()
   
   ```

   

## 编码解码模块事件分发器

**TcpConnection** 主要负责的是 **读数据** 和 **写数据**，并不关心事件处理的逻辑。事实上，**TcpConnection** 就是向**事件处理器**输入数据，等处理完成后又从事件处理器中拿出数据回送给客户端。

### 编码器

**AbstractCodeC**是一个抽象的**虚基类**, 它提供有：**encode**(编码)、**decode**(解码) 这两个基本的方法。

所有协议的编解码器都必须**继承**这个基类，并实现自己的编解码逻辑，即重写 **encode**、**decode** 方法。

```
class AbstractCodeC {
 public:
  typedef std::shared_ptr<AbstractCodeC> ptr;

  AbstractCodeC() {}

  virtual ~AbstractCodeC() {}

  // 编码, buf 是输入缓冲区， data 是协议格式
  // 即将 AbstractData 格式的数据包编码后输入到 buf 中
  virtual void encode(TcpBuffer* buf, AbstractData* data) = 0;

  // 解码, buf 是输出缓冲区， data 是协议格式
  // 即从 buf 中解析出一个 AbstractData 格式的数据包
  virtual void decode(TcpBuffer* buf, AbstractData* data) = 0;

  virtual ProtocalType getProtocalType() = 0;
};
```

```
class TinyPbCodeC: public AbstractCodeC {
 public:
  // overwrite
  // 编码, buf 是输入缓冲区， data 是协议格式
  // 即将 AbstractData 格式的数据包编码后输入到 buf 中
  void encode(TcpBuffer* buf, AbstractData* data);

  // overwrite

  // 解码, buf 是输出缓冲区， data 是协议格式
  // 即从 buf 中解析出一个 AbstractData 格式的数据包
  void decode(TcpBuffer* buf, AbstractData* data);

  // overwrite
  virtual ProtocalType getProtocalType();

  const char* encodePbData(TinyPbStruct* data, int& len);
};
```

### 事件分发器

数据解码完成了，就产生了一个对应协议类型的数据包。接下来就要把这个数据包交给对应协议的**事件分发器**来处理了。不同协议有不同的分发处理器，这里还是以 **TinyPB** 协议的方法处理来作例子：

###  抽象事件分发器 AbstractDispatcher

**AbstractDispatcher** 也是一个抽象基类，所有协议的事件分发器都必须继承这个基类，并重写 **dispatch** 方法实现自己的事件分发逻辑：

```cpp
class AbstractDispatcher {
 public:
  typedef std::shared_ptr<AbstractDispatcher> ptr;

  AbstractDispatcher() {}

  virtual ~AbstractDispatcher() {}

  virtual void dispatch(AbstractData* data, TcpConnection* conn) = 0;
};
```

**dispatch** 方法接收一个 **AbstractData** 类型的数据包，处理完事件后将回包数据放到对应的 **TcpConnection** 连接中，**TcpConnection** 负责后面的回送响应数据包给客户端。

```
void TinyPbRpcDispacther::dispatch(AbstractData* data, TcpConnection* conn) {
  // tmp 是请求数据包， TinyPbStruct 类型的
  TinyPbStruct* tmp = dynamic_cast<TinyPbStruct*>(data);

  InfoLog << "begin to dispatch client tinypb request, msgno=" << tmp->msg_req;

  std::string service_name;
  std::string method_name;

  // 初始化响应数据包
  TinyPbStruct reply_pk;
  // 设置接口名
  reply_pk.service_full_name = tmp->service_full_name;
  reply_pk.msg_req = tmp->msg_req;
  if (reply_pk.msg_req.empty()) {
    reply_pk.msg_req = MsgReqUtil::genMsgNumber();
  }
  // 解析 service_full_name，得到 service_name 和 method_name
  if (!parseServiceFullName(tmp->service_full_name, service_name, method_name)) {
    // 解析失败，返回对应错误码, 不进行其他的处理
    ErrorLog << reply_pk.msg_req << "|parse service name " << tmp->service_full_name << "error";

    reply_pk.err_code = ERROR_PARSE_SERVICE_NAME;
    std::stringstream ss;
    ss << "cannot parse service_name:[" << tmp->service_full_name << "]";
    reply_pk.err_info = ss.str();
    // 编码响应数据包，回送给 TcpConnection 对象
    conn->getCodec()->encode(conn->getOutBuffer(), dynamic_cast<AbstractData*>(&reply_pk));
    return;
  }

  Coroutine::GetCurrentCoroutine()->getRunTime()->m_interface_name = tmp->service_full_name;
  // 找到对应的 service
  auto it = m_service_map.find(service_name);
  if (it == m_service_map.end() || !((*it).second)) {
    reply_pk.err_code = ERROR_SERVICE_NOT_FOUND;
    std::stringstream ss;
    ss << "not found service_name:[" << service_name << "]"; 
    ErrorLog << reply_pk.msg_req << "|" << ss.str();
    reply_pk.err_info = ss.str();

    conn->getCodec()->encode(conn->getOutBuffer(), dynamic_cast<AbstractData*>(&reply_pk));

    InfoLog << "end dispatch client tinypb request, msgno=" << tmp->msg_req;
    return;

  }
  service_ptr service = (*it).second;

  // 找到对应的 method
  const google::protobuf::MethodDescriptor* method = service->GetDescriptor()->FindMethodByName(method_name);
  if (!method) {
    reply_pk.err_code = ERROR_METHOD_NOT_FOUND;
    std::stringstream ss;
    ss << "not found method_name:[" << method_name << "]"; 
    ErrorLog << reply_pk.msg_req << "|" << ss.str();
    reply_pk.err_info = ss.str();
    conn->getCodec()->encode(conn->getOutBuffer(), dynamic_cast<AbstractData*>(&reply_pk));
    return;
  }
  // 根据 method 对象反射出 request 和 response 对象
  google::protobuf::Message* request = service->GetRequestPrototype(method).New();
  DebugLog << reply_pk.msg_req << "|request.name = " << request->GetDescriptor()->full_name();
  // 将客户端数据包里面的 protobuf 字节流反序列化为 request 对象 
  if(!request->ParseFromString(tmp->pb_data)) {
    reply_pk.err_code = ERROR_FAILED_SERIALIZE;
    std::stringstream ss;
    ss << "faild to parse request data, request.name:[" << request->GetDescriptor()->full_name() << "]";
    reply_pk.err_info = ss.str();
    ErrorLog << reply_pk.msg_req << "|" << ss.str();
    delete request;
    conn->getCodec()->encode(conn->getOutBuffer(), dynamic_cast<AbstractData*>(&reply_pk));
    return;
  }

  InfoLog << "============================================================";
  InfoLog << reply_pk.msg_req <<"|Get client request data:" << request->ShortDebugString();
  InfoLog << "============================================================";

  // 初始化 response 对象
  google::protobuf::Message* response = service->GetResponsePrototype(method).New();

  DebugLog << reply_pk.msg_req << "|response.name = " << response->GetDescriptor()->full_name();

  TinyPbRpcController rpc_controller;
  rpc_controller.SetMsgReq(reply_pk.msg_req);
  rpc_controller.SetMethodName(method_name);
  rpc_controller.SetMethodFullName(tmp->service_full_name);

  std::function<void()> reply_package_func = [](){};

  TinyPbRpcClosure closure(reply_package_func);
  // 调用业务处理方法
  service->CallMethod(method, &rpc_controller, request, response, &closure);

  InfoLog << "Call [" << reply_pk.service_full_name << "] succ, now send reply package";
  // 将业务处理后的 response 对象序列化为字节流，放到响应数据包中
  if (!(response->SerializeToString(&(reply_pk.pb_data)))) {
    reply_pk.pb_data = "";
    ErrorLog << reply_pk.msg_req << "|reply error! encode reply package error";
    reply_pk.err_code = ERROR_FAILED_SERIALIZE;
    reply_pk.err_info = "failed to serilize relpy data";
  } else {
    InfoLog << "============================================================";
    InfoLog << reply_pk.msg_req << "|Set server response data:" << response->ShortDebugString();
    InfoLog << "============================================================";
  }

  delete request;
  delete response;

  // 编码响应数据包，回送给 TcpConnection 对象
  conn->getCodec()->encode(conn->getOutBuffer(), dynamic_cast<AbstractData*>(&reply_pk));

}
```

