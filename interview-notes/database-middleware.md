# 数据库 / 中间件八股文

## 一、MySQL

### 1. InnoDB 和 MyISAM 的区别？
- InnoDB 支持事务、行级锁、外键、崩溃恢复（redo log），聚簇索引；MySQL 5.5 起默认。
- MyISAM 不支持事务，只有表级锁，非聚簇索引，`count(*)` 直接读取保存的行数。

### 2. 索引的类型？
- 数据结构：B+ 树索引、哈希索引、全文索引。
- 物理存储：聚簇索引（主键索引，叶子节点存整行数据）、二级索引（叶子节点存主键值）。
- 逻辑：主键索引、唯一索引、普通索引、联合索引、前缀索引。

### 3. 什么是回表、覆盖索引、索引下推？
- 回表：通过二级索引查到主键，再回到聚簇索引取整行数据。
- 覆盖索引：查询的列都在索引中，无需回表（Extra 显示 Using index）。
- 索引下推（ICP，5.6）：在存储引擎层用联合索引中的其他列先过滤，减少回表次数。

### 4. 最左前缀原则？
- 联合索引 (a, b, c) 按 a → b → c 排序，查询必须从最左列开始匹配。
- 范围查询（`>`、`<`）右边的列无法用到索引（`>=`、`<=`、`between`、`like 'x%'` 视情况可用）。MySQL 8.0 支持索引跳跃扫描。

### 5. 索引失效的场景？
- 不满足最左前缀；对索引列做函数运算或计算；隐式类型转换（字符串列用数字查询）；`like '%xx'` 左模糊；`OR` 两边有一列无索引；`!=`、`not in` 可能失效；优化器判断全表扫描更快。

### 6. 事务的 ACID 是怎么保证的？
- 原子性：undo log（回滚）。
- 持久性：redo log（WAL 先写日志）。
- 隔离性：MVCC + 锁。
- 一致性：由以上三者共同保证，是最终目的。

### 7. 并发事务问题与隔离级别？
- 脏读、不可重复读、幻读。
- 读未提交、读已提交（RC）、可重复读（RR，InnoDB 默认）、串行化。
- InnoDB 在 RR 下通过 MVCC（快照读）+ 临键锁（当前读）很大程度上避免了幻读。

### 8. MVCC 原理？
- 每行记录有隐藏字段 `trx_id`（最近修改的事务 ID）和 `roll_pointer`（指向 undo log 版本链）。
- ReadView 包含：活跃事务 ID 列表 `m_ids`、`min_trx_id`、`max_trx_id`、`creator_trx_id`，沿版本链找到第一个对当前事务可见的版本。
- RC：每次 SELECT 生成新 ReadView；RR：事务第一次 SELECT 时生成，之后复用。

### 9. InnoDB 的锁？
- 粒度：表锁、行锁；意向锁（表级，加快判断）。
- 共享锁 S、排他锁 X。
- 行锁算法：记录锁 Record Lock、间隙锁 Gap Lock、临键锁 Next-Key Lock（记录 + 前面的间隙，RR 下默认）。
- 行锁是加在索引上的，没走索引的更新会锁住全表的记录。

### 10. redo log、undo log、binlog 区别？
- redo log：InnoDB 物理日志，记录"某页做了什么修改"，循环写，用于崩溃恢复。
- undo log：逻辑日志，用于回滚和 MVCC。
- binlog：Server 层逻辑日志，追加写，用于主从复制和数据恢复；格式有 statement、row、mixed。
- 两阶段提交：redo prepare → 写 binlog → redo commit，保证两份日志一致。

### 11. 主从复制原理？
- 主库写 binlog → 从库 IO 线程拉取 binlog 写入 relay log → 从库 SQL 线程回放。
- 主从延迟解决：强制读主库、半同步复制、并行复制、业务上容忍。

### 12. 慢 SQL 优化思路？
- 开启慢查询日志定位 → `EXPLAIN` 看 type（system > const > eq_ref > ref > range > index > ALL）、key、rows、Extra（Using filesort、Using temporary 要注意）。
- 加合适的索引、避免 `select *`、深分页用"子查询 / 上次 ID"优化、大表拆分、读写分离。

### 13. 分库分表？
- 垂直拆分：按业务拆库、按字段拆表；水平拆分：按 ID 哈希 / 范围分片。
- 带来的问题：分布式 ID（雪花算法）、跨库 join、分布式事务、分页排序。常用 ShardingSphere。

### 14. `char` 和 `varchar`，`count(*)` 和 `count(1)`？
- char 定长、会补空格；varchar 变长，额外 1~2 字节记录长度。
- `count(*)` ≈ `count(1)` > `count(主键)` > `count(字段)`（后者不统计 NULL）。推荐 `count(*)`。

## 二、Redis

### 1. Redis 为什么这么快？
- 基于内存；单线程处理命令，避免上下文切换和锁竞争；IO 多路复用（epoll）；高效的数据结构。
- Redis 6.0 引入多线程处理网络 IO，命令执行仍是单线程。

