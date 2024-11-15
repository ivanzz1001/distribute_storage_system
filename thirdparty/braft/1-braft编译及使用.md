# braft编译及使用

参看:

- [braft官网](https://github.com/baidu/braft)

- [brpc官网](https://github.com/apache/brpc)


## 1. 下载及编译

braft主要依赖于brpc, 关于brpc的编译我们已经讲解过。这里我们也同样在brpc-compile目录下来编译braft。

braft的github官方地址：https://github.com/baidu/braft


1） 下载braft源码

```
# git clone https://github.com/baidu/braft.git
# cd braft
#git tag
v1.0.0
v1.0.1
v1.0.2
v1.1.0
v1.1.1
v1.1.2
```

这里我们发现braft变动并不算频繁，因此直接使用master版本来编译。笔者在编译braft master分支时最后一个commit为：

```
git log
commit ab0017f0b98d429138d83a04d3ed351197d671a9 (HEAD -> master, origin/master, origin/HEAD)
Merge: 9d9a17d 1bd6c49
Author: doc001 <wenffie@gmail.com>
Date:   Fri Oct 25 15:22:09 2024 +0800

    Merge pull request #475 from thweetkomputer/master
    
    fix typo in route_table

commit 9d9a17db9cc4b35d9493e5ac3f7ebb81c0e8daa8
Merge: 921b00c 2c78931
Author: doc001 <wenffie@gmail.com>
Date:   Fri Oct 25 14:56:01 2024 +0800

    Merge pull request #472 from ideal/master
    
    fix typo in example
```

2) 修改CMakeLists.txt文件 

直接编译存在大量报错，使用本目录中提供的修改过的CMakeLists.txt文件

同时需要修改test/test_node.cpp文件，将如下行注掉：
```
//butil::ShadowingAtExitManager exit_manager_
```


2) 编译braft

<pre>
# mkdir build output-inst && cd build && pwd
/root/cpp_proj/brpc-compile/braft/build


# cmake -DBRPC_WITH_GLOG=ON -DBUILD_UNIT_TESTS=1 -DLEVELDB_WITH_SNAPPY=1 -DCMAKE_PREFIX_PATH=/root/cpp_proj/brpc-compile/gflags/output-inst\;/root/cpp_proj/brpc-compile/glog/output-inst\;/root/cpp_proj/brpc-compile/googletest/output-inst\;/root/cpp_proj/brpc-compile/protobuf/output-inst\;/root/cpp_proj/brpc-compile/leveldb/output-inst\;/root/cpp_proj/brpc-compile/snappy/output-inst\;/root/cpp_proj/brpc-compile/brpc/output-inst -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/braft/output-inst ..

# make
# make install
# tree ../output-inst/
../output-inst/
├── include
│   └── braft
│       ├── ballot_box.h
│       ├── ballot.h
│       ├── builtin_service_impl.h
│       ├── builtin_service.pb.h
│       ├── cli.h
│       ├── cli.pb.h
│       ├── cli_service.h
│       ├── closure_helper.h
│       ├── closure_queue.h
│       ├── configuration.h
│       ├── configuration_manager.h
│       ├── enum.pb.h
│       ├── errno.pb.h
│       ├── file_reader.h
│       ├── file_service.h
│       ├── file_service.pb.h
│       ├── file_system_adaptor.h
│       ├── fsm_caller.h
│       ├── fsync.h
│       ├── lease.h
│       ├── local_file_meta.pb.h
│       ├── local_storage.pb.h
│       ├── log_entry.h
│       ├── log.h
│       ├── log_manager.h
│       ├── macros.h
│       ├── memory_log.h
│       ├── node.h
│       ├── node_manager.h
│       ├── protobuf_file.h
│       ├── raft.h
│       ├── raft_meta.h
│       ├── raft.pb.h
│       ├── raft_service.h
│       ├── remote_file_copier.h
│       ├── repeated_timer_task.h
│       ├── replicator.h
│       ├── route_table.h
│       ├── snapshot_executor.h
│       ├── snapshot.h
│       ├── snapshot_throttle.h
│       ├── storage.h
│       └── util.h
└── lib
    ├── libbraft.a
    └── libbraft.so

3 directories, 45 files

# find ./ -name braft_cli
./tools/output/bin/braft_cli
./tools/output/bin/braft_cli
</pre>
