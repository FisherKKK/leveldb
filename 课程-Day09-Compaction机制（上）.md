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

### 2.2 CompactMemTable详细流程

#### 2.2.1 完整代码分析

```cpp
// db/db_impl.cc, lines 549-580
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();  // 前置条件：必须持有互斥锁
  assert(imm_ != nullptr);  // 确保有Immutable MemTable需要刷盘

  // ========== 阶段1：准备刷盘 ==========
  // 创建VersionEdit记录本次变更
  VersionEdit edit;

  // 获取当前Version（快照）
  Version* base = versions_->current();
  base->Ref();  // 增加引用计数，防止被删除

  // 核心调用：将Immutable MemTable写入SSTable
  Status s = WriteLevel0Table(imm_, &edit, base);

  base->Unref();  // 减少引用计数

  // ========== 阶段2：检查关闭状态 ==========
  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) {
    s = Status::IOError("Deleting DB during memtable compaction");
  }

  // ========== 阶段3：应用版本变更 ==========
  if (s.ok()) {
    // 标记旧的WAL文件可以删除
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);

    // 写入MANIFEST并切换到新Version
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  // ========== 阶段4：清理与提交 ==========
  if (s.ok()) {
    // 成功：清理Immutable MemTable
    imm_->Unref();
    imm_ = nullptr;
    has_imm_.store(false, std::memory_order_release);

    // 删除旧文件（包括WAL）
    RemoveObsoleteFiles();
  } else {
    // 失败：记录错误
    RecordBackgroundError(s);
  }
}
```

#### 2.2.2 执行流程图

```
CompactMemTable完整流程：

线程: 后台Compaction线程
状态: 持有mutex_

┌─────────────────────────────────────────────┐
│ 1. 检查状态                                │
│    assert(imm_ != nullptr)                 │
│    ✓ 有Immutable MemTable需要处理          │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 2. 准备刷盘                                │
│    VersionEdit edit;                        │
│    Version* base = versions_->current();    │
│    base->Ref();  ← 保护当前Version          │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 3. WriteLevel0Table                         │
│    ├─ 创建文件编号: 123                     │
│    ├─ pending_outputs_.insert(123)         │
│    ├─ mutex_.Unlock()                      │
│    ├─ BuildTable(...)                      │
│    │   ├─ 遍历MemTable                     │
│    │   ├─ 写入SSTable                      │
│    │   └─ fsync (持久化)                   │
│    ├─ mutex_.Lock()                        │
│    └─ pending_outputs_.erase(123)          │
│    ✓ 生成 /tmp/db/000123.ldb (2MB)        │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 4. 应用版本变更                            │
│    edit.SetLogNumber(logfile_number_);     │
│    versions_->LogAndApply(&edit, &mutex_);  │
│    ├─ 写入MANIFEST                         │
│    │   add file 123 at level-0             │
│    │   log_number: 456                     │
│    ├─ fsync MANIFEST                       │
│    ├─ 创建新Version                        │
│    │   └─ files_[0].push_back(file_123)    │
│    └─ current_ = new_version               │
│    ✓ 数据库状态更新完成                    │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 5. 清理工作                                │
│    imm_->Unref();                           │
│    imm_ = nullptr;                          │
│    has_imm_.store(false);                   │
│    RemoveObsoleteFiles();                   │
│    ├─ 删除WAL: 000456.log                  │
│    └─ 删除其他过期文件                     │
│    ✓ 清理完成                              │
└─────────────────────────────────────────────┘
```

#### 2.2.3 线程安全分析

**问题：为什么CompactMemTable需要持锁？**

