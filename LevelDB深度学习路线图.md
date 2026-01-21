# LevelDB深度学习路线图

## 📚 课程体系总览

这是一个为期14天的LevelDB深度学习课程，从零开始带你掌握世界级键值存储系统的设计与实现。

### 🎯 学习目标

完成本课程后，你将能够：
- 深入理解LSM-Tree存储引擎的设计原理
- 掌握高性能数据结构（SkipList）的无锁并发实现
- 理解数据库持久化和崩溃恢复机制
- 学会分析和优化数据库性能
- 具备设计类似系统的能力

---

## 📅 课程大纲

### 第一阶段：基础架构（Day 1-4）

#### Day 1: LevelDB概述与架构总览
**核心概念**: LSM-Tree、分层存储、读写路径
- [ ] 理解LevelDB的设计目标
- [ ] 掌握LSM-Tree的基本原理
- [ ] 了解整体架构和数据流
- [ ] 编译运行第一个LevelDB程序

**关键文件**:
- `include/leveldb/db.h` - 公共API
- `db/db_impl.h` - 核心实现

**实战练习**:
```bash
# 编译LevelDB
cd /home/dev/leveldb
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .

# 运行基准测试
./db_bench --benchmarks=fillseq,readrandom --num=100000
```

---

#### Day 2: 核心数据结构 - Slice与Status
**核心概念**: 零拷贝、无异常错误处理
- [ ] 掌握Slice的零拷贝设计
- [ ] 理解Status错误处理机制
- [ ] 学习Options配置系统

**关键文件**:
- `include/leveldb/slice.h` - 字节数组引用
- `include/leveldb/status.h` - 错误状态
- `include/leveldb/options.h` - 配置选项

**深度思考**:
- 为什么Google要禁用C++异常？
- Slice与std::string_view的对比
- 零拷贝在高性能系统中的应用

---

#### Day 3: SkipList无锁数据结构
**核心概念**: 跳表、无锁并发、内存顺序
- [ ] 理解SkipList的概率平衡原理
- [ ] 掌握release-acquire内存模型
- [ ] 学习无锁数据结构设计

**关键文件**:
- `db/skiplist.h` - 完整实现（300行经典代码）
- `util/arena.h` - 内存池分配器
- `util/random.h` - 随机数生成

**性能分析**:
```cpp
// 基准测试：SkipList vs std::map
// 插入100万条记录
SkipList: ~400ns per insert
std::map: ~600ns per insert
```

---

#### Day 4: MemTable内存写缓冲
**核心概念**: InternalKey编码、多版本控制、引用计数
- [ ] 理解InternalKey的编码格式
- [ ] 掌握Sequence Number的作用
- [ ] 学习MemTable的生命周期管理

**关键文件**:
- `db/memtable.h` - MemTable实现
- `db/dbformat.h` - InternalKey格式
- `db/write_batch.h` - 批量写入

**数据格式**:
```
InternalKey = [UserKey][SequenceNumber(7 bytes)][Type(1 byte)]
MemTable Entry = [KeyLength][InternalKey][ValueLength][Value]
```

---

### 第二阶段：存储引擎（Day 5-8）

#### Day 5: SSTable文件格式与Block结构
**核心概念**: 前缀压缩、两级索引、重启点
- [ ] 理解SSTable的分层结构
- [ ] 掌握前缀压缩算法
- [ ] 学习BlockHandle和Footer

**关键文件**:
- `table/table.cc` - SSTable读取
- `table/table_builder.cc` - SSTable构建
- `table/block.cc` - Block结构
- `table/format.h` - 文件格式

**文件解剖**:
```
SSTable结构：
[Data Blocks] → 存储键值对（前缀压缩）
[Meta Block] → 统计信息
[Meta Index Block] → Meta块索引
[Index Block] → Data块索引
[Footer] → 48字节元信息
```

---

#### Day 6: Write-Ahead Log (WAL)
**核心概念**: 持久化、崩溃恢复、Record分片
- [ ] 理解WAL的持久化保证
- [ ] 掌握log文件格式
- [ ] 学习崩溃恢复流程

**关键文件**:
- `db/log_writer.h/cc` - WAL写入
- `db/log_reader.h/cc` - WAL读取
- `db/log_format.h` - Record格式

**性能权衡**:
```cpp
WriteOptions options;
options.sync = false;  // 快速：100k ops/sec
options.sync = true;   // 安全：100 ops/sec（慢1000倍）
```

---

#### Day 7: 读取路径与迭代器
**核心概念**: 多层查找、迭代器模式、快照
- [ ] 理解Get操作的完整流程
- [ ] 掌握MergingIterator实现
- [ ] 学习Snapshot机制

