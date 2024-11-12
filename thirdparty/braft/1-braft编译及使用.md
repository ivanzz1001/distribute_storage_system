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

2) 