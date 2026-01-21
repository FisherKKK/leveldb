# Day 13: 并发控制与线程安全

## 学习目标
- 理解LevelDB的并发模型
- 掌握读写锁策略
- 学习后台线程管理
- 理解无锁优化技术

## 1. 并发模型

### 1.1 线程模型

```
LevelDB线程类型：

前台线程（用户线程）：
- 多个读线程：并发读取
- 多个写线程：串行化写入（通过mutex_）

后台线程：
- Compaction线程：1个（env_->Schedule）
- 其他后台任务：异步执行
```

### 1.2 锁策略

```cpp
// db/db_impl.h
class DBImpl : public DB {
 private:
  port::Mutex mutex_;  // 保护以下状态
  port::CondVar background_work_finished_signal_;
  
  MemTable* mem_ GUARDED_BY(mutex_);
  MemTable* imm_ GUARDED_BY(mutex_);
  std::atomic<bool> has_imm_;  // 无锁快速检查
  
  WritableFile* logfile_ GUARDED_BY(mutex_);
  uint64_t logfile_number_ GUARDED_BY(mutex_);
  log::Writer* log_ GUARDED_BY(mutex_);
  
  VersionSet* versions_ GUARDED_BY(mutex_);
  
  std::deque<Writer*> writers_ GUARDED_BY(mutex_);
  WriteBatch* tmp_batch_ GUARDED_BY(mutex_);
  
  Status bg_error_ GUARDED_BY(mutex_);
  
  std::atomic<bool> shutting_down_;
  port::CondVar background_work_finished_signal_ GUARDED_BY(mutex_);
};
```

## 2. 读取并发

### 2.1 无锁读取

```cpp
Status DBImpl::Get(const ReadOptions& options, const Slice& key,
                    std::string* value) {
  Status s;
  MutexLock l(&mutex_);
  
  // 1. 获取快照和引用
  SequenceNumber snapshot;
  if (options.snapshot != nullptr) {
    snapshot = static_cast<const SnapshotImpl*>(options.snapshot)->sequence_number();
  } else {
    snapshot = versions_->LastSequence();
  }
  
  MemTable* mem = mem_;
  MemTable* imm = imm_;
  Version* current = versions_->current();
  mem->Ref();
  if (imm != nullptr) imm->Ref();
  current->Ref();
  
  // 2. 释放锁，无锁查找
  {
    mutex_.Unlock();
    
    if (mem->Get(lkey, value, &s)) {
      // Done
    } else if (imm != nullptr && imm->Get(lkey, value, &s)) {
      // Done
    } else {
      s = current->Get(options, lkey, value, &stats);
    }
    
    mutex_.Lock();
  }
  
  // 3. 释放引用
  mem->Unref();
  if (imm != nullptr) imm->Unref();
  current->Unref();
  
  return s;
}
```

**关键点：**
- 持锁获取引用
- 释放锁后查找
- SkipList支持无锁读取
- Version引用计数保护

### 2.2 SkipList无锁读

```cpp
// db/skiplist.h
// Thread safety:
// Writes require external synchronization (mutex)
// Reads require no locking (lock-free)

Node* Next(int n) {
  assert(n >= 0);
  return next_[n].load(std::memory_order_acquire);
}

void SetNext(int n, Node* x) {
  assert(n >= 0);
  next_[n].store(x, std::memory_order_release);
}
```

## 3. 写入并发

### 3.1 Group Commit

```cpp
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
  Writer w(&mutex_);
  w.batch = updates;
  w.sync = options.sync;
  w.done = false;

  MutexLock l(&mutex_);
  writers_.push_back(&w);
  
  // 等待轮到自己
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
  
  if (w.done) {
    return w.status;  // 被代写
  }

  // 合并批次
  WriteBatch* write_batch = BuildBatchGroup(&last_writer);
  
  // 释放锁执行I/O
  {
    mutex_.Unlock();
    status = log_->AddRecord(...);
    if (options.sync) {
      status = logfile_->Sync();
    }
    status = WriteBatchInternal::InsertInto(write_batch, mem_);
    mutex_.Lock();
  }

  // 唤醒被代写的线程
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

  // 唤醒下一批
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
}
```

