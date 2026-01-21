# Day 4: MemTable内存写缓冲

## 学习目标
- 理解MemTable在写入路径中的作用
- 掌握InternalKey的编码格式
- 学习MemTable的查找和插入操作
- 理解Sequence Number的作用

## 1. MemTable概述

### 1.1 MemTable的角色

**MemTable在LevelDB中的位置：**

```
写入流程：
Client → WriteBatch → WAL → MemTable → SSTable
                      ↓       ↓
                    持久化   内存缓冲

读取流程：
Client → MemTable → Immutable MemTable → SSTable
         (最新)      (正在刷盘)          (磁盘)
```

**核心职责：**
1. **写缓冲**：接收所有写入操作
2. **排序**：维护键的有序性
3. **快速查询**：O(log N)查找最新值
4. **批量刷盘**：积累到阈值后写入SSTable

### 1.2 为什么需要MemTable？

**问题1：直接写磁盘太慢**
```
每次写入都刷盘：
- 随机I/O：~100 ops/sec
- 顺序I/O：~10,000 ops/sec

解决方案：
- 写入内存MemTable：~100,000 ops/sec
- 后台异步批量刷盘
```

**问题2：如何保证顺序性？**
```
用户写入顺序：key3, key1, key2
需要在读取时按序：key1, key2, key3

解决方案：
- MemTable使用SkipList维护有序
- 插入时自动排序
```

**问题3：如何保证持久性？**
```
内存数据断电丢失

解决方案：
- 先写WAL日志（Write-Ahead Log）
- 再写MemTable
- 崩溃后从WAL恢复
```

## 2. MemTable结构

### 2.1 类定义

```cpp
// db/memtable.h
class MemTable {
 public:
  explicit MemTable(const InternalKeyComparator& comparator);

  // 引用计数
  void Ref() { ++refs_; }
  void Unref() {
    --refs_;
    assert(refs_ >= 0);
    if (refs_ <= 0) {
      delete this;
    }
  }

  // 内存使用
  size_t ApproximateMemoryUsage();

  // 返回迭代器
  Iterator* NewIterator();

  // 添加条目
  void Add(SequenceNumber seq, ValueType type,
           const Slice& key, const Slice& value);

  // 查找
  bool Get(const LookupKey& key, std::string* value, Status* s);

 private:
  friend class MemTableIterator;
  friend class MemTableBackwardIterator;

  struct KeyComparator {
    const InternalKeyComparator comparator;
    explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) {}
    int operator()(const char* a, const char* b) const;
  };

  typedef SkipList<const char*, KeyComparator> Table;

  ~MemTable();  // Private，通过Unref()销毁

  KeyComparator comparator_;
  int refs_;              // 引用计数
  Arena arena_;           // 内存池
  Table table_;           // SkipList
};
```

**关键设计：**
- 使用SkipList存储：`SkipList<const char*, KeyComparator>`
- 键类型是`const char*`：指向Arena中的编码数据
- Arena分配器：快速分配，批量释放
- 引用计数：管理生命周期

### 2.2 内存布局

```
MemTable对象:
┌─────────────────────────────────┐
│ comparator_  (KeyComparator)    │
├─────────────────────────────────┤
│ refs_        (int)              │
├─────────────────────────────────┤
│ arena_       (Arena)            │  ← 管理所有条目的内存
├─────────────────────────────────┤
│ table_       (SkipList)         │  ← SkipList头节点
└─────────────────────────────────┘
           ↓
Arena内存池:
┌─────────────────────────────────┐
│ [Entry 1] [Entry 2] ... [Entry N] │
│ [SkipList Nodes]                │
└─────────────────────────────────┘
```

## 3. InternalKey编码

### 3.1 为什么需要InternalKey？

**用户视角 vs 内部视角：**

