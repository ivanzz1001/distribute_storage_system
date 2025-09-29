
# libconhash的使用

参考文档：
- [libconhash GitHub官网](https://github.com/swordflychen/libconhash)

## 1. libconhash介绍

libconhash是一个用C语言编写的一致性hash库，可在Windows及Linux平台编译使用，其具有如下Feature:

- 易用性&高性能：libconhash内部使用红黑树来管理所有节点以获得高性能

- 可支持自定义hash函数，默认采用MD5算法

- 根据节点的处理容量，具有良好的scale能力


## 2. Linux环境下的编译

1. 下载源码

    ```bash
    # git clone https://github.com/swordflychen/libconhash.git
    # cd libconhash
    ```

2. 编译

   ```bash
   # install_dir=/usr/local/conhash
   # make && make install INSTALL_DIR=${install_dir}
   ```





