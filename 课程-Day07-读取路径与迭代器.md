# Day 7: 读取路径与迭代器

## 学习目标
- 理解Get操作的完整流程
- 掌握迭代器的层次结构
- 学习MergingIterator的实现
- 理解Snapshot快照机制

## 1. 读取路径概述

### 1.1 Get操作流程

**完整的查找路径：**

```
Client: db->Get(key)
  ↓
┌─────────────────────────────────────┐
│ 1. 检查MemTable                     │  ← 最新数据
│    if found: return                 │
└──────────┬──────────────────────────┘
           ↓ 未找到
┌─────────────────────────────────────┐
│ 2. 检查Immutable MemTable           │  ← 正在刷盘
│    if found: return                 │
└──────────┬──────────────────────────┘
           ↓ 未找到
┌─────────────────────────────────────┐
│ 3. 查找Level-0 SSTables             │  ← 可能有重叠
│    遍历所有L0文件（最多4个）         │
│    if found: return                 │
└──────────┬──────────────────────────┘
           ↓ 未找到
┌─────────────────────────────────────┐
│ 4. 查找Level-1+ SSTables            │  ← 二分查找
│    每层最多查一个文件                │
│    if found: return                 │
└──────────┬──────────────────────────┘
           ↓ 未找到
┌─────────────────────────────────────┐
│ 5. 返回 NotFound                    │
└─────────────────────────────────────┘
```

### 1.2 为什么按这个顺序？

**数据新鲜度原则：**

```
时间轴（从新到旧）：
T4: MemTable          ← 最新写入
T3: Immutable         ← 几秒前
T2: Level-0 SSTable   ← 几分钟前
T1: Level-1+ SSTable  ← 更早

查找原则：
- 同一个键可能有多个版本
- 总是返回最新的版本
- 因此必须从新到旧查找
```

**示例：**

```cpp
// 时间线
T1: db->Put("user:100", "Alice");     // 写入Level-1
T2: db->Put("user:100", "Bob");       // 写入Level-0
T3: db->Put("user:100", "Charlie");   // 写入MemTable

// 查询
db->Get("user:100") → "Charlie"  ✓ 正确（最新）

// 如果反向查找
查Level-1 → "Alice"  ✗ 错误（过期数据）
```

### 1.3 性能优化手段

```
优化1：Bloom Filter
- Level-1+文件：先查Bloom Filter
- 快速判断键不存在，避免读取文件

优化2：Table Cache
- 缓存打开的SSTable文件句柄
- 避免重复打开文件

优化3：Block Cache
- 缓存最近读取的数据块
- 减少磁盘I/O

优化4：Level-0特殊处理
- 按文件的newest到oldest顺序查找
- 找到后立即返回
```

## 2. DBImpl::Get实现

### 2.1 入口函数

```cpp
// db/db_impl.cc, lines 1088-1153
Status DBImpl::Get(const ReadOptions& options, const Slice& key,
                    std::string* value) {
  Status s;
  MutexLock l(&mutex_);
  SequenceNumber snapshot;
  if (options.snapshot != nullptr) {
    snapshot =
        static_cast<const SnapshotImpl*>(options.snapshot)->sequence_number();
  } else {
    snapshot = versions_->LastSequence();
  }

  MemTable* mem = mem_;
  MemTable* imm = imm_;
  Version* current = versions_->current();
  mem->Ref();
  if (imm != nullptr) imm->Ref();
  current->Ref();

  bool have_stat_update = false;
  Version::GetStats stats;

  // 释放锁后查找
  {
    mutex_.Unlock();
    // 1. 查找MemTable
    LookupKey lkey(key, snapshot);
    if (mem->Get(lkey, value, &s)) {
      // 找到
    } else if (imm != nullptr && imm->Get(lkey, value, &s)) {
      // 在Immutable中找到
    } else {
      // 在SSTables中查找
      s = current->Get(options, lkey, value, &stats);
      have_stat_update = true;
    }
    mutex_.Lock();
  }

  if (have_stat_update && current->UpdateStats(stats)) {
    MaybeScheduleCompaction();
  }
  mem->Unref();
  if (imm != nullptr) imm->Unref();
  current->Unref();
  return s;
}
```

