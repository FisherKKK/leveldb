# Day 8: 写入路径与WriteBatch

## 学习目标
- 理解Put/Delete操作的完整流程
- 掌握WriteBatch的编码格式
- 学习Group Commit批量写入优化
- 理解MakeRoomForWrite流控机制

## 1. 写入路径概述

### 1.1 完整写入流程

**单个Put操作的生命周期：**

```
Client: db->Put("key", "value")
  ↓
┌─────────────────────────────────────┐
│ 1. 封装为WriteBatch                 │
│    batch.Put("key", "value")        │
└──────────┬──────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 2. Write(WriteBatch*)               │
│    ├─ 获取互斥锁                    │
│    ├─ Group Commit合并              │
│    └─ MakeRoomForWrite              │
└──────────┬──────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 3. 写WAL日志                        │
│    log_->AddRecord(batch)           │
│    if sync: fsync()                 │
└──────────┬──────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 4. 插入MemTable                     │
│    InsertInto(batch, mem_)          │
└──────────┬──────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 5. 返回Status                       │
│    唤醒等待线程                     │
└─────────────────────────────────────┘
```

### 1.2 为什么需要WriteBatch？

**问题1：原子性**
```cpp
// 需求：原子地更新多个键
db->Put("user:100:name", "Alice");
db->Put("user:100:age", "25");
// 问题：两次Put之间可能崩溃，导致不一致

// 解决方案：WriteBatch
WriteBatch batch;
batch.Put("user:100:name", "Alice");
batch.Put("user:100:age", "25");
db->Write(WriteOptions(), &batch);  // 原子写入
```

**问题2：性能优化**
```cpp
// 单个Put开销：
// - 获取锁：~100ns
// - 写WAL：~1μs
// - 写MemTable：~100ns
// - fsync（如果sync=true）：~10ms
// 总计：~10ms（如果sync）

// 批量写入开销：
// - 获取锁：~100ns
// - 写WAL（1000个操作）：~100μs
// - 写MemTable（1000个操作）：~100μs
// - fsync：~10ms
// 总计：~10ms（1000个操作共享fsync）
// 平均每个操作：~10μs（快1000倍！）
```

### 1.3 写入并发控制

```cpp
// db/db_impl.h, lines 174-206
class DBImpl : public DB {
 private:
  port::Mutex mutex_;                  // 保护以下状态
  MemTable* mem_;                      // 当前MemTable
  MemTable* imm_;                      // Immutable MemTable
  std::deque<Writer*> writers_;        // 等待写入的队列
  WriteBatch* tmp_batch_;              // 临时批次
};
```

**并发模型：**
```
写线程1: Put("a")    ┐
写线程2: Put("b")    ├─→ [等待队列] → Group Commit → 批量执行
写线程3: Put("c")    ┘

读线程：可以并发（无锁读取MemTable）
```

## 2. WriteBatch编码格式

### 2.1 内存布局

```cpp
// include/leveldb/write_batch.h, lines 28-60
class LEVELDB_EXPORT WriteBatch {
 public:
  WriteBatch();
  ~WriteBatch();

  void Put(const Slice& key, const Slice& value);
  void Delete(const Slice& key);
  void Clear();

  size_t ApproximateSize() const;

  void Append(const WriteBatch& source);

  Status Iterate(Handler* handler) const;

 private:
  friend class WriteBatchInternal;
  std::string rep_;  // 编码的数据
};
```

**rep_编码格式：**

