# brpc server的实现


 我们在前面大体分析了一下example/echo_c++示例，但是对于brpc内部的实现机理仍然不够清楚，本节我们来深入分析一下。

参看： 

- [brpc server](https://github.com/apache/brpc/blob/master/docs/cn/server.md)

- [Brpc 服务端收包源码分析](https://www.jianshu.com/p/6a12105c4725)


## 1. 向brpc::Server中添加service
在example/echo_c++示例的server.cpp中，首先创建了一个brpc::Server实例，接着向其添加了一个service，我们来看这一过程(src/brpc/server.cpp)：

```
int Server::AddService(google::protobuf::Service* service,
                       ServiceOwnership ownership) {
    ServiceOptions options;
    options.ownership = ownership;
    return AddServiceInternal(service, false, options);
}
int Server::AddServiceInternal(google::protobuf::Service* service,
                               bool is_builtin_service,
                               const ServiceOptions& svc_opt) {

	....

	const google::protobuf::ServiceDescriptor* sd = service->GetDescriptor();

	//1: 构建_method_map
	Tabbed* tabbed = dynamic_cast<Tabbed*>(service);
    for (int i = 0; i < sd->method_count(); ++i) {
        const google::protobuf::MethodDescriptor* md = sd->method(i);
        MethodProperty mp;
        mp.is_builtin_service = is_builtin_service;
        mp.own_method_status = true;
        mp.params.is_tabbed = !!tabbed;
        mp.params.allow_default_url = svc_opt.allow_default_url;
        mp.params.allow_http_body_to_pb = svc_opt.allow_http_body_to_pb;
        mp.params.pb_bytes_to_base64 = svc_opt.pb_bytes_to_base64;
        mp.params.pb_single_repeated_to_array = svc_opt.pb_single_repeated_to_array;
        mp.params.enable_progressive_read = svc_opt.enable_progressive_read;
        if (mp.params.enable_progressive_read) {
            _has_progressive_read_method = true;
        }
        mp.service = service;
        mp.method = md;
        mp.status = new MethodStatus;
        _method_map[md->full_name()] = mp;
        if (is_idl_support && sd->name() != sd->full_name()/*has ns*/) {
            MethodProperty mp2 = mp;
            mp2.own_method_status = false;
            // have to map service_name + method_name as well because ubrpc
            // does not send the namespace before service_name.
            std::string full_name_wo_ns;
            full_name_wo_ns.reserve(sd->name().size() + 1 + md->name().size());
            full_name_wo_ns.append(sd->name());
            full_name_wo_ns.push_back('.');
            full_name_wo_ns.append(md->name());
            if (_method_map.seek(full_name_wo_ns) == NULL) {
                _method_map[full_name_wo_ns] = mp2;
            } else {
                LOG(ERROR) << '`' << full_name_wo_ns << "' already exists";
                RemoveMethodsOf(service);
                return -1;
            }
        }
    }


	//2: 构建_fullname_service_map
	_fullname_service_map[sd->full_name()] = ss;

	//3: 构建_service_map
    _service_map[sd->name()] = ss;

}
```

从上面我们可以看到，Server::AddServiceInternal()主要做了如下几件事情：

- 构建_method_map

- 构建_fullname_service_map

- 构建_service_map

这些map在server后续接收request请求，然后查找对应的回调函数时具有重要的作用。

>ps: 这里涉及到google::protobuf::ServiceDescriptor以及google::protobuf::MethodDescriptor等与protobuf反射相关的重要内容，这一块的具体工作原理暂不太熟悉，后续单独再用一篇文章来深入分析。



## 2. brpc::Server的启动

我们来看位于src/brpc/server.cpp中的brpc::Server的启动函数`server.Start()`：


```
int Server::Start(const butil::EndPoint& endpoint, const ServerOptions* opt) {
    return StartInternal(
        endpoint, PortRange(endpoint.port, endpoint.port), opt);
}

int Server::StartInternal(const butil::EndPoint& endpoint,
                          const PortRange& port_range,
                          const ServerOptions *opt) {

	...

	_listen_addr = endpoint;
    for (int port = port_range.min_port; port <= port_range.max_port; ++port) {
        _listen_addr.port = port;
        butil::fd_guard sockfd(tcp_listen(_listen_addr));
        if (sockfd < 0) {
            if (port != port_range.max_port) { // not the last port, try next
                continue;
            }
            if (port_range.min_port != port_range.max_port) {
                LOG(ERROR) << "Fail to listen " << _listen_addr.ip
                           << ":[" << port_range.min_port << '-'
                           << port_range.max_port << ']';
            } else {
                LOG(ERROR) << "Fail to listen " << _listen_addr;
            }
            return -1;
        }
        if (_listen_addr.port == 0) {
            // port=0 makes kernel dynamically select a port from
            // https://en.wikipedia.org/wiki/Ephemeral_port
            _listen_addr.port = get_port_from_fd(sockfd);
            if (_listen_addr.port <= 0) {
                LOG(ERROR) << "Fail to get port from fd=" << sockfd;
                return -1;
            }
        }
        if (_am == NULL) {
            _am = BuildAcceptor();
            if (NULL == _am) {
                LOG(ERROR) << "Fail to build acceptor";
                return -1;
            }
            _am->_use_rdma = _options.use_rdma;
            _am->_bthread_tag = _options.bthread_tag;
        }
        // Set `_status' to RUNNING before accepting connections
        // to prevent requests being rejected as ELOGOFF
        _status = RUNNING;
        time(&_last_start_time);
        GenerateVersionIfNeeded();
        g_running_server_count.fetch_add(1, butil::memory_order_relaxed);

        // Pass ownership of `sockfd' to `_am'
        if (_am->StartAccept(sockfd, _options.idle_timeout_sec,
                             _default_ssl_ctx,
                             _options.force_ssl) != 0) {
            LOG(ERROR) << "Fail to start acceptor";
            return -1;
        }
        sockfd.release();
        break; // stop trying
    }
   
	...
}