**关键点：**

```cpp
// 1. 获取快照序列号
SequenceNumber snapshot;
if (options.snapshot != nullptr) {
  snapshot = options.snapshot->sequence_number();
} else {
  snapshot = versions_->LastSequence();  // 当前最新
}

// 2. 增加引用计数（防止被删除）
mem->Ref();
if (imm != nullptr) imm->Ref();
current->Ref();

// 3. 释放锁后查找（允许并发）
mutex_.Unlock();
// ... 执行查找 ...
mutex_.Lock();

// 4. 释放引用
mem->Unref();
if (imm != nullptr) imm->Unref();
current->Unref();
```

### 2.2 MemTable查找

```cpp
// db/memtable.cc, lines 89-125
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  Slice memkey = key.memtable_key();
  Table::Iterator iter(&table_);
  iter.Seek(memkey.data());
  if (iter.Valid()) {
    // entry format: klength(varint32) + userkey + tag(8) + vlength(varint32) + value
    const char* entry = iter.key();
    uint32_t key_length;
    const char* key_ptr = GetVarint32Ptr(entry, entry + 5, &key_length);
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8), key.user_key()) == 0) {
      // 找到匹配的用户键
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      switch (static_cast<ValueType>(tag & 0xff)) {
        case kTypeValue: {
          Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
          value->assign(v.data(), v.size());
          return true;
        }
        case kTypeDeletion:
          *s = Status::NotFound(Slice());
          return true;
      }
    }
  }
  return false;
}
```

**查找逻辑：**

```
1. 构造LookupKey
   user_key + snapshot序列号

2. 在SkipList中Seek
   找到 >= LookupKey 的第一个条目

3. 检查user_key是否匹配
   if 匹配:
     检查类型：
       kTypeValue → 返回值
       kTypeDeletion → 返回NotFound
   else:
     未找到

4. 返回结果
```

### 2.3 Version::Get实现

```cpp
// db/version_set.cc, lines 329-440
Status Version::Get(const ReadOptions& options, const LookupKey& k,
                     std::string* value, GetStats* stats) {
  Slice ikey = k.internal_key();
  Slice user_key = k.user_key();
  const Comparator* ucmp = vset_->icmp_.user_comparator();
  Status s;

  stats->seek_file = nullptr;
  stats->seek_file_level = -1;
  FileMetaData* last_file_read = nullptr;
  int last_file_read_level = -1;

  // 逐层查找
  std::vector<FileMetaData*> tmp;
  FileMetaData* tmp2;
  for (int level = 0; level < config::kNumLevels; level++) {
    size_t num_files = files_[level].size();
    if (num_files == 0) continue;

    // 获取候选文件
    FileMetaData* const* files = &files_[level][0];
    if (level == 0) {
      // Level-0: 可能有重叠，需要检查所有文件
      tmp.reserve(num_files);
      for (uint32_t i = 0; i < num_files; i++) {
        FileMetaData* f = files[i];
        if (ucmp->Compare(user_key, f->smallest.user_key()) >= 0 &&
            ucmp->Compare(user_key, f->largest.user_key()) <= 0) {
          tmp.push_back(f);
        }
      }
      if (tmp.empty()) continue;

      // 按newest到oldest排序
      std::sort(tmp.begin(), tmp.end(), NewestFirst);
      files = &tmp[0];
      num_files = tmp.size();
    } else {
      // Level-1+: 二分查找
      uint32_t index = FindFile(vset_->icmp_, files_[level], ikey);
      if (index >= num_files) {
        files = nullptr;
        num_files = 0;
      } else {
        tmp2 = files[index];
        if (ucmp->Compare(user_key, tmp2->smallest.user_key()) < 0) {
          files = nullptr;
          num_files = 0;
        } else {
          files = &tmp2;
          num_files = 1;
        }
      }
    }

    // 在候选文件中查找
    for (uint32_t i = 0; i < num_files; ++i) {
      if (last_file_read != nullptr && stats->seek_file == nullptr) {
        stats->seek_file = last_file_read;
        stats->seek_file_level = last_file_read_level;
      }

      FileMetaData* f = files[i];
      last_file_read = f;
      last_file_read_level = level;

      Saver saver;
      saver.state = kNotFound;
      saver.ucmp = ucmp;
      saver.user_key = user_key;
      saver.value = value;
      s = vset_->table_cache_->Get(options, f->number, f->file_size, ikey,
                                    &saver, SaveValue);
      if (!s.ok()) {
        return s;
      }
      switch (saver.state) {
        case kNotFound:
          break;
        case kFound:
          return s;
        case kDeleted:
          s = Status::NotFound(Slice());
          return s;
        case kCorrupt:
          s = Status::Corruption("corrupted key for ", user_key);
          return s;
      }
    }
  }

  return Status::NotFound(Slice());
}
```

