# brpc example

参看:

- [brpc官网](https://github.com/apache/brpc)

- [brpc server](https://github.com/apache/brpc/blob/master/docs/cn/server.md)

## 1. echo_c++的编译及运行

1) 修改CMakeLists.txt

参看本目录中修改后的CMakeLists.txt文件

>ps: ZLIB库的查找是依赖于cmake安装目录/usr/share/cmake-3.22/Modules/FindZLIB.cmake来进行的

2) 执行如下脚本编译
```
# cd example/echo_c++
# cmake -DCMAKE_PREFIX_PATH=/root/cpp_proj/brpc-compile/gflags/output-inst\;/root/cpp_proj/brpc-compile/googletest/output-inst\;/root/cpp_proj/brpc-compile/protobuf/output-inst\;/root/cpp_proj/brpc-compile/leveldb/output-inst\;/root/cpp_proj/brpc-compile/snappy/output-inst\;/root/cpp_proj/brpc-compile/brpc/output-inst -B build

# cmake --build build 
```

3) 运行脚本

```
# ./echo_server &
# ./echo_client
I1109 21:57:34.477359 461176     0 /root/cpp_proj/brpc-compile/brpc/example/echo_c++/client.cpp:78] Received response from 0.0.0.0:8000 to 127.0.0.1:55746: hello world (attached=) latency=4151us
I1109 21:57:35.479628 461176     0 /root/cpp_proj/brpc-compile/brpc/example/echo_c++/client.cpp:78] Received response from 0.0.0.0:8000 to 127.0.0.1:55746: hello world (attached=) latency=1690us
I1109 21:57:36.480605 461176     0 /root/cpp_proj/brpc-compile/brpc/example/echo_c++/client.cpp:78] Received response from 0.0.0.0:8000 to 127.0.0.1:55746: hello world (attached=) latency=754us
I1109 21:57:37.481621 461176     0 /root/cpp_proj/brpc-compile/brpc/example/echo_c++/client.cpp:78] Received response from 0.0.0.0:8000 to 127.0.0.1:55746: hello world (attached=) latency=806us
```

## 2. echo_c++源代码分析


### 2.1 echo.proto
echo.proto定义了EchoService所实现的方法，下面我们来看：

```
syntax="proto2";
package example;

option cc_generic_services = true;

message EchoRequest {
      required string message = 1;
};

message EchoResponse {
      required string message = 1;
};

service EchoService {
      rpc Echo(EchoRequest) returns (EchoResponse);
};
```

从上面可以看到EchoSerive中有一个`Echo()`方法。使用protoc对该proto文件编译后生成echo.pb.h文件，该文件包含了EchoService服务所对应的`服务端`(server)及`客户端`实现， 下面我们分别来看。


1) EchoService服务的server端实现

```
class EchoService : public ::google::protobuf::Service {
 protected:
  EchoService() = default;

 public:
  using Stub = EchoService_Stub;

  EchoService(const EchoService&) = delete;
  EchoService& operator=(const EchoService&) = delete;
  virtual ~EchoService() = default;

  static const ::google::protobuf::ServiceDescriptor* descriptor();

  virtual void Echo(::google::protobuf::RpcController* controller,
                        const ::example::EchoRequest* request,
                        ::example::EchoResponse* response,
                        ::google::protobuf::Closure* done);

  // implements Service ----------------------------------------------
  const ::google::protobuf::ServiceDescriptor* GetDescriptor() override;

  void CallMethod(const ::google::protobuf::MethodDescriptor* method,
                  ::google::protobuf::RpcController* controller,
                  const ::google::protobuf::Message* request,
                  ::google::protobuf::Message* response,
                  ::google::protobuf::Closure* done) override;

  const ::google::protobuf::Message& GetRequestPrototype(
      const ::google::protobuf::MethodDescriptor* method) const override;

  const ::google::protobuf::Message& GetResponsePrototype(
      const ::google::protobuf::MethodDescriptor* method) const override;
};
```