### 3.2 写入序列化

```
多个写线程 → writers_队列 → 串行执行

优点：
✓ 简化并发控制
✓ Group Commit优化
✓ 保证顺序一致性

缺点：
✗ 写入串行化
✗ 锁竞争

权衡：
- LSM-Tree写入本身很快
- Group Commit缓解瓶颈
- 实际场景够用
```

## 4. 后台线程

### 4.1 Compaction调度

```cpp
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  if (background_compaction_scheduled_) {
    // 已调度
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // 关闭中
  } else if (!bg_error_.ok()) {
    // 有错误
  } else if (imm_ == nullptr && !versions_->NeedsCompaction()) {
    // 不需要
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
  if (!shutting_down_.load(std::memory_order_acquire)) {
    BackgroundCompaction();
  }
  background_compaction_scheduled_ = false;
  background_work_finished_signal_.SignalAll();
  MaybeScheduleCompaction();
}
```

### 4.2 条件变量

```cpp
// 等待Compaction完成
while (imm_ != nullptr) {
  background_work_finished_signal_.Wait();
}

// Compaction完成后唤醒
void DBImpl::BackgroundCall() {
  BackgroundCompaction();
  background_work_finished_signal_.SignalAll();
}
```

## 5. 原子操作

### 5.1 has_imm_优化

```cpp
// 快速检查（无需加锁）
std::atomic<bool> has_imm_;

// Get操作
if (has_imm_.load(std::memory_order_relaxed)) {
  // 可能需要等待Compaction
}

// 设置
has_imm_.store(true, std::memory_order_release);
has_imm_.store(false, std::memory_order_release);
```

### 5.2 shutting_down_

```cpp
std::atomic<bool> shutting_down_;

// 关闭时设置
void DBImpl::~DBImpl() {
  shutting_down_.store(true, std::memory_order_release);
  // 等待后台线程
  while (background_compaction_scheduled_) {
    background_work_finished_signal_.Wait();
  }
}

// 检查
if (shutting_down_.load(std::memory_order_acquire)) {
  return Status::IOError("DB is shutting down");
}
```

## 6. 并发性能

```
读并发：
- 多线程并发读取：线性扩展
- SkipList无锁读：无竞争
- 受限于：Cache大小、磁盘I/O

写并发：
- 串行化写入：单线程
- Group Commit：批量提升
- 吞吐量：~100K ops/sec (sync=false)
           ~100 ops/sec (sync=true)
```

## 7. 内存顺序详解

### 7.1 C++11内存模型

LevelDB使用C++11原子操作和内存顺序保证并发安全：

```cpp
// 六种内存顺序
enum memory_order {
  memory_order_relaxed,   // 最弱：仅保证原子性
  memory_order_consume,   // 数据依赖顺序
  memory_order_acquire,   // 读同步点
  memory_order_release,   // 写同步点
  memory_order_acq_rel,   // 读写同步
  memory_order_seq_cst    // 最强：顺序一致性
};
```

### 7.2 LevelDB中的使用

**1. SkipList的release-acquire**

```cpp
// db/skiplist.h
void SetNext(int n, Node* x) {
  next_[n].store(x, std::memory_order_release);
}

Node* Next(int n) {
  return next_[n].load(std::memory_order_acquire);
}
```

**保证**：
- 写线程的所有写入在`release`之前完成
- 读线程在`acquire`之后看到所有写入
- 形成同步关系（synchronizes-with）

**示例**：
```
写线程：
  T1: node->key = "foo";
  T2: node->value = "bar";
  T3: prev->SetNext(0, node);  // release

读线程：
  T4: node = head->Next(0);    // acquire
  T5: 读取 node->key, node->value  // 保证看到完整数据
```

**2. has_imm_的relaxed**

