# ceph 10.2.10源代码编译

参考如下文章:

- [Ceph 源码编译](https://blog.csdn.net/wxb880114/article/details/130215330)

- [ceph源代码tarball](https://download.ceph.com/tarballs/)

- [ceph-18.2.0版本编译安装](https://www.cnblogs.com/mjxi/p/17653697.html)

- [在 CentOS 上编译 Ceph](https://gukaifeng.cn/posts/zai-centos-shang-bian-yi-ceph/)



## Linux镜像下载

- ### centos7镜像下载
    - [centos官网](https://www.centos.org/download/)


- ### ubuntu镜像下载

   - 官方下载地址：https://www.ubuntu.com/download
   
   - 中科大源: http://mirrors.ustc.edu.cn/ubuntu-releases/

   - 南京大学: http://mirrors.nju.edu.cn/ubuntu-releases/

   - 上海交通大学: http://ftp.sjtu.edu.cn/ubuntu-releases/

   - 清华大学: https://mirror.tuna.tsinghua.edu.cn/ubuntu-releases/

   - 阿里云开源镜像站: http://mirrors.aliyun.com/ubuntu-releases/

   - 浙江大学: http://mirrors.zju.edu.cn/ubuntu-releases/

   - 不知名镜像网站: http://mirror.pnl.gov/releases/

   - 各个版本下载网址: http://mirrors.melbourne.co.uk/ubuntu-releases/

## 1. 使用centos7编译ceph10.2.10

1) 源代码下载
<pre>
# wget https://download.ceph.com/tarballs/ceph-10.2.10.tar.gz
</pre>

2) 安装相关工具
<pre>
# yum install gcc gcc-c++
</pre>



3) 编译

- 修正源代码相关错误

在编译之前，执行如下命令修正相关错误:
<pre>
# sed -i src/test/librgw_file.cc -e "s/CLEANUP/CLEANUP1/"
# sed -i src/test/librgw_file_aw.cc -e "s/CLEANUP/CLEANUP4/"
# sed -i src/test/librgw_file_cd.cc -e "s/CLEANUP/CLEANUP3/"
# sed -i src/test/librgw_file_gp.cc -e "s/CLEANUP/CLEANUP2/"
# sed -i src/test/librgw_file_marker.cc -e "s/CLEANUP/CLEANUP6/"
# sed -i src/test/librgw_file_nfsns.cc -e "s/CLEANUP/CLEANUP5/"
</pre>

- 修正`pip no such option: --use-wheel`错误

新版本的python pip已经不支持```--use-wheel```选项，因此执行如下命令去掉该选项：
<pre>
# grep -rn "use-wheel" ./
# sed -i 's/--use-wheel//g' ./src/ceph-detect-init/CMakeLists.txt
# sed -i 's/--use-wheel//g' ./src/ceph-detect-init/Makefile.am
# sed -i 's/--use-wheel//g' ./src/ceph-detect-init/tox.ini
# sed -i 's/--use-wheel//g' ./src/ceph-disk/CMakeLists.txt
# sed -i 's/--use-wheel//g' ./src/ceph-disk/Makefile.am
# sed -i 's/--use-wheel//g' ./src/ceph-disk/tox.ini
# sed -i 's/--use-wheel//g' ./src/tools/setup-virtualenv.sh
# sed -i 's/--use-wheel//g' ./src/Makefile.in
</pre>



- 编译源码
<pre>
# ./install-deps.sh
# ./autogen.sh
# ./configure --with-rbd --with-debug --with-rados --with-radosgw --with-cephfs --enable-client --enable-server
# make -j2
</pre>

### 1.1 ceph10.2.10测试

1) 启动开发集群

在ceph的src目录下有一个名称为vstart.sh的脚本文件，开发人员可以使用该部署脚本快速的部署一个调试环境。一旦编译完成，使用如下的命令来启动ceph部署：
<pre>
# cd src
# ./vstart.sh -d -n -x
</pre>

我们也可以指定mon、mds、osd的个数：
<pre>
# MON=3 MDS=0 OSD=3  ./vstart.sh -d -n -x
# ./ceph -w
</pre>

>ps: 所采用的ceph配置文件为./src/ceph.conf

这里我们针对相关的参数做一个简单的说明：
<pre>
-m  指出monitor节点的ip地址和默认端口6789

-n  指出此次部署为全新部署

-d  指出使用debug模式（便于调试代码）

-r  指出启动radosgw进程

--mon_num  指出部署的monitor个数

--osd_num  指出部署的osd个数

--mds_num  指出部署的mds个数

--blustore  指出ceph后端存储使用最新的bluestore
</pre>


2) 停止开发集群
<pre>
# ./stop.sh
</pre>