```

从上面我们可以看到，首先调用tcp_listen()创建了一个fd，之后调用BuildAcceptor()创建了一个acceptor，然后调用该acceptor的StartAccept()方法来接收客户端的连接。

### 2.1 创建Acceptor


BuildAcceptor()是一个比较重要的函数，其中蕴含着后续在socket fd上接收到数据时如何解析这一重要的功能实现， 下面我们来看(src/brpc/server.cpp):

```
Acceptor* Server::BuildAcceptor() {
    std::set<std::string> whitelist;
    for (butil::StringSplitter sp(_options.enabled_protocols.c_str(), ' ');
         sp; ++sp) {
        std::string protocol(sp.field(), sp.length());
        whitelist.insert(protocol);
    }
    const bool has_whitelist = !whitelist.empty();
    Acceptor* acceptor = new (std::nothrow) Acceptor(_keytable_pool);
    if (NULL == acceptor) {
        LOG(ERROR) << "Fail to new Acceptor";
        return NULL;
    }
    InputMessageHandler handler;
    std::vector<Protocol> protocols;
    ListProtocols(&protocols);
    for (size_t i = 0; i < protocols.size(); ++i) {
        if (protocols[i].process_request == NULL) {
            // The protocol does not support server-side.
            continue;
        }
        if (has_whitelist &&
            !is_http_protocol(protocols[i].name) &&
            !whitelist.erase(protocols[i].name)) {
            // the protocol is not allowed to serve.
            RPC_VLOG << "Skip protocol=" << protocols[i].name;
            continue;
        }
        // `process_request' is required at server side
        handler.parse = protocols[i].parse;
        handler.process = protocols[i].process_request;
        handler.verify = protocols[i].verify;
        handler.arg = this;
        handler.name = protocols[i].name;
        if (acceptor->AddHandler(handler) != 0) {
            LOG(ERROR) << "Fail to add handler into Acceptor("
                       << acceptor << ')';
            delete acceptor;
            return NULL;
        }
    }
    if (!whitelist.empty()) {
        std::ostringstream err;
        err << "ServerOptions.enabled_protocols has unknown protocols=`";
        for (std::set<std::string>::const_iterator it = whitelist.begin();
             it != whitelist.end(); ++it) {
            err << *it << ' ';
        }
        err << '\'';
        delete acceptor;
        LOG(ERROR) << err.str();
        return NULL;
    }
    return acceptor;
}