```cpp
std::atomic<bool> has_imm_;

// 写入（持有mutex_）
imm_ = mem_;
has_imm_.store(true, std::memory_order_relaxed);

// 读取（可能无锁）
if (has_imm_.load(std::memory_order_relaxed)) {
  MutexLock l(&mutex_);
  // 重新检查
  if (imm_ != nullptr) {
    // ...
  }
}
```

**为什么用relaxed？**
- 写操作在mutex_保护下
- 读操作会double-check
- 只需要原子性，不需要同步

**3. shutting_down_的release-acquire**

```cpp
std::atomic<bool> shutting_down_;

// 析构函数
shutting_down_.store(true, std::memory_order_release);

// 后台线程检查
if (shutting_down_.load(std::memory_order_acquire)) {
  return;
}
```

**保证**：
- 析构前的清理操作对后台线程可见
- 后台线程不会访问已销毁的对象

---

## 8. Group Commit完整实现

### 8.1 Writer结构

```cpp
// db/db_impl.cc
struct DBImpl::Writer {
  explicit Writer(port::Mutex* mu)
      : batch(nullptr), sync(false), done(false), cv(mu) {}

  Status status;
  WriteBatch* batch;
  bool sync;
  bool done;
  port::CondVar cv;
};
```

### 8.2 BuildBatchGroup实现

```cpp
// db/db_impl.cc, lines 1118-1170
WriteBatch* DBImpl::BuildBatchGroup(Writer** last_writer) {
  mutex_.AssertHeld();
  assert(!writers_.empty());

  Writer* first = writers_.front();
  WriteBatch* result = first->batch;
  assert(result != nullptr);

  size_t size = WriteBatchInternal::ByteSize(first->batch);

  // 允许合并的最大大小
  size_t max_size = 1 << 20;  // 1MB
  if (size <= (128 << 10)) {
    max_size = size + (128 << 10);  // 加128KB
  }

  *last_writer = first;
  std::deque<Writer*>::iterator iter = writers_.begin();
  ++iter;  // 跳过first

  for (; iter != writers_.end(); ++iter) {
    Writer* w = *iter;

    // 如果需要sync，只合并到下一个sync
    if (first->sync && !w->sync) {
      break;
    }

    // 大小超限，停止合并
    if (size + WriteBatchInternal::ByteSize(w->batch) > max_size) {
      break;
    }

    // 合并批次
    size += WriteBatchInternal::ByteSize(w->batch);
    if (result == first->batch) {
      // 需要拷贝第一个batch
      result = tmp_batch_;
      assert(WriteBatchInternal::Count(result) == 0);
      WriteBatchInternal::Append(result, first->batch);
    }
    WriteBatchInternal::Append(result, w->batch);
    *last_writer = w;
  }
  return result;
}
```

### 8.3 Group Commit流程图

```
多个写线程同时到达：
Thread1: Put("a", "1")
Thread2: Put("b", "2")
Thread3: Put("c", "3")
Thread4: Put("d", "4")

┌─────────────────────────────┐
│ writers_ queue:             │
│ [Thread1] → [Thread2] →     │
│ [Thread3] → [Thread4]       │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ Thread1 作为leader:         │
│ BuildBatchGroup():          │
│ - 合并4个batch              │
│ - 总大小 < 1MB              │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ 写入WAL（合并批次）:         │
│ [a=1, b=2, c=3, d=4]        │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ 写入MemTable（合并批次）     │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ Thread1 唤醒其他线程:       │
│ - Thread2.done = true       │
│ - Thread3.done = true       │
│ - Thread4.done = true       │
└─────────────────────────────┘

结果：
- 4个写操作只写入一次WAL
- 4个线程都返回成功
- 吞吐量提升4倍
```

### 8.4 Group Commit优化效果

**基准测试：**