### 2. 数据类型及底层结构？
- String（SDS）、List（quicklist / listpack）、Hash（listpack / hashtable）、Set（intset / listpack / hashtable）、ZSet（listpack / 跳表 + hashtable）。
- 其他：Bitmap、HyperLogLog（基数统计）、GEO、Stream。

### 3. 持久化：RDB 和 AOF？
- RDB：某时刻的快照，`bgsave` fork 子进程写文件（写时复制），文件小、恢复快，但可能丢失较多数据。
- AOF：记录每条写命令，`appendfsync` 有 always / everysec（默认）/ no，数据更安全，文件大，需要 AOF 重写。
- 混合持久化（4.0+）：AOF 重写时前半段写 RDB，后半段写增量 AOF。

### 4. 过期删除与内存淘汰策略？
- 过期删除：惰性删除（访问时检查）+ 定期删除（随机抽样）。
- 内存淘汰：noeviction（默认）、allkeys-lru、allkeys-lfu、allkeys-random、volatile-lru、volatile-lfu、volatile-random、volatile-ttl。

### 5. 缓存穿透、击穿、雪崩？
- 穿透：查询不存在的数据，请求直达数据库 → 缓存空值、布隆过滤器、参数校验。
- 击穿：热点 key 过期瞬间大量请求打到数据库 → 互斥锁、逻辑过期 / 热点 key 永不过期。
- 雪崩：大量 key 同时过期或 Redis 宕机 → 过期时间加随机值、多级缓存、集群高可用、限流降级。

### 6. 如何保证缓存与数据库一致性？
- 常用 Cache Aside：读时先读缓存，未命中读库再写缓存；写时**先更新数据库，再删除缓存**。
- 删除失败的兜底：重试、消息队列、订阅 binlog（Canal）异步删除；延迟双删。
- 强一致性要求高的场景不适合用缓存。

### 7. Redis 分布式锁？
- `SET key uniqueValue NX PX 30000` 加锁；释放时用 Lua 脚本判断 value 是自己的再删除。
- 问题：锁过期但业务未执行完 → 看门狗自动续期（Redisson）；主从切换导致锁丢失 → RedLock（有争议）。

### 8. 高可用方案？
- 主从复制：全量同步（RDB）+ 增量同步（repl_backlog_buffer）。
- 哨兵 Sentinel：监控、主观 / 客观下线、选举 leader 哨兵、故障转移。
- Cluster：16384 个哈希槽，`CRC16(key) % 16384`，去中心化，支持水平扩展。

### 9. 大 key 和热 key 问题？
- 大 key：value 过大导致阻塞、网络拥塞 → 拆分、使用 `UNLINK` 异步删除、`redis-cli --bigkeys` 排查。
- 热 key：单节点压力大 → 本地缓存、key 加随机后缀分散到多个节点。

## 三、消息队列

### 1. 消息队列的作用？
- 异步、解耦、削峰填谷。
- 带来的问题：系统复杂度增加、可用性依赖 MQ、消息丢失 / 重复 / 顺序、一致性问题。

### 2. Kafka、RocketMQ、RabbitMQ 对比？
- Kafka：吞吐量最高，适合日志、大数据、流处理；功能相对简单。
- RocketMQ：阿里开源，支持事务消息、延迟消息、顺序消息，适合电商业务。
- RabbitMQ：基于 AMQP，路由灵活（Exchange：direct / topic / fanout / headers），延迟低，吞吐量相对较低。

### 3. 如何保证消息不丢失？
- 生产者：同步发送 / 回调确认，失败重试（Kafka `acks=all`）。
- Broker：同步刷盘、多副本（Kafka `min.insync.replicas`）。
- 消费者：业务处理成功后再手动提交 offset / ACK。

### 4. 如何处理重复消费？
- 无法完全避免，需保证**消费幂等**：唯一业务 ID + 数据库唯一索引、Redis `SETNX` 去重、状态机判断、乐观锁版本号。

### 5. 如何保证消息顺序？
- 全局有序：只用一个分区 / 队列（牺牲性能）。
- 局部有序：按业务 key（如订单 ID）哈希到同一分区，消费者单线程或按 key 分组处理。

### 6. 消息积压怎么办？
- 先排查消费者是否故障；临时扩容消费者（分区数也要足够）、新建临时 Topic 扩大分区快速转发；优化消费逻辑（批量、异步）。

### 7. Kafka 为什么吞吐量高？
- 顺序写磁盘、利用 Page Cache、零拷贝（sendfile）、批量发送与压缩、分区并行。

### 8. Kafka 的核心概念？
- Broker、Topic、Partition（并行单位）、Replica（Leader / Follower）、ISR（与 Leader 保持同步的副本集合）、Consumer Group（一个分区只能被组内一个消费者消费）、offset。
- Kafka 3.x 起可用 KRaft 模式替代 ZooKeeper，4.0 已完全移除 ZooKeeper。

### 9. 事务消息 / 分布式事务？
- RocketMQ 事务消息：发送半消息 → 执行本地事务 → 提交或回滚；Broker 会回查未确认的事务状态。
- 分布式事务方案：2PC、TCC、本地消息表、最大努力通知、Seata（AT 模式）。