```cpp
// 用户操作
db->Put("user_key", "value1");  // sequence=100
db->Put("user_key", "value2");  // sequence=101
db->Delete("user_key");         // sequence=102

// 内部存储（MemTable中）
InternalKey("user_key", 100, kTypeValue) → "value1"
InternalKey("user_key", 101, kTypeValue) → "value2"
InternalKey("user_key", 102, kTypeDeletion) → ""

// 为什么保留多个版本？
// 1. 支持快照读取（读取sequence=100时的状态）
// 2. 延迟删除（等待Compaction时清理）
```

### 3.2 InternalKey结构

```cpp
// db/dbformat.h, lines 69-98

// InternalKey = UserKey + Tag
// Tag = (SequenceNumber << 8) | ValueType

struct ParsedInternalKey {
  Slice user_key;           // 用户键
  SequenceNumber sequence;  // 序列号（56位）
  ValueType type;           // 类型（8位）
};

enum ValueType {
  kTypeDeletion = 0x0,  // 删除标记
  kTypeValue = 0x1      // 普通值
};

// SequenceNumber: 56位整数
typedef uint64_t SequenceNumber;
static const SequenceNumber kMaxSequenceNumber = ((0x1ull << 56) - 1);
```

**编码格式：**

```
InternalKey内存布局:
┌─────────────────────┬──────────────┐
│   User Key          │    Tag       │
│   (变长)            │   (8 bytes)  │
└─────────────────────┴──────────────┘

Tag (64位):
┌────────────────────────────┬──────────┐
│   SequenceNumber (56位)    │  Type(8位)│
└────────────────────────────┴──────────┘

示例：
User Key: "user_key" (8字节)
Sequence: 100
Type: kTypeValue (1)

Tag = (100 << 8) | 1 = 25601
InternalKey = "user_key" + "\x01\x64\x00\x00\x00\x00\x00\x00"
```

### 3.3 InternalKey比较器

**比较规则（db/dbformat.cc, lines 36-61）：**

```cpp
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // 1. 先比较user_key（升序）
  int r = user_comparator_->Compare(ExtractUserKey(akey),
                                     ExtractUserKey(bkey));
  if (r != 0) return r;

  // 2. user_key相同，比较sequence（降序！）
  const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
  const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
  if (anum > bnum) {
    return -1;  // 注意：sequence大的排在前面
  } else if (anum < bnum) {
    return +1;
  } else {
    return 0;
  }
}
```

**为什么sequence降序？**

```
相同user_key的多个版本：
InternalKey("key", seq=102, kTypeDeletion)    ← 最新，排最前
InternalKey("key", seq=101, kTypeValue)       ← 较新
InternalKey("key", seq=100, kTypeValue)       ← 最旧

查找时：
- Seek("key")会定位到sequence最大的版本
- 先找到最新版本，可以快速判断是否被删除
```

## 4. MemTable条目格式

### 4.1 编码格式

```cpp
// db/memtable.cc, lines 76-99

// MemTable条目格式:
//   klength  : varint32（internal_key的长度）
//   key bytes: char[klength]（internal_key）
//   vlength  : varint32（value的长度）
//   value bytes: char[vlength]

void MemTable::Add(SequenceNumber s, ValueType type,
                   const Slice& key,
                   const Slice& value) {
  size_t key_size = key.size();
  size_t val_size = value.size();
  size_t internal_key_size = key_size + 8;  // +8 for tag

  // 计算编码长度
  const size_t encoded_len = VarintLength(internal_key_size) +
                             internal_key_size +
                             VarintLength(val_size) +
                             val_size;

  // 从Arena分配内存
  char* buf = arena_.Allocate(encoded_len);
  char* p = EncodeVarint32(buf, internal_key_size);

  // 写入user_key
  std::memcpy(p, key.data(), key_size);
  p += key_size;

  // 写入tag (sequence + type)
  EncodeFixed64(p, (s << 8) | type);
  p += 8;

  // 写入value
  p = EncodeVarint32(p, val_size);
  std::memcpy(p, value.data(), val_size);

  // 插入SkipList
  table_.Insert(buf);
}
```