```cpp
// bench_group_commit.cc
#include "leveldb/db.h"
#include <thread>
#include <vector>

void SingleThreadWrite(leveldb::DB* db, int count) {
  leveldb::WriteOptions options;
  options.sync = false;

  for (int i = 0; i < count; i++) {
    std::string key = "key" + std::to_string(i);
    db->Put(options, key, "value");
  }
}

void MultiThreadWrite(leveldb::DB* db, int thread_count, int per_thread) {
  std::vector<std::thread> threads;
  for (int t = 0; t < thread_count; t++) {
    threads.emplace_back([db, t, per_thread]() {
      leveldb::WriteOptions options;
      options.sync = false;
      for (int i = 0; i < per_thread; i++) {
        std::string key = "key_" + std::to_string(t) + "_" + std::to_string(i);
        db->Put(options, key, "value");
      }
    });
  }
  for (auto& t : threads) {
    t.join();
  }
}

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;
  leveldb::DB::Open(options, "/tmp/bench_gc", &db);

  const int kOps = 100000;

  // 单线程
  auto start = std::chrono::high_resolution_clock::now();
  SingleThreadWrite(db, kOps);
  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
  std::cout << "Single thread: " << kOps / (duration.count() / 1000.0)
            << " ops/sec\n";

  // 4线程
  start = std::chrono::high_resolution_clock::now();
  MultiThreadWrite(db, 4, kOps / 4);
  end = std::chrono::high_resolution_clock::now();
  duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
  std::cout << "4 threads (Group Commit): " << kOps / (duration.count() / 1000.0)
            << " ops/sec\n";

  delete db;
  return 0;
}
```

**预期输出：**
```
Single thread: 80000 ops/sec
4 threads (Group Commit): 200000 ops/sec  (2.5x提升)
```

---

## 9. 条件变量深度分析

### 9.1 等待Compaction完成

```cpp
// db/db_impl.cc
Status DBImpl::MakeRoomForWrite(bool force) {
  mutex_.AssertHeld();
  assert(!writers_.empty());
  bool allow_delay = !force;
  Status s;

  while (true) {
    if (!bg_error_.ok()) {
      s = bg_error_;
      break;
    } else if (allow_delay &&
               versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
      // Level-0文件过多，延迟写入
      mutex_.Unlock();
      env_->SleepForMicroseconds(1000);
      allow_delay = false;
      mutex_.Lock();
    } else if (!force && mem_->ApproximateMemoryUsage() <= options_.write_buffer_size) {
      // 还有空间
      break;
    } else if (imm_ != nullptr) {
      // 等待之前的Immutable刷盘
      Log(options_.info_log, "Current memtable full; waiting...\n");
      background_work_finished_signal_.Wait();
    } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
      // Level-0文件太多，停止写入
      Log(options_.info_log, "Too many L0 files; waiting...\n");
      background_work_finished_signal_.Wait();
    } else {
      // 转换MemTable
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

**流控机制：**

```
Level-0文件数量 → 写入策略

0-3个:   正常写入
4-7个:   延迟1ms（kL0_SlowdownWritesTrigger=8）
8个:     停止写入，等待Compaction（kL0_StopWritesTrigger=12）
```

### 9.2 虚假唤醒处理

条件变量可能虚假唤醒，因此使用while循环：

```cpp
// 错误写法
if (imm_ != nullptr) {
  background_work_finished_signal_.Wait();  // 虚假唤醒后可能imm_还是非空
}

// 正确写法
while (imm_ != nullptr) {
  background_work_finished_signal_.Wait();  // 循环检查条件
}
```

---

## 10. 并发测试

### 10.1 读并发测试

```cpp
// concurrent_read_test.cc
#include "leveldb/db.h"
#include <thread>
#include <vector>
#include <atomic>

