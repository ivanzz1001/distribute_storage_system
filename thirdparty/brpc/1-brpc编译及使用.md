# brpc编译及使用

参看:

- [brpc官网](https://github.com/apache/brpc)


## 1. 下载及编译

brpc具有如下依赖：

- gflags: 用于定义一些全局的选项

- protobuf: 用于序列化messags、服务接口

- leveldb: rpcz用于记录相关跟踪信息

  - snappy：要静态链接leveldb的话，需要此库

- gtest: 如果要运行brpc相关测试，需要此库

- glog: 使用glog版的brpc，添加选项--with-glog

- google-perftools: 如果要在样例中启用cpu/heap的profiler，需要此库


我们知道上述依赖之后，就分别下载对应的源代码来进行编译。首先来大概看一下我们编译的操作系统环境信息：

<pre>
# uname -a
Linux ivanzz1001-virtual-machine 6.5.0-41-generic #41~22.04.2-Ubuntu SMP PREEMPT_DYNAMIC Mon Jun  3 11:32:55 UTC 2 x86_64 x86_64 x86_64 GNU/Linux 

# lsb_release  -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.4 LTS
Release:        22.04
Codename:       jammy

# cmake --version
cmake version 3.22.1

CMake suite maintained and supported by Kitware (kitware.com/cmake).

# gcc --version
gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
</pre>

创建的应的编译目录：
<pre>
# mkdir -p brpc-compile
</pre>



### 1.1 编译gflags

glfags的github官网地址为: https://github.com/gflags/gflags


1) 下载源代码
<pre>
# git clone https://github.com/gflags/gflags.git
# cd gflags
# git tag
v0.1
v0.2
v0.3
v0.4
v0.5
v0.6
v0.7
v0.8
v0.9
v1.0rc1
v1.0rc2
v1.1
v1.2
v1.3
v1.4
v1.5
v1.6
v1.7
v2.0
v2.1.0
v2.1.1
v2.1.2
v2.2.0
v2.2.1
v2.2.2
# git checkout tags/v2.2.2
</pre>

2) 编译gflags
<pre>
# mkdir build output-inst && cd build && pwd
/root/cpp_proj/brpc-compile/gflags/build

# cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/gflags/output-inst ..
# make
# make install

# tree ../output-inst/
../output-inst/
├── bin
│   └── gflags_completions.sh
├── include
│   └── gflags
│       ├── gflags_completions.h
│       ├── gflags_declare.h
│       ├── gflags_gflags.h
│       └── gflags.h
└── lib
    ├── cmake
    │   └── gflags
    │       ├── gflags-config.cmake
    │       ├── gflags-config-version.cmake
    │       ├── gflags-nonamespace-targets.cmake
    │       ├── gflags-nonamespace-targets-relwithdebinfo.cmake
    │       ├── gflags-targets.cmake
    │       └── gflags-targets-relwithdebinfo.cmake
    ├── libgflags.a
    ├── libgflags_nothreads.a
    └── pkgconfig
        └── gflags.pc

7 directories, 14 files
</pre>

### 1.2 编译gtest/gmock

gtest的Github官网地址：https://github.com/google/googletest

>gmock与gtest位于同一代码仓库

参看：

- [linux下搭建gtest和gmock测试框架](https://cloud.tencent.com/developer/article/1507088)


1） 下载源代码
<pre>
# git clone https://github.com/google/googletest.git
# cd googletest
# git tag
release-1.0.0
release-1.0.1
release-1.1.0
release-1.10.0
release-1.11.0
release-1.12.0
release-1.12.1
release-1.2.0
release-1.2.1
release-1.3.0
release-1.4.0
release-1.5.0
release-1.6.0
release-1.7.0
release-1.8.0
release-1.8.1
v1.10.x
v1.12.0
v1.12.0-pre
v1.13.0
v1.13.0-pre
v1.14.0
v1.14.0-pre
v1.15.0
v1.15.0-pre
v1.15.1
v1.15.2
# git checkout tags/v1.15.2
</pre>

2) 编译gtest/gmock
<pre>
# mkdir build output-inst && cd build && pwd
/root/cpp_proj/brpc-compile/googletest/build