**关键文件**:
- `db/db_impl.cc` - Get实现
- `table/iterator.cc` - 迭代器基类
- `table/merger.cc` - 合并迭代器

**查找路径**:
```
MemTable → Immutable → Level-0 (4个文件) → Level-1+ (每层1个文件)
```

---

#### Day 8: 写入路径与WriteBatch
**核心概念**: 批量写入、Group Commit、原子性
- [ ] 理解WriteBatch的原子性保证
- [ ] 掌握Group Commit优化
- [ ] 学习写入流控机制

**关键文件**:
- `db/write_batch.cc` - 批量操作
- `db/db_impl.cc::Write()` - 写入实现

**吞吐量对比**:
```
单条写入：~10k ops/sec
批量写入(1000条)：~100k ops/sec（提升10倍）
```

---

### 第三阶段：核心机制（Day 9-11）

#### Day 9: Compaction机制（上）
**核心概念**: Minor Compaction、触发条件、选择策略
- [ ] 理解MemTable刷盘过程
- [ ] 掌握Compaction触发条件
- [ ] 学习文件选择算法

**关键文件**:
- `db/db_impl.cc::CompactMemTable()` - Minor Compaction
- `db/version_set.cc::PickCompaction()` - 文件选择

---

#### Day 10: Compaction机制（下）
**核心概念**: Major Compaction、合并算法、写放大
- [ ] 理解多文件合并算法
- [ ] 掌握删除标记的清理
- [ ] 分析写放大问题

**关键文件**:
- `db/db_impl.cc::DoCompactionWork()` - 执行Compaction
- `db/version_set.cc::Finalize()` - 计算Compaction分数

**写放大分析**:
```
理论最坏情况：
Level-0 → Level-1: 10倍写放大
Level-1 → Level-2: 10倍写放大
总写放大：100倍

实际场景：5-10倍写放大
```

---

#### Day 11: Version与VersionSet版本管理
**核心概念**: MVCC、VersionEdit、MANIFEST
- [ ] 理解Version的快照机制
- [ ] 掌握VersionEdit的增量更新
- [ ] 学习MANIFEST文件格式

**关键文件**:
- `db/version_set.h` - 版本管理
- `db/version_edit.h` - 变更记录

**版本链**:
```
dummy → v1(refs=0) → v2(refs=3) → v3(refs=1, current)
        可删除         被3个读操作引用   当前版本
```

---

### 第四阶段：性能优化（Day 12-14）

#### Day 12: Cache与Bloom Filter优化
**核心概念**: LRU缓存、TableCache、Bloom Filter
- [ ] 理解两级缓存体系
- [ ] 掌握LRU算法实现
- [ ] 学习Bloom Filter原理

**关键文件**:
- `util/cache.cc` - LRU缓存
- `util/bloom.cc` - Bloom Filter
- `db/table_cache.cc` - Table缓存

**优化效果**:
```
无Bloom Filter：每次查询需要读取SSTable
有Bloom Filter：99%的不存在键可以跳过文件读取
```

---

#### Day 13: 并发控制与线程安全
**核心概念**: 读写锁、后台线程、条件变量
- [ ] 理解LevelDB的锁策略
- [ ] 掌握后台Compaction线程
- [ ] 学习无锁读取优化

**关键文件**:
- `db/db_impl.h` - 锁和条件变量
- `port/port_stdcxx.h` - 平台抽象

**并发模型**:
```
写操作：单线程（持有mutex_）
读操作：多线程（引用计数，无锁）
后台Compaction：单线程
```

---

#### Day 14: 性能优化技巧与最佳实践
**核心概念**: 参数调优、问题诊断、生产配置
- [ ] 掌握关键参数调优
- [ ] 学习性能分析方法
- [ ] 理解常见问题解决方案

**调优参数**:
```cpp
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB
options.block_cache = NewLRUCache(512 * 1024 * 1024);  // 512MB
options.filter_policy = NewBloomFilterPolicy(10);  // 10 bits/key
```

---

## 🛠️ 实战项目

### 项目1：LevelDB性能基准测试工具
**难度**: ⭐⭐
**时间**: 2-3小时
**目标**: 编写自定义基准测试，分析不同配置的性能

```cpp
// bench_custom.cc
#include "leveldb/db.h"
#include <chrono>

void BenchmarkRandomWrites(leveldb::DB* db, int count) {
  auto start = std::chrono::high_resolution_clock::now();
  for (int i = 0; i < count; i++) {
    // 实现随机写入测试
  }
  auto end = std::chrono::high_resolution_clock::now();
  // 计算并输出吞吐量
}
```

---

### 项目2：LevelDB监控工具
**难度**: ⭐⭐⭐
**时间**: 4-6小时
**目标**: 实时监控数据库状态

