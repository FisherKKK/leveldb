# Day 9: Compaction机制（上）

## 学习目标
- 理解Compaction的作用和类型
- 掌握Minor Compaction触发和执行
- 学习Major Compaction的基本概念
- 理解Size-tiered策略

## 1. Compaction概述

### 1.1 为什么需要Compaction?

**LSM-Tree的问题：**

```
写入流程：
MemTable → Level-0 → Level-1 → Level-2 → ...

问题1：Level-0文件堆积
- MemTable刷盘产生Level-0文件
- Level-0文件键范围可能重叠
- 读取需要查所有Level-0文件 → 性能下降

问题2：数据冗余
- 同一个键有多个版本
- 删除标记占用空间
- 浪费磁盘空间

问题3：读放大
- 查找需要遍历多层
- 层数越多，读取越慢
```

**Compaction解决方案：**

```
Compaction = 压缩 + 整理

作用：
1. 合并重叠文件
2. 删除过期版本
3. 清理删除标记
4. 重新组织数据层次

结果：
- 减少文件数量
- 提升读性能
- 回收磁盘空间
```

### 1.2 Compaction类型

```cpp
// Minor Compaction (MemTable → Level-0)
MemTable (4MB)
  ↓ Minor Compaction
Level-0 SSTable (2MB)

// Major Compaction (Level-N → Level-N+1)
Level-0: [file1, file2, file3, file4]  (重叠)
  ↓ Major Compaction
Level-1: [file5, file6, file7]  (不重叠)
  ↓ Major Compaction
Level-2: [file8, file9, ..., fileN]  (更大)
```

**对比：**

| 类型 | 输入 | 输出 | 触发条件 | 频率 |
|------|------|------|---------|------|
| Minor | MemTable | Level-0 SSTable | MemTable满 | 高 |
| Major | Level-N + Level-N+1 | Level-N+1 | Level-N文件过多 | 中低 |

### 1.3 Compaction的权衡

```
收益：
✓ 减少文件数量 → 提升读性能
✓ 删除冗余数据 → 节省空间
✓ 整理数据层次 → 维护LSM-Tree结构

代价：
✗ 写放大：数据被多次重写
✗ CPU开销：排序、合并
✗ I/O开销：读取旧文件、写入新文件
✗ 影响前台性能：后台线程竞争资源
```

## 2. Minor Compaction

### 2.1 触发条件

```cpp
// db/db_impl.cc, lines 1311-1413
Status DBImpl::MakeRoomForWrite(bool force) {
  // ...
  if (!force && (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
    // MemTable未满，继续写入
    break;
  } else {
    // MemTable满，切换为Immutable
    imm_ = mem_;
    mem_ = new MemTable(internal_comparator_);
    mem_->Ref();
    MaybeScheduleCompaction();  // 触发Minor Compaction
  }
}
```

**触发流程：**

```
写入操作
  ↓
MakeRoomForWrite()
  ↓
MemTable大小 >= write_buffer_size (4MB)
  ↓
mem_ → imm_ (标记为Immutable)
  ↓
分配新的MemTable
  ↓
MaybeScheduleCompaction()
  ↓
后台线程执行CompactMemTable()
```

### 2.2 CompactMemTable实现

```cpp
// db/db_impl.cc, lines 766-823
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != nullptr);

  // 保存imm_到Level-0
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
  Status s = WriteLevel0Table(imm_, &edit, base);
  base->Unref();

  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) {
    s = Status::IOError("Deleting DB during memtable compaction");
  }

  // 应用版本变更
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  if (s.ok()) {
    // 删除旧的WAL文件
    DeleteObsoleteFiles();
    // 清空imm_
    imm_->Unref();
    imm_ = nullptr;
    has_imm_.store(false, std::memory_order_release);
  } else {
    RecordBackgroundError(s);
  }
}
```

### 2.3 WriteLevel0Table实现

```cpp
// db/db_impl.cc, lines 825-915
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                 Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  Iterator* iter = mem->NewIterator();
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  Status s;
  {
    mutex_.Unlock();
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number, (unsigned long long)meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);

  // 选择合适的层级
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size, meta.smallest,
                  meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}
```

**BuildTable实现：**

