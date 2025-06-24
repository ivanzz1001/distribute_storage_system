# foundationDB各组件介绍


在FoundationDB架构中，系统的核心组件主要分为三类进程角色(process roles):

- Storage
- Transaction
- Stateless

它们分别承担不同的职责，形成了 FoundationDB 的分布式事务 + 存储系统。


# 1. Storage进程(storage role)

## 1.1 功能

- 持久化存储所有键值数据(Key-Value)

- 提供版本化读数据能力

- 接收并响应读取请求(包括range read)

- 存储的数据是MVCC(多版本并发控制）的

## 1.2 特点

- 数据分片(shard)后分配到不同storage进程

- 每个key有一个primary storage replica

- 可以与transaction process同进程运行


# 2. Transaction 进程（transaction / proxy / resolver / log）

## 2.1 功能（由多个角色组成）

| 子角色           | 功能                                  |
| ------------- | -------------------------------------- |
| **Proxy**     | 接收客户端事务、管理事务状态、协调写入        |
| **Resolver**  | 检查事务冲突（读写冲突检测）                |
| **Log**       | 写入事务日志（TLog），持久化顺序日志，用于恢复 |
| **Committer** | 管理 commit 阶段、数据版本推进（MVCC）      |


## 2.2 特点：

- 保证事务的 串行化一致性

- 所有写操作都必须经过 Transaction 系统

- 写前日志记录后，由 storage 异步落盘

# 3. Stateless 进程（stateless / fdbserver without assigned role）

## 3.1 📌 功能：
- 不承担存储或事务任务

- 通常是备用进程，可动态分配角色

- 常用于部署弹性架构：随需应变地成为 proxy、log 或 storage

## 3.2 特点

- 在配置中留白角色就是 stateless（如使用 process = stateless）

- 可作为负载缓冲或冗余备份进程

# 4. 对比总结

| 角色          | 主要职责              | 是否持久化     | 是否参与事务  | 数据访问类型    |
| ----------- | --------------------- | ------------ | ----------- | -------------- |
| Storage     | 存储键值对、提供读服务       | ✅ 是       | ❌ 只负责存取 | 读/写（最终一致） |
| Transaction | 管理事务生命周期、写日志、冲突检测 | ✅ 是（TLog） | ✅ 是     | 管理写入、冲突   |
| Stateless   | 空闲备用进程，可动态分配角色    | ❌ 否       | ❌ 否     | -         |


# 5. 示例场景：

- 如果你有高读取负载 ➜ 增加 storage 进程数量

- 如果你有大量并发事务 ➜ 增加 proxy / resolver / log

- 如果你需要弹性伸缩 ➜ 部署更多 stateless 进程作为候选