**查找流程可视化：**

```
Version::Get("user_key")
  ↓
Level 0: [file4, file3, file2, file1]
  ├─ 检查键范围：file4, file2匹配
  ├─ 按newest排序：[file4, file2]
  ├─ 查file4 → NotFound
  ├─ 查file2 → NotFound
  └─ 继续下一层

Level 1: [file5, file6, file7, file8]
  ├─ 二分查找：定位到file6
  ├─ 检查键范围：匹配
  ├─ 查file6 → Found: "value"
  └─ 返回结果 ✓
```

## 3. 迭代器层次结构

### 3.1 Iterator接口

```cpp
// include/leveldb/iterator.h, lines 20-60
class LEVELDB_EXPORT Iterator {
 public:
  Iterator();
  virtual ~Iterator();

  // 迭代器位置是否有效
  virtual bool Valid() const = 0;

  // 定位到第一个键
  virtual void SeekToFirst() = 0;

  // 定位到最后一个键
  virtual void SeekToLast() = 0;

  // 定位到 >= target 的第一个键
  virtual void Seek(const Slice& target) = 0;

  // 前进到下一个键
  virtual void Next() = 0;

  // 后退到上一个键
  virtual void Prev() = 0;

  // 返回当前键
  virtual Slice key() const = 0;

  // 返回当前值
  virtual Slice value() const = 0;

  // 返回错误状态
  virtual Status status() const = 0;

  // 注册清理函数（用于资源释放）
  typedef void (*CleanupFunction)(void* arg1, void* arg2);
  void RegisterCleanup(CleanupFunction function, void* arg1, void* arg2);

 private:
  struct CleanupNode {
    CleanupFunction function;
    void* arg1;
    void* arg2;
    CleanupNode* next;
  };
  CleanupNode cleanup_head_;
};
```

### 3.2 迭代器类型层次

```
Iterator接口
  ├─ MemTableIterator        ← SkipList迭代器
  ├─ Block::Iter             ← 单个数据块迭代器
  ├─ TwoLevelIterator        ← 两层迭代器（索引+数据）
  │   └─ Table::Iterator     ← SSTable迭代器
  ├─ MergingIterator         ← 多路归并迭代器
  └─ DBIter                  ← 用户层迭代器（过滤删除标记）
```

**使用示例：**

```cpp
// 用户代码
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  std::cout << it->key().ToString() << ": "
            << it->value().ToString() << std::endl;
}
delete it;

// 内部实现链
DBIter
  → MergingIterator
      ├─ MemTableIterator (mem_)
      ├─ MemTableIterator (imm_)
      └─ Version::NewConcatenatingIterator
          ├─ Level-0: MergingIterator
          │   ├─ Table::Iterator (file1)
          │   ├─ Table::Iterator (file2)
          │   └─ Table::Iterator (file3)
          └─ Level-1+: ConcatenatingIterator
              ├─ Table::Iterator (file4)
              ├─ Table::Iterator (file5)
              └─ ...
```