```cpp
// db/builder.cc, lines 25-95
Status BuildTable(const std::string& dbname, Env* env, const Options& options,
                  TableCache* table_cache, Iterator* iter, FileMetaData* meta) {
  Status s;
  meta->file_size = 0;
  iter->SeekToFirst();

  std::string fname = TableFileName(dbname, meta->number);
  if (iter->Valid()) {
    WritableFile* file;
    s = env->NewWritableFile(fname, &file);
    if (!s.ok()) {
      return s;
    }

    TableBuilder* builder = new TableBuilder(options, file);
    meta->smallest.DecodeFrom(iter->key());
    Slice key;
    for (; iter->Valid(); iter->Next()) {
      key = iter->key();
      builder->Add(key, iter->value());
    }
    if (!key.empty()) {
      meta->largest.DecodeFrom(key);
    }

    // 完成构建
    s = builder->Finish();
    if (s.ok()) {
      meta->file_size = builder->FileSize();
      assert(meta->file_size > 0);
    }
    delete builder;

    // 确保写入磁盘
    if (s.ok()) {
      s = file->Sync();
    }
    if (s.ok()) {
      s = file->Close();
    }
    delete file;

    if (s.ok()) {
      // 验证SSTable
      Iterator* it = table_cache->NewIterator(ReadOptions(), meta->number,
                                               meta->file_size);
      s = it->status();
      delete it;
    }
  }

  // 检查输入迭代器
  if (!iter->status().ok()) {
    s = iter->status();
  }

  if (s.ok() && meta->file_size > 0) {
    // 保留文件
  } else {
    env->RemoveFile(fname);
  }
  return s;
}
```

### 2.4 PickLevelForMemTableOutput

```cpp
// db/version_set.cc, lines 1237-1274
int Version::PickLevelForMemTableOutput(const Slice& smallest_user_key,
                                         const Slice& largest_user_key) {
  int level = 0;
  if (!OverlapInLevel(0, &smallest_user_key, &largest_user_key)) {
    // Level-0没有重叠，尝试推送到更高层
    InternalKey start(smallest_user_key, kMaxSequenceNumber, kValueTypeForSeek);
    InternalKey limit(largest_user_key, 0, static_cast<ValueType>(0));
    std::vector<FileMetaData*> overlaps;
    while (level < config::kMaxMemCompactLevel) {
      if (OverlapInLevel(level + 1, &smallest_user_key, &largest_user_key)) {
        break;
      }
      if (level + 2 < config::kNumLevels) {
        // 检查Level+2的重叠（避免过多Compaction）
        GetOverlappingInputs(level + 2, &start, &limit, &overlaps);
        const int64_t sum = TotalFileSize(overlaps);
        if (sum > MaxGrandParentOverlapBytes(vset_->options_)) {
          break;
        }
      }
      level++;
    }
  }
  return level;
}
```

**层级选择策略：**

```
目标：尽可能将SSTable推送到更高层

条件1：Level-0无重叠
条件2：Level-1无重叠
条件3：Level-2重叠 < 20MB (MaxGrandParentOverlapBytes)

示例：
MemTable: [100, 200]

Level-0: [10, 50], [60, 90]  ← 无重叠
Level-1: [110, 150]          ← 有重叠
结果：放到Level-0

Level-0: [10, 50], [60, 90]  ← 无重叠
Level-1: [110, 150], [250, 300]  ← 无重叠
Level-2: [105, 120], [180, 220]  ← 重叠<20MB
结果：放到Level-2

为什么这样做？
- 减少Level-0文件数量
- 减少后续Major Compaction
- 平衡各层大小
```

## 3. Major Compaction概述

### 3.1 触发条件

```cpp
// db/version_set.cc, lines 1321-1375
void VersionSet::Finalize(Version* v) {
  int best_level = -1;
  double best_score = -1;

  for (int level = 0; level < config::kNumLevels - 1; level++) {
    double score;
    if (level == 0) {
      // Level-0: 基于文件数量
      score = v->files_[level].size() /
              static_cast<double>(config::kL0_CompactionTrigger);
    } else {
      // Level-1+: 基于总大小
      const uint64_t level_bytes = TotalFileSize(v->files_[level]);
      score = static_cast<double>(level_bytes) / MaxBytesForLevel(options_, level);
    }

    if (score > best_score) {
      best_level = level;
      best_score = score;
    }
  }

  v->compaction_level_ = best_level;
  v->compaction_score_ = best_score;
}
```