从上面可以看到，EchoService继承自::google::protobuf::Service，其中有一个virtual方法Echo，提示我们应该在具体的服务实现中实现该方法。

2） EchoSerive的client端实现

```
class EchoService_Stub final : public EchoService {
 public:
  EchoService_Stub(::google::protobuf::RpcChannel* channel);
  EchoService_Stub(::google::protobuf::RpcChannel* channel,
                   ::google::protobuf::Service::ChannelOwnership ownership);

  EchoService_Stub(const EchoService_Stub&) = delete;
  EchoService_Stub& operator=(const EchoService_Stub&) = delete;

  ~EchoService_Stub() override;

  inline ::google::protobuf::RpcChannel* channel() { return channel_; }

  // implements EchoService ------------------------------------------
  void Echo(::google::protobuf::RpcController* controller,
                        const ::example::EchoRequest* request,
                        ::example::EchoResponse* response,
                        ::google::protobuf::Closure* done) override;

 private:
  ::google::protobuf::RpcChannel* channel_;
  bool owns_channel_;
};
```

从上面我们可以看到， client端也有一个Echo()方法，客户端就是用该方法来向服务端发起echo请求的。




### 2.2 echo_server的实现

具体的echo_server实现需要继承自EchoService，下面我们来看：

```
// Your implementation of example::EchoService
// Notice that implementing brpc::Describable grants the ability to put
// additional information in /status.
namespace example {
class EchoServiceImpl : public EchoService {
public:
    EchoServiceImpl() {}
    virtual ~EchoServiceImpl() {}
    virtual void Echo(google::protobuf::RpcController* cntl_base,
                      const EchoRequest* request,
                      EchoResponse* response,
                      google::protobuf::Closure* done) {
        // This object helps you to call done->Run() in RAII style. If you need
        // to process the request asynchronously, pass done_guard.release().
        brpc::ClosureGuard done_guard(done);

        brpc::Controller* cntl =
            static_cast<brpc::Controller*>(cntl_base);

        // optional: set a callback function which is called after response is sent
        // and before cntl/req/res is destructed.
        cntl->set_after_rpc_resp_fn(std::bind(&EchoServiceImpl::CallAfterRpc,
            std::placeholders::_1, std::placeholders::_2, std::placeholders::_3));

        // The purpose of following logs is to help you to understand
        // how clients interact with servers more intuitively. You should 
        // remove these logs in performance-sensitive servers.
        LOG(INFO) << "Received request[log_id=" << cntl->log_id() 
                  << "] from " << cntl->remote_side() 
                  << " to " << cntl->local_side()
                  << ": " << request->message()
                  << " (attached=" << cntl->request_attachment() << ")";

        // Fill response.
        response->set_message(request->message());

        // You can compress the response by setting Controller, but be aware
        // that compression may be costly, evaluate before turning on.
        // cntl->set_response_compress_type(brpc::COMPRESS_TYPE_GZIP);

        if (FLAGS_echo_attachment) {
            // Set attachment which is wired to network directly instead of
            // being serialized into protobuf messages.
            cntl->response_attachment().append(cntl->request_attachment());
        }
    }

    // optional
    static void CallAfterRpc(brpc::Controller* cntl,
                        const google::protobuf::Message* req,
                        const google::protobuf::Message* res) {
        // at this time res is already sent to client, but cntl/req/res is not destructed
        std::string req_str;
        std::string res_str;
        json2pb::ProtoMessageToJson(*req, &req_str, NULL);
        json2pb::ProtoMessageToJson(*res, &res_str, NULL);
        LOG(INFO) << "req:" << req_str
                    << " res:" << res_str;
    }
};
}  // namespace example
```

从上面我们可以看到，服务端就是把收到的request message原模原样的赛到response中去。我们如何来启动这个服务呢？接下来看：