```

从上面我们看到，首先会调用ListProtocols()来列出当前brpc库所支持的所有协议，接着会构造对应的handler加入到acceptor中。


1） 注册Protocols

上面有ListProtocols()函数，那里面的protocol是如何注册进去的呢？ 我们来看GlobalInitializeOrDieImpl()的实现(src/brpc/global.cpp)：

```
static void GlobalInitializeOrDieImpl() {
    //////////////////////////////////////////////////////////////////
    // Be careful about usages of gflags inside this function which //
    // may be called before main() only seeing gflags with default  //
    // values even if the gflags will be set after main().          //
    //////////////////////////////////////////////////////////////////

    // Ignore SIGPIPE.
    struct sigaction oldact;
    if (sigaction(SIGPIPE, NULL, &oldact) != 0 ||
            (oldact.sa_handler == NULL && oldact.sa_sigaction == NULL)) {
        CHECK(SIG_ERR != signal(SIGPIPE, SIG_IGN));
    }

#if GOOGLE_PROTOBUF_VERSION < 3022000
    // Make GOOGLE_LOG print to comlog device
    SetLogHandler(&BaiduStreamingLogHandler);
#endif

    // Set bthread create span function
    bthread_set_create_span_func(CreateBthreadSpan);

    // Setting the variable here does not work, the profiler probably check
    // the variable before main() for only once.
    // setenv("TCMALLOC_SAMPLE_PARAMETER", "524288", 0);

    // Initialize openssl library
    SSL_library_init();
    // RPC doesn't require openssl.cnf, users can load it by themselves if needed
    SSL_load_error_strings();
    if (SSLThreadInit() != 0 || SSLDHInit() != 0) {
        exit(1);
    }

    // Defined in http_rpc_protocol.cpp
    InitCommonStrings();

    // Leave memory of these extensions to process's clean up.
    g_ext = new(std::nothrow) GlobalExtensions();
    if (NULL == g_ext) {
        exit(1);
    }
    // Naming Services
#ifdef BAIDU_INTERNAL
    NamingServiceExtension()->RegisterOrDie("bns", &g_ext->bns);
#endif
    NamingServiceExtension()->RegisterOrDie("file", &g_ext->fns);
    NamingServiceExtension()->RegisterOrDie("list", &g_ext->lns);
    NamingServiceExtension()->RegisterOrDie("dlist", &g_ext->dlns);
    NamingServiceExtension()->RegisterOrDie("http", &g_ext->dns);
    NamingServiceExtension()->RegisterOrDie("https", &g_ext->dns_with_ssl);
    NamingServiceExtension()->RegisterOrDie("redis", &g_ext->dns);
    NamingServiceExtension()->RegisterOrDie("remotefile", &g_ext->rfns);
    NamingServiceExtension()->RegisterOrDie("consul", &g_ext->cns);
    NamingServiceExtension()->RegisterOrDie("discovery", &g_ext->dcns);
    NamingServiceExtension()->RegisterOrDie("nacos", &g_ext->nns);

    // Load Balancers
    LoadBalancerExtension()->RegisterOrDie("rr", &g_ext->rr_lb);
    LoadBalancerExtension()->RegisterOrDie("wrr", &g_ext->wrr_lb);
    LoadBalancerExtension()->RegisterOrDie("random", &g_ext->randomized_lb);
    LoadBalancerExtension()->RegisterOrDie("wr", &g_ext->wr_lb);
    LoadBalancerExtension()->RegisterOrDie("la", &g_ext->la_lb);
    LoadBalancerExtension()->RegisterOrDie("c_murmurhash", &g_ext->ch_mh_lb);
    LoadBalancerExtension()->RegisterOrDie("c_md5", &g_ext->ch_md5_lb);
    LoadBalancerExtension()->RegisterOrDie("c_ketama", &g_ext->ch_ketama_lb);
    LoadBalancerExtension()->RegisterOrDie("_dynpart", &g_ext->dynpart_lb);

    // Compress Handlers
    const CompressHandler gzip_compress =
        { GzipCompress, GzipDecompress, "gzip" };
    if (RegisterCompressHandler(COMPRESS_TYPE_GZIP, gzip_compress) != 0) {
        exit(1);
    }
    const CompressHandler zlib_compress =
        { ZlibCompress, ZlibDecompress, "zlib" };
    if (RegisterCompressHandler(COMPRESS_TYPE_ZLIB, zlib_compress) != 0) {
        exit(1);
    }
    const CompressHandler snappy_compress =
        { SnappyCompress, SnappyDecompress, "snappy" };
    if (RegisterCompressHandler(COMPRESS_TYPE_SNAPPY, snappy_compress) != 0) {
        exit(1);
    }

    // Protocols
    Protocol baidu_protocol = { ParseRpcMessage,
                                SerializeRequestDefault, PackRpcRequest,
                                ProcessRpcRequest, ProcessRpcResponse,
                                VerifyRpcRequest, NULL, NULL,
                                CONNECTION_TYPE_ALL, "baidu_std" };
    if (RegisterProtocol(PROTOCOL_BAIDU_STD, baidu_protocol) != 0) {
        exit(1);
    }

    Protocol streaming_protocol = { ParseStreamingMessage,
                                    NULL, NULL, ProcessStreamingMessage,
                                    ProcessStreamingMessage,
                                    NULL, NULL, NULL,
                                    CONNECTION_TYPE_SINGLE, "streaming_rpc" };

    if (RegisterProtocol(PROTOCOL_STREAMING_RPC, streaming_protocol) != 0) {
        exit(1);
    }

    Protocol http_protocol = { ParseHttpMessage,
                               SerializeHttpRequest, PackHttpRequest,
                               ProcessHttpRequest, ProcessHttpResponse,
                               VerifyHttpRequest, ParseHttpServerAddress,
                               GetHttpMethodName,
                               CONNECTION_TYPE_POOLED_AND_SHORT,
                               "http" };
    if (RegisterProtocol(PROTOCOL_HTTP, http_protocol) != 0) {
        exit(1);
    }

    Protocol http2_protocol = { ParseH2Message,
                                SerializeHttpRequest, PackH2Request,
                                ProcessHttpRequest, ProcessHttpResponse,
                                VerifyHttpRequest, ParseHttpServerAddress,
                                GetHttpMethodName,
                                CONNECTION_TYPE_SINGLE,
                                "h2" };
    if (RegisterProtocol(PROTOCOL_H2, http2_protocol) != 0) {
        exit(1);
    }

    Protocol hulu_protocol = { ParseHuluMessage,
                               SerializeRequestDefault, PackHuluRequest,
                               ProcessHuluRequest, ProcessHuluResponse,
                               VerifyHuluRequest, NULL, NULL,
                               CONNECTION_TYPE_ALL, "hulu_pbrpc" };
    if (RegisterProtocol(PROTOCOL_HULU_PBRPC, hulu_protocol) != 0) {
        exit(1);
    }

    // Only valid at client side
    Protocol nova_protocol = { ParseNsheadMessage,
                               SerializeNovaRequest, PackNovaRequest,
                               NULL, ProcessNovaResponse,
                               NULL, NULL, NULL,
                               CONNECTION_TYPE_POOLED_AND_SHORT,  "nova_pbrpc" };
    if (RegisterProtocol(PROTOCOL_NOVA_PBRPC, nova_protocol) != 0) {
        exit(1);
    }

    // Only valid at client side
    Protocol public_pbrpc_protocol = { ParseNsheadMessage,
                                       SerializePublicPbrpcRequest,
                                       PackPublicPbrpcRequest,
                                       NULL, ProcessPublicPbrpcResponse,
                                       NULL, NULL, NULL,
                                       // public_pbrpc server implementation
                                       // doesn't support full duplex
                                       CONNECTION_TYPE_POOLED_AND_SHORT,
                                       "public_pbrpc" };
    if (RegisterProtocol(PROTOCOL_PUBLIC_PBRPC, public_pbrpc_protocol) != 0) {
        exit(1);
    }

    Protocol sofa_protocol = { ParseSofaMessage,
                               SerializeRequestDefault, PackSofaRequest,
                               ProcessSofaRequest, ProcessSofaResponse,
                               VerifySofaRequest, NULL, NULL,
                               CONNECTION_TYPE_ALL, "sofa_pbrpc" };
    if (RegisterProtocol(PROTOCOL_SOFA_PBRPC, sofa_protocol) != 0) {
        exit(1);
    }

    // Only valid at server side. We generalize all the protocols that
    // prefixes with nshead as `nshead_protocol' and specify the content
    // parsing after nshead by ServerOptions.nshead_service.
    Protocol nshead_protocol = { ParseNsheadMessage,
                                 SerializeNsheadRequest, PackNsheadRequest,
                                 ProcessNsheadRequest, ProcessNsheadResponse,
                                 VerifyNsheadRequest, NULL, NULL,
                                 CONNECTION_TYPE_POOLED_AND_SHORT, "nshead" };
    if (RegisterProtocol(PROTOCOL_NSHEAD, nshead_protocol) != 0) {
        exit(1);
    }

    Protocol mc_binary_protocol = { ParseMemcacheMessage,
                                    SerializeMemcacheRequest,
                                    PackMemcacheRequest,
                                    NULL, ProcessMemcacheResponse,
                                    NULL, NULL, GetMemcacheMethodName,
                                    CONNECTION_TYPE_ALL, "memcache" };
    if (RegisterProtocol(PROTOCOL_MEMCACHE, mc_binary_protocol) != 0) {
        exit(1);
    }

    Protocol redis_protocol = { ParseRedisMessage,
                                SerializeRedisRequest,
                                PackRedisRequest,
                                ProcessRedisRequest, ProcessRedisResponse,
                                NULL, NULL, GetRedisMethodName,
                                CONNECTION_TYPE_ALL, "redis" };
    if (RegisterProtocol(PROTOCOL_REDIS, redis_protocol) != 0) {
        exit(1);
    }

    Protocol mongo_protocol = { ParseMongoMessage,
                                NULL, NULL,
                                ProcessMongoRequest, NULL,
                                NULL, NULL, NULL,
                                CONNECTION_TYPE_POOLED, "mongo" };
    if (RegisterProtocol(PROTOCOL_MONGO, mongo_protocol) != 0) {
        exit(1);
    }

// Use Macro is more straight forward than weak link technology(becasue of static link issue)
#ifdef ENABLE_THRIFT_FRAMED_PROTOCOL
    Protocol thrift_binary_protocol = {
        policy::ParseThriftMessage,
        policy::SerializeThriftRequest, policy::PackThriftRequest,
        policy::ProcessThriftRequest, policy::ProcessThriftResponse,
        policy::VerifyThriftRequest, NULL, NULL,
        CONNECTION_TYPE_POOLED_AND_SHORT, "thrift" };
    if (RegisterProtocol(PROTOCOL_THRIFT, thrift_binary_protocol) != 0) {
        exit(1);
    }
#endif

    // Only valid at client side
    Protocol ubrpc_compack_protocol = {
        ParseNsheadMessage,
        SerializeUbrpcCompackRequest, PackUbrpcRequest,
        NULL, ProcessUbrpcResponse,
        NULL, NULL, NULL,
        CONNECTION_TYPE_POOLED_AND_SHORT,  "ubrpc_compack" };
    if (RegisterProtocol(PROTOCOL_UBRPC_COMPACK, ubrpc_compack_protocol) != 0) {
        exit(1);
    }
    Protocol ubrpc_mcpack2_protocol = {
        ParseNsheadMessage,
        SerializeUbrpcMcpack2Request, PackUbrpcRequest,
        NULL, ProcessUbrpcResponse,
        NULL, NULL, NULL,
        CONNECTION_TYPE_POOLED_AND_SHORT,  "ubrpc_mcpack2" };
    if (RegisterProtocol(PROTOCOL_UBRPC_MCPACK2, ubrpc_mcpack2_protocol) != 0) {
        exit(1);
    }

    // Only valid at client side
    Protocol nshead_mcpack_protocol = {
        ParseNsheadMessage,
        SerializeNsheadMcpackRequest, PackNsheadMcpackRequest,
        NULL, ProcessNsheadMcpackResponse,
        NULL, NULL, NULL,
        CONNECTION_TYPE_POOLED_AND_SHORT,  "nshead_mcpack" };
    if (RegisterProtocol(PROTOCOL_NSHEAD_MCPACK, nshead_mcpack_protocol) != 0) {
        exit(1);
    }

    Protocol rtmp_protocol = {
        ParseRtmpMessage,
        SerializeRtmpRequest, PackRtmpRequest,
        ProcessRtmpMessage, ProcessRtmpMessage,
        NULL, NULL, NULL,
        (ConnectionType)(CONNECTION_TYPE_SINGLE|CONNECTION_TYPE_SHORT),
        "rtmp" };
    if (RegisterProtocol(PROTOCOL_RTMP, rtmp_protocol) != 0) {
        exit(1);
    }

    Protocol esp_protocol = {
        ParseEspMessage,
        SerializeEspRequest, PackEspRequest,
        NULL, ProcessEspResponse,
        NULL, NULL, NULL,
        CONNECTION_TYPE_POOLED_AND_SHORT, "esp"};
    if (RegisterProtocol(PROTOCOL_ESP, esp_protocol) != 0) {
        exit(1);
    }

    std::vector<Protocol> protocols;
    ListProtocols(&protocols);
    for (size_t i = 0; i < protocols.size(); ++i) {
        if (protocols[i].process_response) {
            InputMessageHandler handler;
            // `process_response' is required at client side
            handler.parse = protocols[i].parse;
            handler.process = protocols[i].process_response;
            // No need to verify at client side
            handler.verify = NULL;
            handler.arg = NULL;
            handler.name = protocols[i].name;
            if (get_or_new_client_side_messenger()->AddHandler(handler) != 0) {
                exit(1);
            }
        }
    }

    // Concurrency Limiters
    ConcurrencyLimiterExtension()->RegisterOrDie("auto", &g_ext->auto_cl);
    ConcurrencyLimiterExtension()->RegisterOrDie("constant", &g_ext->constant_cl);
    ConcurrencyLimiterExtension()->RegisterOrDie("timeout", &g_ext->timeout_cl);

    if (FLAGS_usercode_in_pthread) {
        // Optional. If channel/server are initialized before main(), this
        // flag may be false at here even if it will be set to true after
        // main(). In which case, the usercode pool will not be initialized
        // until the pool is used.
        InitUserCodeBackupPoolOnceOrDie();
    }

    // We never join GlobalUpdate, let it quit with the process.
    bthread_t th;
    CHECK(bthread_start_background(&th, NULL, GlobalUpdate, NULL) == 0)
        << "Fail to start GlobalUpdate";
}