**评分公式：**

```
Level-0得分：
score = 文件数量 / kL0_CompactionTrigger (4)

Level-N得分 (N >= 1)：
score = 总字节数 / MaxBytesForLevel(N)

MaxBytesForLevel(N)：
Level-1: 10MB
Level-2: 100MB
Level-3: 1000MB (1GB)
Level-N: 10^N MB

触发条件：
score >= 1.0
```

**示例：**

```
Level-0: 5个文件
score = 5 / 4 = 1.25  ← 需要Compaction

Level-1: 15MB
score = 15MB / 10MB = 1.5  ← 需要Compaction

Level-2: 80MB
score = 80MB / 100MB = 0.8  ← 不需要

选择：Level-1 (score=1.5最高)
```

### 3.2 Size-tiered策略

```
LSM-Tree分层策略：

Level-0:
- 文件数量：最多4个 (kL0_CompactionTrigger)
- 单个文件：~2MB
- 总大小：~8MB
- 特点：键范围可能重叠

Level-1:
- 目标大小：10MB
- 单个文件：~2MB
- 文件数量：~5个
- 特点：键范围不重叠

Level-2:
- 目标大小：100MB (10x Level-1)
- 单个文件：~2MB
- 文件数量：~50个

Level-N:
- 目标大小：10^N MB
- 扩大系数：10x
```

**为什么是10倍？**

```
权衡：
- 扩大系数越大：
  ✓ 层数越少
  ✓ 读放大越小
  ✗ 写放大越大
  ✗ 空间放大越大

- 扩大系数越小：
  ✓ 写放大越小
  ✓ 空间放大越小
  ✗ 层数越多
  ✗ 读放大越大

LevelDB选择10x：
- 平衡读写放大
- 限制层数（7层可存10TB）
- 工业界常用值
```

## 4. 后台线程调度

### 4.1 MaybeScheduleCompaction

```cpp
// db/db_impl.cc, lines 534-558
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  if (background_compaction_scheduled_) {
    // 已经调度
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // 正在关闭
  } else if (!bg_error_.ok()) {
    // 后台错误
  } else if (imm_ == nullptr && manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
    // 无需Compaction
  } else {
    background_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);
  }
}

void DBImpl::BGWork(void* db) {
  reinterpret_cast<DBImpl*>(db)->BackgroundCall();
}

void DBImpl::BackgroundCall() {
  MutexLock l(&mutex_);
  assert(background_compaction_scheduled_);
  if (shutting_down_.load(std::memory_order_acquire)) {
    // 取消
  } else if (!bg_error_.ok()) {
    // 错误
  } else {
    BackgroundCompaction();
  }
  background_compaction_scheduled_ = false;

  // 继续调度
  MaybeScheduleCompaction();
  background_work_finished_signal_.SignalAll();
}
```

### 4.2 BackgroundCompaction

```cpp
// db/db_impl.cc, lines 610-698
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  if (imm_ != nullptr) {
    // 优先处理Immutable MemTable
    CompactMemTable();
    return;
  }

  Compaction* c;
  bool is_manual = (manual_compaction_ != nullptr);
  InternalKey manual_end;
  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    c = versions_->CompactRange(m->level, m->begin, m->end);
    m->done = (c == nullptr);
    if (c != nullptr) {
      manual_end = c->input(0, c->num_input_files(0) - 1)->largest;
    }
  } else {
    c = versions_->PickCompaction();
  }

  Status status;
  if (c == nullptr) {
    // 无需Compaction
  } else if (!is_manual && c->IsTrivialMove()) {
    // 优化：直接移动文件到下一层
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0);
    c->edit()->RemoveFile(c->level(), f->number);
    c->edit()->AddFile(c->level() + 1, f->number, f->file_size, f->smallest,
                       f->largest);
    status = versions_->LogAndApply(c->edit(), &mutex_);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    VersionSet::LevelSummaryStorage tmp;
    Log(options_.info_log, "Moved #%lld to level-%d %lld bytes %s: %s\n",
        static_cast<unsigned long long>(f->number), c->level() + 1,
        static_cast<unsigned long long>(f->file_size),
        status.ToString().c_str(), versions_->LevelSummary(&tmp));
  } else {
    CompactionState* compact = new CompactionState(c);
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    CleanupCompaction(compact);
    c->ReleaseInputs();
    RemoveObsoleteFiles();
  }
  delete c;

  if (status.ok()) {
    // 完成
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // 忽略关闭时的错误
  } else {
    Log(options_.info_log, "Compaction error: %s", status.ToString().c_str());
  }

  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    if (!status.ok()) {
      m->done = true;
    }
    if (!m->done) {
      m->tmp_storage = manual_end;
      m->begin = &m->tmp_storage;
    }
    manual_compaction_ = nullptr;
  }
}
```