```cpp
// 潜在竞争条件
Thread1 (写入)           Thread2 (Compaction)
    │                        │
    │ db->Put(k, v)          │
    │ ├─ 检查mem_            │
    │ ├─ 写入mem_            │ CompactMemTable()
    │ └─ 返回                │ ├─ imm_ = nullptr  ❌
    │                        │ └─ imm_->Unref()  ❌
    │                        │

// 正确的加锁顺序
Thread1 (写入)           Thread2 (Compaction)
    │                        │
    │ mutex_.Lock()          │ mutex_.Lock()
    │ db->Put(k, v)          │ 等待...
    │ ├─ 检查mem_            │  (阻塞)
    │ ├─ 写入mem_            │
    │ mutex_.Unlock()        │ ✓ 获得锁
    │                        │ CompactMemTable()
    │                        │ ├─ imm_ = nullptr  ✓
    │                        │ └─ imm_->Unref()  ✓
```

**关键不变式：**

```cpp
// 不变式1：读写互斥
mutex_.Lock()
  // 要么读，要么写，不能同时
  if (writing) {
    // 前台写入操作
  } else {
    // Compaction操作
  }
mutex_.Unlock()

// 不变式2：imm_的生命周期
if (imm_ != nullptr) {
  // imm_必须保持有效，直到：
  // 1. WriteLevel0Table完成
  // 2. LogAndApply完成
  // 3. imm_ = nullptr
}

// 不变式3：Version切换的原子性
current_->Ref();      // 增加引用
WriteLevel0Table();   // 使用current_
current_->Unref();    // 释放引用
// 确保current_在使用期间不被删除
```

#### 2.2.4 WAL文件的清理

```cpp
// WAL文件生命周期：
// 创建 → 活跃使用 → Immutable → 过期 → 删除

写入阶段:
┌──────────────────────────────────────┐
│  WAL: 000123.log (活跃)              │
│  ├─ 记录所有写入操作                 │
│  └─ 用于崩溃恢复                     │
└──────────────────────────────────────┘
            │ MemTable满
            ▼
切换阶段:
┌──────────────────────────────────────┐
│  mem_  (新的活跃MemTable)            │
│  imm_ (Immutable，准备刷盘)          │
│                                      │
│  WAL: 000123.log (可以删除)          │
│  └─ 数据已在imm_中                   │
└──────────────────────────────────────┘
            │ CompactMemTable
            ▼
刷盘阶段:
┌──────────────────────────────────────┐
│  imm_ → SSTable: 000456.ldb          │
│                                      │
│  edit.SetLogNumber(456);             │
│  └─ 标记000456.log之前都过期         │
└──────────────────────────────────────┘
            │ RemoveObsoleteFiles
            ▼
清理阶段:
┌──────────────────────────────────────┐
│  删除: 000123.log                    │
│  删除: 其他旧.log文件                │
│                                      │
│  当前WAL: 000456.log (活跃)          │
└──────────────────────────────────────┘
```

**代码实现：**

```cpp
// db/filename.cc
// WAL文件命名规则: {log_number}.log
// 例如: 000123.log, 000456.log

// db/version_set.cc:LogAndApply
Status VersionSet::LogAndApply(VersionEdit* edit, port::Mutex* mu) {
  // ...
  if (edit->has_log_number_) {
    // 记录新的log_number
    // 之前的WAL文件（log_number更小的）都可以删除了
    log_number_ = edit->log_number_;
  }

  // 写入MANIFEST
  manifest_log_->AddRecord(edit->EncodeTo());
  manifest_file_->Sync();

  // 应用到当前Version
  builder->Apply(edit);
  // ...
}

// db/db_impl.cc:RemoveObsoleteFiles
void DBImpl::RemoveObsoleteFiles() {
  // 遍历数据库目录
  std::vector<std::string> filenames;
  env_->GetChildren(dbname_, &filenames);

  for (const auto& filename : filenames) {
    uint64_t number;
    FileType type;
    if (!ParseFileName(filename, &number, &type)) {
      continue;
    }

    // 删除过期的WAL文件
    if (type == kLogFile) {
      if (number >= log_number_) {
        // 当前或未来的WAL，不能删除
        continue;
      }
      // 删除旧的WAL
      env_->DeleteFile(dbname_ + "/" + filename);
    }
  }
}
```

#### 2.2.5 错误处理

