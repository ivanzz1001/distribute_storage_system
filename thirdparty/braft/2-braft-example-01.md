# braft exmple(atomic)

参看:

- [braft官网](https://github.com/baidu/braft)

- [brpc官网](https://github.com/apache/brpc)

- [百度云说：解密braft的前世今生](https://www.sohu.com/a/236579130_470007)


## 1. atomic的编译及运行

1) 修改CMakeLists.txt

参看本目录中修改后的CMakeLists.txt文件

2) 执行如下脚本编译
<pre>
# cd example/atomic
# cmake -DCMAKE_PREFIX_PATH=/root/cpp_proj/brpc-compile/gflags/output-inst\;/root/cpp_proj/brpc-compile/glog/output-inst\;/root/cpp_proj/brpc-compile/googletest/output-inst\;/root/cpp_proj/brpc-compile/protobuf/output-inst\;/root/cpp_proj/brpc-compile/leveldb/output-inst\;/root/cpp_proj/brpc-compile/snappy/output-inst\;/root/cpp_proj/brpc-compile/brpc/output-inst\;/root/cpp_proj/brpc-compile/braft/output-inst -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/braft/output-inst -B build

# cmake --build build 
</pre>

3) 运行脚本

首先将build目录下生成的bin文件拷贝到atomic目录：
<pre>
# pwd
/root/cpp_proj/brpc-compile/braft/example/atomic

# cp build/atomic_* ./
# ls
atomic_client  atomic.proto  atomic_server  atomic_test  build  client.cpp  CMakeLists.txt  jepsen_control.sh  run_client.sh  run_server.sh  server.cpp  stop.sh  test.cpp
</pre>

接着修改`run_client.sh`脚本，将log_each_request设置为true:

```
DEFINE_string log_each_request 'true' 'Print log for each request'
```
然后执行如下命令：

<pre>
# ./run_server.sh
# ./run_client.sh 
I1115 00:39:26.984579 10378 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101312 new_value=101312 latency=2854
I1115 00:39:27.989756 10374 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101312 new_value=101313 latency=4101
I1115 00:39:28.994940 10378 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101313 new_value=101314 latency=4484
I1115 00:39:30.001124 10378 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101314 new_value=101315 latency=5142
I1115 00:39:31.006011 10374 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101315 new_value=101316 latency=4052
I1115 00:39:32.010081 10378 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101316 new_value=101317 latency=3332
I1115 00:39:33.015067 10374 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101317 new_value=101318 latency=3875
I1115 00:39:34.024154 10378 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101318 new_value=101319 latency=5105
I1115 00:39:35.031252 10374 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101319 new_value=101320 latency=4555
I1115 00:39:36.038519 10378 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101320 new_value=101321 latency=4787
I1115 00:39:37.045630 10374 4294969344 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:115] Received response from 127.0.1.1:8301:0:0 old_value=101321 new_value=101322 latency=3616
^CI1115 00:39:37.103479 10370     0 /root/cpp_proj/brpc-compile/braft/example/atomic/client.cpp:173] Counter client is going to quit
</pre>

运行起来之后，我们可以看到在当前目录下生成了一个runtime目录：
<pre>
# tree ./runtime
├── 0
│   ├── atomic_server
│   ├── data
│   │   ├── log
│   │   │   └── log_meta
│   │   ├── raft_meta
│   │   │   └── raft_meta
│   │   └── snapshot
│   │       └── snapshot_00000000000000101325
│   │           ├── data
│   │           └── __raft_snapshot_meta
│   └── std.log
├── 1
│   ├── atomic_server
│   ├── data
│   │   ├── log
│   │   │   └── log_meta
│   │   ├── raft_meta
│   │   │   └── raft_meta
│   │   └── snapshot
│   │       └── snapshot_00000000000000101325
│   │           ├── data
│   │           └── __raft_snapshot_meta
│   └── std.log
└── 2
    ├── atomic_server
    ├── data
    │   ├── log
    │   │   └── log_meta
    │   ├── raft_meta
    │   │   └── raft_meta
    │   └── snapshot
    │       └── snapshot_00000000000000101325
    │           ├── data
    │           └── __raft_snapshot_meta
    └── std.log