# cmake -DBUILD_GMOCK=ON -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/googletest/output-inst ..
# make
# make install
# tree ../output-inst/
../output-inst/
├── include
│   ├── gmock
│   │   ├── gmock-actions.h
│   │   ├── gmock-cardinalities.h
│   │   ├── gmock-function-mocker.h
│   │   ├── gmock.h
│   │   ├── gmock-matchers.h
│   │   ├── gmock-more-actions.h
│   │   ├── gmock-more-matchers.h
│   │   ├── gmock-nice-strict.h
│   │   ├── gmock-spec-builders.h
│   │   └── internal
│   │       ├── custom
│   │       │   ├── gmock-generated-actions.h
│   │       │   ├── gmock-matchers.h
│   │       │   ├── gmock-port.h
│   │       │   └── README.md
│   │       ├── gmock-internal-utils.h
│   │       ├── gmock-port.h
│   │       └── gmock-pp.h
│   └── gtest
│       ├── gtest-assertion-result.h
│       ├── gtest-death-test.h
│       ├── gtest.h
│       ├── gtest-matchers.h
│       ├── gtest-message.h
│       ├── gtest-param-test.h
│       ├── gtest_pred_impl.h
│       ├── gtest-printers.h
│       ├── gtest_prod.h
│       ├── gtest-spi.h
│       ├── gtest-test-part.h
│       ├── gtest-typed-test.h
│       └── internal
│           ├── custom
│           │   ├── gtest.h
│           │   ├── gtest-port.h
│           │   ├── gtest-printers.h
│           │   └── README.md
│           ├── gtest-death-test-internal.h
│           ├── gtest-filepath.h
│           ├── gtest-internal.h
│           ├── gtest-param-util.h
│           ├── gtest-port-arch.h
│           ├── gtest-port.h
│           ├── gtest-string.h
│           └── gtest-type-util.h
└── lib
    ├── cmake
    │   └── GTest
    │       ├── GTestConfig.cmake
    │       ├── GTestConfigVersion.cmake
    │       ├── GTestTargets.cmake
    │       └── GTestTargets-relwithdebinfo.cmake
    ├── libgmock.a
    ├── libgmock_main.a
    ├── libgtest.a
    ├── libgtest_main.a
    └── pkgconfig
        ├── gmock_main.pc
        ├── gmock.pc
        ├── gtest_main.pc
        └── gtest.pc

11 directories, 52 files
</pre>

### 1.3 编译glog

glog的github官网地址: https://github.com/google/glog

1) 下载源代码
<pre>
# git clone https://github.com/google/glog.git
# cd glog
# git tag
v0.1
v0.1.1
v0.1.2
v0.2
v0.2.1
v0.3.0
v0.3.1
v0.3.2
v0.3.3
v0.3.4
v0.3.5
v0.4.0
v0.5.0
v0.5.0-rc1
v0.5.0-rc2
v0.6.0
v0.6.0-rc1
v0.6.0-rc2
v0.7.0
v0.7.0-rc1
v0.7.1
# git checkout tags/v0.7.1
</pre>

2) 编译glog
<pre>
# mkdir build output-inst && cd build && pwd
/root/cpp_proj/brpc-compile/glog/build

# cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/glog/output-inst ..

# make
# make install
# tree ../output-inst/
../output-inst/
├── include
│   └── glog
│       ├── export.h
│       ├── flags.h
│       ├── logging.h
│       ├── log_severity.h
│       ├── platform.h
│       ├── raw_logging.h
│       ├── stl_logging.h
│       ├── types.h
│       └── vlog_is_on.h
├── lib
│   ├── cmake
│   │   └── glog
│   │       ├── glog-config.cmake
│   │       ├── glog-config-version.cmake
│   │       ├── glog-modules.cmake
│   │       ├── glog-targets.cmake
│   │       └── glog-targets-relwithdebinfo.cmake
│   ├── libglog.so -> libglog.so.2
│   ├── libglog.so.0.7.1
│   └── libglog.so.2 -> libglog.so.0.7.1
└── share
    └── glog
        └── cmake
            └── FindUnwind.cmake

8 directories, 18 files
</pre>


### 1.4 编译google-perftools

google-perftools的Github官方地址: https://github.com/gperftools/gperftools

参看:

- [你了解过Gperftools性能分析神器吗？](https://zhuanlan.zhihu.com/p/678172638)

1) 下载源代码

<pre>
# git clone https://github.com/gperftools/gperftools.git
</pre>

### 1.5 编译libsnappy

snappy的github官方地址：https://github.com/google/snappy

参看：