```cpp
// CompactMemTable的错误处理策略

if (s.ok()) {
  // 成功路径
  imm_->Unref();
  imm_ = nullptr;
  has_imm_.store(false, std::memory_order_release);
  RemoveObsoleteFiles();
} else {
  // 失败路径
  RecordBackgroundError(s);
  // 注意：imm_仍然存在！
  // 下次Compaction会重试
}

// 错误类型：
// 1. IOError: 磁盘满
//    → imm_保留，等待重试
// 2. Corruption: SSTable损坏
//    → imm_保留，等待重试
// 3. Shutdown: 数据库关闭
//    → 取消Compaction

// db/db_impl.cc:RecordBackgroundError
void DBImpl::RecordBackgroundError(const Status& s) {
  mutex_.AssertHeld();
  if (bg_error_.ok()) {
    // 只记录第一个错误
    bg_error_ = s;
    // 通知所有等待的线程
    background_work_finished_signal_.SignalAll();
  }
}

// 前台操作检查错误
Status DBImpl::Put(const WriteOptions& options, const Slice& key,
                   const Slice& value) {
  // ...
  {
    MutexLock l(&mutex_);
    if (!bg_error_.ok()) {
      // 后台有错误，前台写入失败
      return bg_error_;
    }
    // 正常写入流程
  }
}
```

### 2.3 WriteLevel0Table深度分析

#### 2.3.1 完整代码逐行解析

```cpp
// db/db_impl.cc, lines 505-547
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  // ========== 阶段1：准备工作 ==========
  mutex_.AssertHeld();  // 确保持有互斥锁

  // 记录开始时间（用于性能统计）
  const uint64_t start_micros = env_->NowMicros();

  // 创建文件元数据
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();  // 分配文件编号

  // 关键：将文件编号加入pending_outputs_
  // 防止在文件完全写入前被删除
  pending_outputs_.insert(meta.number);

  // 创建MemTable迭代器（用于遍历所有键值对）
  Iterator* iter = mem->NewIterator();

  // 记录日志
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  // ========== 阶段2：构建SSTable（关键区） ==========
  Status s;
  {
    mutex_.Unlock();  // ⚠️ 释放锁，允许其他线程操作数据库

    // 核心调用：将MemTable内容写入SSTable
    // db/builder.cc:BuildTable
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);

    mutex_.Lock();    // ⚠️ 重新获取锁
  }

  // ========== 阶段3：完成处理 ==========
  // 记录完成日志
  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number,
      (unsigned long long)meta.file_size,
      s.ToString().c_str());

  // 清理迭代器
  delete iter;

  // 从pending集合中移除（文件现在可以被安全删除）
  pending_outputs_.erase(meta.number);

  // ========== 阶段4：选择目标层级 ==========
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    // 获取键范围
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();

    // 智能选择：可能跳过Level-0直接写到Level-2
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }

    // 添加到VersionEdit（记录这次变更）
    edit->AddFile(level, meta.number, meta.file_size,
                  meta.smallest, meta.largest);
  }

  // ========== 阶段5：统计信息 ==========
  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);

  return s;
}
```

#### 2.3.2 内存布局分析

```
WriteLevel0Table内存布局：

┌─────────────────────────────────────────────┐
│  DBImpl::WriteLevel0Table 栈帧              │
├─────────────────────────────────────────────┤
│  FileMetaData meta (栈上)                   │
│  ├─ number: 123                             │
│  ├─ file_size: 0 (初始)                    │
│  └─ smallest/largest: InternalKey           │
├─────────────────────────────────────────────┤
│  Iterator* iter (堆上)                      │
│  └─ 指向 MemTable的SkipList                 │
├─────────────────────────────────────────────┤
│  pending_outputs_ (全局集合)                │
│  └─ {123} ← 防止删除                        │
└─────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│  BuildTable 调用                            │
├─────────────────────────────────────────────┤
│  WritableFile* file                         │
│  └─ fd: /tmp/db/123.ldb                     │
├─────────────────────────────────────────────┤
│  TableBuilder* builder                      │
│  ├─ buffer: std::string (4KB block)        │
│  └─ 压缩器                                  │
└─────────────────────────────────────────────┘
```