18 directories, 18 files
</pre>


## 2. atomic源码分析
因为braft库已经帮我们实现了leader选举，append log entry等动作，因此我们要做的就是将自己包装成一个raft node节点，然后实现自己的state machine即可。现在我们来看看Node类(braft/src/braft/raft.h):

```
class Node {
public:
    Node(const GroupId& group_id, const PeerId& peer_id);
    virtual ~Node();

    // get node id
    NodeId node_id();

    // get leader PeerId, for redirect
    PeerId leader_id();

    // Return true if this is the leader of the belonging group
    bool is_leader();

    // Return true if this is the leader, and leader lease is valid. It's always
    // false when |raft_enable_leader_lease == false|.
    // In the following situations, the returned true is unbeleivable:
    //    -  Not all nodes in the raft group set |raft_enable_leader_lease| to true,
    //       and tranfer leader/vote interfaces are used;
    //    -  In the raft group, the value of |election_timeout_ms| in one node is larger
    //       than |election_timeout_ms + max_clock_drift_ms| in another peer.
    bool is_leader_lease_valid();

    // Get leader lease status for more complex checking
    void get_leader_lease_status(LeaderLeaseStatus* status);

    // init node
    int init(const NodeOptions& options);

    // shutdown local replica.
    // done is user defined function, maybe response to client or clean some resource
    // [NOTE] code after apply can't access resource in done
    void shutdown(Closure* done);

    // Block the thread until the node is successfully stopped.
    void join();

    // [Thread-safe and wait-free]
    // apply task to the replicated-state-machine
    //
    // About the ownership:
    // |task.data|: for the performance consideration, we will take away the 
    //              content. If you want keep the content, copy it before call
    //              this function
    // |task.done|: If the data is successfully committed to the raft group. We
    //              will pass the ownership to StateMachine::on_apply.
    //              Otherwise we will specify the error and call it.
    //
    void apply(const Task& task);

    // list peers of this raft group, only leader retruns ok
    // [NOTE] when list_peers concurrency with add_peer/remove_peer, maybe return peers is staled.
    // because add_peer/remove_peer immediately modify configuration in memory
    butil::Status list_peers(std::vector<PeerId>* peers);

    // Add a new peer to the raft group. done->Run() would be invoked after this
    // operation finishes, describing the detailed result.
    void add_peer(const PeerId& peer, Closure* done);

    // Remove the peer from the raft group. done->Run() would be invoked after
    // this operation finishes, describing the detailed result.
    void remove_peer(const PeerId& peer, Closure* done);

    // Change the configuration of the raft group to |new_peers| , done->Run()
    // would be invoked after this operation finishes, describing the detailed
    // result.
    void change_peers(const Configuration& new_peers, Closure* done);

    // Reset the configuration of this node individually, without any repliation
    // to other peers before this node beomes the leader. This function is
    // supposed to be inovoked when the majority of the replication group are
    // dead and you'd like to revive the service in the consideration of
    // availability.
    // Notice that neither consistency nor consensus are guaranteed in this
    // case, BE CAREFULE when dealing with this method.
    butil::Status reset_peers(const Configuration& new_peers);

    // Start a snapshot immediately if possible. done->Run() would be invoked
    // when the snapshot finishes, describing the detailed result.
    void snapshot(Closure* done);

    // user trigger vote
    // reset election_timeout, suggest some peer to become the leader in a
    // higher probability
    butil::Status vote(int election_timeout);

    // Reset the |election_timeout_ms| for the very node, the |max_clock_drift_ms|
    // is also adjusted to keep the sum of |election_timeout_ms| and |the max_clock_drift_ms|
    // unchanged.
    butil::Status reset_election_timeout_ms(int election_timeout_ms);

    // Forcely reset |election_timeout_ms| and |max_clock_drift_ms|. It may break
    // leader lease safety, should be careful.
    // Following are suggestions for you to change |election_timeout_ms| safely.
    // 1. Three steps to safely upgrade |election_timeout_ms| to a larger one:
    //     - Enlarge |max_clock_drift_ms| in all peers to make sure
    //       |old election_timeout_ms + new max_clock_drift_ms| larger than
    //       |new election_timeout_ms + old max_clock_drift_ms|.
    //     - Wait at least |old election_timeout_ms + new max_clock_drift_ms| times to make
    //       sure all previous elections complete.
    //     - Upgrade |election_timeout_ms| to new one, meanwhiles |max_clock_drift_ms|
    //       can set back to the old value.
    // 2. Three steps to safely upgrade |election_timeout_ms| to a smaller one:
    //     - Adjust |election_timeout_ms| and |max_clock_drift_ms| at the same time,
    //       to make the sum of |election_timeout_ms + max_clock_drift_ms| unchanged.
    //     - Wait at least |election_timeout_ms + max_clock_drift_ms| times to make
    //       sure all previous elections complete.
    //     - Upgrade |max_clock_drift_ms| back to the old value.
    void reset_election_timeout_ms(int election_timeout_ms, int max_clock_drift_ms);

    // Try transferring leadership to |peer|.
    // If peer is ANY_PEER, a proper follower will be chosen as the leader for
    // the next term.
    // Returns 0 on success, -1 otherwise.
    int transfer_leadership_to(const PeerId& peer);

    // Read the first committed user log from the given index.
    // Return OK on success and user_log is assigned with the very data. Be awared
    // that the user_log may be not the exact log at the given index, but the
    // first available user log from the given index to last_committed_index.
    // Otherwise, appropriate errors are returned:
    //     - return ELOGDELETED when the log has been deleted;
    //     - return ENOMOREUSERLOG when we can't get a user log even reaching last_committed_index.
    // [NOTE] in consideration of safety, we use last_applied_index instead of last_committed_index 
    // in code implementation.
    butil::Status read_committed_user_log(const int64_t index, UserLog* user_log);

    // Get the internal status of this node, the information is mostly the same as we
    // see from the website.
    void get_status(NodeStatus* status);

    // Make this node enter readonly mode.
    // Readonly mode should only be used to protect the system in some extreme cases.
    // For example, in a storage system, too many write requests flood into the system
    // unexpectly, and the system is in the danger of exhaust capacity. There's not enough
    // time to add new machines, and wait for capacity balance. Once many disks become
    // full, quorum dead happen to raft groups. One choice in this example is readonly
    // mode, to let leader reject new write requests, but still handle reads request,
    // and configuration changes.
    // If a follower become readonly, the leader stop replicate new logs to it. This
    // may cause the data far behind the leader, in the case that the leader is still
    // writable. After the follower exit readonly mode, the leader will resume to
    // replicate missing logs.
    // A leader is readonly, if the node itself is readonly, or writable nodes (nodes that
    // are not marked as readonly) in the group is less than majority. Once a leader become
    // readonly, no new users logs will be acceptted.
    void enter_readonly_mode();

    // Node leave readonly node.
    void leave_readonly_mode();

    // Check if this node is readonly.
    // There are two situations that if a node is readonly:
    //      - This node is marked as readonly, by calling enter_readonly_mode();
    //      - This node is a leader, and the count of writable nodes in the group
    //        is less than the majority.
    bool readonly();

private:
    NodeImpl* _impl;
};
```

