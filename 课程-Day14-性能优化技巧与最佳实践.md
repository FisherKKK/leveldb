# Day 14: 性能优化技巧与最佳实践

## 学习目标
- 掌握参数调优指南
- 学习性能瓶颈分析方法
- 理解生产环境配置
- 了解常见问题解决方案

---

## 目录

1. [参数调优](#1-参数调优)
2. [性能分析](#2-性能分析)
3. [生产环境配置](#3-生产环境配置)
4. [常见问题](#4-常见问题)
5. [最佳实践](#5-最佳实践)
6. [真实生产案例](#6-真实生产案例)
7. [监控与告警](#7-监控与告警)
8. [容量规划](#8-容量规划)
9. [灾难恢复](#9-灾难恢复)
10. [性能调优实战](#10-性能调优实战)

---

## 1. 参数调优

### 1.1 关键参数详解

```cpp
Options options;

// 1. write_buffer_size (默认4MB)
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB

// 影响分析：
// - 越大：
//   ✓ 减少L0文件数量
//   ✓ 减少Minor Compaction频率
//   ✓ 减少写放大
//   ✗ 增加内存使用
//   ✗ 崩溃恢复时间更长
//
// - 越小：
//   ✓ 内存使用少
//   ✓ 恢复快
//   ✗ 更多L0文件
//   ✗ 更多Compaction
//
// 建议：
// - 写密集：64-128MB
// - 读密集：16-32MB
// - 内存受限：8-16MB

// 2. max_file_size (默认2MB)
options.max_file_size = 4 * 1024 * 1024;  // 4MB

// 影响分析：
// - 越大：
//   ✓ 减少文件数量
//   ✓ 减少打开文件数
//   ✗ Compaction粒度变大
//   ✗ 读取可能浪费
//
// - 越小：
//   ✓ Compaction更细粒度
//   ✓ 更精确的读取范围
//   ✗ 文件数量多
//   ✗ 管理开销大
//
// 建议：2-10MB

// 3. block_size (默认4KB)
options.block_size = 16 * 1024;  // 16KB

// 影响分析：
// - 越大：
//   ✓ 压缩率更好
//   ✓ 元数据开销小
//   ✗ 读取更多无用数据
//   ✗ 缓存效率低
//
// - 越小：
//   ✓ 精确读取
//   ✓ 缓存效率高
//   ✗ 压缩率低
//   ✗ 索引更大
//
// 建议：
// - 大值：16-32KB
// - 小值：4-8KB
// - 点查询多：8KB

// 4. block_cache (默认8MB)
options.block_cache = NewLRUCache(512 * 1024 * 1024);  // 512MB

// 缓存大小选择：
// - 工作集大小：缓存应覆盖热数据
// - 可用内存：通常是总内存的20-40%
// - 命中率目标：>90%
//
// 计算：
// working_set = num_hot_keys * (avg_key_size + avg_value_size)
// cache_size = working_set * 1.2  // 留20%余量

// 5. max_open_files (默认1000)
options.max_open_files = 5000;

// 影响：
// - 文件句柄缓存
// - 超过限制使用OS缓存
//
// 建议：
// - 检查ulimit -n
// - 设置为ulimit的80%

// 6. filter_policy (默认nullptr)
options.filter_policy = NewBloomFilterPolicy(10);  // 10 bits/key

// bits per key选择：
// - 5：   误判率~3.5%，  空间开销小
// - 10：  误判率~1%，   推荐
// - 20：  误判率~0.01%，高精度
//
// 性能提升：
// - 无Bloom Filter：每次Get需读取所有SSTable
// - 有Bloom Filter：99%查询跳过文件

// 7. compression (默认kSnappyCompression)
options.compression = kSnappyCompression;

// 压缩算法对比：
// Algorithm | Ratio | Speed | CPU
// ----------|-------|-------|-----
// None      | 1.0x  | 100%  | 0%
// Snappy    | 2-3x  | 80%   | 低
// Zstd      | 3-5x  | 30%   | 中
// LZ4       | 2-3x  | 90%   | 低

// 8. write_buffer_number (默认2)
options.max_write_buffer_number = 3;  // 最多3个MemTable

// 触发条件：
// - 当有2个Immutable MemTable时暂停写入
// - 等待Compaction完成

// 9. min_write_buffer_number_to_merge (默认1)
options.min_write_buffer_number_to_merge = 2;

// 含义：
// - 积累2个Immutable MemTable后一起刷盘
// - 减少L0文件数量
// - 增加Compaction粒度
```

### 1.2 场景优化配置

**场景1：高吞吐写入（日志、消息队列）**

```cpp
Options options;

// 目标：最大化写入吞吐
// 权衡：牺牲读性能、空间

options.write_buffer_size = 128 * 1024 * 1024;     // 大MemTable
options.max_write_buffer_number = 4;               // 允许更多积压
options.min_write_buffer_number_to_merge = 1;      // 立即刷盘
options.max_file_size = 16 * 1024 * 1024;          // 大文件
options.compression = kNoCompression;              // 不压缩
options.filter_policy = nullptr;                   // 不用Bloom Filter
options.max_open_files = 10000;                    // 多文件句柄

WriteOptions write_opts;
write_opts.sync = false;                           // 异步写入
write_opts.disableWAL = true;                      // 禁用WAL（极致性能）

// 预期性能：
// - 吞吐：100k-500k ops/sec
// - 延迟：1-5ms (p99)
// - 写放大：5-10x
```

**场景2：低延迟读取（缓存、索引）**

```cpp
Options options;

// 目标：最小化读延迟
// 权衡：空间、写放大

options.write_buffer_size = 32 * 1024 * 1024;      // 中等MemTable
options.block_size = 4 * 1024;                     // 小Block
options.block_cache = NewLRUCache(4LL * 1024 * 1024 * 1024);  // 4GB缓存
options.filter_policy = NewBloomFilterPolicy(10);  // Bloom Filter
options.compression = kSnappyCompression;          // Snappy
options.max_open_files = 10000;

ReadOptions read_opts;
read_opts.verify_checksums = false;                // 跳过校验（生产）
read_opts.fill_cache = true;                       // 填充缓存

// 预期性能：
// - 延迟：0.1-1ms (p99)
// - QPS：50k-200k
// - 缓存命中率：>95%
```

**场景3：空间受限（嵌入式、边缘设备）**

```cpp
Options options;

// 目标：最小化磁盘占用
// 权衡：CPU、性能

options.write_buffer_size = 8 * 1024 * 1024;       // 小MemTable
options.block_cache = NewLRUCache(32 * 1024 * 1024);  // 32MB缓存
options.compression = kZstdCompression;            // 最佳压缩
options.filter_policy = NewBloomFilterPolicy(5);   // 小Bloom Filter
options.max_open_files = 500;                      // 少文件

// 预期效果：
// - 压缩比：3-5x
// - 空间节省：60-80%
// - 性能下降：30-50%
```

**场景4：数据归档（冷数据存储）**

```cpp
Options options;

// 目标：高压缩比、低成本
// 权衡：写性能

options.write_buffer_size = 64 * 1024 * 1024;
options.max_file_size = 32 * 1024 * 1024;          // 超大文件
options.compression = kZstdCompression;            // Zstd
options.block_cache = NewLRUCache(64 * 1024 * 1024);  // 小缓存
options.filter_policy = NewBloomFilterPolicy(5);   // 粗糙Filter

// 定期Compaction
db->CompactRange(nullptr, nullptr);

// 预期效果：
// - 压缩比：5-10x
// - 存储成本：降低80%
```

---

## 2. 性能分析

### 2.1 性能监控工具

```cpp
// 详细监控类
class LevelDBMonitor {
 public:
  LevelDBMonitor(DB* db) : db_(db) {}

  void PrintStats() {
    std::string stats;
    db_->GetProperty("leveldb.stats", &stats);

    std::string ssdtables;
    db_->GetProperty("leveldb.sstables", &ssdtables);

    std::string smemory;
    db_->GetProperty("leveldb.approximate-memory-usage", &smemory);

    std::cout << "=== LevelDB Statistics ===\n";
    std::cout << stats << "\n";
    std::cout << "SSTables: " << ssdtables << "\n";
    std::cout << "Memory: " << smemory << " bytes\n";

    ParseCompactionStats(stats);
  }

  void ParseCompactionStats(const std::string& stats) {
    // 解析Compaction统计
    // 格式：
    //   Level  Files Size(MB) Time(sec) Read(MB) Write(MB)
    //   0        3        6         5       13        13
    //   1       17       20        10       50        48

    std::istringstream iss(stats);
    std::string line;
    while (std::getline(iss, line)) {
      // 解析每层统计
      // 计算写放大
    }
  }

  double GetWriteAmplification() {
    // 写放大 = 总写入 / 用户写入
    // 从stats中读取
    std::string stats;
    db_->GetProperty("leveldb.stats", &stats);
    // 解析并计算
    return calculated_wa;
  }

 private:
  DB* db_;
};
```

### 2.2 perf性能分析

```bash
# CPU性能分析
perf record -F 99 -p $(pidof db_bench) -g -- sleep 60
perf report

# 火焰图
perf script | ./FlameGraph/stackcollapse-perf.pl | \
  ./FlameGraph/flamegraph.pl > flamegraph.svg

# 典型热点：
# - MemTable插入
# - Block解压
# - SkipList查找
# - Bloom Filter计算
```

### 2.3 iostat I/O分析

```bash
# 监控I/O
iostat -x 1

# 关键指标：
# %util:    设备利用率
# await:    平均I/O等待（ms）
# r/s, w/s: 读写次数
# rkB/s, wkB/s: 读写吞吐

# 分析：
# - %util > 80%: I/O瓶颈
# - await > 20ms: 延迟高
# - w/s高但wkB/s低: 随机写入
```

---

## 3. 生产环境配置

### 3.1 通用生产配置

```cpp
// 生产环境最佳实践
Options GetProductionOptions(const ProductionConfig& config) {
  Options options;

  // 基础配置
  options.create_if_missing = true;
  options.error_if_exists = false;
  options.paranoid_checks = true;    // 生产环境启用

  // 性能配置（根据机器配置调整）
  int64_t avail_mem = GetAvailableMemory();

  // MemTable: 可用内存的5-10%
  options.write_buffer_size = std::min(
      int64_t(64 * 1024 * 1024),
      avail_mem / 10);

  // Block Cache: 可用内存的20-30%
  int64_t cache_size = avail_mem / 4;
  options.block_cache = NewLRUCache(cache_size);

  // 其他配置
  options.max_file_size = 2 * 1024 * 1024;      // 2MB
  options.block_size = 16 * 1024;               // 16KB
  options.max_open_files = std::min(5000, GetMaxFiles());

  // 优化配置
  options.filter_policy = NewBloomFilterPolicy(10);
  options.compression = kSnappyCompression;

  // 写缓冲配置
  options.max_write_buffer_number = 3;
  options.min_write_buffer_number_to_merge = 2;

  return options;
}
```

### 3.2 关键指标监控

```cpp
// 监控指标收集
struct DBMetrics {
  // 吞吐量
  double write_qps;       // 写QPS
  double read_qps;        // 读QPS

  // 延迟
  double write_p50;       // 写p50延迟
  double write_p99;       // 写p99延迟
  double read_p50;        // 读p50延迟
  double read_p99;        // 读p99延迟

  // 存储
  int64_t db_size;        // DB大小
  int64_t mem_usage;      // 内存使用

  // Compaction
  double compaction_bytes_read;    // 读取字节数
  double compaction_bytes_written; // 写入字节数
  double write_amplification;      // 写放大

  // 各层统计
  struct LevelStats {
    int num_files;
    int64_t size_bytes;
    double compaction_time_sec;
  };
  LevelStats levels[7];
};

DBMetrics CollectMetrics(DB* db) {
  DBMetrics metrics;

  // 收集统计
  std::string stats;
  db->GetProperty("leveldb.stats", &stats);
  // 解析stats填充metrics

  return metrics;
}
```

---

## 4. 常见问题

### 4.1 写入慢

```
问题症状：
- 写入吞吐量 < 10k ops/sec
- p99延迟 > 100ms
- CPU使用率低

诊断步骤：
1. 检查L0文件数
   > db->GetProperty("leveldb.stats", &stats)
   > Level 0文件数 > 4：触发流控

2. 检查Compaction状态
   > Compaction时间过长
   > 写入等待Compaction

3. 检查sync选项
   > sync=true: fsync延迟

解决方案：

方案1：增大MemTable（推荐）
options.write_buffer_size = 128 * 1024 * 1024;

方案2：异步写入
WriteOptions wopts;
wopts.sync = false;

方案3：批量写入
WriteBatch batch;
for (int i = 0; i < 1000; i++) {
  batch.Put(key, value);
}
db->Write(wopts, &batch);

方案4：禁用WAL（慎用）
WriteOptions wopts;
wopts.disableWAL = true;

性能提升：
- 方案1: 2-5x
- 方案2: 10-100x
- 方案3: 5-10x
- 方案4: 2-3x
```

### 4.2 读取慢

```
问题症状：
- 读p99延迟 > 10ms
- 缓存命中率 < 50%
- 磁盘I/O高

诊断步骤：
1. 检查缓存命中率
   > 小缓存或数据量大

2. 检查Bloom Filter
   > 未启用：每次读所有文件

3. 检查L0文件数
   > L0文件多：需查找所有文件

解决方案：

方案1：增大缓存
options.block_cache = NewLRUCache(2GB);

方案2：启用Bloom Filter
options.filter_policy = NewBloomFilterPolicy(10);

方案3：预热缓存
Iterator* it = db->NewIterator(ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // 遍历预热
}
delete it;

方案4：手动Compaction
db->CompactRange(nullptr, nullptr);

性能提升：
- 方案1: 5-20x（缓存命中）
- 方案2: 10-100x（跳过文件）
- 方案3: 2-5x
- 方案4: 2-10x
```

### 4.3 空间增长快

```
问题症状：
- 磁盘使用 > 数据大小3倍
- 写放大 > 20x
- 空间回收慢

诊断步骤：
1. 检查压缩
   > 未压缩：空间大

2. 检查Snapshot
   > 旧Snapshot阻止清理

3. 检查Compaction
   > Compaction慢

解决方案：

方案1：启用压缩
options.compression = kSnappyCompression;

方案2：释放Snapshot
db->ReleaseSnapshot(snapshot);

方案3：手动Compaction
db->CompactRange(nullptr, nullptr);

方案4：调整Compaction参数
options.write_buffer_size = 64 * 1024 * 1024;

空间节省：
- 方案1: 50-70%
- 方案2: 5-20%
- 方案3: 10-30%
- 方案4: 减少20-50%写放大
```

### 4.4 Compaction影响性能

```
问题症状：
- 前台操作卡顿
- CPU/I/O高
- 延迟抖动

原因：
- 后台Compaction竞争资源
- 大Compaction阻塞前台

解决方案：

方案1：调整Compaction时间
// 避开高峰期
ScheduleCompaction(2:00 AM);

方案2：限制Compaction带宽
// RocksDB支持
// LevelDB需要修改源码

方案3：增大文件大小
options.max_file_size = 16 * 1024 * 1024;

方案4：分层Compaction
// 高峰期：只Compaction L0->L1
// 低峰期：Compaction所有层
```

---

## 5. 最佳实践

### 5.1 设计模式

**模式1：批量操作**

```cpp
// 批量写入模板
template<typename Func>
void BatchWrite(DB* db, Func generator, int batch_size) {
  WriteBatch batch;
  int count = 0;

  for (auto&& [key, value] : generator()) {
    batch.Put(key, value);

    if (++count >= batch_size) {
      db->Write(WriteOptions(), &batch);
      batch.Clear();
      count = 0;
    }
  }

  if (count > 0) {
    db->Write(WriteOptions(), &batch);
  }
}

// 使用
BatchWrite(db, []() {
  return GenerateData();
}, 1000);
```

**模式2：快照隔离**

```cpp
// 一致性读取模板
class ScopedSnapshot {
 public:
  ScopedSnapshot(DB* db) : db_(db), snapshot_(db->GetSnapshot()) {}

  ~ScopedSnapshot() {
    db_->ReleaseSnapshot(snapshot_);
  }

  const Snapshot* get() const { return snapshot_; }

 private:
  DB* db_;
  const Snapshot* snapshot_;
};

// 使用
void ReadConsistent(DB* db) {
  ScopedSnapshot snapshot(db);

  ReadOptions opts;
  opts.snapshot = snapshot.get();

  std::string v1, v2;
  db->Get(opts, "key1", &v1);
  db->Get(opts, "key2", &v2);
  // v1和v2来自同一快照
}
```

**模式3：迭代器安全使用**

```cpp
// 安全的迭代器遍历
void SafeIterate(DB* db, std::function<void(Slice, Slice)> visitor) {
  ReadOptions opts;
  opts.verify_checksums = false;
  opts.fill_cache = false;  // 扫描不污染缓存

  std::unique_ptr<Iterator> it(db->NewIterator(opts));

  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    visitor(it->key(), it->value());

    // 检查状态
    if (!it->status().ok()) {
      std::cerr << "Iterator error: " << it->status().ToString() << "\n";
      break;
    }
  }
}
```

### 5.2 运维建议

**备份策略**

```bash
#!/bin/bash
# backup.sh

DB_DIR="/data/leveldb"
BACKUP_DIR="/backup/leveldb-$(date +%Y%m%d-%H%M%S)"

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 方法1：直接复制（需要停写或使用快照）
cp -r "$DB_DIR"/* "$BACKUP_DIR/"

# 方法2：rsync（增量）
rsync -av --delete "$DB_DIR/" "$BACKUP_DIR/"

# 方法3：使用LVM快照
lvcreate -L 10G -s -n leveldb_snap /dev/vg0/leveldb
mount /dev/vg0/leveldb_snap /mnt/snap
rsync -av /mnt/snap/ "$BACKUP_DIR/"
umount /mnt/snap
lvremove -f /dev/vg0/leveldb_snap

# 压缩
tar czf "$BACKUP_DIR.tar.gz" "$BACKUP_DIR"
rm -rf "$BACKUP_DIR"

# 清理旧备份（保留7天）
find /backup -name "leveldb-*" -mtime +7 -delete
```

**监控脚本**

```python
#!/usr/bin/env python3
# monitor.py

import time
import leveldb
import psutil

def check_stats(db_path):
    db = leveldb.LevelDB(db_path)

    # 获取统计
    stats = db.GetProperty("leveldb.stats").decode()
    mem_usage = int(db.GetProperty("leveldb.approximate-memory-usage").decode())

    print(f"Memory Usage: {mem_usage / 1024 / 1024:.2f} MB")

    # 检查L0文件
    if "Level 0" in stats:
        l0_files = int(stats.split("\n")[2].split()[1])
        if l0_files > 4:
            print(f"WARNING: {l0_files} files in Level 0")

    # 检查磁盘使用
    disk = psutil.disk_usage(db_path)
    print(f"Disk Usage: {disk.percent}%")

    if disk.percent > 80:
        print("WARNING: Disk usage > 80%")

if __name__ == "__main__":
    while True:
        check_stats("/data/leveldb")
        time.sleep(60)
```

---

## 6. 真实生产案例

### 案例1：Chrome浏览器数据库

**场景：** 存储浏览历史、IndexedDB等

**配置：**
```cpp
options.write_buffer_size = 32 * 1024 * 1024;    // 32MB
options.max_open_files = 1000;
options.block_cache = NewLRUCache(8 * 1024 * 1024);  // 8MB
options.filter_policy = NewBloomFilterPolicy(10);
options.compression = kSnappyCompression;
```

**问题：** IndexedDB数据库过大

**解决：**
1. 实施配额限制
2. 定期Compaction
3. 用户提示清理

**结果：** 平均大小减少40%

### 案例2：Bitcoin核心

**场景：** 存储UTXO集、区块索引

**配置：**
```cpp
options.write_buffer_size = 64 * 1024 * 1024;
options.block_cache = NewLRUCache(128 * 1024 * 1024);  // 128MB
options.filter_policy = NewBloomFilterPolicy(10);
options.compression = kSnappyCompression;
```

**问题：** 初始同步慢

**解决：**
1. 预编译数据库
2. 下载验证过的快照
3. 并行验证

**结果：** 同步时间从3天降到6小时

### 案例3：消息队列系统

**场景：** 持久化消息

**配置：**
```cpp
options.write_buffer_size = 128 * 1024 * 1024;   // 大MemTable
options.compression = kNoCompression;             // 不压缩
WriteOptions wopts;
wopts.sync = false;                               // 异步
```

**问题：** 写入瓶颈

**解决：**
1. 分区：多个LevelDB实例
2. 批量：1000条/批次
3. 零拷贝：共享内存传输

**结果：** 吞吐从50k提升到500k ops/sec

---

## 7. 监控与告警

### 7.1 关键指标

```yaml
# Prometheus监控指标
metrics:
  - name: leveldb_write_qps
    type: gauge
    help: Write operations per second

  - name: leveldb_read_qps
    type: gauge
    help: Read operations per second

  - name: leveldb_write_latency_p99
    type: histogram
    help: Write latency p99

  - name: leveldb_cache_hit_ratio
    type: gauge
    help: Block cache hit ratio

  - name: leveldb_compaction_write_amplification
    type: gauge
    help: Write amplification factor

  - name: leveldb_disk_usage_bytes
    type: gauge
    help: Total disk usage

  - name: leveldb_level0_files
    type: gauge
    help: Number of Level 0 files
```

### 7.2 告警规则

```yaml
# 告警规则
groups:
  - name: leveldb_alerts
    rules:
      # 高延迟告警
      - alert: LevelDBHighLatency
        expr: leveldb_write_latency_p99 > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High write latency"

      # L0文件过多
      - alert: LevelDBTooManyL0Files
        expr: leveldb_level0_files > 8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Too many Level 0 files"

      # 缓存命中率低
      - alert: LevelDBLowCacheHit
        expr: leveldb_cache_hit_ratio < 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low cache hit ratio"

      # 磁盘空间不足
      - alert: LevelDBDiskSpaceLow
        expr: leveldb_disk_usage_bytes > total_disk * 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space critically low"
```

---

## 8. 容量规划

### 8.1 存储容量计算

```cpp
// 容量规划工具
class CapacityPlanner {
 public:
  struct Estimate {
    int64_t total_bytes;
    int64_t wal_bytes;
    int64_t data_bytes;
    int64_t index_bytes;
    int64_t write_amplification;
  };

  static Estimate Calculate(
      int64_t num_keys,
      int64_t avg_key_size,
      int64_t avg_value_size,
      double compression_ratio = 0.5) {

    Estimate est;

    // 原始数据大小
    int64_t raw_size = num_keys * (avg_key_size + avg_value_size);

    // 压缩后数据
    est.data_bytes = raw_size * compression_ratio;

    // 索引开销（~10%）
    est.index_bytes = est.data_bytes * 0.1;

    // WAL（等于原始数据）
    est.wal_bytes = raw_size;

    // 写放大（10x）
    est.write_amplification = 10;
    est.total_bytes = est.data_bytes * est.write_amplification;

    return est;
  }
};

// 使用示例
auto est = CapacityPlanner::Calculate(
    1000000000,    // 10亿条
    16,            // 16字节key
    100,           // 100字节value
    0.5);          // 50%压缩率

std::cout << "Estimated storage: " << est.total_bytes / 1e9 << " GB\n";
// 输出：Estimated storage: 550 GB
```

### 8.2 内存容量规划

```cpp
// 内存规划
int64_t EstimateMemoryUsage(
    int64_t num_keys,
    int64_t avg_key_size,
    int64_t avg_value_size) {

  // MemTable内存
  // SkipList节点: 32字节/key
  int64_t memtable = num_keys * (32 + avg_key_size + avg_value_size);

  // Block Cache
  // 假设热数据10%
  int64_t cache = num_keys * 0.1 * (avg_key_size + avg_value_size);

  // 其他开销（~10%）
  int64_t overhead = (memtable + cache) * 0.1;

  return memtable + cache + overhead;
}
```

---

## 9. 灾难恢复

### 9.1 数据恢复流程

```bash
#!/bin/bash
# restore.sh

BACKUP_DIR="$1"
DB_DIR="$2"

# 1. 停止服务
systemctl stop leveldb-service

# 2. 验证备份
if [ ! -f "$BACKUP_DIR/CURRENT" ]; then
  echo "Invalid backup: missing CURRENT file"
  exit 1
fi

# 3. 验证MANIFEST
if [ ! -f "$BACKUP_DIR/MANIFEST-"* ]; then
  echo "Invalid backup: missing MANIFEST file"
  exit 1
fi

# 4. 备份当前数据
mv "$DB_DIR" "$DB_DIR.bak"

# 5. 恢复数据
cp -r "$BACKUP_DIR"/* "$DB_DIR/"

# 6. 验证完整性
# 使用db_verify工具
db_verify "$DB_DIR"

if [ $? -eq 0 ]; then
  echo "Restore successful"
  systemctl start leveldb-service
else
  echo "Restore failed, rolling back"
  rm -rf "$DB_DIR"
  mv "$DB_DIR.bak" "$DB_DIR"
  systemctl start leveldb-service
  exit 1
fi
```

### 9.2 MANIFEST损坏修复

```cpp
// 尝试恢复MANIFEST
Status RecoverManifest(const std::string& dbname) {
  // 1. 查找所有MANIFEST文件
  std::vector<std::string> manifests = FindManifestFiles(dbname);

  // 2. 从最新的MANIFEST尝试恢复
  for (const auto& manifest : manifests) {
    Status s = TryRecoverFromManifest(dbname, manifest);
    if (s.ok()) {
      return s;
    }
  }

  // 3. 从CURRENT重新构建
  return RebuildFromCurrent(dbname);
}
```

---

## 10. 性能调优实战

### 10.1 调优流程

```
1. 建立基线
   - 使用默认配置运行基准测试
   - 记录性能指标
   - 识别瓶颈

2. 逐个调优
   - 修改一个参数
   - 测量影响
   - 决定保留或回滚

3. 验证稳定性
   - 长时间运行测试
   - 压力测试
   - 故障注入

4. 生产部署
   - 灰度发布
   - 监控指标
   - 准备回滚
```

### 10.2 调优决策树

```
开始
  |
  v
[写吞吐低？] --是--> [增大write_buffer_size] --> [测试]
  | 否                                 |
  v                                   v
[读延迟高？] --是--> [增大block_cache] --> [测试]
  | 否                                 |
  v                                   v
[空间大？] --是--> [启用压缩] --> [测试]
  | 否                                 |
  v                                   v
[写放大高？] --是--> [增大max_file_size] --> [测试]
  | 否                                 |
  v                                   v
[Compaction慢？] --是--> [调整层级大小] --> [测试]
  | 否                                 |
  v                                   v
完成
```

---

## 总结

今天我们深入学习了：

1. ✅ **参数调优**：详细理解每个参数的影响
2. ✅ **场景优化**：不同场景的配置策略
3. ✅ **性能分析**：监控工具和诊断方法
4. ✅ **生产配置**：推荐的配置和监控
5. ✅ **最佳实践**：设计模式和运维建议
6. ✅ **真实案例**：Chrome、Bitcoin等
7. ✅ **监控告警**：关键指标和告警规则
8. ✅ **容量规划**：存储和内存计算
9. ✅ **灾难恢复**：备份和恢复流程
10. ✅ **调优实战**：调优流程和决策树

**LevelDB基础学习完成！**

经过14天的深入学习，你已经掌握：
- 基础数据结构（Slice, Status, SkipList, MemTable）
- 存储引擎（WAL, SSTable, LSM-Tree）
- 核心机制（Compaction, Version管理）
- 性能优化（Cache, Bloom Filter, 并发）
- 生产实践（配置、监控、问题解决）

**继续进阶：**

1. **高级课程**
   - 内存管理与Arena优化
   - 性能分析与优化实战
   - 故障排查实战手册
   - 数据布局与压缩优化
   - 无锁数据结构与并发
   - 网络序列化与零拷贝

2. **源码精读**
   - 逐行阅读核心模块
   - 理解每个设计决策
   - 对比RocksDB改进

3. **实践项目**
   - 实现简化版LSM-Tree
   - 性能基准测试工具
   - 数据库监控系统

4. **深入研究**
   - 阅读LSM-Tree论文
   - 研究RocksDB高级特性
   - 贡献开源项目

**祝你在数据库领域继续成长！**

---

## 思考题

1. 如何在设计时平衡写放大和空间放大？
2. 在什么情况下应该禁用压缩？
3. 如何为你的应用选择合适的write_buffer_size？
4. 如何设计一个自动化调优系统？

## 下一步学习

- 高级课程：内存管理与Arena优化
- IO优化课程：深入理解I/O性能
- C++性能优化：编译器、CPU、算法优化