```
WriteBatch内存布局:
┌──────────────┬──────────────┬─────────────────────────────┐
│  Sequence    │   Count      │   Records...                │
│  (8 bytes)   │  (4 bytes)   │   [Record1][Record2]...     │
└──────────────┴──────────────┴─────────────────────────────┘

Record格式:
Put Record:
┌──────┬──────────────┬──────────────┬────────┐
│ Type │ Key Length   │ Value Length │ Data   │
│ (1B) │ (varint32)   │ (varint32)   │ ...    │
└──────┴──────────────┴──────────────┴────────┘
       │←─ Key Bytes ─→│←─ Value Bytes ─→│

Delete Record:
┌──────┬──────────────┬────────┐
│ Type │ Key Length   │ Data   │
│ (1B) │ (varint32)   │ ...    │
└──────┴──────────────┴────────┘
       │←─ Key Bytes ─→│

Type值:
  kTypeDeletion = 0x0
  kTypeValue = 0x1
```

### 2.2 WriteBatchInternal

```cpp
// db/write_batch_internal.h, lines 12-32
class WriteBatchInternal {
 public:
  // 返回批次中的操作数量
  static int Count(const WriteBatch* batch);

  // 设置操作数量
  static void SetCount(WriteBatch* batch, int n);

  // 返回批次的序列号
  static SequenceNumber Sequence(const WriteBatch* batch);

  // 设置序列号
  static void SetSequence(WriteBatch* batch, SequenceNumber seq);

  // 返回编码内容
  static Slice Contents(const WriteBatch* batch) { return Slice(batch->rep_); }

  // 设置编码内容
  static void SetContents(WriteBatch* batch, const Slice& contents);

  // 插入到MemTable
  static Status InsertInto(const WriteBatch* batch, MemTable* memtable);

  // 追加到另一个批次
  static void Append(WriteBatch* dst, const WriteBatch* src);
};
```

### 2.3 编码实现

```cpp
// db/write_batch.cc, lines 27-82
void WriteBatch::Put(const Slice& key, const Slice& value) {
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  rep_.push_back(static_cast<char>(kTypeValue));
  PutLengthPrefixedSlice(&rep_, key);
  PutLengthPrefixedSlice(&rep_, value);
}

void WriteBatch::Delete(const Slice& key) {
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  rep_.push_back(static_cast<char>(kTypeDeletion));
  PutLengthPrefixedSlice(&rep_, key);
}

// util/coding.cc
void PutLengthPrefixedSlice(std::string* dst, const Slice& value) {
  PutVarint32(dst, value.size());
  dst->append(value.data(), value.size());
}

void PutVarint32(std::string* dst, uint32_t v) {
  char buf[5];
  char* ptr = EncodeVarint32(buf, v);
  dst->append(buf, ptr - buf);
}
```

**编码示例：**

```cpp
WriteBatch batch;
batch.Put("foo", "bar");
batch.Delete("old");

// rep_内存（十六进制）：
// [Sequence: 8 bytes]    00 00 00 00 00 00 00 64  (seq=100)
// [Count: 4 bytes]       02 00 00 00              (count=2)
// [Type]                 01                       (kTypeValue)
// [Key Length]           03                       (3 bytes)
// [Key Data]             66 6F 6F                 ("foo")
// [Value Length]         03                       (3 bytes)
// [Value Data]           62 61 72                 ("bar")
// [Type]                 00                       (kTypeDeletion)
// [Key Length]           03                       (3 bytes)
// [Key Data]             6F 6C 64                 ("old")
```

### 2.4 解码与应用