**示例：**

```
Add(seq=100, type=kTypeValue, key="foo", value="bar")

编码结果：
┌─────┬────────────┬──────────┬─────┬────────────┐
│ 11  │  "foo"     │  Tag     │  3  │  "bar"     │
│(vlnt)│  (3 bytes) │(8 bytes) │(vlnt)│ (3 bytes) │
└─────┴────────────┴──────────┴─────┴────────────┘
  ↑                                    ↑
  internal_key_size=11               value_size=3
  (3字节key + 8字节tag)

Tag = (100 << 8) | 1 = 0x6401 (小端序)
```

### 4.2 解码过程

```cpp
// db/memtable.cc, lines 102-136
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  Slice memkey = key.memtable_key();
  Table::Iterator iter(&table_);
  iter.Seek(memkey.data());

  if (iter.Valid()) {
    const char* entry = iter.key();

    // 解码klength
    uint32_t key_length;
    const char* key_ptr = GetVarint32Ptr(entry, entry + 5, &key_length);

    // 比较user_key
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8),
            key.user_key()) == 0) {

      // 提取tag
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      switch (static_cast<ValueType>(tag & 0xff)) {
        case kTypeValue: {
          // 解码value
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

## 5. MemTable操作

### 5.1 插入操作

```cpp
// 通过DBImpl调用
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
  // 1. 写WAL
  status = log_->AddRecord(WriteBatchInternal::Contents(updates));

  // 2. 写MemTable
  if (status.ok()) {
    status = WriteBatchInternal::InsertInto(updates, mem_);
  }

  return status;
}

// WriteBatch迭代插入
class MemTableInserter : public WriteBatch::Handler {
  SequenceNumber sequence_;
  MemTable* mem_;

 public:
  void Put(const Slice& key, const Slice& value) override {
    mem_->Add(sequence_, kTypeValue, key, value);
    sequence_++;
  }

  void Delete(const Slice& key) override {
    mem_->Add(sequence_, kTypeDeletion, key, Slice());
    sequence_++;
  }
};
```

### 5.2 查找操作

**LookupKey结构：**

```cpp
// db/dbformat.h, lines 184-216
class LookupKey {
 public:
  LookupKey(const Slice& user_key, SequenceNumber sequence);

  Slice memtable_key() const { return Slice(start_, end_ - start_); }
  Slice internal_key() const { return Slice(kstart_, end_ - kstart_); }
  Slice user_key() const { return Slice(kstart_, end_ - kstart_ - 8); }

 private:
  const char* start_;   // klength的开始
  const char* kstart_;  // key的开始
  const char* end_;     // 数据的结束
  char space_[200];     // 栈上缓冲区，避免堆分配
};

// 内存布局
// [klength][user_key][tag]
//  ^start  ^kstart   ^end
```

**查找流程：**

```cpp
Status DBImpl::Get(const ReadOptions& options,
                   const Slice& key,
                   std::string* value) {
  SequenceNumber snapshot = versions_->LastSequence();

  // 构造LookupKey
  LookupKey lkey(key, snapshot);

  // 1. 查找MemTable
  if (mem_->Get(lkey, value, &s)) {
    // 找到或确认删除
  }
  // 2. 查找Immutable MemTable
  else if (imm_ != nullptr && imm_->Get(lkey, value, &s)) {
    // 找到
  }
  // 3. 查找SSTable
  else {
    s = current->Get(options, lkey, value, &stats);
  }

  return s;
}
```

### 5.3 迭代器

```cpp
// db/memtable.cc, lines 138-185
class MemTableIterator : public Iterator {
 public:
  explicit MemTableIterator(MemTable::Table* table) : iter_(table) {}

  bool Valid() const override { return iter_.Valid(); }

  void Seek(const Slice& k) override {
    iter_.Seek(EncodeKey(&tmp_, k));
  }

