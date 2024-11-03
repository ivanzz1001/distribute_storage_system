# brpc example

参看:

- [brpc官网](https://github.com/apache/brpc)


## 1. echo_c++的编译及运行

1) 修改CMakeLists.txt

参看本目录中修改后的CMakeLists.txt文件



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
```