功能：
- 显示各层文件数量和大小
- 监控Compaction进度
- 统计读写操作延迟
- 缓存命中率分析

---

### 项目3：简化版LSM-Tree实现
**难度**: ⭐⭐⭐⭐⭐
**时间**: 2-3周
**目标**: 从零实现一个简化版的LSM-Tree存储引擎

核心功能：
1. MemTable（基于SkipList）
2. WAL日志
3. SSTable文件（简化版）
4. 两层Compaction（L0 → L1）

参考代码框架：
```cpp
class SimpleLSM {
 private:
  SkipList* memtable_;
  WALWriter* wal_;
  std::vector<SSTable*> level0_;
  std::vector<SSTable*> level1_;

 public:
  void Put(const std::string& key, const std::string& value);
  bool Get(const std::string& key, std::string* value);
  void Compact();
};
```

---

## 🔍 深度阅读资源

### 必读论文
1. **The Log-Structured Merge-Tree (LSM-Tree)**
   - 作者: Patrick O'Neil et al.
   - 链接: http://www.cs.umb.edu/~poneil/lsmtree.pdf
   - 重要性: LSM-Tree的开山之作

2. **Bigtable: A Distributed Storage System**
   - 作者: Google
   - LevelDB的设计灵感来源

3. **LevelDB实现文档**
   - doc/impl.md - 实现细节
   - doc/table_format.md - 表格式
   - doc/log_format.md - 日志格式

### 相关系统对比

| 特性 | LevelDB | RocksDB | BadgerDB |
|------|---------|---------|----------|
| 语言 | C++ | C++ | Go |
| 写放大 | 中等 | 可调 | 低（WiscKey） |
| 并发 | 单写多读 | 多线程 | 多线程 |
| 压缩 | Snappy | 多种 | Zstd |
| 应用 | Chrome, Bitcoin | Meta产品 | Dgraph |

### 推荐书籍
1. **Database Internals** - Alex Petrov
2. **Designing Data-Intensive Applications** - Martin Kleppmann

---

## 💡 学习建议

### 初学者路径（2周）
1. 先通读所有14天课程（理解概念）
2. 重点深入Day 1, 3, 5, 6（核心模块）
3. 完成所有代码示例
4. 做项目1和项目2

### 进阶路径（1个月）
1. 完整学习所有课程
2. 阅读所有源码文件
3. 完成项目3（简化LSM实现）
4. 阅读相关论文
5. 对比研究RocksDB的改进

### 专家路径（3个月）
1. 深度源码分析（逐行理解）
2. 性能分析和优化实验
3. 实现完整的LSM-Tree引擎
4. 贡献到开源项目

---

## 📊 学习检查清单

### 基础概念（Day 1-4）
- [ ] 能够解释LSM-Tree的工作原理
- [ ] 理解Slice的零拷贝设计
- [ ] 能够实现一个简单的SkipList
- [ ] 理解InternalKey的编码格式

### 存储引擎（Day 5-8）
- [ ] 能够手绘SSTable文件结构
- [ ] 理解WAL的持久化保证
- [ ] 能够解释读取路径的层次
- [ ] 理解WriteBatch的原子性

### 核心机制（Day 9-11）
- [ ] 能够解释Compaction的触发条件
- [ ] 理解写放大的来源
- [ ] 理解Version的MVCC机制

### 性能优化（Day 12-14）
- [ ] 能够调优LevelDB参数
- [ ] 理解Bloom Filter的工作原理
- [ ] 能够诊断性能问题

---

## 🎓 考核方式

### 理论考核
1. 画出LevelDB的完整架构图
2. 解释LSM-Tree相比B+Tree的优劣
3. 分析一次Get操作的完整路径
4. 计算在特定场景下的写放大系数

### 实践考核
1. 编写性能基准测试程序
2. 实现一个简化版的MemTable
3. 调优LevelDB以达到特定性能指标
4. 分析并解决一个性能问题

---

## 🤝 社区和帮助

### 官方资源
- GitHub: https://github.com/google/leveldb
- 文档: leveldb/doc/
- 问题跟踪: GitHub Issues

### 学习交流
- 遇到问题先查阅源码注释
- 参考单元测试用例
- 使用gdb调试源码

---

## 📝 学习笔记模板

```markdown
# Day X 学习笔记

## 今日重点
-

## 关键代码片段
```cpp
// 代码示例
```

## 遇到的问题
1.

## 解决方案
1.

## 思考题答案
1.

## 下一步计划
-
```

---

**祝你学习愉快，深入理解LevelDB的精妙设计！** 🚀

如有问题，请随时查阅源码或参考单元测试。记住：**源码是最好的老师**。