```

我们看到会通过调用RegisterProtocol()注册了特别多的协议：

- PROTOCOL_BAIDU_STD: 参看[baidu-std协议描述](https://github.com/apache/brpc/blob/master/docs/cn/baidu_std.md)

- PROTOCOL_STREAMING_RPC

- PROTOCOL_HTTP

- PROTOCOL_H2: 即h2/gRPC协议

- PROTOCOL_HULU_PBRPC

- PROTOCOL_NOVA_PBRPC

- PROTOCOL_PUBLIC_PBRPC

- PROTOCOL_SOFA_PBRPC

- PROTOCOL_NSHEAD

- PROTOCOL_MEMCACHE

- PROTOCOL_REDIS

- PROTOCOL_MONGO

还有很多，这里我们就不再一一列出。brpc server端支持很多协议，在我们的示例exxample/echo_c++中client与server通信使用的就是`PROTOCOL_BAIDU_STD`协议：

```
DEFINE_string(protocol, "baidu_std", "Protocol type. Defined in src/brpc/options.proto");
```

### 2.2 StartAccept()接收客户端连接

StartAccept()用于接收客户端的连接，我们来看其代码实现(src/brpc/acceptor.cpp)：

```
int Acceptor::StartAccept(int listened_fd, int idle_timeout_sec,
                          const std::shared_ptr<SocketSSLContext>& ssl_ctx,
                          bool force_ssl) {
    if (listened_fd < 0) {
        LOG(FATAL) << "Invalid listened_fd=" << listened_fd;
        return -1;
    }

    if (!ssl_ctx && force_ssl) {
        LOG(ERROR) << "Fail to force SSL for all connections "
                      " because ssl_ctx is NULL";
        return -1;
    }
    
    BAIDU_SCOPED_LOCK(_map_mutex);
    if (_status == UNINITIALIZED) {
        if (Initialize() != 0) {
            LOG(FATAL) << "Fail to initialize Acceptor";
            return -1;
        }
        _status = READY;
    }
    if (_status != READY) {
        LOG(FATAL) << "Acceptor hasn't stopped yet: status=" << status();
        return -1;
    }
    if (idle_timeout_sec > 0) {
        bthread_attr_t tmp = BTHREAD_ATTR_NORMAL;
        tmp.tag = _bthread_tag;
        if (bthread_start_background(&_close_idle_tid, &tmp, CloseIdleConnections, this) != 0) {
            LOG(FATAL) << "Fail to start bthread";
            return -1;
        }
    }
    _idle_timeout_sec = idle_timeout_sec;
    _force_ssl = force_ssl;
    _ssl_ctx = ssl_ctx;
    
    // Creation of _acception_id is inside lock so that OnNewConnections
    // (which may run immediately) should see sane fields set below.
    SocketOptions options;
    options.fd = listened_fd;
    options.user = this;
    options.bthread_tag = _bthread_tag;
    options.on_edge_triggered_events = OnNewConnections;
    if (Socket::Create(options, &_acception_id) != 0) {
        // Close-idle-socket thread will be stopped inside destructor
        LOG(FATAL) << "Fail to create _acception_id";
        return -1;
    }
    
    _listened_fd = listened_fd;
    _status = RUNNING;
    return 0;
}