## 4. MergingIterator实现

### 4.1 核心思想

**多路归并：**

```
输入：多个有序序列
MemTable:     [2, 5, 8]
Immutable:    [1, 6, 9]
Level-0 f1:   [3, 7]
Level-0 f2:   [4]

输出：合并的有序序列
Result:       [1, 2, 3, 4, 5, 6, 7, 8, 9]

实现：最小堆（priority queue）
```

### 4.2 类定义

```cpp
// table/merger.cc, lines 24-177
class MergingIterator : public Iterator {
 public:
  MergingIterator(const Comparator* comparator, Iterator** children, int n)
      : comparator_(comparator),
        children_(new IteratorWrapper[n]),
        n_(n),
        current_(nullptr),
        direction_(kForward) {
    for (int i = 0; i < n; i++) {
      children_[i].Set(children[i]);
    }
  }

  ~MergingIterator() override { delete[] children_; }

  bool Valid() const override { return (current_ != nullptr); }

  void SeekToFirst() override {
    for (int i = 0; i < n_; i++) {
      children_[i].SeekToFirst();
    }
    FindSmallest();
    direction_ = kForward;
  }

  void SeekToLast() override {
    for (int i = 0; i < n_; i++) {
      children_[i].SeekToLast();
    }
    FindLargest();
    direction_ = kReverse;
  }

  void Seek(const Slice& target) override {
    for (int i = 0; i < n_; i++) {
      children_[i].Seek(target);
    }
    FindSmallest();
    direction_ = kForward;
  }

  void Next() override {
    assert(Valid());
    if (direction_ != kForward) {
      // 切换方向
      for (int i = 0; i < n_; i++) {
        IteratorWrapper* child = &children_[i];
        if (child != current_) {
          child->Seek(key());
          if (child->Valid() &&
              comparator_->Compare(key(), child->key()) == 0) {
            child->Next();
          }
        }
      }
      direction_ = kForward;
    }

    current_->Next();
    FindSmallest();
  }

  void Prev() override {
    assert(Valid());
    if (direction_ != kReverse) {
      // 切换方向
      for (int i = 0; i < n_; i++) {
        IteratorWrapper* child = &children_[i];
        if (child != current_) {
          child->Seek(key());
          if (child->Valid()) {
            child->Prev();
          } else {
            child->SeekToLast();
          }
        }
      }
      direction_ = kReverse;
    }

    current_->Prev();
    FindLargest();
  }

  Slice key() const override {
    assert(Valid());
    return current_->key();
  }

  Slice value() const override {
    assert(Valid());
    return current_->value();
  }

 private:
  void FindSmallest() {
    IteratorWrapper* smallest = nullptr;
    for (int i = 0; i < n_; i++) {
      IteratorWrapper* child = &children_[i];
      if (child->Valid()) {
        if (smallest == nullptr) {
          smallest = child;
        } else if (comparator_->Compare(child->key(), smallest->key()) < 0) {
          smallest = child;
        }
      }
    }
    current_ = smallest;
  }

  void FindLargest() {
    IteratorWrapper* largest = nullptr;
    for (int i = 0; i < n_; i++) {
      IteratorWrapper* child = &children_[i];
      if (child->Valid()) {
        if (largest == nullptr) {
          largest = child;
        } else if (comparator_->Compare(child->key(), largest->key()) > 0) {
          largest = child;
        }
      }
    }
    current_ = largest;
  }

  const Comparator* comparator_;
  IteratorWrapper* children_;  // 子迭代器数组
  int n_;                      // 子迭代器数量
  IteratorWrapper* current_;   // 当前最小的迭代器
  Direction direction_;        // 当前方向
};
```

### 4.3 FindSmallest实现

