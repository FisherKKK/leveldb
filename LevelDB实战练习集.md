# LevelDB实战练习集

## 概述

本文档提供了一系列实战练习，帮助你将理论知识转化为实际技能。练习难度从入门到高级逐步递增。

---

## 目录

1. [入门练习](#入门练习)
2. [进阶练习](#进阶练习)
3. [高级练习](#高级练习)
4. [挑战项目](#挑战项目)
5. [性能测试](#性能测试)

---

## 入门练习

### 练习1：Hello LevelDB

**目标**: 熟悉LevelDB基本操作

```cpp
// exercises/hello_leveldb.cc
#include "leveldb/db.h"
#include <iostream>

int main() {
  // 1. 打开数据库
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;

  leveldb::Status status = leveldb::DB::Open(options, "/tmp/hello_db", &db);

  if (!status.ok()) {
    std::cerr << "Open failed: " << status.ToString() << std::endl;
    return 1;
  }

  // 2. 写入数据
  status = db->Put(leveldb::WriteOptions(), "hello", "world");
  if (!status.ok()) {
    std::cerr << "Put failed: " << status.ToString() << std::endl;
    delete db;
    return 1;
  }

  // 3. 读取数据
  std::string value;
  status = db->Get(leveldb::ReadOptions(), "hello", &value);
  if (status.ok()) {
    std::cout << "hello = " << value << std::endl;
  }

  // 4. 删除数据
  status = db->Delete(leveldb::WriteOptions(), "hello");

  // 5. 关闭数据库
  delete db;

  return 0;
}
```

**任务**:
1. 编译并运行此程序
2. 添加更多键值对
3. 尝试读取不存在的键
4. 查看数据库目录的文件结构

**预期输出**:
```
hello = world
```

---

### 练习2：批量写入

**目标**: 学习使用WriteBatch提高写入性能

```cpp
// exercises/batch_write.cc
#include "leveldb/db.h"
#include <iostream>
#include <chrono>

void SingleWrite(leveldb::DB* db, int count) {
  auto start = std::chrono::high_resolution_clock::now();

  for (int i = 0; i < count; i++) {
    std::string key = "key" + std::to_string(i);
    std::string value = "value" + std::to_string(i);
    db->Put(leveldb::WriteOptions(), key, value);
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "Single write: " << duration.count() << "ms\n";
}

void BatchWrite(leveldb::DB* db, int count) {
  auto start = std::chrono::high_resolution_clock::now();

  leveldb::WriteBatch batch;
  for (int i = 0; i < count; i++) {
    std::string key = "key" + std::to_string(i);
    std::string value = "value" + std::to_string(i);
    batch.Put(key, value);
  }
  db->Write(leveldb::WriteOptions(), &batch);

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "Batch write: " << duration.count() << "ms\n";
}

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;

  leveldb::DB::Open(options, "/tmp/batch_db", &db);

  const int count = 10000;

  SingleWrite(db, count);
  BatchWrite(db, count);

  delete db;
  return 0;
}
```

**任务**:
1. 运行并比较两种写入方式的性能
2. 尝试不同的batch size: 100, 1000, 10000
3. 绘制性能曲线图

**思考题**:
- 为什么批量写入更快？
- 最优的batch size是多少？

---

### 练习3：迭代器使用

**目标**: 掌握迭代器遍历数据

```cpp
// exercises/iterator.cc
#include "leveldb/db.h"
#include <iostream>

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;

  leveldb::DB::Open(options, "/tmp/iterator_db", &db);

  // 写入测试数据
  for (int i = 0; i < 10; i++) {
    db->Put(leveldb::WriteOptions(),
            "key" + std::to_string(i),
            "value" + std::to_string(i));
  }

  // 遍历所有数据
  std::cout << "Forward iteration:\n";
  leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    std::cout << it->key().ToString() << " = "
              << it->value().ToString() << "\n";
  }
  delete it;

  // 范围查询
  std::cout << "\nRange query (key3 to key7):\n";
  it = db->NewIterator(leveldb::ReadOptions());
  for (it->Seek("key3");
       it->Valid() && it->key().ToString() < "key8";
       it->Next()) {
    std::cout << it->key().ToString() << " = "
              << it->value().ToString() << "\n";
  }
  delete it;

  // 反向遍历
  std::cout << "\nBackward iteration:\n";
  it = db->NewIterator(leveldb::ReadOptions());
  for (it->SeekToLast(); it->Valid(); it->Prev()) {
    std::cout << it->key().ToString() << " = "
              << it->value().ToString() << "\n";
  }
  delete it;

  delete db;
  return 0;
}
```

**任务**:
1. 实现Seek功能（查找特定key）
2. 实现Prefix Scan（查找前缀相同的key）
3. 计算遍历100万条记录的时间

---

## 进阶练习

### 练习4：自定义Comparator

**目标**: 学习自定义键比较逻辑

```cpp
// exercises/custom_comparator.cc
#include "leveldb/db.h"
#include "leveldb/comparator.h"
#include <iostream>

// 反向比较器（降序）
class ReverseComparator : public leveldb::Comparator {
 public:
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    // 反向比较
    return a.compare(b);
  }

  const char* Name() const {
    return "ReverseComparator";
  }

  void FindShortestSeparator(std::string* start,
                             const leveldb::Slice& limit) const {
    // 简化实现
  }

  void FindShortSuccessor(std::string* key) const {
    // 简化实现
  }
};

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;
  options.comparator = new ReverseComparator();

  leveldb::Status status = leveldb::DB::Open(options, "/tmp/reverse_db", &db);

  if (!status.ok()) {
    std::cerr << "Open failed: " << status.ToString() << std::endl;
    return 1;
  }

  // 写入数据
  for (int i = 0; i < 5; i++) {
    std::string key = "key" + std::to_string(i);
    db->Put(leveldb::WriteOptions(), key, "value");
  }

  // 遍历（应该看到反向顺序）
  leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    std::cout << it->key().ToString() << "\n";
  }
  delete it;

  delete db;
  delete options.comparator;
  return 0;
}
```

**任务**:
1. 实现一个按数字大小比较的Comparator
2. 实现一个不区分大小写的Comparator
3. 理解Comparator对数据排序的影响

---

### 练习5：Snapshot使用

**目标**: 学习快照隔离

```cpp
// exercises/snapshot.cc
#include "leveldb/db.h"
#include <iostream>
#include <thread>
#include <chrono>

void writer(leveldb::DB* db) {
  for (int i = 0; i < 5; i++) {
    std::string key = "counter";
    std::string value;

    db->Get(leveldb::ReadOptions(), key, &value);
    int count = std::stoi(value);
    count++;

    db->Put(leveldb::WriteOptions(), key, std::to_string(count));

    std::cout << "Writer: counter = " << count << "\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
  }
}

void reader(leveldb::DB* db, const leveldb::Snapshot* snapshot) {
  for (int i = 0; i < 5; i++) {
    std::string value;
    leveldb::ReadOptions options;
    options.snapshot = snapshot;

    db->Get(options, "counter", &value);
    std::cout << "Reader: counter = " << value << " (snapshot)\n";

    std::this_thread::sleep_for(std::chrono::milliseconds(100));
  }
}

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;

  leveldb::DB::Open(options, "/tmp/snapshot_db", &db);

  db->Put(leveldb::WriteOptions(), "counter", "0");

  // 创建快照
  const leveldb::Snapshot* snapshot = db->GetSnapshot();

  // 启动读写线程
  std::thread w(writer, db);
  std::thread r(reader, db, snapshot);

  w.join();
  r.join();

  // 释放快照
  db->ReleaseSnapshot(snapshot);

  delete db;
  return 0;
}
```

**任务**:
1. 运行并观察输出
2. 理解快照隔离的效果
3. 尝试不使用快照的读取，对比差异

---

### 练习6：性能测试工具

**目标**: 实现简单的性能测试工具

```cpp
// exercises/benchmark.cc
#include "leveldb/db.h"
#include <iostream>
#include <chrono>
#include <random>
#include <iomanip>

class Benchmark {
 public:
  Benchmark(const std::string& db_path) : db_path_(db_path) {}

  void SetUp() {
    leveldb::Options options;
    options.create_if_missing = true;
    options.write_buffer_size = 64 * 1024 * 1024;  // 64MB
    options.block_cache = leveldb::NewLRUCache(128 * 1024 * 1024);  // 128MB
    options.filter_policy = leveldb::NewBloomFilterPolicy(10);

    leveldb::Status status = leveldb::DB::Open(options, db_path_, &db_);
    if (!status.ok()) {
      std::cerr << "Open failed: " << status.ToString() << std::endl;
      exit(1);
    }
  }

  void TearDown() {
    delete db_;
  }

  void WriteBenchmark(int num_keys, int key_size, int value_size) {
    std::cout << "\n=== Write Benchmark ===\n";
    std::cout << "Keys: " << num_keys << "\n";
    std::cout << "Key size: " << key_size << " bytes\n";
    std::cout << "Value size: " << value_size << " bytes\n";

    auto start = std::chrono::high_resolution_clock::now();

    leveldb::WriteBatch batch;
    int batch_size = 1000;
    int count = 0;

    for (int i = 0; i < num_keys; i++) {
      std::string key = GenerateKey(i, key_size);
      std::string value = GenerateValue(value_size);

      batch.Put(key, value);

      if (++count >= batch_size) {
        db_->Write(leveldb::WriteOptions(), &batch);
        batch.Clear();
        count = 0;
      }
    }

    if (count > 0) {
      db_->Write(leveldb::WriteOptions(), &batch);
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    double seconds = duration.count() / 1000.0;
    double ops_per_sec = num_keys / seconds;

    std::cout << "Time: " << duration.count() << " ms\n";
    std::cout << "Throughput: " << std::fixed << std::setprecision(2)
              << ops_per_sec << " ops/sec\n";
  }

  void ReadBenchmark(int num_keys, int key_size) {
    std::cout << "\n=== Read Benchmark ===\n";
    std::cout << "Keys: " << num_keys << "\n";

    // 预热
    for (int i = 0; i < num_keys; i++) {
      std::string key = GenerateKey(i, key_size);
      std::string value;
      db_->Get(leveldb::ReadOptions(), key, &value);
    }

    // 测试
    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < num_keys; i++) {
      std::string key = GenerateKey(i, key_size);
      std::string value;
      db_->Get(leveldb::ReadOptions(), key, &value);
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    double seconds = duration.count() / 1000.0;
    double ops_per_sec = num_keys / seconds;

    std::cout << "Time: " << duration.count() << " ms\n";
    std::cout << "Throughput: " << std::fixed << std::setprecision(2)
              << ops_per_sec << " ops/sec\n";
  }

 private:
  std::string GenerateKey(int num, int size) {
    std::string key = "key" + std::to_string(num);
    key.resize(size, '0');
    return key;
  }

  std::string GenerateValue(int size) {
    return std::string(size, 'x');
  }

  std::string db_path_;
  leveldb::DB* db_;
};

int main() {
  Benchmark bench("/tmp/benchmark_db");

  bench.SetUp();

  // 写入测试
  bench.WriteBenchmark(100000, 16, 100);

  // 读取测试
  bench.ReadBenchmark(100000, 16);

  bench.TearDown();

  return 0;
}
```

**任务**:
1. 编译并运行基准测试
2. 尝试不同的参数组合
3. 绘制性能曲线

---

## 高级练习

### 练习7：实现简化版MemTable

**目标**: 理解MemTable的内部结构

```cpp
// exercises/simple_memtable.h
#pragma once
#include <string>
#include <map>

class SimpleMemTable {
 public:
  SimpleMemTable() : sequence_(0) {}

  void Put(const std::string& key, const std::string& value) {
    sequence_++;
    auto internal_key = MakeInternalKey(key, sequence_, kTypeValue);
    table_[internal_key] = value;
  }

  void Delete(const std::string& key) {
    sequence_++;
    auto internal_key = MakeInternalKey(key, sequence_, kTypeDeletion);
    table_[internal_key] = "";
  }

  bool Get(const std::string& key, std::string* value) {
    // 从后向前查找（最新的序列号）
    for (auto it = table_.rbegin(); it != table_.rend(); ++it) {
      auto [user_key, seq, type] = ParseInternalKey(it->first);

      if (user_key == key) {
        if (type == kTypeDeletion) {
          return false;  // 已删除
        }
        *value = it->second;
        return true;
      }

      if (user_key < key) {
        break;  // 不会再有更小的key
      }
    }
    return false;
  }

 private:
  enum ValueType { kTypeDeletion = 0, kTypeValue = 1 };

  std::string MakeInternalKey(const std::string& key,
                              uint64_t seq,
                              ValueType type) {
    std::string result;
    result.reserve(key.size() + 9);
    result.append(key);
    result.append(reinterpret_cast<char*>(&seq), 8);
    result.push_back(static_cast<char>(type));
    return result;
  }

  std::tuple<std::string, uint64_t, ValueType> ParseInternalKey(
      const std::string& internal_key) {
    std::string key = internal_key.substr(0, internal_key.size() - 9);
    uint64_t seq;
    std::memcpy(&seq, internal_key.data() + internal_key.size() - 9, 8);
    ValueType type = static_cast<ValueType>(internal_key.back());
    return {key, seq, type};
  }

  std::map<std::string, std::string> table_;
  uint64_t sequence_;
};
```

**任务**:
1. 实现迭代器
2. 添加序列化功能（可以写入文件）
3. 性能测试：对比std::map和SkipList

---

### 练习8：LRU Cache实现

**目标**: 理解LRU缓存原理

```cpp
// exercises/lru_cache.h
#pragma once
#include <unordered_map>
#include <list>
#include <cstddef>

template<typename K, typename V>
class LRUCache {
 public:
  explicit LRUCache(size_t capacity) : capacity_(capacity) {}

  void Put(const K& key, const V& value) {
    auto it = cache_.find(key);

    if (it != cache_.end()) {
      // 更新现有项
      it->second.second = value;
      lru_list_.erase(it->second.first);
      lru_list_.push_front(key);
      it->second.first = lru_list_.begin();
    } else {
      // 新增项
      if (cache_.size() >= capacity_) {
        // 淘汰最旧的项
        auto oldest = lru_list_.back();
        cache_.erase(oldest);
        lru_list_.pop_back();
      }

      lru_list_.push_front(key);
      cache_[key] = {lru_list_.begin(), value};
    }
  }

  bool Get(const K& key, V* value) {
    auto it = cache_.find(key);

    if (it == cache_.end()) {
      return false;
    }

    // 更新LRU顺序
    lru_list_.erase(it->second.first);
    lru_list_.push_front(key);
    it->second.first = lru_list_.begin();

    *value = it->second.second;
    return true;
  }

  size_t Size() const {
    return cache_.size();
  }

 private:
  using ListType = std::list<K>;
  using CacheItem = std::pair<typename ListType::iterator, V>;
  using CacheMap = std::unordered_map<K, CacheItem>;

  size_t capacity_;
  ListType lru_list_;
  CacheMap cache_;
};

// 测试代码
#include <iostream>

void TestLRUCache() {
  LRUCache<int, std::string> cache(3);

  cache.Put(1, "one");
  cache.Put(2, "two");
  cache.Put(3, "three");

  std::string value;
  if (cache.Get(1, &value)) {
    std::cout << "Found: " << value << "\n";
  }

  cache.Put(4, "four");  // 应该淘汰key 2

  if (!cache.Get(2, &value)) {
    std::cout << "Key 2 was evicted (correct)\n";
  }
}
```

**任务**:
1. 实现线程安全的LRU Cache
2. 添加统计信息（命中率、未命中次数）
3. 对比不同缓存大小的性能

---

### 练习9：Bloom Filter实现

**目标**: 理解Bloom Filter原理

```cpp
// exercises/bloom_filter.h
#pragma once
#include <vector>
#include <string>

class BloomFilter {
 public:
  BloomFilter(size_t bits, size_t num_hashes)
      : bits_(bits), num_hashes_(num_hashes), data_(bits / 8 + 1, 0) {}

  void Add(const std::string& key) {
    for (size_t i = 0; i < num_hashes_; i++) {
      size_t hash = Hash(key, i);
      size_t byte = hash / 8;
      size_t bit = hash % 8;
      data_[byte] |= (1 << bit);
    }
  }

  bool MightContain(const std::string& key) const {
    for (size_t i = 0; i < num_hashes_; i++) {
      size_t hash = Hash(key, i);
      size_t byte = hash / 8;
      size_t bit = hash % 8;
      if (!(data_[byte] & (1 << bit))) {
        return false;
      }
    }
    return true;
  }

 private:
  size_t Hash(const std::string& key, size_t seed) const {
    // 简化的hash函数
    size_t h = seed;
    for (char c : key) {
      h = h * 31 + c;
    }
    return h % bits_;
  }

  size_t bits_;
  size_t num_hashes_;
  std::vector<uint8_t> data_;
};

// 测试误判率
#include <iostream>
#include <random>
#include <set>

void TestBloomFilter() {
  const int num_items = 100000;
  const int bits_per_item = 10;
  const int num_hashes = 7;

  BloomFilter filter(num_items * bits_per_item, num_hashes);
  std::set<std::string> existing;

  // 添加100k个随机key
  std::mt19937 rng(42);
  for (int i = 0; i < num_items; i++) {
    std::string key = "key" + std::to_string(rng());
    filter.Add(key);
    existing.insert(key);
  }

  // 测试误判率
  int false_positives = 0;
  int tests = 10000;

  for (int i = 0; i < tests; i++) {
    std::string key = "key" + std::to_string(rng());

    bool in_filter = filter.MightContain(key);
    bool exists = existing.count(key) > 0;

    if (in_filter && !exists) {
      false_positives++;
    }
  }

  double fp_rate = static_cast<double>(false_positives) / tests;
  std::cout << "False positive rate: " << fp_rate * 100 << "%\n";
  std::cout << "Expected: ~1%\n";
}
```

**任务**:
1. 实现更好的哈希函数（如MurmurHash）
2. 计算理论误判率并与实际对比
3. 研究bits per key对误判率的影响

---

## 挑战项目

### 项目1：简化版LSM-Tree

**难度**: ⭐⭐⭐⭐⭐

**目标**: 从零实现一个简化的LSM-Tree存储引擎

**要求**:
1. MemTable（基于SkipList或std::map）
2. WAL（Write-Ahead Log）
3. SSTable（简化版，无压缩）
4. 两层Compaction（MemTable → L0 → L1）
5. Get/Put/Delete接口
6. 迭代器

**框架**:

```cpp
// exercises/simple_lsm.h
#pragma once
#include "simple_memtable.h"
#include <string>
#include <vector>
#include <memory>

class SimpleLSM {
 public:
  SimpleLSM(const std::string& data_dir);
  ~SimpleLSM();

  // 基本操作
  void Put(const std::string& key, const std::string& value);
  bool Get(const std::string& key, std::string* value);
  void Delete(const std::string& key);

  // 管理操作
  void Flush();
  void Compact();

  // 迭代器
  class Iterator;
  Iterator* NewIterator();

 private:
  std::string data_dir_;
  std::unique_ptr<SimpleMemTable> memtable_;
  std::unique_ptr<SimpleMemTable> immutable_memtable_;

  // SSTable管理
  struct SSTable {
    int level;
    std::string filename;
    // 最小/最大key
    std::string smallest_key;
    std::string largest_key;
  };
  std::vector<SSTable> level0_;
  std::vector<SSTable> level1_;

  // WAL
  int wal_fd_;
};
```

**评分标准**:
- [ ] MemTable正确实现
- [ ] WAL持久化
- [ ] SSTable格式
- [ ] Compaction逻辑
- [ ] 正确性测试通过
- [ ] 性能测试达标

**参考资料**:
- LevelDB源码
- LSM-Tree论文

---

### 项目2：LevelDB监控系统

**难度**: ⭐⭐⭐

**目标**: 实现实时监控系统

**功能**:
1. 实时统计信息展示
2. 性能指标图表
3. 告警功能
4. Web界面

**技术栈**:
- 后端：C++ + HTTP服务器
- 前端：HTML + JavaScript + Chart.js

**界面示例**:

```
┌─────────────────────────────────────┐
│     LevelDB Monitor Dashboard       │
├─────────────────────────────────────┤
│                                     │
│  Throughput (ops/sec)               │
│  ████████░░ 80k write               │
│  ██████████ 100k read               │
│                                     │
│  Latency (ms)                       │
│  Write: p50=2 p99=10                │
│  Read:  p50=1 p99=5                 │
│                                     │
│  Storage                            │
│  Level 0: 3 files, 12 MB            │
│  Level 1: 15 files, 120 MB          │
│  Total: 1.2 GB                      │
│                                     │
│  Cache Hit Ratio: 95.2%             │
│  Write Amplification: 8.5x          │
│                                     │
└─────────────────────────────────────┘
```

---

### 项目3：性能对比工具

**难度**: ⭐⭐⭐

**目标**: 对比不同配置的性能

**功能**:
1. 自动化测试
2. 生成对比报告
3. 可视化图表

**对比维度**:
- write_buffer_size
- block_cache大小
- 有/无Bloom Filter
- 压缩算法

**输出示例**:

```
Configuration Performance Report
=================================

Test: Random Write (1M keys)
Data: key=16B, value=100B

Config | Throughput | Latency p99 | WA
-------|-----------|------------|-----
A      | 50k ops/s | 20ms       | 10x
B      | 80k ops/s | 15ms       | 8x
C      | 30k ops/s | 30ms       | 15x

Recommendation: Config B
```

---

## 性能测试

### 测试1：YCSB基准

使用YCSB（Yahoo! Cloud Serving Benchmark）测试框架：

```bash
# 加载工作负载
./db_bench --benchmarks=fillseq \
           --num=1000000 \
           --value_size=100

# 运行测试
./db_bench --benchmarks=readrandom \
           --num=1000000 \
           --threads=16

# 工作负载类型：
# - readrandom: 随机读
# - writerandom: 随机写
# - readwhilewriting: 读写混合
# - deleteseq: 顺序删除
```

### 测试2：压力测试

```cpp
// exercises/stress_test.cc
void StressTest(int num_threads, int duration_sec) {
  leveldb::DB* db;
  // ... 打开数据库 ...

  std::vector<std::thread> threads;

  auto start = std::chrono::steady_clock::now();

  for (int i = 0; i < num_threads; i++) {
    threads.emplace_back([&, i]() {
      std::mt19937 rng(i);

      while (true) {
        auto now = std::chrono::steady_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(
            now - start).count();
        if (elapsed >= duration_sec) break;

        // 随机操作
        int op = rng() % 3;
        std::string key = "key" + std::to_string(rng() % 100000);

        if (op == 0) {
          db->Put(leveldb::WriteOptions(), key, "value");
        } else if (op == 1) {
          std::string value;
          db->Get(leveldb::ReadOptions(), key, &value);
        } else {
          db->Delete(leveldb::WriteOptions(), key);
        }
      }
    });
  }

  for (auto& t : threads) {
    t.join();
  }

  delete db;
}
```

---

## 总结

本练习集涵盖了从入门到高级的各个方面：

**入门级**:
- Hello LevelDB
- 批量写入
- 迭代器使用

**进阶级**:
- 自定义Comparator
- Snapshot隔离
- 性能测试工具

**高级**:
- MemTable实现
- LRU Cache
- Bloom Filter

**挑战项目**:
- 简化版LSM-Tree
- 监控系统
- 性能对比工具

通过完成这些练习，你将：
1. 掌握LevelDB的核心概念
2. 理解LSM-Tree的工作原理
3. 具备实现类似系统的能力

**继续学习**:
- 阅读LevelDB完整源码
- 对比研究RocksDB
- 参与开源社区

祝你学习愉快！🚀
