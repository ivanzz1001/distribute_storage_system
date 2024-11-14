# braft编译及使用

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