```cpp
void FindSmallest() {
  IteratorWrapper* smallest = nullptr;
  for (int i = 0; i < n_; i++) {
    IteratorWrapper* child = &children_[i];
    if (child->Valid()) {
      if (smallest == nullptr) {
        smallest = child;
      } else if (comparator_->Compare(child->key(), smallest->key()) < 0) {
        smallest = child;
      }
    }
  }
  current_ = smallest;
}
```

**示例：**

```
子迭代器状态：
children_[0]:  key=5, valid=true
children_[1]:  key=2, valid=true
children_[2]:  key=8, valid=true
children_[3]:  valid=false

FindSmallest():
  i=0: smallest = children_[0] (key=5)
  i=1: 2 < 5, smallest = children_[1] (key=2)
  i=2: 8 > 2, 不更新
  i=3: invalid, 跳过

结果: current_ = children_[1] (key=2)
```

### 4.4 Next操作

```cpp
void Next() override {
  assert(Valid());
  if (direction_ != kForward) {
    // 方向切换逻辑（复杂）
    for (int i = 0; i < n_; i++) {
      IteratorWrapper* child = &children_[i];
      if (child != current_) {
        child->Seek(key());
        if (child->Valid() &&
            comparator_->Compare(key(), child->key()) == 0) {
          child->Next();
        }
      }
    }
    direction_ = kForward;
  }

  current_->Next();
  FindSmallest();
}
```

**执行流程：**

```
初始状态：
children_[0]: [2, 5, 8]  ← current (key=2)
children_[1]: [3, 6, 9]
children_[2]: [4, 7]

调用 Next():
  1. current_->Next()
     children_[0]: [2, →5, 8]

  2. FindSmallest()
     比较: 5 vs 3 vs 4
     结果: children_[1] (key=3)

新状态：
children_[0]: [2, 5, 8]
children_[1]: [3, 6, 9]  ← current (key=3)
children_[2]: [4, 7]
```

## 5. Snapshot机制

### 5.1 Snapshot原理

```cpp
// db/snapshot.h, lines 17-46
class SnapshotList {
 public:
  SnapshotList() : list_(this) {}

  bool empty() const { return list_.next_ == &list_; }
  SnapshotImpl* oldest() const {
    assert(!empty());
    return list_.next_;
  }
  SnapshotImpl* newest() const {
    assert(!empty());
    return list_.prev_;
  }

  const SnapshotImpl* New(SequenceNumber sequence_number) {
    assert(empty() || newest()->sequence_number_ <= sequence_number);

    SnapshotImpl* snapshot = new SnapshotImpl(sequence_number);
    snapshot->list_ = this;
    snapshot->next_ = &list_;
    snapshot->prev_ = list_.prev_;
    snapshot->prev_->next_ = snapshot;
    snapshot->next_->prev_ = snapshot;
    return snapshot;
  }

  void Delete(const SnapshotImpl* snapshot) {
    assert(snapshot->list_ == this);
    snapshot->prev_->next_ = snapshot->next_;
    snapshot->next_->prev_ = snapshot->prev_;
    delete snapshot;
  }

 private:
  SnapshotImpl list_;  // 双向链表头
};

class SnapshotImpl : public Snapshot {
 public:
  SequenceNumber sequence_number() const { return sequence_number_; }

 private:
  friend class SnapshotList;
  SequenceNumber sequence_number_;
  SnapshotImpl* prev_;
  SnapshotImpl* next_;
  SnapshotList* list_;
};
```

### 5.2 Snapshot使用

```cpp
// 创建快照
const Snapshot* snapshot = db->GetSnapshot();

// 使用快照读取
ReadOptions options;
options.snapshot = snapshot;
std::string value;
db->Get(options, "key", &value);  // 读取快照时刻的数据

// 释放快照
db->ReleaseSnapshot(snapshot);
```

**工作原理：**