```

从上面我们可以看到通过options设置了当前监听socket的边沿触发方式，当收到新的客户端连接时就会回调OnNewConnections()函数。


1） epoll_wait()等待监听socket上的连接事件

在 Acceptor::StartAccept()中调用Socket::Create()创建了一个VersionedRefWithId<Socket>对象，当该对象创建完成之后会回调Socket::OnCreated()函数(src/brpc/socket.cpp):

```
int Socket::OnCreated(const SocketOptions& options) {
	if (_io_event.Init((void*)id()) != 0) {
        LOG(ERROR) << "Fail to init IOEvent";
        SetFailed(ENOMEM, "%s", "Fail to init IOEvent");
        return -1;
    }


	...

	if (ResetFileDescriptor(options.fd) != 0) {
        const int saved_errno = errno;
        PLOG(ERROR) << "Fail to ResetFileDescriptor";
        SetFailed(saved_errno, "Fail to ResetFileDescriptor: %s",
                     berror(saved_errno));
        return -1;
    }

}


int Socket::ResetFileDescriptor(int fd) {
	...

    if (_on_edge_triggered_events) {
        if (_io_event.AddConsumer(fd) != 0) {
            PLOG(ERROR) << "Fail to add SocketId=" << id() 
                        << " into EventDispatcher";
            _fd.store(-1, butil::memory_order_release);
            return -1;
        }
    }

	...
}

```

在ResetFileDescriptor()中调用AddConsumer()将fd加到epoll中，我们来看代码(src/brpc/event_dispatcher.h):

```
// See comments of `EventDispatcher::AddConsumer'.
int AddConsumer(int fd) {
	if (!_init) {
		LOG(ERROR) << "IOEvent has not been initialized";
		return -1;
	}
	return GetGlobalEventDispatcher(fd, _bthread_tag)
		.AddConsumer(_event_data_id, fd);
}

EventDispatcher& GetGlobalEventDispatcher(int fd, bthread_tag_t tag) {
    pthread_once(&g_edisp_once, InitializeGlobalDispatchers);
    if (FLAGS_task_group_ntags == 1 && FLAGS_event_dispatcher_num == 1) {
        return g_edisp[0];
    }
    int index = butil::fmix32(fd) % FLAGS_event_dispatcher_num;
    return g_edisp[tag * FLAGS_event_dispatcher_num + index];
}


void InitializeGlobalDispatchers() {
    g_edisp = new EventDispatcher[FLAGS_task_group_ntags * FLAGS_event_dispatcher_num];
    for (int i = 0; i < FLAGS_task_group_ntags; ++i) {
        for (int j = 0; j < FLAGS_event_dispatcher_num; ++j) {
            bthread_attr_t attr =
                FLAGS_usercode_in_pthread ? BTHREAD_ATTR_PTHREAD : BTHREAD_ATTR_NORMAL;
            attr.tag = (BTHREAD_TAG_DEFAULT + i) % FLAGS_task_group_ntags;
            CHECK_EQ(0, g_edisp[i * FLAGS_event_dispatcher_num + j].Start(&attr));
        }
    }
    // This atexit is will be run before g_task_control.stop() because above
    // Start() initializes g_task_control by creating bthread (to run epoll/kqueue).
    CHECK_EQ(0, atexit(StopAndJoinGlobalDispatchers));
}
```

从上面我们可以看到，监听socket被加入到了一个全局的EventDispatcher中。该全局的Dispatcher是调用InitializeGlobalDispatchers()来完成构建及初始化的，其会调用EventDispatcher::Start()方法来等待对应fd上的连接事件。


2) OnNewConnections()接受新的客户端连接

当监听socket接收到客户端的连接时，就会回调OnNewConnections()函数(src/brpc/acceptor.cpp):

```
void Acceptor::OnNewConnections(Socket* acception) {
    int progress = Socket::PROGRESS_INIT;
    do {
        OnNewConnectionsUntilEAGAIN(acception);
        if (acception->Failed()) {
            return;
        }
    } while (acception->MoreReadEvents(&progress));
}