void ReaderThread(leveldb::DB* db, int thread_id,
                  std::atomic<int>* success_count) {
  leveldb::ReadOptions options;
  std::string value;

  for (int i = 0; i < 10000; i++) {
    std::string key = "key" + std::to_string(i % 1000);
    leveldb::Status s = db->Get(options, key, &value);
    if (s.ok()) {
      (*success_count)++;
    }
  }
}

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;
  leveldb::DB::Open(options, "/tmp/concurrent_read", &db);

  // 预填充数据
  for (int i = 0; i < 1000; i++) {
    std::string key = "key" + std::to_string(i);
    db->Put(leveldb::WriteOptions(), key, "value");
  }

  // 启动多个读线程
  const int kNumReaders = 8;
  std::vector<std::thread> readers;
  std::atomic<int> success_count(0);

  auto start = std::chrono::high_resolution_clock::now();

  for (int i = 0; i < kNumReaders; i++) {
    readers.emplace_back(ReaderThread, db, i, &success_count);
  }

  for (auto& t : readers) {
    t.join();
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "Total reads: " << success_count.load() << "\n";
  std::cout << "Time: " << duration.count() << " ms\n";
  std::cout << "Throughput: " << (success_count.load() * 1000.0 / duration.count())
            << " ops/sec\n";

  delete db;
  return 0;
}
```

### 10.2 读写并发测试

```cpp
// concurrent_rw_test.cc
#include "leveldb/db.h"
#include <thread>
#include <atomic>

void Writer(leveldb::DB* db, std::atomic<bool>* running) {
  leveldb::WriteOptions options;
  int count = 0;
  while (running->load()) {
    std::string key = "key" + std::to_string(count++ % 1000);
    db->Put(options, key, "value_" + std::to_string(count));
  }
}

void Reader(leveldb::DB* db, std::atomic<bool>* running,
            std::atomic<int>* read_count) {
  leveldb::ReadOptions options;
  std::string value;
  while (running->load()) {
    std::string key = "key" + std::to_string(rand() % 1000);
    db->Get(options, key, &value);
    (*read_count)++;
  }
}

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;
  leveldb::DB::Open(options, "/tmp/concurrent_rw", &db);

  std::atomic<bool> running(true);
  std::atomic<int> read_count(0);

  // 1个写线程 + 8个读线程
  std::thread writer(Writer, db, &running);
  std::vector<std::thread> readers;
  for (int i = 0; i < 8; i++) {
    readers.emplace_back(Reader, db, &running, &read_count);
  }

  // 运行10秒
  std::this_thread::sleep_for(std::chrono::seconds(10));
  running.store(false);

  writer.join();
  for (auto& t : readers) {
    t.join();
  }

  std::cout << "Total reads in 10s: " << read_count.load() << "\n";
  std::cout << "Read throughput: " << (read_count.load() / 10) << " ops/sec\n";

  delete db;
  return 0;
}
```

---

## 11. 死锁预防

### 11.1 锁顺序规则

LevelDB遵循严格的锁获取顺序防止死锁：

```cpp
// 规则：永远按照固定顺序获取锁

// 正确：先mutex_，后其他资源
void DBImpl::SomeOperation() {
  MutexLock l(&mutex_);
  // 访问mem_, imm_, versions_等
}

// 错误：在持有其他锁时获取mutex_
void BadOperation() {
  SomeLock other_lock;
  MutexLock l(&mutex_);  // 可能死锁
}
```

### 11.2 避免嵌套锁

LevelDB通过释放锁来避免嵌套锁：

```cpp
Status DBImpl::Get(...) {
  MutexLock l(&mutex_);

  // 获取引用
  mem->Ref();

  // 释放锁
  {
    mutex_.Unlock();
    // 执行I/O操作，不持有锁
    mem->Get(...);
    mutex_.Lock();
  }

  // 释放引用
  mem->Unref();
}
```

---

## 12. 常见并发问题

### 问题1：ABA问题

**问题**：
```cpp
// 线程1
Node* node = head->next;  // 读取A
// 线程2删除A，再插入A
head->next = node;  // 以为还是原来的A
```

**LevelDB解决**：
- 不删除节点（SkipList）
- 引用计数（Version）
- 内存顺序保证

### 问题2：内存重排序

**问题**：
```cpp
// 编译器可能重排序
node->value = data;  // 可能在SetNext之后执行
prev->SetNext(node);
```

**LevelDB解决**：
```cpp
// 使用memory_order_release
prev->SetNext(node);  // 保证之前的写入完成
```

### 问题3：虚假共享

**问题**：
```cpp
struct Counters {
  int counter1;  // 同一缓存行
  int counter2;  // 同一缓存行
};
// 多线程修改不同计数器，但缓存行伪共享
```

**LevelDB解决**：
- 分片Cache（ShardedLRUCache）
- 减少共享状态
- 使用原子操作

---

## 13. 与其他系统对比

### 13.1 LevelDB vs RocksDB并发模型

| 特性 | LevelDB | RocksDB |
|------|---------|---------|
| 写并发 | 串行化 | 多写线程（Write Ahead Log Pipeline） |
| 读并发 | 无锁读 | 无锁读 |
| Compaction | 单线程 | 多线程并发Compaction |
| 写吞吐 | ~100K ops/s | ~500K ops/s |

**RocksDB的改进**：

```cpp
// RocksDB的并发写入
class DBImpl {
  // 多个WAL写线程
  WriteThread write_thread_;