```cpp
// db/write_batch.cc, lines 117-171
Status WriteBatch::Iterate(Handler* handler) const {
  Slice input(rep_);
  if (input.size() < kHeader) {
    return Status::Corruption("malformed WriteBatch (too small)");
  }

  input.remove_prefix(kHeader);  // 跳过Sequence和Count
  Slice key, value;
  int found = 0;
  while (!input.empty()) {
    found++;
    char tag = input[0];
    input.remove_prefix(1);
    switch (tag) {
      case kTypeValue:
        if (GetLengthPrefixedSlice(&input, &key) &&
            GetLengthPrefixedSlice(&input, &value)) {
          handler->Put(key, value);
        } else {
          return Status::Corruption("bad WriteBatch Put");
        }
        break;
      case kTypeDeletion:
        if (GetLengthPrefixedSlice(&input, &key)) {
          handler->Delete(key);
        } else {
          return Status::Corruption("bad WriteBatch Delete");
        }
        break;
      default:
        return Status::Corruption("unknown WriteBatch tag");
    }
  }
  if (found != WriteBatchInternal::Count(this)) {
    return Status::Corruption("WriteBatch has wrong count");
  } else {
    return Status::OK();
  }
}

// 插入到MemTable
namespace {
class MemTableInserter : public WriteBatch::Handler {
 public:
  SequenceNumber sequence_;
  MemTable* mem_;

  void Put(const Slice& key, const Slice& value) override {
    mem_->Add(sequence_, kTypeValue, key, value);
    sequence_++;
  }
  void Delete(const Slice& key) override {
    mem_->Add(sequence_, kTypeDeletion, key, Slice());
    sequence_++;
  }
};
}  // namespace

Status WriteBatchInternal::InsertInto(const WriteBatch* b, MemTable* memtable) {
  MemTableInserter inserter;
  inserter.sequence_ = WriteBatchInternal::Sequence(b);
  inserter.mem_ = memtable;
  return b->Iterate(&inserter);
}
```

## 3. Write操作实现

### 3.1 DBImpl::Write入口

```cpp
// db/db_impl.cc, lines 1155-1254
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
  Writer w(&mutex_);
  w.batch = updates;
  w.sync = options.sync;
  w.done = false;

  MutexLock l(&mutex_);
  writers_.push_back(&w);
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();  // 等待轮到自己
  }
  if (w.done) {
    return w.status;  // 被其他线程代写了
  }

  // 执行写入
  Status status = MakeRoomForWrite(updates == nullptr);
  uint64_t last_sequence = versions_->LastSequence();
  Writer* last_writer = &w;
  if (status.ok() && updates != nullptr) {
    // Group Commit: 合并多个WriteBatch
    WriteBatch* write_batch = BuildBatchGroup(&last_writer);
    WriteBatchInternal::SetSequence(write_batch, last_sequence + 1);
    last_sequence += WriteBatchInternal::Count(write_batch);

    // 释放锁，执行I/O
    {
      mutex_.Unlock();
      status = log_->AddRecord(WriteBatchInternal::Contents(write_batch));
      bool sync_error = false;
      if (status.ok() && options.sync) {
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
        status = WriteBatchInternal::InsertInto(write_batch, mem_);
      }
      mutex_.Lock();
      if (sync_error) {
        RecordBackgroundError(status);
      }
    }
    if (write_batch == tmp_batch_) tmp_batch_->Clear();

    versions_->SetLastSequence(last_sequence);
  }

  // 唤醒已完成的写入者
  while (true) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }

  // 唤醒下一批写入者
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
}
```

**流程图：**

```
Write(batch)
  ↓
加入writers_队列
  ↓
等待轮到自己 (while not front)
  ↓
MakeRoomForWrite()  ← 检查是否需要切换MemTable
  ↓
BuildBatchGroup()   ← Group Commit合并
  ↓
释放锁
  ↓
log_->AddRecord()   ← 写WAL
  ↓
logfile_->Sync()    ← fsync (if sync=true)
  ↓
InsertInto(mem_)    ← 写MemTable
  ↓
获取锁
  ↓
唤醒已完成的线程
  ↓
唤醒下一批等待者
```

### 3.2 Group Commit优化

