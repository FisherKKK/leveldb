# Day 14: 性能优化技巧与最佳实践

## 学习目标
- 掌握参数调优指南
- 学习性能瓶颈分析方法
- 理解生产环境配置
- 了解常见问题解决方案

## 1. 参数调优

### 1.1 关键参数

```cpp
Options options;

// 1. write_buffer_size (默认4MB)
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB
// 影响：
// - 越大：减少L0文件，减少Compaction
// - 越小：更频繁刷盘，更多Compaction
// 建议：16MB-128MB

// 2. max_file_size (默认2MB)
options.max_file_size = 4 * 1024 * 1024;  // 4MB
// 影响：
// - 越大：减少文件数量
// - 越小：Compaction更细粒度
// 建议：2MB-10MB

// 3. block_size (默认4KB)
options.block_size = 16 * 1024;  // 16KB
// 影响：
// - 越大：压缩率更好，读取更多数据
// - 越小：随机读更快
// 建议：4KB-64KB

// 4. block_cache (默认8MB)
options.block_cache = NewLRUCache(512 * 1024 * 1024);  // 512MB
// 影响：直接影响读性能
// 建议：根据内存设置，越大越好

// 5. max_open_files (默认1000)
options.max_open_files = 5000;
// 影响：TableCache大小
// 建议：根据ulimit设置

// 6. filter_policy (默认nullptr)
options.filter_policy = NewBloomFilterPolicy(10);  // 10 bits/key
// 影响：
// - 减少99%的无效读取
// - 空间开销：10 bits/key ≈ 1.25 bytes/key
// 建议：总是启用

// 7. compression (默认kSnappyCompression)
options.compression = kSnappyCompression;
// 选项：
// - kNoCompression: 最快，最大空间
// - kSnappyCompression: 平衡（推荐）
// - kZstdCompression: 最佳压缩率，较慢
```

### 1.2 场景优化

**写密集型：**
```cpp
Options options;
options.write_buffer_size = 128 * 1024 * 1024;  // 大MemTable
options.max_file_size = 10 * 1024 * 1024;       // 大文件
options.compression = kNoCompression;            // 快速写入
options.max_open_files = 10000;                  // 多文件
```

**读密集型：**
```cpp
Options options;
options.block_cache = NewLRUCache(2 * 1024 * 1024 * 1024);  // 大Cache
options.filter_policy = NewBloomFilterPolicy(10);           // Bloom Filter
options.block_size = 4 * 1024;                             // 小Block
options.compression = kSnappyCompression;                  // 平衡
```

**空间受限：**
```cpp
Options options;
options.compression = kZstdCompression;  // 最佳压缩
options.write_buffer_size = 16 * 1024 * 1024;  // 小MemTable
options.block_cache = NewLRUCache(64 * 1024 * 1024);  // 小Cache
```

## 2. 性能分析

### 2.1 性能监控

```cpp
// 获取统计信息
std::string stats;
db->GetProperty("leveldb.stats", &stats);
std::cout << stats << std::endl;

// 输出示例：
//                                Compactions
// Level  Files Size(MB) Time(sec) Read(MB) Write(MB)
// --------------------------------------------------
//   0        3        6         5       13        13
//   1       17       20        10       50        48
//   2      205      200        30      400       395
```

### 2.2 瓶颈分析

**写入瓶颈：**
```
症状：写入延迟高、吞吐量低
原因：
1. L0文件过多 → 流控
2. Compaction慢 → MemTable切换慢
3. sync=true → fsync延迟

解决：
1. 增大write_buffer_size
2. 增大max_file_size
3. 使用批量写入
4. 考虑sync=false
```

**读取瓶颈：**
```
症状：读取延迟高
原因：
1. Cache未命中 → 磁盘I/O
2. 无Bloom Filter → 无效查找
3. L0文件过多 → 查找慢

解决：
1. 增大block_cache
2. 启用Bloom Filter
3. 触发Compaction减少L0文件
```

**空间瓶颈：**
```
症状：磁盘空间增长快
原因：
1. 写放大 → Compaction重写
2. 未压缩 → 占用大
3. 旧版本未清理

解决：
1. 启用压缩
2. 手动触发Compaction
3. 及时释放Snapshot
```

## 3. 生产环境配置

### 3.1 推荐配置

```cpp
// 通用生产环境配置
Options options;

// 基础配置
options.create_if_missing = true;
options.error_if_exists = false;
options.paranoid_checks = true;  // 严格检查

// 性能配置
options.write_buffer_size = 64 * 1024 * 1024;           // 64MB
options.max_file_size = 2 * 1024 * 1024;                // 2MB
options.block_size = 16 * 1024;                         // 16KB
options.block_cache = NewLRUCache(512 * 1024 * 1024);   // 512MB
options.max_open_files = 5000;                          // 5000个

// 优化配置
options.filter_policy = NewBloomFilterPolicy(10);       // Bloom Filter
options.compression = kSnappyCompression;               // Snappy压缩

// 写入配置
WriteOptions write_options;
write_options.sync = false;  // 性能模式（可能丢失最后几秒数据）

// 读取配置
ReadOptions read_options;
read_options.verify_checksums = true;   // 验证校验和
read_options.fill_cache = true;         // 填充缓存
```