  // 多个Compaction线程
  std::vector<std::thread> compaction_threads_;
};
```

### 13.2 LevelDB vs MySQL InnoDB

| 特性 | LevelDB | MySQL InnoDB |
|------|---------|--------------|
| 锁粒度 | DB级锁 | 行级锁 |
| 读并发 | 无锁 | MVCC |
| 写并发 | 串行 | 并发写入 |
| 适用场景 | 嵌入式KV | 事务数据库 |

---

## 14. 性能调优

### 14.1 减少锁竞争

```cpp
// 1. 使用原子操作代替锁
std::atomic<bool> has_imm_;  // 代替mutex_

// 2. 缩短临界区
{
  MutexLock l(&mutex_);
  // 只做必要的操作
  mem->Ref();
}
// 在锁外执行耗时操作
DoSlowOperation();

// 3. 读写分离
// 读操作无锁
// 写操作持锁
```

### 14.2 Group Commit优化

```cpp
// 增大批次大小
size_t max_batch_size = 4 << 20;  // 4MB

// 减少sync频率
int sync_interval = 1000;  // 每1000次写入sync一次
```

### 14.3 并发基准测试

```bash
# 多线程写入测试
./db_bench --benchmarks=fillrandom --num=1000000 --threads=4

# 多线程读取测试
./db_bench --benchmarks=readrandom --num=1000000 --threads=8 \
  --use_existing_db=1

# 读写混合测试
./db_bench --benchmarks=readwhilewriting --num=1000000 \
  --threads=8 --writes_per_second=10000
```

---

## 总结

今天我们深入学习了：
1. ✅ **内存顺序**：release-acquire保证并发安全
2. ✅ **Group Commit完整实现**：BuildBatchGroup详解
3. ✅ **条件变量**：流控机制和虚假唤醒
4. ✅ **并发测试**：读并发、读写并发测试
5. ✅ **死锁预防**：锁顺序和避免嵌套锁
6. ✅ **常见问题**：ABA、重排序、虚假共享
7. ✅ **系统对比**：与RocksDB、InnoDB的对比

**关键要点：**
- 读操作通过SkipList无锁和Version引用计数实现并发
- 写操作串行化但Group Commit提供批量优化
- 内存顺序保证正确性，原子操作提升性能
- 条件变量实现流控，防止Level-0文件过多

**思考题答案：**
1. **为什么读可以无锁？**
   - SkipList使用release-acquire保证内存顺序
   - Version引用计数防止数据被删除
   - 节点一旦插入就不删除

2. **写串行化有什么影响？**
   - 限制写吞吐量在单线程级别
   - Group Commit缓解影响
   - 适合读多写少场景

3. **如何实现并发写入？**
   - 参考RocksDB的WAL Pipeline
   - 使用多个MemTable
   - 并发写入不同分区

**明天预告：Day 14 - 性能优化与最佳实践**
我们将总结所有性能优化技巧和生产环境最佳实践。
