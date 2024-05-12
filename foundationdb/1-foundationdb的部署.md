# foundationdb的部署

参看:

- [FoundationDB集群以及YCSB压测](https://zhuanlan.zhihu.com/p/609794912)

- [Getting Started on Linux](https://apple.github.io/foundationdb/getting-started-linux.html)

- [foundationDB 二进制包下载](https://github.com/apple/foundationdb/releases)

- [FoundationDB官方文档](https://apple.github.io/foundationdb/index.html#)

- [关于 FoundationDB 的使用 以及 性能压测](https://blog.csdn.net/Z_Stand/article/details/125703518)

- [Installing FoundationDB](https://pierrez.github.io/fdb-book/getting_started/installing-fdb.html)

- [GCDW元数据服务FoundationDB的集群模式配置和高可用测试](https://www.modb.pro/db/445779)

当前操作系统: Ubuntu22.04




## 1. 单机版FoundationDB部署

### 1.1 下载安装foundationdb

1) 下载foundationdb安装文件

这里我们下载foundationdb 7.3.40版本：
<pre>
# wget https://github.com/apple/foundationdb/releases/download/7.3.40/foundationdb-server_7.3.40-1_amd64.deb
# wget https://github.com/apple/foundationdb/releases/download/7.3.40/foundationdb-clients_7.3.40-1_amd64.deb

# ls
foundationdb-clients_7.3.40-1_amd64.deb  foundationdb-server_7.3.40-1_amd64.deb
</pre>

2) 安装foundationdb

```
# dpkg -i foundationdb-clients_7.3.40-1_amd64.deb 
Selecting previously unselected package foundationdb-clients.
(Reading database ... 288857 files and directories currently installed.)
Preparing to unpack foundationdb-clients_7.3.40-1_amd64.deb ...
Unpacking foundationdb-clients (7.3.40) ...
Setting up foundationdb-clients (7.3.40) ...
Adding group `foundationdb' (GID 139) ...
Done.
Adding system user `foundationdb' (UID 131) ...
Adding new user `foundationdb' (UID 131) with group `foundationdb' ...
Not creating home directory `/var/lib/foundationdb'.
Processing triggers for libc-bin (2.35-0ubuntu3.7) ...

# dpkg -i foundationdb-server_7.3.40-1_amd64.deb 
(Reading database ... 288879 files and directories currently installed.)
Preparing to unpack foundationdb-server_7.3.40-1_amd64.deb ...
Failed to stop foundationdb.service: Unit foundationdb.service not loaded.
invoke-rc.d: initscript foundationdb, action "stop" failed.
Unpacking foundationdb-server (7.3.40) over (7.3.40) ...
Setting up foundationdb-server (7.3.40) ...
>>> configure new single memory
Database created
>>> status
Using cluster file `/etc/foundationdb/fdb.cluster'.

Unable to retrieve all status information.

Configuration:
  Redundancy mode        - single
  Storage engine         - memory
  Log engine             - ssd-2
  Encryption at-rest     - disabled
  Coordinators           - 1
  Desired Commit Proxies - 3
  Desired GRV Proxies    - 1
  Desired Resolvers      - 1
  Desired Logs           - 3
  Usable Regions         - 1

Cluster:
  FoundationDB processes - 1
  Zones                  - unknown
  Machines               - 0
  Fault Tolerance        - 0 machines
  Server time            - 05/05/24 16:52:54

Data:
  Replication health     - unknown
  Moving data            - unknown
  Sum of key-value sizes - unknown
  Disk space used        - unknown

Operating space:
  Unable to retrieve operating space status

Workload:
  Read rate              - 0 Hz
  Write rate             - 0 Hz
  Transactions started   - 0 Hz
  Transactions committed - 0 Hz
  Conflict rate          - 0 Hz

Backup and DR:
  Running backups        - 0
  Running DRs            - 0

Client time: 05/05/24 16:52:54

```
上面FoundationDB运行在single-server模式，在该模式下数据是单副本的，因此不具备容灾能力。FoundationDB默认使用```内存存储引擎```(memory storage-engine)，如果要支持硬盘持久化能力，通常要求数据量大小与内存相匹配， 参看[System requirements](https://apple.github.io/foundationdb/configuration.html#system-requirements)

现在我们来看看FoundationDB的运行状态：
```
# ps -ef | grep fdb
root        2248       1  0 16:52 ?        00:00:00 /usr/lib/foundationdb/fdbmonitor --conffile /etc/foundationdb/foundationdb.conf --lockfile /var/run/fdbmonitor.pid --daemonize
foundat+    2249    2248  0 16:52 ?        00:00:05 /usr/lib/foundationdb/backup_agent/backup_agent --cluster-file /etc/foundationdb/fdb.cluster --logdir /var/log/foundationdb
foundat+    2250    2248  2 16:52 ?        00:00:31 /usr/sbin/fdbserver --cluster-file /etc/foundationdb/fdb.cluster --datadir /var/lib/foundationdb/data/4500 --listen-address public --logdir /var/log/foundationdb --public-address auto:4500

# systemctl status foundationdb
● foundationdb.service - LSB: start and stop foundationdb
     Loaded: loaded (/etc/init.d/foundationdb; generated)
     Active: active (running) since Sun 2024-05-05 16:52:50 CST; 4min 22s ago
       Docs: man:systemd-sysv-generator(8)
      Tasks: 10 (limit: 4554)
     Memory: 28.1M
        CPU: 7.997s
     CGroup: /system.slice/foundationdb.service
             ├─2248 /usr/lib/foundationdb/fdbmonitor --conffile /etc/foundationdb/foundationdb.conf --lockfile /var/run/fdbmonitor.pid --daemonize
             ├─2249 /usr/lib/foundationdb/backup_agent/backup_agent --cluster-file /etc/foundationdb/fdb.cluster --logdir /var/log/foundationdb
             └─2250 /usr/sbin/fdbserver --cluster-file /etc/foundationdb/fdb.cluster --datadir /var/lib/foundationdb/data/4500 --listen-address public --log>

5月 05 16:52:50 ivanzz1001-virtual-machine fdbmonitor[2247]: LogGroup="default" Process="fdbmonitor": Started FoundationDB Process Monitor 7.3 (v7.3.40)
5月 05 16:52:50 ivanzz1001-virtual-machine fdbmonitor[2248]: LogGroup="default" Process="fdbmonitor": Watching conf file /etc/foundationdb/foundationdb.conf
5月 05 16:52:50 ivanzz1001-virtual-machine fdbmonitor[2248]: LogGroup="default" Process="fdbmonitor": Watching conf dir /etc/foundationdb/ (2)
```
从上面我们看到同时启动了3个进程：

- fdbmonitor

- backup_agent

- fdbserver

其中fdbmonitor是backup_agent、fdbserver的父进程，这说明backup_agent与fdbserver是由fdbmonitor来启动的。


3) 卸载foundationdb
<pre>
# sudo dpkg --purge foundationdb-server
# sudo dpkg --purge foundationdb-clients

// 卸载完软件要清理下环境，要不再次安装容易出现问题
# sudo rm -rf /var/lib/foundationdb /var/log/foundationdb /etc/foundationdb/fdb.cluster
</pre>
 
### 1.2 FoundationDB测试验证
为了验证上述本地单机版FoundationDB是否成功安装，我们使用```fdbcli```命令行来查看数据库运行状态：
```
# fdbcli
Using cluster file `/etc/foundationdb/fdb.cluster'.

The database is available.

Welcome to the fdbcli. For help, type `help'.
fdb> status

Using cluster file `/etc/foundationdb/fdb.cluster'.

Configuration:
  Redundancy mode        - single
  Storage engine         - memory
  Log engine             - ssd-2
  Encryption at-rest     - disabled
  Coordinators           - 1
  Desired Commit Proxies - 3
  Desired GRV Proxies    - 1
  Desired Resolvers      - 1
  Desired Logs           - 3
  Usable Regions         - 1

Cluster:
  FoundationDB processes - 1
  Zones                  - 1
  Machines               - 1
  Memory availability    - 2.9 GB per process on machine with least available
                           >>>>> (WARNING: 4.0 GB recommended) <<<<<
  Fault Tolerance        - 0 machines
  Server time            - 05/05/24 17:24:13

Data:
  Replication health     - Healthy
  Moving data            - 0.000 GB
  Sum of key-value sizes - 0 MB
  Disk space used        - 105 MB

Operating space:
  Storage server         - 1.0 GB free on most full server
  Log server             - 28.1 GB free on most full server

Workload:
  Read rate              - 22 Hz
  Write rate             - 1 Hz
  Transactions started   - 7 Hz
  Transactions committed - 0 Hz
  Conflict rate          - 0 Hz

Backup and DR:
  Running backups        - 0
  Running DRs            - 0

Client time: 05/05/24 17:24:13

fdb> 
```

1） 插入数据

- 检查当前数据库的写模式

```
fdb> help writemode

writemode <on|off>

Enables or disables sets and clears.

Setting or clearing keys from the CLI is not recommended.

```

- 写入数据
```
fdb> writemode on
fdb> set msg_info "hello, world"
Committed (2919359892)
```

2) 读取数据
```
fdb> get msg_info
`msg_info' is `hello, world'
```

3) 扫描keys

我们先来看一下```getrange```命令的使用:
```
fdb> help getrange

getrange <BEGINKEY> [ENDKEY] [LIMIT]

Fetch key/value pairs in a range of keys.

Displays up to LIMIT keys and values for keys between BEGINKEY (inclusive) and
ENDKEY (exclusive). If ENDKEY is omitted, then the range will include all keys
starting with BEGINKEY. LIMIT defaults to 25 if omitted.

For information on escaping keys, type `help escaping'.
```

接下来我们扫描出步骤2中写入的```msg_info```这个key:
```
fdb> getrange \x00 \xfe 10

Range limited to 10 keys
`msg_info' is `hello, world'
```


如下我们来使用一下其set/get命令：
```
fdb> help set

set <KEY> <VALUE>

Set a value for a given key.

If KEY is not already present in the database, it will be created.

For information on escaping keys and values, type `help escaping'.

fdb> set test_key "test value"
ERROR: writemode must be enabled to set or clear keys in the database.

```


### 1.3 管理FoundationDB服务


- 启动与停止


在安装完成之后，FoundationDB被设置为了自动启动。我们可以使用如下的一些命令来手动启动与关闭数据库。
<pre>
# service foundation start
# service foundation stop
# service foundation restart
</pre>

上述命令会启动和关闭```fdbmonitor```进程，然后通过该进程启动与关闭fdbserver、backup_agent。关于fdbmonitor与fdbserver更多详细信息，请参看: [administration fdbmonitor](https://apple.github.io/foundationdb/administration.html#administration-fdbmonitor)



## 2. 搭建foundationDB集群

我们使用三台虚拟机来搭建FoundationDB集群，IP地址分别为:

- 192.168.180.131
- 192.168.180.132
- 192.168.180.133

### 2.1 搭建步骤

1) 安装foundationdb

在上述3台虚拟机上分别安装foundationdb服务。


2）让FoundationDB使用公网IP

- 在192.168.180.131上执行




- 在192.168.180.132上执行

- 在192.168.180.133上执行