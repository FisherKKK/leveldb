# 高级课程：分布式存储与LSM扩展

## 课程概述

本课程讲解LevelDB/LSM-Tree在分布式存储系统中的应用，探讨如何将单机存储引擎扩展到分布式环境，以及业界主流分布式数据库的设计思想。

---

## 学习目标

完成本课程后，你将能够：
- 理解单机存储到分布式的扩展挑战
- 掌握数据分区和分片策略
- 学习分布式一致性协议
- 了解RocksDB的分布式改进
- 理解业界主流系统的架构设计

---

## 目录

1. [从单机到分布式](#1-从单机到分布式)
2. [数据分区策略](#2-数据分区策略)
3. [分布式一致性](#3-分布式一致性)
4. [RocksDB的扩展](#4-rocksdb的扩展)
5. [分布式LSM系统](#5-分布式lsm系统)
6. [性能优化技巧](#6-性能优化技巧)
7. [实战：构建分布式KV](#7-实战构建分布式kv)
8. [业界案例分析](#8-业界案例分析)

---

## 1. 从单机到分布式

### 1.1 为什么需要分布式？

**单机LevelDB的局限**:

```
容量限制：
- 单机磁盘：~10TB
- 内存：~1TB
- QPS：~100k

扩展需求：
- 数据量：PB级别
- 访问量：M QPS
- 可用性：99.99%+
```

**分布式存储的目标**:

1. **水平扩展**：通过增加节点提升容量和性能
2. **高可用性**：节点故障不影响服务
3. **数据一致性**：保证数据的正确性
4. **低成本**：使用廉价硬件

### 1.2 CAP理论

```
Consistency (一致性):  所有节点同时看到相同数据
Availability (可用性): 每个请求都能得到响应
Partition Tolerance (分区容错): 系统在网络分区时仍能运行

CAP定理：在分布式系统中，只能同时满足两项

CA: 单机数据库（如MySQL主从）
CP: 强一致性系统（如HBase, MongoDB）
AP: 最终一致性系统（如Cassandra, Dynamo）
```

**LSM-Tree的定位**:

```
LevelDB: CA (单机，无分区)
RocksDB: CA (单机，无分区)
Cassandra: AP (LSM + 最终一致性)
HBase: CP (LSM + 强一致性)
```

---

## 2. 数据分区策略

### 2.1 范围分区 (Range Partitioning)

**原理**: 将key排序后按范围分配到不同节点

```cpp
// 范围分区示例
// Node 1: [a, g)
// Node 2: [g, n)
// Node 3: [n, z)

struct RangePartition {
  std::string start;
  std::string end;
  int node_id;
};

std::vector<RangePartition> partitions = {
  {"",    "g",  0},
  {"g",   "n",  1},
  {"n",   "z",  2},
  {"z",   "~",  3}
};

int FindNode(const std::string& key) {
  for (const auto& p : partitions) {
    if (key >= p.start && key < p.end) {
      return p.node_id;
    }
  }
  return -1;
}
```

**优点**:
- 范围查询高效
- 数据局部性好

**缺点**:
- 热点问题（某些key访问频繁）
- 负载不均衡
- 节点增删需要大量数据迁移

**应用**: HBase, Bigtable

### 2.2 哈希分区 (Hash Partitioning)

**原理**: 对key进行哈希，取模分配

```cpp
// 哈希分区示例
int num_nodes = 10;
std::hash<std::string> hasher;

int FindNode(const std::string& key) {
  size_t hash = hasher(key);
  return hash % num_nodes;
}
```

**优点**:
- 数据分布均匀
- 负载均衡
- 简单实现

**缺点**:
- 范围查询需要访问所有节点
- 节点增删需要大量数据迁移

**应用**: Memcached, Redis Cluster

### 2.3 一致性哈希 (Consistent Hashing)

**原理**: 将节点和数据都映射到哈希环上

```cpp
// 一致性哈希实现
class ConsistentHash {
 public:
  void AddNode(int node_id, int num_virtual_nodes = 100) {
    std::hash<std::string> hasher;
    for (int i = 0; i < num_virtual_nodes; i++) {
      std::string vnode = "node" + std::to_string(node_id) +
                         "-" + std::to_string(i);
      size_t hash = hasher(vnode);
      ring_[hash] = node_id;
    }
  }

  void RemoveNode(int node_id) {
    auto it = ring_.begin();
    while (it != ring_.end()) {
      if (it->second == node_id) {
        it = ring_.erase(it);
      } else {
        ++it;
      }
    }
  }

  int FindNode(const std::string& key) {
    if (ring_.empty()) return -1;

    std::hash<std::string> hasher;
    size_t hash = hasher(key);

    auto it = ring_.lower_bound(hash);
    if (it == ring_.end()) {
      it = ring_.begin();  // 环绕
    }
    return it->second;
  }

 private:
  std::map<size_t, int> ring_;  // hash -> node_id
};
```

**优点**:
- 最小化节点增删时的数据迁移
- 平衡负载

**缺点**:
- 实现复杂
- 仍需虚拟节点平衡负载

**应用**: Dynamo, Cassandra, Riak

**虚拟节点的作用**:

```
无虚拟节点（不均匀）:
Node A: ████████████████ 50%
Node B: ██████ 25%
Node C: ██████ 25%

有虚拟节点（均匀）:
Node A (100个虚拟): ████████████ 33%
Node B (100个虚拟): ████████████ 33%
Node C (100个虚拟): ████████████ 33%
```

---

## 3. 分布式一致性

### 3.1 复制策略

**主从复制 (Master-Slave)**:

```
写入流程：
Client → Master → Slave1, Slave2, ...

读取策略：
- 主读：强一致性，延迟高
- 从读：最终一致性，延迟低
```

**多主复制 (Multi-Master)**:

```
多个Master都可以写入

挑战：
- 冲突解决
- 数据一致性
- 复杂度高
```

**无主复制 (Leaderless)**:

```
客户端写入多个节点

写入参数：
- W: 写入副本数
- R: 读取副本数
- N: 总副本数

一致性保证：
- R + W > N: 强一致性
- R + W <= N: 最终一致性
```

### 3.2 一致性协议

**两阶段提交 (2PC)**:

```
阶段1：准备
Coordinator → Participant: prepare
Participant → Coordinator: ready/abort

阶段2：提交/回滚
Coordinator → Participant: commit/abort
```

**问题**:
- 阻塞协议
- 单点故障
- 性能差

**Paxos**:

```
基本流程：
1. Proposer提出提案
2. Acceptor接受/拒绝
3. Learner学习结果

保证：
- 安全性：只有一个值被选定
- 活性：只要多数节点可达就能选定值
```

**Raft** (更易理解的协议):

```
角色：
- Leader: 处理所有写入
- Follower: 接收Leader的复制
- Candidate: 选举时的临时角色

Term (任期):
- 逻辑时钟
- 每次选举递增

日志复制：
1. Leader接收写入
2. 追加到本地日志
3. 并行复制到Follower
4. 多数确认后提交
```

**Raft与LSM-Tree结合**:

```cpp
// 在Raft之上构建LSM-Tree
class RaftLSMNode {
 public:
  // 写入流程
  void Put(const std::string& key, const std::string& value) {
    if (role_ != LEADER) {
      return RedirectToLeader(key, value);
    }

    // 1. 追加到Raft日志
    LogEntry entry;
    entry.type = PUT;
    entry.key = key;
    entry.value = value;

    AppendLog(entry);

    // 2. 等待多数确认
    WaitForCommit(entry.index);

    // 3. 应用到本地LSM
    ApplyToLSM(entry);
  }

  // 日志应用
  void ApplyToLSM(const LogEntry& entry) {
    switch (entry.type) {
      case PUT:
        lsm_->Put(entry.key, entry.value);
        break;
      case DELETE:
        lsm_->Delete(entry.key);
        break;
    }
  }

 private:
  NodeRole role_;
  uint64_t current_term_;
  RaftLog raft_log_;
  LevelDB* lsm_;  // 本地LSM存储
};
```

---

## 4. RocksDB的扩展

### 4.1 RocksDB vs LevelDB

| 特性 | LevelDB | RocksDB |
|------|---------|---------|
| 开发者 | Google | Facebook |
| 并发写入 | 单线程 | 多线程 |
| Compaction | 单线程 | 多线程 |
| Column Family | 不支持 | 支持 |
| 压缩 | Snappy | 多种 |
| Merge Operator | 不支持 | 支持 |
| 事务 | 不支持 | 支持 |
| 备份检查点 | 基础 | 增强 |

### 4.2 Column Family

**概念**: 逻辑隔离的多个键空间

```cpp
// RocksDB Column Family示例
#include "rocksdb/db.h"

rocksdb::DB* db;
rocksdb::Options options;

options.create_if_missing = true;

// 创建多个Column Family
std::vector<rocksdb::ColumnFamilyDescriptor> column_families;
column_families.push_back(
    rocksdb::ColumnFamilyDescriptor(
        rocksdb::kDefaultColumnFamilyName,
        rocksdb::ColumnFamilyOptions()));
column_families.push_back(
    rocksdb::ColumnFamilyDescriptor(
        "user_cf",
        rocksdb::ColumnFamilyOptions()));
column_families.push_back(
    rocksdb::ColumnFamilyDescriptor(
        "order_cf",
        rocksdb::ColumnFamilyOptions()));

// 打开数据库
std::vector<rocksdb::ColumnFamilyHandle*> handles;
rocksdb::Status status = rocksdb::DB::Open(
    options,
    "/tmp/rocksdb_cf",
    column_families,
    &handles,
    &db);

// 写入不同CF
rocksdb::WriteOptions wopts;
db->Put(wopts, handles[0], "key1", "value1");  // default
db->Put(wopts, handles[1], "user1", "data1");  // user_cf
db->Put(wopts, handles[2], "order1", "data1"); // order_cf
```

**优势**:
- 逻辑隔离（不同业务）
- 独立Compaction（避免相互影响）
- 不同配置（压缩、缓存）

### 4.3 Merge Operator

**用途**: 支持原子的读-修改-写操作

```cpp
// 自定义Merge Operator
class CountMergeOperator : public rocksdb::MergeOperator {
 public:
  bool FullMerge(const rocksdb::Slice& key,
                 const rocksdb::Slice* existing_value,
                 const std::deque<std::string>& operands,
                 std::string* new_value,
                 rocksdb::Logger* logger) const override {

    int count = existing_value ?
        std::stoi(existing_value->ToString()) : 0;

    for (const auto& op : operands) {
      count += std::stoi(op);
    }

    *new_value = std::to_string(count);
    return true;
  }

  const char* Name() const override {
    return "CountMergeOperator";
  }
};

// 使用Merge
options.merge_operator = std::make_shared<CountMergeOperator>();

// 多次Merge（非阻塞）
db->Merge(wopts, "counter", "1");  // +1
db->Merge(wopts, "counter", "1");  // +1
db->Merge(wopts, "counter", "1");  // +1

// 读取时合并
std::string value;
db->Get(ropts, "counter", &value);
// value = "3"
```

### 4.4 事务支持

```cpp
// RocksDB事务
rocksdb::WriteOptions wopts;
rocksdb::TransactionOptions txn_options;

rocksdb::Transaction* txn = db->BeginTransaction(wopts, txn_options);

// 多个操作
txn->Put("key1", "value1");
txn->Put("key2", "value2");
txn->Delete("key3");

// 提交或回滚
rocksdb::Status s = txn->Commit();
if (!s.ok()) {
  txn->Rollback();
}

delete txn;
```

---

## 5. 分布式LSM系统

### 5.1 Cassandra架构

**数据模型**:
```
Keyspace (类似数据库)
  └── Column Family (表)
        └── Row (行)
              └── Column (列)
```

**分区策略**: 一致性哈希 + 虚拟节点

**复制**:
- 无主复制
- 可配置复制因子
- Snitch机制（感知拓扑）

**读取路径**:
```
1. 客户端请求Coordinator节点
2. Coordinator根据一致性哈希确定副本
3. 并行查询多个副本
4. 返回最新数据（根据timestamp）
```

**写入路径**:
```
1. 写入CommitLog (WAL)
2. 写入MemTable
3. MemTable满时刷盘为SSTable
4. 后台Compaction
```

### 5.2 HBase架构

```
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   ZooKeeper │  │ 元数据管理
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼────┐       ┌────▼────┐       ┌────▼────┐
   │RegionServer│    │RegionServer│    │RegionServer│
   └────┬────┘       └────┬────┘       └────┬────┘
        │                  │                  │
   ┌────▼────┐       ┌────▼────┐       ┌────▼────┐
   │ Region  │       │ Region  │       │ Region  │
   │ (LSM)   │       │ (LSM)   │       │ (LSM)   │
   └────┬────┘       └────┬────┘       └────┬────┘
        │                  │                  │
   ┌────▼────┐       ┌────▼────┐       ┌────▼────┐
   │  HDFS   │       │  HDFS   │       │  HDFS   │
   └─────────┘       └─────────┘       └─────────┘
```

**特点**:
- CP系统（强一致性）
- LSM-Tree存储
- HDFS作为底层存储
- Region自动分裂

### 5.3 RocksDB在TiKV中的应用

**TiKV架构**:
```
Client
  │
  ▼
PD (Placement Driver) - 集群管理
  │
  ▼
TiKV节点
  ├── Region 1 (RocksDB)
  ├── Region 2 (RocksDB)
  └── Region 3 (RocksDB)
  ├── Raft层
  └── RMS (RocksDB)
```

**分层架构**:
```
SQL Layer (TiDB)
    │
    ▼
Key-Value Layer (TiKV)
    │
    ▼
Storage Engine (RocksDB)
```

---

## 6. 性能优化技巧

### 6.1 分区优化

**热点数据处理**:

```cpp
// 1. 添加随机前缀
std::string AddHotspotPrefix(const std::string& key, int buckets) {
  int bucket = std::hash<std::string>{}(key) % buckets;
  return std::to_string(bucket) + "_" + key;
}

// 2. 写入时分桶
Put("1_user123", data);
Put("2_user123", data);  // 同一个key的不同分桶
Put("3_user123", data);

// 3. 读取时扫描所有分桶
for (int i = 0; i < buckets; i++) {
  std::string key = std::to_string(i) + "_user123";
  Get(key, &value);
}
```

### 6.2 Compaction优化

**分层Compaction**:

```cpp
// RocksDB Universal Compaction
options.compaction_style = rocksdb::kCompactionStyleUniversal;

// 优势：
// - 减少写放大
// - 适合写多读少场景

// Tiered vs Leveled:
// Tiered: 相似大小文件合并 (Cassandra风格)
// Leveled: 层级大小指数增长 (LevelDB/RocksDB风格)
```

### 6.3 布隆滤波器优化

**分区布隆滤波器**:

```cpp
// 按key前缀分区布隆滤波器
class PartitionedBloomFilter {
 public:
  PartitionedBloomFilter(int num_partitions) {
    for (int i = 0; i < num_partitions; i++) {
      filters_.push_back(
          std::make_unique<BloomFilter>(bits_per_key_));
    }
  }

  void Add(const std::string& key) {
    int partition = GetPartition(key);
    filters_[partition]->Add(key);
  }

  bool MightContain(const std::string& key) {
    int partition = GetPartition(key);
    return filters_[partition]->MightContain(key);
  }

 private:
  int GetPartition(const std::string& key) {
    return std::hash<std::string>{}(key) % filters_.size();
  }

  std::vector<std::unique_ptr<BloomFilter>> filters_;
};
```

---

## 7. 实战：构建分布式KV

### 7.1 系统设计

**架构**:

```
┌───────────┐
│  Client   │
└─────┬─────┘
      │
┌─────▼───────────────────┐
│  Router (一致性哈希)     │
└─────┬───────────────────┘
      │
      ├────────┬────────┬────────
      │        │        │
  ┌───▼───┐┌──▼───┐┌──▼───┐
  │Node 0 ││Node 1││Node 2│
  │LevelDB││LevelDB││LevelDB│
  └───────┘└──────┘└──────┘
```

### 7.2 路由层实现

```cpp
// distributed/router.h
#pragma once
#include "leveldb/db.h"
#include <vector>
#include <memory>

class ConsistentHashRouter {
 public:
  void AddNode(const std::string& addr) {
    // 一致性哈希添加节点
    hash_ring_.AddNode(addr);
    nodes_[addr] = OpenLevelDB("/data/" + addr);
  }

  std::string Get(const std::string& key) {
    std::string target_node = hash_ring_.FindNode(key);
    return nodes_[target_node]->Get(key);
  }

  void Put(const std::string& key, const std::string& value) {
    std::string target_node = hash_ring_.FindNode(key);
    nodes_[target_node]->Put(key, value);
  }

 private:
  leveldb::DB* OpenLevelDB(const std::string& path) {
    leveldb::DB* db;
    leveldb::Options options;
    options.create_if_missing = true;
    leveldb::DB::Open(options, path, &db);
    return db;
  }

  ConsistentHash hash_ring_;
  std::unordered_map<std::string, leveldb::DB*> nodes_;
};
```

### 7.3 复制层实现

```cpp
// distributed/replicated_kv.h
#pragma once
#include "router.h"

class ReplicatedKVStore {
 public:
  ReplicatedKVStore(int replication_factor = 3)
      : replication_factor_(replication_factor) {}

  void Put(const std::string& key, const std::string& value) {
    // 找到所有副本节点
    auto replicas = FindReplicas(key);

    // 并行写入
    std::vector<std::future<leveldb::Status>> futures;
    for (const auto& node : replicas) {
      futures.push_back(std::async(std::launch::async,
          [&node, &key, &value]() {
            return node->Put(key, value);
      }));
    }

    // 等待多数确认
    int success = 0;
    for (auto& f : futures) {
      if (f.get().ok()) success++;
    }

    if (success < replication_factor_ / 2 + 1) {
      throw std::runtime_error("Write failed: insufficient replicas");
    }
  }

  std::string Get(const std::string& key) {
    auto replicas = FindReplicas(key);

    // 读取任意副本
    for (const auto& node : replicas) {
      std::string value;
      if (node->Get(key, &value).ok()) {
        return value;
      }
    }

    throw std::runtime_error("Key not found");
  }

 private:
  std::vector<leveldb::DB*> FindReplicas(const std::string& key) {
    // 在一致性哈希环上顺时针找N个节点
    std::vector<leveldb::DB*> replicas;
    std::string node = hash_ring_.FindNode(key);

    for (int i = 0; i < replication_factor_; i++) {
      replicas.push_back(nodes_[node]);
      node = hash_ring_.GetNextNode(node);
    }

    return replicas;
  }

  int replication_factor_;
  ConsistentHashRouter hash_ring_;
  std::unordered_map<std::string, leveldb::DB*> nodes_;
};
```

---

## 8. 业界案例分析

### 案例1：Facebook RocksDB

**背景**:
- MySQL InnoDB在SSD上性能不佳
- 需要更好的压缩率和写放大

**解决方案**:
- 基于LevelDB开发RocksDB
- 多线程Compaction
- Column Family隔离
- 适配SSD特性

**效果**:
- 写放大降低50%
- 压缩比提升30%
- 支持更大规模数据

### 案例2：Google Bigtable

**背景**:
- 需要PB级存储
- 亿级QPS
- 毫秒级延迟

**架构**:
```
Bigtable
  ├── Tablet (数据分片)
  │   ├── SSTable (LSM-Tree)
  │   └── CommitLog (GFS)
  ├── Chubby (锁服务)
  └── GFS (底层存储)
```

**特点**:
- LSM-Tree存储
- 范围分区
- Master协调
- Chubby保证一致性

### 案例3：Apache Cassandra

**背景**:
- 需要多数据中心
- 高可用性
- 最终一致性可接受

**设计**:
- 无中心架构
- 一致性哈希
- Gossip协议
- 可调一致性

**数据写入**:
```
1. 客户端连接任意节点
2. 节点根据一致性哈希确定副本位置
3. 写入所有副本（异步或同步）
4. 返回确认
```

---

## 总结

本课程深入学习了：

1. ✅ **分布式扩展挑战**：CAP、一致性、分区
2. ✅ **分区策略**：范围、哈希、一致性哈希
3. ✅ **一致性协议**：2PC、Paxos、Raft
4. ✅ **RocksDB扩展**：Column Family、Merge、事务
5. ✅ **分布式LSM系统**：Cassandra、HBase、TiKV
6. ✅ **性能优化**：热点、Compaction、布隆滤波器
7. ✅ **实战构建**：路由、复制
8. ✅ **业界案例**：Facebook、Google、Apache

**关键要点**:
- 分布式需要权衡一致性、可用性、分区容错
- 一致性哈希是最常用的分区策略
- Raft是理解分布式一致性的最佳起点
- LSM-Tree在分布式系统中有广泛应用
- RocksDB是LevelDB的超集，适合更复杂场景

**进一步学习**:
- Raft论文：[In Search of an Understandable Consensus Algorithm](https://raft.github.io/)
- Dynamo论文：[Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- Bigtable论文：[Bigtable: A Distributed Storage System](https://research.google/pubs/pub27898/)
- 实践：使用RocksDB构建分布式KV

**思考题**:
1. 如何在保证一致性的同时提高可用性？
2. LSM-Tree在分布式环境下有哪些新的挑战？
3. 如何设计一个支持多数据中心的KV系统？
4. 节点故障时如何保证数据不丢失？

---

## 参考资料

### 论文
1. LSM-Tree: "The Log-Structured Merge-Tree"
2. Dynamo: Amazon's Dynamo
3. Bigtable: Google's Bigtable
4. Raft: In Search of an Understandable Consensus Algorithm
5. Snowflake: Snowflake: A Linearizable Key-Value Store

### 开源项目
1. [RocksDB](https://github.com/facebook/rocksdb)
2. [Cassandra](https://cassandra.apache.org/)
3. [HBase](https://hbase.apache.org/)
4. [TiKV](https://github.com/tikv/tikv)

### 书籍
1. "Designing Data-Intensive Applications" - Martin Kleppmann
2. "Database Internals" - Alex Petrov
3. "Distributed Systems" - Maarten van Steen & Andrew S. Tanenbaum