- [Snappy使用](https://blog.csdn.net/weixin_45010335/article/details/140136092)

1） 下载源代码

<pre>
# git clone https://github.com/google/snappy.git
# cd snappy
# git tag
1.1.10
1.1.3
1.1.4
1.1.5
1.1.6
1.1.7
1.1.8
1.1.9
1.2.0
1.2.1
# git checkout tags/1.2.1
</pre>

2) 编译snappy

</pre>
# mkdir build output-inst && cd build && pwd
/root/cpp_proj/brpc-compile/snappy/build
# cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=RelWithDebInfo -DSNAPPY_BUILD_TESTS=OFF -DSNAPPY_BUILD_BENCHMARKS=OFF -DCMAKE_PREFIX_PATH=/root/cpp_proj/brpc-compile/googletest/output-inst -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/snappy/output-inst ..
# make
# make install
# tree ../output-inst/
../output-inst/
├── include
│   ├── snappy-c.h
│   ├── snappy.h
│   ├── snappy-sinksource.h
│   └── snappy-stubs-public.h
└── lib
    ├── cmake
    │   └── Snappy
    │       ├── SnappyConfig.cmake
    │       ├── SnappyConfigVersion.cmake
    │       ├── SnappyTargets.cmake
    │       └── SnappyTargets-relwithdebinfo.cmake
    └── libsnappy.a

4 directories, 9 files
</pre>

### 1.5 编译leveldb

leveldb的github官网地址: https://github.com/google/leveldb

1) 下载源代码

<pre>
# git clone https://github.com/google/leveldb.git
# cd leveldb
# git tag
1.21
1.22
1.23
v1.10
v1.11
v1.12
v1.13
v1.14
v1.15
v1.16
v1.17
v1.18
v1.19
v1.20
v1.3
v1.4
v1.5
v1.6
v1.7
v1.8
v1.9
# git checkout tags/1.23
</pre>


2) 编译leveldb

<pre>
# mkdir build output-inst && cd build && pwd
/root/cpp_proj/brpc-compile/leveldb/build

# cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DLEVELDB_BUILD_TESTS=OFF -DLEVELDB_BUILD_BENCHMARKS=OFF -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/leveldb/output-inst ..

# make
# make install
# tree ../output-inst/
../output-inst/
├── include
│   └── leveldb
│       ├── cache.h
│       ├── c.h
│       ├── comparator.h
│       ├── db.h
│       ├── dumpfile.h
│       ├── env.h
│       ├── export.h
│       ├── filter_policy.h
│       ├── iterator.h
│       ├── options.h
│       ├── slice.h
│       ├── status.h
│       ├── table_builder.h
│       ├── table.h
│       └── write_batch.h
└── lib
    ├── cmake
    │   └── leveldb
    │       ├── leveldbConfig.cmake
    │       ├── leveldbConfigVersion.cmake
    │       ├── leveldbTargets.cmake
    │       └── leveldbTargets-release.cmake
    └── libleveldb.a

5 directories, 20 files
</pre>

### 1.6 编译protobuf

protobuf的github官方地址: https://github.com/protocolbuffers/protobuf

1) 下载protobuf
<pre>
# git clone https://github.com/protocolbuffers/protobuf.git
# cd protobuf
# git submodule update --init --recursive
# git tag
# git checkout tags/v4.25.0
</pre>

2) 编译protobuf
<pre>
# mkdir build output-inst && cd build && pwd
/root/cpp_proj/brpc-compile/protobuf/build

# cmake -G "Unix Makefiles"  -DCMAKE_CXX_STANDARD=17 -DABSL_PROPAGATE_CXX_STD=ON -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DCMAKE_BUILD_TYPE=RelWithDebInfo -Dprotobuf_BUILD_TESTS=OFF -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/protobuf/output-inst ..


# make -j2
# make install
</pre>

>注意：
>
>1） 关掉protobuf_BUILD_TESTS，否则后面编译brpc时可能出现The following imported targets are referenced, but are missing: protobuf::gmock
>
>2) brpc与protobuf版本具有比较紧密的依赖性，这里使用v4.25.0版本，并使用c++17版本来编译
>
>3) 经测试protobuf使用v21.8也可以编译通过



### 1.7 编译brpc

brpc的github官方地址：https://github.com/apache/brpc