  void SeekToFirst() override { iter_.SeekToFirst(); }
  void SeekToLast() override { iter_.SeekToLast(); }
  void Next() override { iter_.Next(); }
  void Prev() override { iter_.Prev(); }

  Slice key() const override {
    return GetLengthPrefixedSlice(iter_.key());
  }

  Slice value() const override {
    Slice key_slice = GetLengthPrefixedSlice(iter_.key());
    return GetLengthPrefixedSlice(key_slice.data() + key_slice.size());
  }

 private:
  MemTable::Table::Iterator iter_;
  std::string tmp_;  // For passing to EncodeKey
};
```

## 6. MemTable生命周期

### 6.1 创建和转换

```cpp
// db/db_impl.cc

// 初始化
DBImpl::DBImpl(const Options& options, const std::string& dbname)
    : /* ... */ {
  mem_ = new MemTable(internal_comparator_);
  mem_->Ref();
}

// MemTable满时转换为Immutable
Status DBImpl::MakeRoomForWrite(bool force) {
  while (true) {
    if (!force && mem_->ApproximateMemoryUsage() <= options_.write_buffer_size) {
      break;  // 还有空间
    } else if (imm_ != nullptr) {
      // 等待之前的Immutable刷盘完成
      background_work_finished_signal_.Wait();
    } else {
      // 转换：mem_ → imm_，创建新的mem_
      assert(versions_->PrevLogNumber() == 0);
      uint64_t new_log_number = versions_->NewFileNumber();
      WritableFile* lfile = nullptr;
      s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);

      delete log_;
      s = logfile_->Close();

      logfile_ = lfile;
      logfile_number_ = new_log_number;
      log_ = new log::Writer(lfile);

      imm_ = mem_;              // 当前MemTable变为Immutable
      has_imm_.store(true, std::memory_order_release);
      mem_ = new MemTable(internal_comparator_);
      mem_->Ref();

      force = false;
      MaybeScheduleCompaction();  // 触发刷盘
    }
  }
  return s;
}
```

### 6.2 刷盘（Minor Compaction）

```cpp
// db/db_impl.cc, lines 549-594
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != nullptr);

  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();

  // 将Immutable MemTable写入SSTable
  Status s = WriteLevel0Table(imm_, &edit, base);

  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) {
    s = Status::IOError("Deleting DB during memtable compaction");
  }

  if (s.ok()) {
    // 更新MANIFEST
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  if (s.ok()) {
    // 释放Immutable MemTable
    imm_->Unref();
    imm_ = nullptr;
    has_imm_.store(false, std::memory_order_release);
    RemoveObsoleteFiles();  // 删除旧日志
  } else {
    RecordBackgroundError(s);
  }
}
```

## 7. 性能优化

### 7.1 Arena内存分配

**优势：**

```cpp
// 传统方式：每个条目单独分配
for (int i = 0; i < 1000000; i++) {
  char* entry = new char[100];  // 100万次malloc！
  // ...
}

// Arena方式：批量分配
Arena arena;
for (int i = 0; i < 1000000; i++) {
  char* entry = arena.Allocate(100);  // 很少的malloc
  // ...
}

// 性能提升：
// - malloc开销：从O(N)降到O(1)
// - 内存碎片：大块分配，碎片更少
// - 释放开销：批量释放，O(1)
```

### 7.2 栈上缓冲区

```cpp
// db/dbformat.h, lines 184-216
class LookupKey {
 private:
  const char* start_;
  const char* kstart_;
  const char* end_;
  char space_[200];  // 栈上缓冲区

