# LevelDB实战调试与性能分析指南

## 目录
1. [调试技巧](#调试技巧)
2. [性能分析工具](#性能分析工具)
3. [常见问题排查](#常见问题排查)
4. [性能优化案例](#性能优化案例)
5. [生产环境最佳实践](#生产环境最佳实践)

---

## 调试技巧

### 1. 使用GDB调试LevelDB

#### 编译调试版本

```bash
cd /home/dev/leveldb
mkdir -p build-debug && cd build-debug
cmake -DCMAKE_BUILD_TYPE=Debug ..
cmake --build .
```

#### GDB基本操作

```bash
# 启动调试
gdb ./db_bench

# 设置断点
(gdb) break db/db_impl.cc:1088     # DBImpl::Get入口
(gdb) break db/memtable.cc:89      # MemTable::Get
(gdb) break db/version_set.cc:345  # Version::Get

# 条件断点
(gdb) break db_impl.cc:100 if key.ToString() == "user:1001"

# 运行
(gdb) run --benchmarks=readrandom --num=1000

# 查看调用栈
(gdb) backtrace
(gdb) bt

# 查看变量
(gdb) print key
(gdb) print *mem_
(gdb) print versions_->current_

# 单步调试
(gdb) next    # 下一行（不进入函数）
(gdb) step    # 单步（进入函数）
(gdb) finish  # 运行到函数返回

# 查看源码
(gdb) list
(gdb) list db_impl.cc:1088
```

#### 调试Get操作完整路径

```gdb
# 调试脚本：trace_get.gdb
break DBImpl::Get
commands
  silent
  printf "=== DBImpl::Get called, key=%s\n", key.data()
  continue
end

break MemTable::Get
commands
  silent
  printf "  -> Checking MemTable\n"
  continue
end

break Version::Get
commands
  silent
  printf "  -> Checking SSTables\n"
  continue
end

run --benchmarks=readrandom --num=100
```

使用：
```bash
gdb -x trace_get.gdb ./db_bench
```

---

### 2. 日志调试

#### 启用详细日志

```cpp
#include "leveldb/db.h"
#include "leveldb/env.h"

leveldb::Options options;
options.create_if_missing = true;

// 创建日志文件
leveldb::Env* env = leveldb::Env::Default();
env->NewLogger("/tmp/leveldb.log", &options.info_log);

leveldb::DB* db;
leveldb::Status s = leveldb::DB::Open(options, "/tmp/testdb", &db);
```

#### 添加自定义日志

在源码中添加日志：

```cpp
// db/db_impl.cc
Status DBImpl::Get(const ReadOptions& options,
                    const Slice& key,
                    std::string* value) {
  Log(options_.info_log, "Get key: %s", key.ToString().c_str());

  // ... 原有代码 ...

  if (s.ok()) {
    Log(options_.info_log, "Get success: %s", value->c_str());
  } else {
    Log(options_.info_log, "Get failed: %s", s.ToString().c_str());
  }
  return s;
}
```

---

### 3. 使用perf分析性能瓶颈

#### 安装perf

```bash
sudo apt-get install linux-tools-common linux-tools-generic
```

#### CPU性能分析

```bash
# 记录性能数据
sudo perf record -g ./db_bench --benchmarks=fillrandom --num=1000000

# 查看报告
sudo perf report

# 生成火焰图
sudo perf record -F 99 -g ./db_bench --benchmarks=fillrandom --num=1000000
sudo perf script | flamegraph.pl > flamegraph.svg
```

#### 分析结果示例

```
Samples: 10K of event 'cycles'
  50.23%  db_bench  db_bench  [.] leveldb::MemTable::Add
  15.42%  db_bench  db_bench  [.] leveldb::SkipList::Insert
  12.18%  db_bench  libc.so   [.] memcpy
   8.34%  db_bench  db_bench  [.] leveldb::DBImpl::Write
   5.67%  db_bench  db_bench  [.] leveldb::log::Writer::AddRecord
```

**分析**：
- `MemTable::Add`占50%：正常，写入主要瓶颈
- `memcpy`占12%：可能需要优化内存拷贝

---

### 4. 使用Valgrind检测内存问题

#### 内存泄漏检测

```bash
# 编译时启用调试符号
cmake -DCMAKE_BUILD_TYPE=Debug ..

# 运行valgrind
valgrind --leak-check=full --show-leak-kinds=all \
  ./db_bench --benchmarks=fillseq --num=10000
```

#### 常见问题

**案例1：忘记释放MemTable**

```cpp
// 错误代码
MemTable* mem = new MemTable(comparator);
mem->Ref();
// 忘记调用 mem->Unref();

// Valgrind输出
==12345== 4,096 bytes in 1 blocks are definitely lost
==12345==    at 0x4C2FB0F: malloc (vg_replace_malloc.c:299)
==12345==    by 0x4E8A9C: operator new (in /usr/lib/...)
==12345==    by 0x40C123: leveldb::MemTable::MemTable (memtable.cc:30)
```

**修复**：
```cpp
mem->Unref();  // 释放引用
```

---

## 性能分析工具

### 1. db_bench基准测试

#### 常用测试场景

```bash
# 1. 顺序写入（最快）
./db_bench --benchmarks=fillseq --num=1000000

# 2. 随机写入（模拟真实负载）
./db_bench --benchmarks=fillrandom --num=1000000

# 3. 随机读取
./db_bench --benchmarks=readrandom --num=1000000 --use_existing_db=1

# 4. 顺序读取
./db_bench --benchmarks=readseq --num=1000000 --use_existing_db=1

# 5. 混合读写（90%读，10%写）
./db_bench --benchmarks=readwhilewriting --num=1000000 \
  --reads=900000 --writes=100000

# 6. 大值测试
./db_bench --benchmarks=fillrandom --num=100000 --value_size=10000

# 7. 小键测试
./db_bench --benchmarks=fillrandom --num=1000000 --key_size=8
```

#### 自定义配置测试

```bash
# 测试不同write_buffer_size的影响
for size in 4 16 64 128; do
  ./db_bench --benchmarks=fillrandom --num=1000000 \
    --write_buffer_size=$((size * 1024 * 1024)) \
    --db=/tmp/testdb_${size}mb
done

# 测试不同block_cache大小
for cache in 8 64 512 1024; do
  ./db_bench --benchmarks=readrandom --num=1000000 \
    --cache_size=$((cache * 1024 * 1024)) \
    --use_existing_db=1
done
```

---

### 2. 数据库统计信息

#### 获取实时统计

```cpp
#include "leveldb/db.h"
#include <iostream>

void PrintStats(leveldb::DB* db) {
  std::string stats;

  // 1. 总体统计
  db->GetProperty("leveldb.stats", &stats);
  std::cout << "=== LevelDB Stats ===\n" << stats << std::endl;

  // 2. 各层SSTable信息
  db->GetProperty("leveldb.sstables", &stats);
  std::cout << "=== SSTable Info ===\n" << stats << std::endl;

  // 3. 内存使用
  db->GetProperty("leveldb.approximate-memory-usage", &stats);
  std::cout << "Memory Usage: " << stats << " bytes\n";

  // 4. 当前Compaction状态
  db->GetProperty("leveldb.num-files-at-level0", &stats);
  std::cout << "Level-0 files: " << stats << std::endl;
}
```

#### 输出示例

```
=== LevelDB Stats ===
                               Compactions
Level  Files Size(MB) Time(sec) Read(MB) Write(MB)
--------------------------------------------------
  0        3        6         2       10        10
  1       15       20         5       50        48
  2      180      200        15      400       395
  3     1500     2000        50     4000      3950

=== SSTable Info ===
--- level 0 ---
  file 000005.ldb: [apple..banana]
  file 000008.ldb: [cherry..grape]
  file 000011.ldb: [orange..watermelon]

--- level 1 ---
  file 000003.ldb: [apple..banana]
  file 000006.ldb: [cherry..grape]
  ...

Memory Usage: 4194304 bytes
Level-0 files: 3
```

---

### 3. 自定义性能监控工具

创建 `monitor.cc`:

```cpp
#include "leveldb/db.h"
#include <iostream>
#include <thread>
#include <chrono>

class LevelDBMonitor {
 public:
  LevelDBMonitor(leveldb::DB* db) : db_(db), running_(true) {}

  void Start() {
    monitor_thread_ = std::thread([this]() {
      while (running_) {
        PrintMetrics();
        std::this_thread::sleep_for(std::chrono::seconds(5));
      }
    });
  }

  void Stop() {
    running_ = false;
    if (monitor_thread_.joinable()) {
      monitor_thread_.join();
    }
  }

 private:
  void PrintMetrics() {
    std::string stats;
    db_->GetProperty("leveldb.stats", &stats);

    std::cout << "\033[2J\033[H";  // 清屏
    std::cout << "=== LevelDB Monitor ===\n";
    std::cout << "Time: " << GetCurrentTime() << "\n\n";
    std::cout << stats << std::endl;

    // 计算写放大
    ParseAndPrintWriteAmplification(stats);
  }

  void ParseAndPrintWriteAmplification(const std::string& stats) {
    // 解析stats，计算写放大
    // Write Amplification = (Write(MB) at all levels) / (Write(MB) at L0)
    std::cout << "\nWrite Amplification: " << CalculateWA(stats) << "x\n";
  }

  std::string GetCurrentTime() {
    auto now = std::chrono::system_clock::now();
    std::time_t time = std::chrono::system_clock::to_time_t(now);
    return std::ctime(&time);
  }

  leveldb::DB* db_;
  bool running_;
  std::thread monitor_thread_;
};

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;

  leveldb::Status s = leveldb::DB::Open(options, "/tmp/testdb", &db);
  if (!s.ok()) {
    std::cerr << "Failed to open db: " << s.ToString() << std::endl;
    return 1;
  }

  LevelDBMonitor monitor(db);
  monitor.Start();

  // 进行一些操作...

  std::this_thread::sleep_for(std::chrono::minutes(1));
  monitor.Stop();

  delete db;
  return 0;
}
```

---

## 常见问题排查

### 问题1：写入变慢

**症状**：
```
Initial write rate: 100k ops/sec
After 10 minutes: 10k ops/sec (dropped 10x!)
```

**排查步骤**：

```cpp
// 1. 检查Level-0文件数
std::string num_files;
db->GetProperty("leveldb.num-files-at-level0", &num_files);
std::cout << "L0 files: " << num_files << std::endl;

// 如果L0文件 >= 8：触发了写入流控
```

**根本原因**：
- Level-0文件过多（≥8个）
- Compaction跟不上写入速度
- 触发写入流控（stall）

**解决方案**：

```cpp
// 方案1：增大write_buffer_size
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB

// 方案2：使用批量写入
leveldb::WriteBatch batch;
for (int i = 0; i < 1000; i++) {
  batch.Put(key, value);
}
db->Write(leveldb::WriteOptions(), &batch);

// 方案3：手动触发Compaction
db->CompactRange(nullptr, nullptr);
```

---

### 问题2：读取延迟高

**症状**：
```
Average read latency: 50ms (expected: 1-5ms)
```

**排查步骤**：

```cpp
// 1. 检查是否有Bloom Filter
leveldb::Options options;
if (options.filter_policy == nullptr) {
  std::cout << "WARNING: No Bloom Filter!\n";
}

// 2. 检查缓存大小
if (options.block_cache == nullptr) {
  std::cout << "WARNING: No Block Cache!\n";
}

// 3. 检查Level-0文件数
// Level-0文件多 → 需要查找多个文件
```

**解决方案**：

```cpp
// 方案1：启用Bloom Filter
options.filter_policy = leveldb::NewBloomFilterPolicy(10);

// 方案2：增大缓存
options.block_cache = leveldb::NewLRUCache(512 * 1024 * 1024);

// 方案3：预热缓存
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // 遍历所有数据，填充缓存
}
delete it;
```

---

### 问题3：磁盘空间持续增长

**症状**：
```
Initial DB size: 1GB
After deletes: 1.5GB (expected: smaller)
```

**排查步骤**：

```bash
# 查看数据库目录
ls -lh /path/to/db/

# 检查是否有旧文件未删除
# 查看MANIFEST中的文件列表
```

**根本原因**：
- 删除操作只写入删除标记
- 等待Compaction才真正删除数据
- Compaction速度慢

**解决方案**：

```cpp
// 方案1：手动Compaction
db->CompactRange(nullptr, nullptr);

// 方案2：删除后立即Compact特定范围
std::string start = "user:1000";
std::string end = "user:2000";
leveldb::Slice start_slice(start);
leveldb::Slice end_slice(end);
db->CompactRange(&start_slice, &end_slice);

// 方案3：定期后台Compaction
void BackgroundCompaction(leveldb::DB* db) {
  while (true) {
    std::this_thread::sleep_for(std::chrono::hours(1));
    db->CompactRange(nullptr, nullptr);
  }
}
```

---

### 问题4：崩溃后数据丢失

**症状**：
```
写入10000条记录
崩溃后重启：只恢复了9500条
```

**排查步骤**：

```cpp
// 检查WriteOptions配置
leveldb::WriteOptions write_options;
std::cout << "sync: " << write_options.sync << std::endl;
// 如果 sync=false，可能丢失数据
```

**根本原因**：
- `WriteOptions::sync = false`
- WAL在OS缓存中，未刷盘
- 崩溃时丢失未刷盘的数据

**解决方案**：

```cpp
// 方案1：启用sync（性能换安全性）
leveldb::WriteOptions options;
options.sync = true;
db->Put(options, key, value);

// 方案2：批量写入 + sync
leveldb::WriteBatch batch;
for (int i = 0; i < 1000; i++) {
  batch.Put(key, value);
}
leveldb::WriteOptions options;
options.sync = true;
db->Write(options, &batch);  // 1000条写入只fsync一次

// 方案3：定期sync
int write_count = 0;
leveldb::WriteOptions no_sync, with_sync;
no_sync.sync = false;
with_sync.sync = true;

for (int i = 0; i < 100000; i++) {
  db->Put(no_sync, key, value);
  write_count++;

  if (write_count % 1000 == 0) {
    db->Put(with_sync, "checkpoint", "marker");  // 每1000次sync一次
  }
}
```

---

## 性能优化案例

### 案例1：从1万ops/s优化到10万ops/s

**场景**：高频写入应用

**初始配置**：
```cpp
leveldb::Options options;
options.write_buffer_size = 4 * 1024 * 1024;      // 4MB
options.block_cache = leveldb::NewLRUCache(8 * 1024 * 1024);
options.filter_policy = nullptr;  // 没有Bloom Filter
```

**性能**：~10,000 writes/sec

**优化步骤**：

**优化1：增大write_buffer_size**
```cpp
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB
```
**结果**：20,000 writes/sec（提升2倍）
**原因**：减少Level-0文件生成频率，减少Compaction

**优化2：使用批量写入**
```cpp
leveldb::WriteBatch batch;
for (int i = 0; i < 100; i++) {
  batch.Put(keys[i], values[i]);
}
db->Write(leveldb::WriteOptions(), &batch);
```
**结果**：80,000 writes/sec（再提升4倍）
**原因**：减少锁竞争，批量写入WAL

**优化3：关闭sync**
```cpp
leveldb::WriteOptions options;
options.sync = false;
db->Put(options, key, value);
```
**结果**：100,000 writes/sec（再提升25%）
**原因**：避免fsync开销

**最终配置**：
```cpp
leveldb::Options options;
options.write_buffer_size = 64 * 1024 * 1024;
options.block_cache = leveldb::NewLRUCache(512 * 1024 * 1024);

leveldb::WriteOptions write_options;
write_options.sync = false;

// 批量写入
leveldb::WriteBatch batch;
for (int i = 0; i < 100; i++) {
  batch.Put(keys[i], values[i]);
}
db->Write(write_options, &batch);
```

**性能**：100,000 writes/sec（提升10倍）

---

### 案例2：优化读取延迟

**场景**：随机读取密集型应用

**初始配置**：
```cpp
leveldb::Options options;
// 使用默认配置
```

**性能**：平均延迟50ms

**优化步骤**：

**优化1：启用Bloom Filter**
```cpp
options.filter_policy = leveldb::NewBloomFilterPolicy(10);
```
**结果**：平均延迟15ms（提升3.3倍）
**原因**：99%的不存在键直接跳过文件查找

**优化2：增大BlockCache**
```cpp
options.block_cache = leveldb::NewLRUCache(2 * 1024 * 1024 * 1024);  // 2GB
```
**结果**：平均延迟3ms（再提升5倍）
**原因**：缓存命中率从10%提升到90%

**优化3：减少Level-0文件**
```cpp
// 定期触发Compaction
std::thread compaction_thread([db]() {
  while (true) {
    std::this_thread::sleep_for(std::chrono::minutes(10));
    db->CompactRange(nullptr, nullptr);
  }
});
```
**结果**：平均延迟1ms（再提升3倍）
**原因**：减少需要查找的文件数量

**最终性能**：平均延迟1ms（提升50倍）

---

## 生产环境最佳实践

### 1. 推荐配置

```cpp
leveldb::Options GetProductionOptions() {
  leveldb::Options options;

  // 基础配置
  options.create_if_missing = true;
  options.error_if_exists = false;
  options.paranoid_checks = true;  // 严格检查

  // 性能调优
  options.write_buffer_size = 64 * 1024 * 1024;           // 64MB
  options.max_file_size = 2 * 1024 * 1024;                // 2MB
  options.block_size = 16 * 1024;                         // 16KB
  options.block_cache = NewLRUCache(512 * 1024 * 1024);   // 512MB
  options.max_open_files = 5000;

  // 优化
  options.filter_policy = NewBloomFilterPolicy(10);
  options.compression = kSnappyCompression;

  return options;
}
```

### 2. 监控指标

关键指标：
- Level-0文件数量（< 4 正常，>= 8 警告）
- Compaction延迟
- 写入吞吐量
- 读取延迟
- 缓存命中率

### 3. 备份策略

```bash
#!/bin/bash
# backup.sh

DB_PATH="/data/leveldb"
BACKUP_PATH="/backup/leveldb-$(date +%Y%m%d-%H%M%S)"

# 1. 创建硬链接备份（快速，不占额外空间）
mkdir -p $BACKUP_PATH
cp -al $DB_PATH/* $BACKUP_PATH/

# 2. 验证备份
leveldb_check $BACKUP_PATH

# 3. 保留最近7天的备份
find /backup/ -name "leveldb-*" -mtime +7 -exec rm -rf {} \;
```

### 4. 故障恢复

```cpp
// recovery.cc
#include "leveldb/db.h"
#include "leveldb/options.h"

bool RepairDatabase(const std::string& dbname) {
  leveldb::Options options;
  leveldb::Status s = leveldb::RepairDB(dbname, options);
  if (!s.ok()) {
    std::cerr << "Repair failed: " << s.ToString() << std::endl;
    return false;
  }
  return true;
}
```

---

## 总结

本指南涵盖了：
1. ✅ GDB/Valgrind/Perf等调试工具的使用
2. ✅ 性能分析和监控方法
3. ✅ 常见问题的排查和解决
4. ✅ 真实优化案例
5. ✅ 生产环境最佳实践

**记住**：
- 先测量，再优化
- 使用工具定位瓶颈
- 参考源码理解问题本质
- 在生产环境谨慎操作

**下一步**：
- 实践调试技巧
- 运行性能基准测试
- 分析自己的应用场景
- 针对性优化