```cpp
// db/db_impl.cc, lines 1256-1309
WriteBatch* DBImpl::BuildBatchGroup(Writer** last_writer) {
  mutex_.AssertHeld();
  assert(!writers_.empty());
  Writer* first = writers_.front();
  WriteBatch* result = first->batch;
  assert(result != nullptr);

  size_t size = WriteBatchInternal::ByteSize(first->batch);

  // 限制：最多合并1MB
  size_t max_size = 1 << 20;
  if (size <= (128 << 10)) {
    max_size = size + (128 << 10);
  }

  *last_writer = first;
  std::deque<Writer*>::iterator iter = writers_.begin();
  ++iter;  // 跳过first
  for (; iter != writers_.end(); ++iter) {
    Writer* w = *iter;
    if (w->sync && !first->sync) {
      // 不要合并sync和non-sync
      break;
    }

    if (w->batch != nullptr) {
      size += WriteBatchInternal::ByteSize(w->batch);
      if (size > max_size) {
        break;
      }

      // 合并到result
      if (result == first->batch) {
        result = tmp_batch_;
        assert(WriteBatchInternal::Count(result) == 0);
        WriteBatchInternal::Append(result, first->batch);
      }
      WriteBatchInternal::Append(result, w->batch);
    }
    *last_writer = w;
  }
  return result;
}
```

**Group Commit示例：**

```
时间轴：
T1: 线程A调用Write(batch1, sync=false, size=10KB)
T2: 线程B调用Write(batch2, sync=false, size=20KB)
T3: 线程C调用Write(batch3, sync=false, size=30KB)

队列状态：
writers_ = [A, B, C]

线程A执行BuildBatchGroup():
  - 合并A: size=10KB
  - 合并B: size=30KB
  - 合并C: size=60KB
  - 总大小 < max_size(138KB)
  - 返回合并后的批次

结果：
  - 一次fsync写入3个批次
  - 线程B、C被A代写完成
  - 性能提升3倍
```

### 3.3 MakeRoomForWrite流控

```cpp
// db/db_impl.cc, lines 1311-1413
Status DBImpl::MakeRoomForWrite(bool force) {
  mutex_.AssertHeld();
  assert(!writers_.empty());
  bool allow_delay = !force;
  Status s;
  while (true) {
    if (!bg_error_.ok()) {
      s = bg_error_;
      break;
    } else if (allow_delay && versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
      // Level-0文件过多，延迟写入
      mutex_.Unlock();
      env_->SleepForMicroseconds(1000);
      allow_delay = false;
      mutex_.Lock();
    } else if (!force && (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
      // MemTable有空间
      break;
    } else if (imm_ != nullptr) {
      // Immutable正在刷盘，等待完成
      Log(options_.info_log, "Current memtable full; waiting...\n");
      background_work_finished_signal_.Wait();
    } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
      // Level-0文件过多，停止写入
      Log(options_.info_log, "Too many L0 files; waiting...\n");
      background_work_finished_signal_.Wait();
    } else {
      // 切换MemTable
      assert(versions_->PrevLogNumber() == 0);
      uint64_t new_log_number = versions_->NewFileNumber();
      WritableFile* lfile = nullptr;
      s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);
      if (!s.ok()) {
        versions_->ReuseFileNumber(new_log_number);
        break;
      }
      delete log_;
      delete logfile_;
      logfile_ = lfile;
      logfile_number_ = new_log_number;
      log_ = new log::Writer(lfile);
      imm_ = mem_;
      has_imm_.store(true, std::memory_order_release);
      mem_ = new MemTable(internal_comparator_);
      mem_->Ref();
      force = false;
      MaybeScheduleCompaction();
    }
  }
  return s;
}
```

**流控逻辑：**

```
MakeRoomForWrite流程:

1. 检查后台错误
   if bg_error: 返回错误

2. 检查Level-0文件数
   if L0 files >= 8 (SlowdownWrites):
     延迟1ms  ← 轻度限流
   if L0 files >= 12 (StopWrites):
     等待Compaction  ← 完全阻塞

3. 检查MemTable大小
   if size < write_buffer_size:
     继续写入  ← 正常情况
   else:
     需要切换MemTable

4. 检查Immutable状态
   if imm_ != nullptr:
     等待刷盘完成  ← 阻塞写入
   else:
     切换MemTable:
       - 创建新的log文件
       - mem_ → imm_
       - 分配新的mem_
       - 触发后台Compaction
```