## 5. 性能分析

### 5.1 Minor Compaction性能

```
单次Minor Compaction：
- 输入：4MB MemTable
- 输出：~2MB SSTable (Snappy压缩)
- 时间：~100ms

性能瓶颈：
1. 迭代MemTable：~10ms
2. 写入SSTable：~50ms
3. fsync：~30ms
4. 更新Version：~10ms

吞吐量：
- 4MB / 100ms = 40MB/s
- 对前台写入影响：轻微（后台执行）
```

### 5.2 Major Compaction性能

```
Level-0 → Level-1 Compaction：
- 输入Level-0：4个文件 × 2MB = 8MB
- 输入Level-1：重叠文件 ~10MB
- 输出Level-1：~15MB (去重后)
- 时间：~500ms

性能分析：
- 读取：18MB / 500ms = 36MB/s
- 写入：15MB / 500ms = 30MB/s
- CPU：排序、合并
- 写放大：15MB写入 / 8MB输入 = 1.875x

对前台影响：
- I/O竞争：读写磁盘
- CPU竞争：排序合并
- 延迟增加：~10-50ms
```

## 6. 代码实践

### 6.1 触发Compaction测试

```cpp
#include "leveldb/db.h"
#include <iostream>
#include <thread>
#include <chrono>

using namespace leveldb;

int main() {
  DB* db;
  Options options;
  options.create_if_missing = true;
  options.write_buffer_size = 1 << 20;  // 1MB MemTable

  Status s = DB::Open(options, "/tmp/testdb", &db);
  if (!s.ok()) {
    std::cerr << "Open failed: " << s.ToString() << std::endl;
    return 1;
  }

  // 写入数据触发Compaction
  const int num_writes = 100000;
  std::string large_value(128, 'x');

  for (int i = 0; i < num_writes; i++) {
    std::string key = "key" + std::to_string(i);
    s = db->Put(WriteOptions(), key, large_value);
    if (!s.ok()) {
      std::cerr << "Put failed: " << s.ToString() << std::endl;
      break;
    }

    if (i % 10000 == 0) {
      std::cout << "Wrote " << i << " keys\n";
      // 等待Compaction完成
      std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
  }

  std::cout << "Finished writing, waiting for compaction...\n";
  std::this_thread::sleep_for(std::chrono::seconds(5));

  delete db;
  return 0;
}
```

## 总结

今天我们学习了：
1. ✅ **Compaction作用**：合并文件、删除冗余、提升性能
2. ✅ **Minor Compaction**：MemTable → Level-0
3. ✅ **触发条件**：文件数量和层级大小
4. ✅ **Size-tiered策略**：10倍扩大系数

**关键要点：**
- Compaction是LSM-Tree的核心机制
- Minor Compaction频繁但快速
- Major Compaction代价高但必需
- 权衡读写放大和空间放大

**思考题：**
1. 为什么MemTable刷盘可能跳过Level-0直接到Level-2？
2. Level-0为什么基于文件数量而不是大小触发？
3. 如果关闭Compaction会发生什么？
4. 写放大系数如何计算？

**明天预告：Day 10 - Compaction机制（下）**
我们将深入学习Major Compaction的执行细节，包括PickCompaction算法和DoCompactionWork实现。