```
时间线：
T1: db->Put("user:100", "v1")  → seq=10
T2: snapshot = GetSnapshot()   → seq=10
T3: db->Put("user:100", "v2")  → seq=11
T4: db->Get(snapshot, "user:100") → 返回 "v1"

数据存储：
MemTable:
  InternalKey("user:100", 11, kValue) → "v2"
  InternalKey("user:100", 10, kValue) → "v1"

查找逻辑：
  LookupKey("user:100", snapshot_seq=10)
  找到 seq=11 → 跳过（大于快照序列号）
  找到 seq=10 → 返回 "v1" ✓
```

### 5.3 Snapshot影响Compaction

```cpp
// db/version_set.cc, lines 982-1020
bool Version::RecordReadSample(Slice internal_key) {
  ParsedInternalKey ikey;
  if (!ParseInternalKey(internal_key, &ikey)) {
    return false;
  }

  // 如果有快照，不能删除旧版本
  if (vset_->NumLevelFiles(0) > config::kL0_CompactionTrigger) {
    // 考虑触发Compaction
  }

  return true;
}

// db/db_impl.cc
// Compaction时检查快照
bool DBImpl::IsBaseLevelForKey(const Slice& user_key) {
  const Comparator* user_cmp = user_comparator();
  for (int level = 1; level < config::kNumLevels; level++) {
    const std::vector<FileMetaData*>& files = current_->files_[level];
    while (level_ptrs_[level] < files.size()) {
      FileMetaData* f = files[level_ptrs_[level]];
      if (user_cmp->Compare(user_key, f->largest.user_key()) <= 0) {
        // 发现相同的键
        ParsedInternalKey ikey;
        if (ParseInternalKey(f->largest.Encode(), &ikey) &&
            ikey.sequence != kMaxSequenceNumber) {
          // 可能被快照引用，不能删除
          return false;
        }
        break;
      }
      level_ptrs_[level]++;
    }
  }
  return true;
}
```

**快照与Compaction的关系：**

```
场景：存在快照时的Compaction

数据：
  InternalKey("key", 100, kValue) → "v1"
  InternalKey("key", 101, kValue) → "v2"

快照：snapshot_seq = 100

Compaction决策：
  if snapshot_seq >= 100:
    保留 seq=100 的版本（快照可能使用）
  else:
    删除 seq=100 的版本（没有快照引用）
```

## 6. 性能分析

### 6.1 Get操作开销

```
Get操作成本分解：

1. MemTable查找：
   - SkipList查找：O(log N)
   - 内存操作：~100ns

2. Immutable查找：
   - SkipList查找：O(log N)
   - 内存操作：~100ns

3. Level-0查找：
   - Bloom Filter：~100ns × 4 = 400ns
   - Block Cache命中：~500ns
   - Block Cache未命中：~10ms (读磁盘)

4. Level-1+查找：
   - 二分定位文件：O(log M)
   - Bloom Filter：~100ns
   - Block Cache命中：~500ns
   - Block Cache未命中：~10ms

总计（最坏情况）：
- 命中MemTable：~100ns
- 命中Level-0：~1μs
- 未命中需读盘：~10ms
```

### 6.2 Iterator性能

```cpp
// 测试：遍历100万个键

// 场景1：MemTable遍历
Iterator* it = mem->NewIterator();
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // ...
}
// 性能：~50ns/key (纯内存)

// 场景2：SSTable遍历
Iterator* it = table->NewIterator();
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // ...
}
// 性能：~200ns/key (Block Cache命中)
//      ~10μs/key (需要读盘)

// 场景3：完整DB遍历
Iterator* it = db->NewIterator();
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // 合并多个迭代器
}
// 性能：~500ns/key (多层合并开销)
```

## 7. 代码实践

### 7.1 Get操作示例