**触发条件：**

```
MemTable切换条件：
- MemTable大小 >= write_buffer_size (默认4MB)
- force=true (强制切换)

Level-0限流：
- kL0_SlowdownWritesTrigger = 8 files
  → 每次写入延迟1ms
- kL0_StopWritesTrigger = 12 files
  → 完全停止写入，等待Compaction

为什么限流？
- Level-0文件过多会影响读性能（需要查所有文件）
- 防止写入速度超过Compaction速度
- 反压机制，保护系统稳定性
```

## 4. 性能分析

### 4.1 写入性能对比

```cpp
// 测试：100万次Put操作

// 场景1：单个Put，sync=false
for (int i = 0; i < 1000000; i++) {
  db->Put(WriteOptions(), key, value);
}
// 性能：~100,000 ops/sec
// 瓶颈：锁竞争

// 场景2：单个Put，sync=true
WriteOptions options;
options.sync = true;
for (int i = 0; i < 1000000; i++) {
  db->Put(options, key, value);
}
// 性能：~100 ops/sec
// 瓶颈：fsync（每次10ms）

// 场景3：批量写入，sync=false
WriteBatch batch;
for (int i = 0; i < 1000; i++) {
  batch.Put(key, value);
}
db->Write(WriteOptions(), &batch);
// 性能：~500,000 ops/sec
// 优势：减少锁竞争

// 场景4：批量写入，sync=true
WriteBatch batch;
for (int i = 0; i < 1000; i++) {
  batch.Put(key, value);
}
WriteOptions options;
options.sync = true;
db->Write(options, &batch);
// 性能：~100,000 ops/sec (每批1000个)
// 优势：批量fsync
```

### 4.2 Group Commit收益

```
无Group Commit：
  线程A: Write() → fsync → 10ms
  线程B: Write() → fsync → 10ms
  线程C: Write() → fsync → 10ms
  总耗时：30ms，吞吐量：100 ops/sec

有Group Commit：
  线程A: BuildBatchGroup() → 合并A+B+C → fsync → 10ms
  线程B: 被A代写，立即返回
  线程C: 被A代写，立即返回
  总耗时：10ms，吞吐量：300 ops/sec

收益：
  吞吐量提升：3倍
  延迟降低：线程B、C几乎无延迟
```

### 4.3 内存与磁盘使用

```
写入1GB数据的资源使用：

内存：
  - MemTable：4MB (write_buffer_size)
  - Immutable：4MB (刷盘中)
  - Block Cache：8MB (默认)
  - 总计：~16MB

磁盘：
  - WAL文件：4MB × 2 = 8MB
  - SSTable：1GB (最终)
  - 临时空间：~20MB
  - 总计：~1GB + 28MB

写放大：
  - WAL写入：1GB
  - MemTable刷盘：1GB
  - Compaction：~3GB (取决于层数)
  - 总写入：~5GB
  - 写放大系数：5x
```

## 5. 代码实践

### 5.1 WriteBatch示例

```cpp
#include "leveldb/db.h"
#include <iostream>

using namespace leveldb;

int main() {
  DB* db;
  Options options;
  options.create_if_missing = true;

  Status s = DB::Open(options, "/tmp/testdb", &db);
  if (!s.ok()) {
    std::cerr << "Open failed: " << s.ToString() << std::endl;
    return 1;
  }

  // 原子批量操作
  WriteBatch batch;
  batch.Put("user:100:name", "Alice");
  batch.Put("user:100:age", "25");
  batch.Put("user:100:email", "alice@example.com");
  batch.Delete("user:99");

  s = db->Write(WriteOptions(), &batch);
  if (!s.ok()) {
    std::cerr << "Write failed: " << s.ToString() << std::endl;
    return 1;
  }

  std::cout << "Batch write successful\n";

  // 验证
  std::string value;
  s = db->Get(ReadOptions(), "user:100:name", &value);
  std::cout << "name = " << value << "\n";

  delete db;
  return 0;
}
```