从上面我们可以看到Node实例可以apply()任务，可以判断自己是否是leader，可以列出当前raft group的peers，可以通过在init()初始化时传入options来指定状态机。


### 2.1 atomic server端实现

1) **Atomic类的实现**

对于server端的实现我们主要看`Atomic`类：
```
// Implementation of example::Atomic as a braft::StateMachine.
class Atomic : public braft::StateMachine {
public:
    void apply(AtomicOpType type, const google::protobuf::Message* request,
               AtomicResponse* response, google::protobuf::Closure* done);


    // @braft::StateMachine
    void on_apply(braft::Iterator& iter);

    void on_snapshot_save(braft::SnapshotWriter* writer, braft::Closure* done);

    int on_snapshot_load(braft::SnapshotReader* reader);

    void on_leader_start(int64_t term);

    void on_leader_stop(const butil::Status& status);

    void on_shutdown();
    void on_error(const ::braft::Error& e);
    void on_configuration_committed(const ::braft::Configuration& conf);
    void on_stop_following(const ::braft::LeaderChangeContext& ctx);
    void on_start_following(const ::braft::LeaderChangeContext& ctx);

private:
    typedef butil::FlatMap<int64_t, int64_t> ValueMap;

    struct SnapshotClosure {
        std::vector<std::pair<int64_t, int64_t> > values;
        braft::SnapshotWriter* writer;
        braft::Closure* done;
    };

    braft::Node* volatile _node;
    butil::atomic<int64_t> _leader_term;
    ValueMap _value_map;
};
```
Atomic包装了一个`braft::Node`，这样也就使得其可以成为raft group中的一员，我们就可以利用braft node所提供的apply()功能来提交任务，并将成功提交到大部分节点的任务apply到状态机。