```
int main(int argc, char* argv[]) {
    // Parse gflags. We recommend you to use gflags as well.
    GFLAGS_NS::ParseCommandLineFlags(&argc, &argv, true);

    // Generally you only need one Server.
    brpc::Server server;

    // Instance of your service.
    example::EchoServiceImpl echo_service_impl;

    // Add the service into server. Notice the second parameter, because the
    // service is put on stack, we don't want server to delete it, otherwise
    // use brpc::SERVER_OWNS_SERVICE.
    if (server.AddService(&echo_service_impl, 
                          brpc::SERVER_DOESNT_OWN_SERVICE) != 0) {
        LOG(ERROR) << "Fail to add service";
        return -1;
    }

    butil::EndPoint point;
    if (!FLAGS_listen_addr.empty()) {
        if (butil::str2endpoint(FLAGS_listen_addr.c_str(), &point) < 0) {
            LOG(ERROR) << "Invalid listen address:" << FLAGS_listen_addr;
            return -1;
        }
    } else {
        point = butil::EndPoint(butil::IP_ANY, FLAGS_port);
    }
    // Start the server.
    brpc::ServerOptions options;
    options.idle_timeout_sec = FLAGS_idle_timeout_s;
    if (server.Start(point, &options) != 0) {
        LOG(ERROR) << "Fail to start EchoServer";
        return -1;
    }

    // Wait until Ctrl-C is pressed, then Stop() and Join() the server.
    server.RunUntilAskedToQuit();
    return 0;
}

```

从上面我们可以看到定义了一个brpc::Server，然后向其中添加我们的EchoSerivceImpl， 接着调用Start()来启动服务。

>ps: 一个brpc::Server似乎可以添加多个不同的service，当其收到请求时是如何路由到具体的service的呢？对这一块我们后续来分析


### 2.3 echo_client的实现

```
int main(int argc, char* argv[]) {
    // Parse gflags. We recommend you to use gflags as well.
    GFLAGS_NS::ParseCommandLineFlags(&argc, &argv, true);
    
    // A Channel represents a communication line to a Server. Notice that 
    // Channel is thread-safe and can be shared by all threads in your program.
    brpc::Channel channel;
    
    // Initialize the channel, NULL means using default options.
    brpc::ChannelOptions options;
    options.protocol = FLAGS_protocol;
    options.connection_type = FLAGS_connection_type;
    options.timeout_ms = FLAGS_timeout_ms/*milliseconds*/;
    options.max_retry = FLAGS_max_retry;
    if (channel.Init(FLAGS_server.c_str(), FLAGS_load_balancer.c_str(), &options) != 0) {
        LOG(ERROR) << "Fail to initialize channel";
        return -1;
    }

    // Normally, you should not call a Channel directly, but instead construct
    // a stub Service wrapping it. stub can be shared by all threads as well.
    example::EchoService_Stub stub(&channel);

    // Send a request and wait for the response every 1 second.
    int log_id = 0;
    while (!brpc::IsAskedToQuit()) {
        // We will receive response synchronously, safe to put variables
        // on stack.
        example::EchoRequest request;
        example::EchoResponse response;
        brpc::Controller cntl;

        request.set_message("hello world");

        cntl.set_log_id(log_id ++);  // set by user
        // Set attachment which is wired to network directly instead of 
        // being serialized into protobuf messages.
        cntl.request_attachment().append(FLAGS_attachment);

        // Because `done'(last parameter) is NULL, this function waits until
        // the response comes back or error occurs(including timedout).
        stub.Echo(&cntl, &request, &response, NULL);
        if (!cntl.Failed()) {
            LOG(INFO) << "Received response from " << cntl.remote_side()
                << " to " << cntl.local_side()
                << ": " << response.message() << " (attached="
                << cntl.response_attachment() << ")"
                << " latency=" << cntl.latency_us() << "us";
        } else {
            LOG(WARNING) << cntl.ErrorText();
        }
        usleep(FLAGS_interval_ms * 1000L);
    }

    LOG(INFO) << "EchoClient is going to quit";
    return 0;
}
```

从上面的代码我们可以看到，首先初始化了一个brpc::Channel，接着使用该channel构造了一个EchoService_Stub对象，我们使用该对象就可以发送请求了。