### 5.2 Group Commit测试

```cpp
#include "leveldb/db.h"
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>

using namespace leveldb;

void WriterThread(DB* db, int id, int num_writes) {
  WriteOptions options;
  options.sync = true;

  auto start = std::chrono::high_resolution_clock::now();
  for (int i = 0; i < num_writes; i++) {
    std::string key = "key" + std::to_string(id) + "_" + std::to_string(i);
    std::string value = "value" + std::to_string(i);
    Status s = db->Put(options, key, value);
    if (!s.ok()) {
      std::cerr << "Put failed: " << s.ToString() << std::endl;
    }
  }
  auto end = std::chrono::high_resolution_clock::now();

  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);
  std::cout << "Thread " << id << " wrote " << num_writes << " keys in "
            << duration.count() << " ms\n";
}

int main() {
  DB* db;
  Options options;
  options.create_if_missing = true;

  Status s = DB::Open(options, "/tmp/testdb", &db);
  if (!s.ok()) {
    std::cerr << "Open failed: " << s.ToString() << std::endl;
    return 1;
  }

  // 启动多个写线程，观察Group Commit效果
  const int num_threads = 4;
  const int writes_per_thread = 100;
  std::vector<std::thread> threads;

  auto start = std::chrono::high_resolution_clock::now();
  for (int i = 0; i < num_threads; i++) {
    threads.emplace_back(WriterThread, db, i, writes_per_thread);
  }

  for (auto& t : threads) {
    t.join();
  }
  auto end = std::chrono::high_resolution_clock::now();

  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);
  std::cout << "\nTotal time: " << duration.count() << " ms\n";
  std::cout << "Total ops: " << (num_threads * writes_per_thread) << "\n";
  std::cout << "Ops/sec: "
            << (num_threads * writes_per_thread * 1000 / duration.count())
            << "\n";

  delete db;
  return 0;
}
```

### 5.3 流控测试

```cpp
#include "leveldb/db.h"
#include <iostream>
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

  // 快速写入大量数据，观察流控
  const int num_writes = 100000;
  std::string large_value(1024, 'x');  // 1KB值

  auto start = std::chrono::high_resolution_clock::now();
  for (int i = 0; i < num_writes; i++) {
    std::string key = "key" + std::to_string(i);
    s = db->Put(WriteOptions(), key, large_value);
    if (!s.ok()) {
      std::cerr << "Put failed: " << s.ToString() << std::endl;
      break;
    }

    if (i % 10000 == 0) {
      auto now = std::chrono::high_resolution_clock::now();
      auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
          now - start);
      std::cout << "Wrote " << i << " keys in " << duration.count()
                << " ms\n";
    }
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);
  std::cout << "Total: " << num_writes << " keys in " << duration.count()
            << " ms\n";

  delete db;
  return 0;
}
```

## 总结

今天我们学习了：
1. ✅ **写入流程**：WriteBatch → WAL → MemTable
2. ✅ **WriteBatch编码**：Sequence + Count + Records
3. ✅ **Group Commit**：合并多个批次，减少fsync
4. ✅ **MakeRoomForWrite**：流控机制，防止写入过快

**关键要点：**
- WriteBatch保证原子性和性能
- Group Commit是写入性能的关键优化
- MakeRoomForWrite通过延迟和阻塞实现流控
- sync选项在性能和持久性之间权衡

**思考题：**
1. 为什么Group Commit能提升性能？
2. Level-0限流的阈值为什么是8和12？
3. 如果关闭Group Commit，性能会下降多少？
4. WriteBatch的最大大小应该如何设置？

**明天预告：Day 9 - Compaction机制（上）**
我们将学习LevelDB的核心机制——Compaction，包括Minor和Major Compaction的触发和执行。