#### 2.3.3 关键设计点

**设计点1：锁的释放与获取**

```cpp
{
  mutex_.Unlock();
  s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
  mutex_.Lock();
}
```

**为什么需要释放锁？**

```
不释放锁的问题：
┌──────────────┐        ┌──────────────┐
│ Compaction   │        │  前台写入    │
│  Thread      │        │   Thread     │
└──────┬───────┘        └──────┬───────┘
       │ 持有mutex_              │ 等待mutex_
       │ BuildTable...           │ (阻塞!)
       │ 50-100ms                │
       │                         │ ❌ 写入延迟

释放锁的好处：
┌──────────────┐        ┌──────────────┐
│ Compaction   │        │  前台写入    │
│  Thread      │        │   Thread     │
└──────┬───────┘        └──────┬───────┘
       │ 释放mutex_             │ 获取mutex_
       │ BuildTable...          │ 继续写入 ✓
       │ 50-100ms               │
       │ 重新获取mutex_         │
       │                         │ ✅ 并发执行
```

**设计点2：pending_outputs_的作用**

```cpp
pending_outputs_.insert(meta.number);  // 写入前
// ... BuildTable ...
pending_outputs_.erase(meta.number);   // 写入后
```

**防止文件被误删除：**

```cpp
// db/db_impl.cc:DeleteObsoleteFiles
void DBImpl::DeleteObsoleteFiles() {
  // ... 遍历需要删除的文件
  if (pending_outputs_.count(file_number) > 0) {
    // 文件正在写入，跳过删除！
    continue;
  }
  // 安全删除
  env_->DeleteFile(file_name);
}
```

**时序图：**

```
T1: WriteLevel0Table开始
    pending_outputs_.insert(123)
    └─> 文件123被标记为"写入中"

T2: 另一个线程调用DeleteObsoleteFiles()
    检查: pending_outputs_.count(123) > 0
    └─> 跳过删除123.ldb

T3: BuildTable完成
    pending_outputs_.erase(123)
    └─> 文件123可以被删除了

T4: 下次DeleteObsoleteFiles()
    检查: pending_outputs_.count(123) == 0
    └─> 可以删除123.ldb
```

#### 2.3.4 BuildTable深度解析

```cpp
// db/builder.cc, lines 17-80
Status BuildTable(const std::string& dbname, Env* env, const Options& options,
                  TableCache* table_cache, Iterator* iter, FileMetaData* meta) {
  Status s;
  meta->file_size = 0;  // 初始化文件大小

  // ========== 步骤1：准备迭代器 ==========
  iter->SeekToFirst();  // 定位到第一个键值对

  // ========== 步骤2：创建文件 ==========
  std::string fname = TableFileName(dbname, meta->number);
  // 例如: /tmp/db/000123.ldb

  if (iter->Valid()) {  // MemTable不为空
    // 创建可写文件
    WritableFile* file;
    s = env->NewWritableFile(fname, &file);
    if (!s.ok()) {
      return s;  // 创建失败
    }

    // ========== 步骤3：构建SSTable ==========
    TableBuilder* builder = new TableBuilder(options, file);

    // 记录最小键（第一个键）
    meta->smallest.DecodeFrom(iter->key());

    Slice key;
    // 遍历所有键值对
    for (; iter->Valid(); iter->Next()) {
      key = iter->key();
      builder->Add(key, iter->value());  // 添加到builder
    }

    // 记录最大键（最后一个键）
    if (!key.empty()) {
      meta->largest.DecodeFrom(key);
    }

    // ========== 步骤4：完成构建 ==========
    // Flush所有buffer，写入索引块和Footer
    s = builder->Finish();
    if (s.ok()) {
      meta->file_size = builder->FileSize();
      assert(meta->file_size > 0);
    }
    delete builder;

    // ========== 步骤5：同步到磁盘 ==========
    if (s.ok()) {
      s = file->Sync();   // ⚠️ fsync确保数据持久化
    }
    if (s.ok()) {
      s = file->Close();  // 关闭文件
    }
    delete file;
    file = nullptr;

    // ========== 步骤6：验证SSTable ==========
    if (s.ok()) {
      // 打开刚创建的SSTable，验证可读性
      Iterator* it = table_cache->NewIterator(
          ReadOptions(), meta->number, meta->file_size);
      s = it->status();
      delete it;
    }
  }

  // ========== 步骤7：错误处理 ==========
  if (!iter->status().ok()) {
    s = iter->status();
  }

  // 如果失败，删除不完整的文件
  if (s.ok() && meta->file_size > 0) {
    // 保留文件
  } else {
    env->RemoveFile(fname);
  }

  return s;
}
```