由于Atomic继承了braft::StateMachine，因此其本身也是个状态机，在我们初始化指定状态机时指定为`this`即可：
```
// Starts this node
int start() {
    butil::EndPoint addr(butil::my_ip(), FLAGS_port);
    braft::NodeOptions node_options;
    if (node_options.initial_conf.parse_from(FLAGS_conf) != 0) {
        LOG(ERROR) << "Fail to parse configuration `" << FLAGS_conf << '\'';
        return -1;
    }
    node_options.election_timeout_ms = FLAGS_election_timeout_ms;
    node_options.fsm = this;
    node_options.node_owns_fsm = false;
    node_options.snapshot_interval_s = FLAGS_snapshot_interval;
    std::string prefix = "local://" + FLAGS_data_path;
    node_options.log_uri = prefix + "/log";
    node_options.raft_meta_uri = prefix + "/raft_meta";
    node_options.snapshot_uri = prefix + "/snapshot";
    node_options.disable_cli = FLAGS_disable_cli;
    braft::Node* node = new braft::Node(FLAGS_group, braft::PeerId(addr));
    if (node->init(node_options) != 0) {
        LOG(ERROR) << "Fail to init raft node";
        delete node;
        return -1;
    }
    _node = node;
    return 0;
}
```

接下来我们重点看看其中的两个函数: apply()提交任务，on_apply()应用所提交的任务


* apply()提交log entry

```
void apply(AtomicOpType type, const google::protobuf::Message* request,
           AtomicResponse* response, google::protobuf::Closure* done) {
    brpc::ClosureGuard done_guard(done);
    // Serialize request to the replicated write-ahead-log so that all the
    // peers in the group receive this request as well.
    // Notice that _value can't be modified in this routine otherwise it
    // will be inconsistent with others in this group.
    
    // Serialize request to IOBuf
    const int64_t term = _leader_term.load(butil::memory_order_relaxed);
    if (term < 0) {
        return redirect(response);
    }
    butil::IOBuf log;
    log.push_back((uint8_t)type);
    butil::IOBufAsZeroCopyOutputStream wrapper(&log);
    if (!request->SerializeToZeroCopyStream(&wrapper)) {
        LOG(ERROR) << "Fail to serialize request";
        response->set_success(false);
        return;
    }
    // Apply this log as a braft::Task
    braft::Task task;
    task.data = &log;
    // This callback would be iovoked when the task actually excuted or
    // fail
    task.done = new AtomicClosure(this, request, response,
                                  done_guard.release());
    if (FLAGS_check_term) {
        // ABA problem can be avoid if expected_term is set
        task.expected_term = term;
    }
    // Now the task is applied to the group, waiting for the result.
    return _node->apply(task);
}
```

从上面我们可以看到，首先构造了一个braft::Task，接着就调用`_node->apply()`来提交任务。这里我们简单说明一下task.done：当任务提交成功，done这个closesure就会被转到on_apply()中；当任务提交失败，那么done()就会有node自动调用。通过task.done这个闭包，无论任务提交成功或失败，本节点都可以很方便的获取到所提交任务的详细信息，而不需要对log entry进行解码。

- **on_apply()应用log entry**

```
// @braft::StateMachine
void on_apply(braft::Iterator& iter) {
    // A batch of tasks are committed, which must be processed through 
    // |iter|
    for (; iter.valid(); iter.next()) {
        // This guard helps invoke iter.done()->Run() asynchronously to
        // avoid that callback blocks the StateMachine.
        braft::AsyncClosureGuard done_guard(iter.done());

        // Parse data
        butil::IOBuf data = iter.data();
        // Fetch the type of operation from the leading byte.
        uint8_t type = OP_UNKNOWN;
        data.cutn(&type, sizeof(uint8_t));

        AtomicClosure* c = NULL;
        if (iter.done()) {
            c = dynamic_cast<AtomicClosure*>(iter.done());
        }

        const google::protobuf::Message* request = c ? c->request() : NULL;
        AtomicResponse r;
        AtomicResponse* response = c ? c->response() : &r;
        const char* op = NULL;
        // Execute the operation according to type
        switch (type) {
        case OP_GET:
            op = "get";
            get_value(data, request, response);
            break;
        case OP_EXCHANGE:
            op = "exchange";
            exchange(data, request, response);
            break;
        case OP_CAS:
            op = "cas";
            cas(data, request, response);
            break;
        default:
            CHECK(false) << "Unknown type=" << static_cast<int>(type);
            break;
        }

        // The purpose of following logs is to help you understand the way
        // this StateMachine works.
        // Remove these logs in performance-sensitive servers.
        LOG_IF(INFO, FLAGS_log_applied_task) 
                << "Handled operation " << op 
                << " on id=" << response->id()
                << " at log_index=" << iter.index()
                << " success=" << response->success()
                << " old_value=" << response->old_value()
                << " new_value=" << response->new_value();
    }
}
```
从上面我们可以看到，如果是本node提交的task，那么其done直接保存在内存中，我们就可以直接通过done就拿到对应的Request和Response结构； 否则我们就需要解析butil::IOBuf。


2） **AtomicServiceImpl的实现**

由于我们要接收client端的请求，因此我们必须实现一个brpc server及brpc service。我们来看看对应的实现：

```
// Implements example::AtomicService if you are using brpc.
class AtomicServiceImpl : public AtomicService {
public:

