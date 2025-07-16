# FoundationDB的编译

这里我们在Ubuntu22.04上采用docker容器来编译foundationdb。参看如下:

- [foundationdb github](https://github.com/apple/foundationdb)

- [fdb-build-support](https://github.com/FoundationDB/fdb-build-support)

- [foundationdb官网](https://www.foundationdb.org/)

- [site map](https://apple.github.io/foundationdb/contents.html?highlight=log+server)

- [known limitations](https://apple.github.io/foundationdb/known-limitations.html)

- [cpu too high](https://forums.foundationdb.org/t/why-many-clientthreads-will-cause-fdbserver-stateless-grv-proxy-cpu-too-high/4969/4)


## 1. 使用docker编译foundationdb

1) 拉取build镜像并运行

这里我们使用别人已经构建好的docker镜像来编译foundationdb:

>ps: 如果我们想自己构建镜像，官方也提供了执行步骤

```
# docker pull foundationdb/build:centos7-latest
# docker image ls
REPOSITORY           TAG              IMAGE ID       CREATED         SIZE
foundationdb/build   centos7-latest   2a83d112b443   8 hours ago     18.3GB
hello-world          latest           d2c94e258dcb   12 months ago   13.3kB

# docker run -it foundationdb/build:centos7-latest /bin/bash
```

2）镜像中拉取foundationdb源代码并编译

```
source /opt/rh/devtoolset-11/enable
source /opt/rh/rh-python38/enable
source /opt/rh/rh-ruby27/enable

mkdir -p src/foundationdb
git clone https://github.com/apple/foundationdb.git src/foundationdb/ 
mkdir build_output

cmake -S src/foundationdb -B build_output -G Ninja 
ninja -C build_output # If this crashes it probably ran out of memory. Try ninja -j1
```
>注1: 当编译机资源较少时，使用 `ninja -j2 -C build_output`可以避免卡住，反而可以更快的完成编译
>
>注2: 将内存调整到6GB以上，否则链接fdbserver时可能会因内存不足导致链接失败

经过长时间(约3天)的编译，我们大体来看看编译结果(编译后的二进制会放在 bin 目录下，lib库会放在 lib目录下)：
<pre>
# /tmp/build_output
# ls
actorcompiler.exe  bindingtester2.touch  cmake_install.cmake  CPackSourceConfig.cmake  fdbcli       fdbserver    metacluster  tests
bin                bindingtester.touch   contrib              CTestCustom.ctest        fdbclient    flow         packages     versions.target
bindings           build.ninja           correctness          CTestTestfile.cmake      fdb.cluster  flowbench    packaging    version.txt
bindingtester      CMakeCache.txt        coveragetool.exe     documentation            fdbmonitor   lib          sandbox      vexillographer.exe
bindingtester2     CMakeFiles            CPackConfig.cmake    fdbbackup                fdbrpc       License.txt  share

# cd bin
# ls
acac                             coverage.fdbconvert.xml          fdbbackup                   fdb_c_shim_api_tester         fdbrestore
actor_flamegraph                 coverage.fdbdecode.xml           fdb_c90_test                fdb_c_shim_lib_tester         fdbserver
authz_tls_unittest               coverage.fdb_flow_tester.xml     fdb_c_api_tester            fdb_c_shim_unit_tests         linktest
backup_agent                     coverage.fdbrpclinktest.xml      fdb_c_client_config_tester  fdb_c_txn_size_test           mako
coro_tutorial                    coverage.fdbserver.xml           fdb_c_client_memory_test    fdb_c_unit_tests              mkcert
coverage.authz_tls_unittest.xml  coverage.flowlinktest.xml        fdbcli                      fdb_c_unit_tests_version_510  trace_partial_file_suffix_test
coverage.coro_tutorial.xml       coverage.tutorial.xml            fdbconvert                  fdbdecode                     tutorial
coverage.fdbbackup.xml           disconnected_timeout_unit_tests  fdb_c_performance_test      fdbdr
coverage.fdbclientlinktest.xml   dr_agent                         fdb_c_ryw_benchmark         fdb_flow_tester
coverage.fdbcli.xml              fastrestore_tool                 fdb_c_setup_tests           fdbmonitor
</pre>