**BuildTable数据流：**

```
MemTable (SkipList)
    │
    │ Iterator
    ▼
┌─────────────────────────────────┐
│  TableBuilder                   │
├─────────────────────────────────┤
│  Data Block 1 (4KB)             │
│  ├─ key1: value1                │
│  ├─ key2: value2                │
│  └─ ...                         │
├─────────────────────────────────┤
│  Data Block 2 (4KB)             │
│  ├─ ...                         │
├─────────────────────────────────┤
│  ...                            │
├─────────────────────────────────┤
│  Filter Block (Bloom Filter)    │
├─────────────────────────────────┤
│  Index Block                    │
│  ├─ Block1: offset, size        │
│  ├─ Block2: offset, size        │
│  └─ ...                         │
├─────────────────────────────────┤
│  Footer (48 bytes)              │
└─────────────────────────────────┘
    │
    │ WriteableFile
    ▼
/tmp/db/000123.ldb
```

#### 2.3.5 TableBuilder工作原理

```cpp
// table/table_builder.cc (简化版)
class TableBuilder {
 public:
  void Add(const Slice& key, const Slice& value) {
    // 1. 检查是否需要重启Block
    if (block_builder_.FileSize() >= options_.block_size) {
      Flush();  // 当前Block满，写入文件
    }

    // 2. 添加到当前Block（前缀压缩）
    block_builder_.Add(key, value);

    // 3. 更新过滤器
    if (filter_policy_) {
      filter_block_.Add(key);
    }
  }

  Status Finish() {
    // 1. Flush最后一个Data Block
    if (!block_buffer_.empty()) {
      Flush();
    }

    // 2. 写入Filter Block
    WriteFilterBlock();

    // 3. 写入Index Block
    WriteIndexBlock();

    // 4. 写入Footer
    WriteFooter();

    return status_;
  }

 private:
  void Flush() {
    // 压缩Block
    std::string compressed;
    if (compressor_) {
      compressor_->Compress(block_buffer_, &compressed);
    } else {
      compressed = block_buffer_;
    }

    // 写入文件
    file_->Append(compressed);

    // 记录到Index
    index_block_.Add(last_key, offset, size);

    // 清空buffer
    block_buffer_.clear();
  }
};
```

**前缀压缩示例：**

```
原始键值：
  user:001 → "Alice"
  user:002 → "Bob"
  user:003 → "Charlie"

不压缩（每个键完整存储）：
  [12字节"user:001"] [5字节"Alice"]
  [12字节"user:002"] [3字节"Bob"]
  [12字节"user:003"] [9字节"Charlie"]
  总计: 53字节

前缀压缩（只存储差异）：
  [12字节"user:001"] [5字节"Alice"]
  [3字节"002"] [3字节"Bob"]         ← 共享前缀"user:00"
  [3字节"003"] [9字节"Charlie"]
  总计: 37字节 (节省30%)

压缩算法：
  1. 计算当前key与上一次key的公共前缀长度
  2. 存储: [共享长度][非共享长度][非共享部分][value]
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