 public:
  LookupKey(const Slice& user_key, SequenceNumber s) {
    size_t usize = user_key.size();
    size_t needed = usize + 13;  // 保守估计
    char* dst;
    if (needed <= sizeof(space_)) {
      dst = space_;  // 使用栈缓冲，无堆分配！
    } else {
      dst = new char[needed];  // 大键才分配堆内存
    }
    // ... 编码
  }
};
```

### 7.3 引用计数

```cpp
// 避免拷贝MemTable
void ProcessMemTable(MemTable* mem) {
  mem->Ref();  // 增加引用

  // 长时间操作...
  Iterator* iter = mem->NewIterator();
  // ...
  delete iter;

  mem->Unref();  // 减少引用，可能销毁
}

// 多线程共享
std::thread t1([mem]() {
  mem->Ref();
  // 读取操作
  mem->Unref();
});

std::thread t2([mem]() {
  mem->Ref();
  // 读取操作
  mem->Unref();
});
```

## 8. 代码实践

### 8.1 测试MemTable

创建 `memtable_test.cc`：

```cpp
#include "db/memtable.h"
#include "db/dbformat.h"
#include "leveldb/comparator.h"
#include <iostream>

using namespace leveldb;

int main() {
  InternalKeyComparator cmp(BytewiseComparator());
  MemTable* mem = new MemTable(cmp);
  mem->Ref();

  // 插入数据
  mem->Add(100, kTypeValue, "key1", "value1");
  mem->Add(101, kTypeValue, "key2", "value2");
  mem->Add(102, kTypeValue, "key1", "value1_new");  // 更新
  mem->Add(103, kTypeDeletion, "key2", "");          // 删除

  // 查询
  std::string value;
  Status s;

  LookupKey lkey1("key1", 105);  // 读取sequence=105时的状态
  if (mem->Get(lkey1, &value, &s)) {
    std::cout << "key1: " << value << std::endl;  // value1_new
  }

  LookupKey lkey2("key2", 105);
  if (mem->Get(lkey2, &value, &s)) {
    std::cout << "key2: found\n";
  } else {
    std::cout << "key2: deleted\n";  // 输出这个
  }

  // 遍历
  Iterator* iter = mem->NewIterator();
  for (iter->SeekToFirst(); iter->Valid(); iter->Next()) {
    ParsedInternalKey ikey;
    ParseInternalKey(iter->key(), &ikey);
    std::cout << ikey.user_key.ToString()
              << " (seq=" << ikey.sequence << "): "
              << iter->value().ToString() << std::endl;
  }
  delete iter;

  mem->Unref();
  return 0;
}
```

### 8.2 内存使用统计

```cpp
// db/memtable.cc, lines 49-51
size_t MemTable::ApproximateMemoryUsage() {
  return arena_.MemoryUsage();
}

// 使用
MemTable* mem = new MemTable(cmp);
mem->Ref();

for (int i = 0; i < 100000; i++) {
  mem->Add(i, kTypeValue, "key" + std::to_string(i), "value");

  if (i % 10000 == 0) {
    std::cout << "Entries: " << i
              << ", Memory: " << mem->ApproximateMemoryUsage() / 1024
              << " KB\n";
  }
}

mem->Unref();
```

## 总结

今天我们学习了：
1. ✅ **MemTable**：写缓冲，使用SkipList维护有序
2. ✅ **InternalKey**：UserKey + Sequence + Type，支持多版本
3. ✅ **编码格式**：Varint长度前缀，紧凑存储
4. ✅ **生命周期**：Active → Immutable → SSTable

**关键要点：**
- MemTable是写入路径的核心，接收所有写操作
- InternalKey的sequence降序排列，最新版本在前
- Arena分配器大幅减少内存分配开销
- 引用计数管理生命周期，支持并发读取

**思考题：**
1. 为什么删除操作不立即删除数据，而是写入kTypeDeletion标记？
2. 如果两个线程同时读取同一个MemTable，会有并发问题吗？
3. MemTable的大小限制（4MB）是如何选择的？

**明天预告：Day 5 - WriteBatch批量写入**
我们将学习LevelDB如何通过批量写入提升吞吐量，以及Group Commit的实现原理。