    explicit AtomicServiceImpl(Atomic* atomic) : _atomic(atomic) {}
    ~AtomicServiceImpl() {}

    void get(::google::protobuf::RpcController* controller,
             const ::example::GetRequest* request,
             ::example::AtomicResponse* response,
             ::google::protobuf::Closure* done) {
        return _atomic->get(request, response, done);
    }
    void exchange(::google::protobuf::RpcController* controller,
                  const ::example::ExchangeRequest* request,
                  ::example::AtomicResponse* response,
                  ::google::protobuf::Closure* done) {
        return _atomic->exchange(request, response, done);
    }
    void compare_exchange(::google::protobuf::RpcController* controller,
                          const ::example::CompareExchangeRequest* request,
                          ::example::AtomicResponse* response,
                          ::google::protobuf::Closure* done) {
        return _atomic->compare_exchange(request, response, done);
    }

private:
    Atomic* _atomic;
};
```
从这里我们看到AtomicServiceImpl仅仅是将收到的来自client端的请求交给Atomic来处理即可。


## 3. atomic client端实现

对于atomic client端的实现比较简单，就是找到对应raft group的leader，然后向leader的brpc server发送请求即可。我们主要来看一下如何找到leader。首先我们在main()函数中看到如下：

```
int main(int argc, char* argv[]) {
    GFLAGS_NS::ParseCommandLineFlags(&argc, &argv, true);
    butil::AtExitManager exit_manager;

    // Register configuration of target group to RouteTable
    if (braft::rtb::update_configuration(FLAGS_group, FLAGS_conf) != 0) {
        LOG(ERROR) << "Fail to register configuration " << FLAGS_conf
                   << " of group " << FLAGS_group;
        return -1;
    }

    ...
}
```
这里针对某个raft group注册了初始路由表，通过这个初始路由表就可以联系上raft group中的节点，之后就可以找到这个raft group的leader。 我们来看看sender()函数的实现：

```
static void* sender(void* arg) {
    SendArg* sa = (SendArg*)arg;
    int64_t value = 0;
    while (!brpc::IsAskedToQuit()) {
        braft::PeerId leader;
        // Select leader of the target group from RouteTable
        if (braft::rtb::select_leader(FLAGS_group, &leader) != 0) {
            // Leader is unknown in RouteTable. Ask RouteTable to refresh leader
            // by sending RPCs.
            butil::Status st = braft::rtb::refresh_leader(
                        FLAGS_group, FLAGS_timeout_ms);
            if (!st.ok()) {
                // Not sure about the leader, sleep for a while and the ask again.
                LOG(WARNING) << "Fail to refresh_leader : " << st;
                bthread_usleep(FLAGS_timeout_ms * 1000L);
            }
            continue;
        }

        // Now we known who is the leader, construct Stub and then sending
        // rpc
        brpc::Channel channel;
        if (channel.Init(leader.addr, NULL) != 0) {
            LOG(ERROR) << "Fail to init channel to " << leader;
            bthread_usleep(FLAGS_timeout_ms * 1000L);
            continue;
        }
        example::AtomicService_Stub stub(&channel);

        brpc::Controller cntl;
        cntl.set_timeout_ms(FLAGS_timeout_ms);
        example::CompareExchangeRequest request;
        example::AtomicResponse response;
        request.set_id(sa->id);
        request.set_expected_value(value);
        request.set_new_value(value + 1);

        stub.compare_exchange(&cntl, &request, &response, NULL);

        if (cntl.Failed()) {
            LOG(WARNING) << "Fail to send request to " << leader
                         << " : " << cntl.ErrorText();
            // Clear leadership since this RPC failed.
            braft::rtb::update_leader(FLAGS_group, braft::PeerId());
            bthread_usleep(FLAGS_timeout_ms * 1000L);
            continue;
        }

        if (!response.success()) {
            // A redirect response
            if (!response.has_old_value()) {
                LOG(WARNING) << "Fail to send request to " << leader
                             << ", redirecting to "
                             << (response.has_redirect() 
                                    ? response.redirect() : "nowhere");
                // Update route table since we have redirect information
                braft::rtb::update_leader(FLAGS_group, response.redirect());
                continue;
            }
            // old_value unmatches expected value check if this is the initial
            // request
            if (value == 0 || response.old_value() == value + 1) {
                //   ^^^                          ^^^
                // It's initalized value          ^^^
                //                          There was false negative
                value = response.old_value();
            } else {
                CHECK_EQ(value, response.old_value());
                exit(-1);
            }
        } else {
            value = response.new_value();
        }
        g_latency_recorder << cntl.latency_us();
        if (FLAGS_log_each_request) {
            LOG(INFO) << "Received response from " << leader
                      << " old_value=" << response.old_value()
                      << " new_value=" << response.new_value()
                      << " latency=" << cntl.latency_us();
            bthread_usleep(1000L * 1000L);
        }
    }
    return NULL;
}

```

从上面我们可以看到，首先是通过braft::rtb::select_leader()找到了对应raft group的leader节点的地址，接着构造一个brpc::Channel向对应的brpc server发送brpc 请求。