```cpp
#include "leveldb/db.h"
#include <iostream>
#include <chrono>

using namespace leveldb;

void BenchmarkGet(DB* db, int num_reads) {
  ReadOptions options;
  std::string value;

  auto start = std::chrono::high_resolution_clock::now();
  for (int i = 0; i < num_reads; i++) {
    std::string key = "key" + std::to_string(i % 10000);
    Status s = db->Get(options, key, &value);
  }
  auto end = std::chrono::high_resolution_clock::now();

  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);
  std::cout << "Get " << num_reads << " keys: " << duration.count()
            << " ms\n";
  std::cout << "Ops/sec: " << (num_reads * 1000 / duration.count()) << "\n";
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

  // 写入测试数据
  for (int i = 0; i < 10000; i++) {
    std::string key = "key" + std::to_string(i);
    std::string value = "value" + std::to_string(i);
    db->Put(WriteOptions(), key, value);
  }

  BenchmarkGet(db, 100000);

  delete db;
  return 0;
}
```

### 7.2 迭代器示例

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

  // 写入数据
  db->Put(WriteOptions(), "apple", "red");
  db->Put(WriteOptions(), "banana", "yellow");
  db->Put(WriteOptions(), "cherry", "red");

  // 正向遍历
  std::cout << "Forward iteration:\n";
  Iterator* it = db->NewIterator(ReadOptions());
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    std::cout << it->key().ToString() << ": "
              << it->value().ToString() << "\n";
  }
  delete it;

  // 反向遍历
  std::cout << "\nReverse iteration:\n";
  it = db->NewIterator(ReadOptions());
  for (it->SeekToLast(); it->Valid(); it->Prev()) {
    std::cout << it->key().ToString() << ": "
              << it->value().ToString() << "\n";
  }
  delete it;

  // 范围查询
  std::cout << "\nRange iteration (>= 'b'):\n";
  it = db->NewIterator(ReadOptions());
  for (it->Seek("b"); it->Valid(); it->Next()) {
    std::cout << it->key().ToString() << ": "
              << it->value().ToString() << "\n";
  }
  delete it;

  delete db;
  return 0;
}
```

### 7.3 快照示例

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

  // 写入初始值
  db->Put(WriteOptions(), "user:100", "Alice");
  db->Put(WriteOptions(), "user:101", "Bob");

  // 创建快照
  const Snapshot* snapshot = db->GetSnapshot();
  std::cout << "Created snapshot\n";

  // 修改数据
  db->Put(WriteOptions(), "user:100", "Charlie");
  db->Delete(WriteOptions(), "user:101");

  // 当前读取
  std::string value;
  s = db->Get(ReadOptions(), "user:100", &value);
  std::cout << "Current user:100 = " << value << "\n";  // Charlie

  s = db->Get(ReadOptions(), "user:101", &value);
  std::cout << "Current user:101 = "
            << (s.IsNotFound() ? "NotFound" : value) << "\n";  // NotFound

  // 快照读取
  ReadOptions read_options;
  read_options.snapshot = snapshot;
  s = db->Get(read_options, "user:100", &value);
  std::cout << "Snapshot user:100 = " << value << "\n";  // Alice

  s = db->Get(read_options, "user:101", &value);
  std::cout << "Snapshot user:101 = " << value << "\n";  // Bob

  // 释放快照
  db->ReleaseSnapshot(snapshot);

  delete db;
  return 0;
}
```

## 总结

今天我们学习了：
1. ✅ **Get操作**：MemTable → Immutable → Level-0 → Level-1+
2. ✅ **迭代器层次**：MemTableIterator, TwoLevelIterator, MergingIterator
3. ✅ **MergingIterator**：多路归并，FindSmallest/FindLargest
4. ✅ **Snapshot机制**：时间点快照，影响Compaction

**关键要点：**
- 读取路径按数据新鲜度查找（新到旧）
- 迭代器采用组合模式，层层包装
- MergingIterator实现多路有序归并
- Snapshot通过SequenceNumber隔离数据版本

**思考题：**
1. 为什么Level-0需要检查所有文件，而Level-1+只检查一个？
2. MergingIterator的时间复杂度是多少？
3. 如果有大量快照，对性能有什么影响？
4. 如何优化范围查询的性能？

**明天预告：Day 8 - 写入路径与WriteBatch**
我们将深入学习写入操作的完整流程，包括WriteBatch编码和Group Commit优化。