void Acceptor::OnNewConnectionsUntilEAGAIN(Socket* acception) {
    while (1) {
        struct sockaddr_storage in_addr;
        bzero(&in_addr, sizeof(in_addr));
        socklen_t in_len = sizeof(in_addr);
        butil::fd_guard in_fd(accept(acception->fd(), (sockaddr*)&in_addr, &in_len));
        if (in_fd < 0) {
            // no EINTR because listened fd is non-blocking.
            if (errno == EAGAIN) {
                return;
            }
            // Do NOT return -1 when `accept' failed, otherwise `_listened_fd'
            // will be closed. Continue to consume all the events until EAGAIN
            // instead.
            // If the accept was failed, the error may repeat constantly, 
            // limit frequency of logging.
            PLOG_EVERY_SECOND(ERROR)
                << "Fail to accept from listened_fd=" << acception->fd();
            continue;
        }

        Acceptor* am = dynamic_cast<Acceptor*>(acception->user());
        if (NULL == am) {
            LOG(FATAL) << "Impossible! acception->user() MUST be Acceptor";
            acception->SetFailed(EINVAL, "Impossible! acception->user() MUST be Acceptor");
            return;
        }
        
        SocketId socket_id;
        SocketOptions options;
        options.keytable_pool = am->_keytable_pool;
        options.fd = in_fd;
        butil::sockaddr2endpoint(&in_addr, in_len, &options.remote_side);
        options.user = acception->user();
        options.force_ssl = am->_force_ssl;
        options.initial_ssl_ctx = am->_ssl_ctx;
#if BRPC_WITH_RDMA
        if (am->_use_rdma) {
            options.on_edge_triggered_events = rdma::RdmaEndpoint::OnNewDataFromTcp;
        } else {
#else
        {
#endif
            options.on_edge_triggered_events = InputMessenger::OnNewMessages;
        }
        options.use_rdma = am->_use_rdma;
        options.bthread_tag = am->_bthread_tag;
        if (Socket::Create(options, &socket_id) != 0) {
            LOG(ERROR) << "Fail to create Socket";
            continue;
        }
        in_fd.release(); // transfer ownership to socket_id

        // There's a funny race condition here. After Socket::Create, messages
        // from the socket are already handled and a RPC is possibly done
        // before the socket is added into _socket_map below. This is found in
        // ChannelTest.skip_parallel in test/brpc_channel_unittest.cpp (running
        // on machines with few cores) where the _messenger.ConnectionCount()
        // may surprisingly be 0 even if the RPC is already done.

        SocketUniquePtr sock;
        if (Socket::AddressFailedAsWell(socket_id, &sock) >= 0) {
            bool is_running = true;
            {
                BAIDU_SCOPED_LOCK(am->_map_mutex);
                is_running = (am->status() == RUNNING);
                // Always add this socket into `_socket_map' whether it
                // has been `SetFailed' or not, whether `Acceptor' is
                // running or not. Otherwise, `Acceptor::BeforeRecycle'
                // may be called (inside Socket::BeforeRecycled) after `Acceptor'
                // has been destroyed
                am->_socket_map.insert(socket_id, ConnectStatistics());
            }
            if (!is_running) {
                LOG(WARNING) << "Acceptor on fd=" << acception->fd()
                    << " has been stopped, discard newly created " << *sock;
                sock->SetFailed(ELOGOFF, "Acceptor on fd=%d has been stopped, "
                        "discard newly created %s", acception->fd(),
                        sock->description().c_str());
                return;
            }
        } // else: The socket has already been destroyed, Don't add its id
          // into _socket_map
    }
}
```
从上面我们可以看到，当有新的连接到来时就会调用accept()来接受连接，然后调用Socket::Create()创建一个brpc::Socket对象来表示该连接，并为其指定对应的回调函数InputMessenger::OnNewMessages()。

## 3. 读取客户端Request数据

通过上面的分析我们知道是调用InputMessenger::OnNewMessages()来读取客户端连接上的数据的(src/brpc/input_messenger.cpp)：

```
void InputMessenger::OnNewMessages(Socket* m) {
    // Notes:
    // - If the socket has only one message, the message will be parsed and
    //   processed in this bthread. nova-pbrpc and http works in this way.
    // - If the socket has several messages, all messages will be parsed (
    //   meaning cutting from butil::IOBuf. serializing from protobuf is part of
    //   "process") in this bthread. All messages except the last one will be
    //   processed in separate bthreads. To minimize the overhead, scheduling
    //   is batched(notice the BTHREAD_NOSIGNAL and bthread_flush).
    // - Verify will always be called in this bthread at most once and before
    //   any process.
    InputMessenger* messenger = static_cast<InputMessenger*>(m->user());
    int progress = Socket::PROGRESS_INIT;

    // Notice that all *return* no matter successful or not will run last
    // message, even if the socket is about to be closed. This should be
    // OK in most cases.
    InputMessageClosure last_msg;
    bool read_eof = false;
    while (!read_eof) {
        const int64_t received_us = butil::cpuwide_time_us();
        const int64_t base_realtime = butil::gettimeofday_us() - received_us;

        // Calculate bytes to be read.
        size_t once_read = m->_avg_msg_size * 16;
        if (once_read < MIN_ONCE_READ) {
            once_read = MIN_ONCE_READ;
        } else if (once_read > MAX_ONCE_READ) {
            once_read = MAX_ONCE_READ;
        }

        // Read.
        const ssize_t nr = m->DoRead(once_read);
        if (nr <= 0) {
            if (0 == nr) {
                // Set `read_eof' flag and proceed to feed EOF into `Protocol'
                // (implied by m->_read_buf.empty), which may produce a new
                // `InputMessageBase' under some protocols such as HTTP
                LOG_IF(WARNING, FLAGS_log_connection_close) << *m << " was closed by remote side";
                read_eof = true;                
            } else if (errno != EAGAIN) {
                if (errno == EINTR) {
                    continue;  // just retry
                }
                const int saved_errno = errno;
                PLOG(WARNING) << "Fail to read from " << *m;
                m->SetFailed(saved_errno, "Fail to read from %s: %s",
                             m->description().c_str(), berror(saved_errno));
                return;
            } else if (!m->MoreReadEvents(&progress)) {
                return;
            } else { // new events during processing
                continue;
            }
        }

        if (m->_rdma_state == Socket::RDMA_OFF && messenger->ProcessNewMessage(
                    m, nr, read_eof, received_us, base_realtime, last_msg) < 0) {
            return;
        } 
    }

    if (read_eof) {
        m->SetEOF();
    }
}

```
在InputMessenger::OnNewMessages()中主要做两件事情：

- 调用Socket::DoRead()从socket中读取数据

- 调用InputMessenger::ProcessNewMessage()来处理message


1) Socket::DoRead()读取socket数据

我们来看是如何读取socket中的数据的(src/brpc/socket.cpp)：

```
ssize_t Socket::DoRead(size_t size_hint) {
    if (ssl_state() == SSL_UNKNOWN) {
        int error_code = 0;
        _ssl_state = DetectSSLState(fd(), &error_code);
        switch (ssl_state()) {
        case SSL_UNKNOWN:
            if (error_code == 0) {  // EOF
                return 0;
            } else {
                errno = error_code;
                return -1;
            }

        case SSL_CONNECTING:
            if (SSLHandshake(fd(), true) != 0) {
                errno = EINVAL;
                return -1;
            }
            break;

        case SSL_CONNECTED:
            CHECK(false) << "Impossible to reach here";
            break;
            
        case SSL_OFF:
            break;
        }
    }
    // _ssl_state has been set
    if (ssl_state() == SSL_OFF) {
        if (_force_ssl) {
            errno = ESSL;
            return -1;
        }
        CHECK(_rdma_state == RDMA_OFF);
        return _read_buf.append_from_file_descriptor(fd(), size_hint);
    }

    CHECK_EQ(SSL_CONNECTED, ssl_state());
    int ssl_error = 0;
    ssize_t nr = 0;
    {
        BAIDU_SCOPED_LOCK(_ssl_session_mutex);
        nr = _read_buf.append_from_SSL_channel(_ssl_session, &ssl_error, size_hint);
    }
    switch (ssl_error) {
    case SSL_ERROR_NONE:  // `nr' > 0
        break;
            
    case SSL_ERROR_WANT_READ:
        // Regard this error as EAGAIN
        errno = EAGAIN;
        break;
            
    case SSL_ERROR_WANT_WRITE:
        // Disable renegotiation
        errno = EPROTO;
        return -1;

    default: {
        const unsigned long e = ERR_get_error();
        if (nr == 0) {
            // Socket EOF or SSL session EOF
        } else if (e != 0) {
            LOG(WARNING) << "Fail to read from ssl_fd=" << fd()
                         << ": " << SSLError(e);
            errno = ESSL;
        } else {
            // System error with corresponding errno set.
            bool is_fatal_error = (ssl_error != SSL_ERROR_ZERO_RETURN &&
                                   ssl_error != SSL_ERROR_SYSCALL) ||
                                   BIO_fd_non_fatal_error(errno) != 0 ||
                                  nr < 0;
            PLOG_IF(WARNING, is_fatal_error) << "Fail to read from ssl_fd=" << fd();
        }
        break;
    }
    }
    return nr;
}
```
这里我们看非SSL的socket连接，它是调用_read_buf.append_from_file_descriptor()将fd上的数据不断的读取到`_read_buf`中的。

2) InputMessenger::ProcessNewMessage()处理message消息

从socket中读取到的数据会被源源不断的存入`_read_buf`中，然后调用ProcessNewMessage()来进行进一步的处理(src/brpc/input_messenger.cpp)：

```
int InputMessenger::ProcessNewMessage(
        Socket* m, ssize_t bytes, bool read_eof,
        const uint64_t received_us, const uint64_t base_realtime,
        InputMessageClosure& last_msg) {

	...

	while (1) {
        size_t index = 8888;
        ParseResult pr = CutInputMessage(m, &index, read_eof);

		
		...
		msg->_process = _handlers[index].process;
        msg->_arg = _handlers[index].arg;


		....

		if (!m->is_read_progressive()) {
            // Transfer ownership to last_msg
            last_msg.reset(msg.release());
        } else {
            QueueMessage(msg.release(), &num_bthread_created,
                                m->_keytable_pool);
            bthread_flush();
            num_bthread_created = 0;
        }

		...
	}
}
```

在ProcessNewMessage()中首先会调用CutInputMessage()来尝试读取出一个完整的message(ps: 如果当前不完整则证明数据可能还没有读取完毕）， 接着为该message指定process()函数， 之后调用QueueMessage()将收到的message放入队列。

- CutInputMessage()从_read_buf中cut出一个完整的message

我们来看如何从`_read_buf`中cut出一个完整的message(src/brpc/input_messenger.cpp):

```
ParseResult InputMessenger::CutInputMessage(
        Socket* m, size_t* index, bool read_eof) {

	...

	for (int i = 0; i <= max_index; ++i) {
        if (i == preferred || _handlers[i].parse == NULL) {
            // Don't try preferred handler(already tried) or invalid handler
            continue;
        }
        ParseResult result = _handlers[i].parse(&m->_read_buf, m, read_eof, _handlers[i].arg);
        if (result.is_ok() ||
            result.error() == PARSE_ERROR_NOT_ENOUGH_DATA) {
            m->set_preferred_index(i);
            *index = i;
            return result;
        } else if (result.error() != PARSE_ERROR_TRY_OTHERS) {
            // Critical error, return directly.
            LOG_IF(ERROR, result.error() == PARSE_ERROR_TOO_BIG_DATA)
                << "A message from " << m->remote_side()
                << "(protocol=" << _handlers[i].name
                << ") is bigger than " << FLAGS_max_body_size
                << " bytes, the connection will be closed."
                " Set max_body_size to allow bigger messages";
            return result;
        }
        // Clear context before trying next protocol which definitely has
        // an incompatible context with the current one.
        if (m->parsing_context()) {
            m->reset_parsing_context(NULL);
        }
        // Try other protocols.
    }

	...

}
```

从上面我们看到，其会遍历所有的`_handlers`，然后尝试用其绑定的parse()方法来解析`_read_buf`中的数据，如果解析成功，则证明收到了一个完整的message，并且将对应的`_handlers`下标也通过参数一并返回。

那这些handlers是如何添加进InputMessenger中的呢？ 我们搜索AddHandler()函数发现在前面介绍的GlobalInitializeOrDieImpl()中会将注册的所有协议所对应的handler都添加到一个全局的messenger中：

```
static void GlobalInitializeOrDieImpl() {

	std::vector<Protocol> protocols;
    ListProtocols(&protocols);
    for (size_t i = 0; i < protocols.size(); ++i) {
        if (protocols[i].process_response) {
            InputMessageHandler handler;
            // `process_response' is required at client side
            handler.parse = protocols[i].parse;
            handler.process = protocols[i].process_response;
            // No need to verify at client side
            handler.verify = NULL;
            handler.arg = NULL;
            handler.name = protocols[i].name;
            if (get_or_new_client_side_messenger()->AddHandler(handler) != 0) {
                exit(1);
            }
        }
    }

}

InputMessenger* get_or_new_client_side_messenger() {
    pthread_once(&g_messenger_init, InitClientSideMessenger);
    return g_messenger;
}
```

>ps: 看上面的实现，一个brpc::Server可以在一个port上支持多种不同的协议，其可以通过遍历所有protocols的parse()函数来完成解析，但在实际使用过程中我们通常只在一个port上实现一种协议。

- QueueMessage()处理收到的message

我们单独起一节来讲述。


## 4. 处理客户端message

在上面我们讲到通过CutInputMessage()从`_read_buf`中cut出一个完整的message后，就会调用QueueMessage()来进行处理(src/brpc/input_messenger.cpp)：

```
static void QueueMessage(InputMessageBase* to_run_msg,
                         int* num_bthread_created,
                         bthread_keytable_pool_t* keytable_pool) {
    if (!to_run_msg) {
        return;
    }
    // Create bthread for last_msg. The bthread is not scheduled
    // until bthread_flush() is called (in the worse case).
                
    // TODO(gejun): Join threads.
    bthread_t th;
    bthread_attr_t tmp = (FLAGS_usercode_in_pthread ?
                          BTHREAD_ATTR_PTHREAD :
                          BTHREAD_ATTR_NORMAL) | BTHREAD_NOSIGNAL;
    tmp.keytable_pool = keytable_pool;
    tmp.tag = bthread_self_tag();
    if (!FLAGS_usercode_in_coroutine && bthread_start_background(
            &th, &tmp, ProcessInputMessage, to_run_msg) == 0) {
        ++*num_bthread_created;
    } else {
        ProcessInputMessage(to_run_msg);
    }
}

void* ProcessInputMessage(void* void_arg) {
    InputMessageBase* msg = static_cast<InputMessageBase*>(void_arg);
    msg->_process(msg);
    return NULL;
}
```

从上面我们可以看到实际是通过调用msg->_process()来进行处理的。我们以example/echo_c++实例使用的`baidu_std`协议为例，看看其对应的_process()方法(src/brpc/policy/baidu_rpc_protocol.cpp)：


```
void ProcessRpcRequest(InputMessageBase* msg_base) {
   
	...

    RpcMeta meta;
    if (!ParsePbFromIOBuf(&meta, msg->meta)) {
        LOG(WARNING) << "Fail to parse RpcMeta from " << *socket;
        socket->SetFailed(EREQUEST, "Fail to parse RpcMeta from %s",
                          socket->description().c_str());
        return;
    }
    const RpcRequestMeta &request_meta = meta.request();


	RpcPBMessages* messages = NULL;


	do {
		if (NULL != server->options().baidu_master_service) {

			...

		}else{
			 const Server::MethodProperty* mp =
                server_accessor.FindMethodPropertyByFullName(
                    svc_name, request_meta.method_name());
           
			...
			svc = mp->service;


			messages = server->options().rpc_pb_message_factory->Get(*svc, *method);
            if (!ParseFromCompressedData(req_buf, messages->Request(), req_cmp_type)) {
                cntl->SetFailed(EREQUEST, "Fail to parse request message, "
                                          "CompressType=%s, request_size=%d",
                                CompressTypeToCStr(req_cmp_type), req_size);
                server->options().rpc_pb_message_factory->Return(messages);
                break;
            }
            req_buf.clear();
		}

		...

		if (!FLAGS_usercode_in_pthread) {
            return svc->CallMethod(method, cntl.release(), 
                                   messages->Request(),
                                   messages->Response(), done);
        }
        if (BeginRunningUserCode()) {
            svc->CallMethod(method, cntl.release(), 
                            messages->Request(),
                            messages->Response(), done);
            return EndRunningUserCodeInPlace();
        } else {
            return EndRunningCallMethodInPool(
                svc, method, cntl.release(),
                messages->Request(),
                messages->Response(), done);
        }


	}while(false);
}
```

从上面我们可以看到是通过调用FindMethodPropertyByFullName()查找到具体的处理方法所对应的service，然后再调用ParseFromCompressedData()来将req_buf中的数据解析成为message， 最后调用service句柄的CallMethod()来完成具体的方法调用的。