1） 下载brpc源码
<pre>
# git clone https://github.com/apache/brpc.git
# cd brpc
# git tag
0.9.5
0.9.6
0.9.6-rc01
0.9.6-rc02
0.9.6-rc03
0.9.7
0.9.7-rc01
0.9.7-rc02
0.9.7-rc03
0.9.8-rc01
1.0.0
1.0.0-rc01
1.0.0-rc02
1.0.1-rc01
1.1.0
1.10.0
1.11.0
1.2.0
1.3.0
1.4.0
1.5.0
1.6.0
1.6.1
1.7.0
1.8.0
1.9.0
v0.9.0
# git checkout tags/1.11.0
</pre>


2） 修改CMakeLists.txt文件 

在编译brpc时会链接上述我们编译好的protobuf，然后会遇到如下问题：https://github.com/protocolbuffers/protobuf/issues/19007
```
/root/cpp_proj/brpc-compile/protobuf/src/google/protobuf/generated_message_tctable_lite.cc:1419: undefined reference to `utf8_range::IsStructurallyValid(std::basic_string_view<char, std::char_traits<char> >)'
/usr/bin/ld: /root/cpp_proj/brpc-compile/protobuf/output-inst/lib/libprotobuf.a(generated_message_tctable_lite.cc.o): in function `google::protobuf::internal::TcParser::FastUS2(google::protobuf::MessageLite*, char const*, google::protobuf::internal::ParseContext*, google::protobuf::internal::TcFieldData, google::protobuf::internal::TcParseTableBase const*, unsigned long)':
/root/cpp_proj/brpc-compile/protobuf/src/google/protobuf/generated_message_tctable_lite.cc:1419: undefined reference to `utf8_range::IsStructurallyValid(std::basic_string_view<char, std::char_traits<char> >)'
```

看报错提示是找不到符号表，我们需要修改brpc根目录下的CMakeLists.txt文件：

```
find_package(Protobuf REQUIRED)
if(Protobuf_VERSION GREATER 4.21)
    # required by absl
    set(CMAKE_CXX_STANDARD 17)

    find_package(utf8_range REQUIRED)
    set(utf8_RANGE_TARGETS
        utf8_range::utf8_validity
    )

    find_package(absl REQUIRED CONFIG)
    set(protobuf_ABSL_USED_TARGETS
        absl::absl_check
        absl::absl_log
        absl::algorithm
        absl::base
        absl::bind_front
        absl::bits
        absl::btree
        absl::cleanup
        absl::cord
        absl::core_headers
        absl::debugging
        absl::die_if_null
        absl::dynamic_annotations
        absl::flags
        absl::flat_hash_map
        absl::flat_hash_set
        absl::function_ref
        absl::hash
        absl::layout
        absl::log_initialize
        absl::log_severity
        absl::memory
        absl::node_hash_map
        absl::node_hash_set
        absl::optional
        absl::span
        absl::status
        absl::statusor
        absl::strings
        absl::synchronization
        absl::time
        absl::type_traits
        absl::utility
        absl::variant
    )
else()
    use_cxx11()
endif()


set(DYNAMIC_LIB
    ${GFLAGS_LIBRARY}
    ${PROTOBUF_LIBRARIES} ${protobuf_ABSL_USED_TARGETS} ${utf8_RANGE_TARGETS}
    ${LEVELDB_LIB}
    ${PROTOC_LIB}
    ${CMAKE_THREAD_LIBS_INIT}
    ${THRIFT_LIB}
    dl
    z)
```

上面我们主要添加了查找utf8_RANGE_TARGETS这一目标。

参看如下：

- [Undefined reference to absl and utf8_range during linking](https://github.com/protocolbuffers/protobuf/issues/12271)


3) 编译brpc
<pre>
# mkdir build output-inst && cd build && pwd
/root/cpp_proj/brpc-compile/brpc/build

# cmake -DWITH_GLOG=OFF -DWITH_SNAPPY=ON -DCMAKE_PREFIX_PATH=/root/cpp_proj/brpc-compile/gflags/output-inst\;/root/cpp_proj/brpc-compile/googletest/output-inst\;/root/cpp_proj/brpc-compile/protobuf/output-inst\;/root/cpp_proj/brpc-compile/leveldb/output-inst\;/root/cpp_proj/brpc-compile/snappy/output-inst -DCMAKE_INSTALL_PREFIX=/root/cpp_proj/brpc-compile/brpc/output-inst ..

# make
# make install

</pre>

>注意：
>
> 1) WITH_GLOG=ON时，编译出现问题，这里直接关掉