### 3.2 监控指标

```cpp
// 定期收集统计信息
void CollectStats(DB* db) {
  std::string stats, sstables;
  db->GetProperty("leveldb.stats", &stats);
  db->GetProperty("leveldb.sstables", &sstables);
  db->GetProperty("leveldb.approximate-memory-usage", &memory);
  
  // 关键指标：
  // 1. Compaction次数和时间
  // 2. 各层文件数和大小
  // 3. 内存使用
  // 4. Cache命中率
}
```

## 4. 常见问题

### 4.1 写入慢

```
问题：写入吞吐量低于预期

诊断：
1. 检查L0文件数 (leveldb.stats)
2. 检查是否触发流控
3. 检查Compaction时间

解决方案：
// 增大MemTable
options.write_buffer_size = 128 * 1024 * 1024;

// 批量写入
WriteBatch batch;
for (...) {
  batch.Put(key, value);
}
db->Write(WriteOptions(), &batch);

// 异步写入
WriteOptions options;
options.sync = false;
```

### 4.2 读取慢

```
问题：随机读取延迟高

诊断：
1. 检查Cache命中率
2. 检查是否有Bloom Filter
3. 检查L0文件数量

解决方案：
// 增大Cache
options.block_cache = NewLRUCache(2 * 1024 * 1024 * 1024);

// 启用Bloom Filter
options.filter_policy = NewBloomFilterPolicy(10);

// Warm up Cache
Iterator* it = db->NewIterator(ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // 预热缓存
}
delete it;
```

### 4.3 空间增长

```
问题：数据库大小超过预期

诊断：
1. 检查压缩是否启用
2. 检查是否有旧Snapshot
3. 检查写放大系数

解决方案：
// 手动Compaction
db->CompactRange(nullptr, nullptr);

// 释放Snapshot
db->ReleaseSnapshot(snapshot);

// 启用压缩
options.compression = kSnappyCompression;
```

### 4.4 Compaction慢

```
问题：Compaction占用过多CPU/IO

诊断：
1. 检查Compaction统计
2. 检查文件大小配置
3. 检查压缩算法

解决方案：
// 增大文件大小
options.max_file_size = 10 * 1024 * 1024;

// 禁用压缩（如果CPU瓶颈）
options.compression = kNoCompression;

// 限制Compaction频率
options.write_buffer_size = 128 * 1024 * 1024;
```

## 5. 最佳实践

### 5.1 设计模式

**1. 批量操作**
```cpp
// ✓ 好：批量写入
WriteBatch batch;
for (int i = 0; i < 10000; i++) {
  batch.Put(key, value);
}
db->Write(WriteOptions(), &batch);

// ✗ 坏：单个写入
for (int i = 0; i < 10000; i++) {
  db->Put(WriteOptions(), key, value);
}
```

**2. 使用快照**
```cpp
// ✓ 好：一致性读取
const Snapshot* snapshot = db->GetSnapshot();
ReadOptions options;
options.snapshot = snapshot;
// ... 多次读取 ...
db->ReleaseSnapshot(snapshot);

// ✗ 坏：多次读取不一致
db->Get(ReadOptions(), key1, &value1);
// 中间可能有写入
db->Get(ReadOptions(), key2, &value2);
```

**3. 范围删除**
```cpp
// ✓ 好：批量删除
WriteBatch batch;
Iterator* it = db->NewIterator(ReadOptions());
for (it->Seek(start); it->Valid() && it->key().ToString() < end; it->Next()) {
  batch.Delete(it->key());
}
delete it;
db->Write(WriteOptions(), &batch);
```

### 5.2 运维建议

**1. 监控**
- 定期收集leveldb.stats
- 监控磁盘使用
- 监控Compaction时间

**2. 备份**
```bash
# 创建快照
mkdir /backup/leveldb-$(date +%Y%m%d)
cp -r /data/leveldb/* /backup/leveldb-$(date +%Y%m%d)/

# 或使用rsync
rsync -av /data/leveldb/ /backup/leveldb-$(date +%Y%m%d)/
```

**3. 恢复**
- 验证MANIFEST完整性
- 检查SSTable文件
- 测试读取

**4. 升级**
- 先升级测试环境
- 备份数据
- 逐步升级生产环境

## 总结

今天我们学习了：
1. ✅ **参数调优**：根据场景调整配置
2. ✅ **性能分析**：识别和解决瓶颈
3. ✅ **生产配置**：推荐的配置和监控
4. ✅ **最佳实践**：设计模式和运维建议

**LevelDB学习完成！**

经过14天的学习，你已经掌握：
- 基础数据结构（Slice, Status, SkipList, MemTable）
- 存储引擎（WAL, SSTable, LSM-Tree）
- 核心机制（Compaction, Version管理）
- 性能优化（Cache, Bloom Filter, 并发）
- 生产实践（配置、监控、问题解决）

**继续深入：**
1. 阅读完整源码
2. 参与社区讨论
3. 实现类似项目
4. 研究变种（RocksDB, LevelDB变体）

**祝你在数据库领域继续成长！**
