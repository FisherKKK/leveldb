# Day 11: Version与VersionSet版本管理

## 学习目标
- 理解Version表示数据库快照的概念
- 掌握VersionEdit记录变更的机制  
- 学习VersionSet管理版本链
- 理解MANIFEST文件格式

## 1. Version概述

### 1.1 为什么需要Version?

**问题：Compaction如何不影响正在进行的读取?**

```
场景：
T1: 读线程开始读取 file001.ldb
T2: Compaction删除 file001.ldb
T3: 读线程访问 file001.ldb → 崩溃！

解决方案：Version
- 每个Version代表一个数据库状态快照
- 读操作持有Version引用
- Compaction创建新Version
- 旧Version在无引用时才删除
```

### 1.2 Version核心概念

```cpp
// db/version_set.h
class Version {
  friend class VersionSet;

 public:
  void Ref();
  void Unref();

  Iterator* NewConcatenatingIterator(const ReadOptions&, int level) const;
  void AddIterators(const ReadOptions&, std::vector<Iterator*>* iters);
  
  Status Get(const ReadOptions&, const LookupKey& key, std::string* val,
             GetStats* stats);

 private:
  VersionSet* vset_;
  Version* next_;  // 双向链表
  Version* prev_;
  int refs_;       // 引用计数

  std::vector<FileMetaData*> files_[config::kNumLevels];
  FileMetaData* file_to_compact_;
  int file_to_compact_level_;
  double compaction_score_;
  int compaction_level_;
};
```

**核心字段：**
- `files_[]`: 每层的SSTable文件列表
- `refs_`: 引用计数（读操作+1，完成-1）
- `next_/prev_`: 版本链表
- `compaction_score_`: Compaction优先级

## 2. VersionEdit

### 2.1 记录变更

```cpp
// db/version_edit.h
class VersionEdit {
 public:
  void SetComparatorName(const Slice& name);
  void SetLogNumber(uint64_t num);
  void SetPrevLogNumber(uint64_t num);
  void SetNextFile(uint64_t num);
  void SetLastSequence(SequenceNumber seq);
  void SetCompactPointer(int level, const InternalKey& key);

  void AddFile(int level, uint64_t file, uint64_t file_size,
               const InternalKey& smallest, const InternalKey& largest);
  void RemoveFile(int level, uint64_t file);

 private:
  typedef std::set<std::pair<int, uint64_t>> DeletedFileSet;

  std::string comparator_;
  uint64_t log_number_;
  uint64_t prev_log_number_;
  uint64_t next_file_number_;
  SequenceNumber last_sequence_;
  bool has_comparator_;
  bool has_log_number_;
  bool has_prev_log_number_;
  bool has_next_file_number_;
  bool has_last_sequence_;

  std::vector<std::pair<int, InternalKey>> compact_pointers_;
  DeletedFileSet deleted_files_;
  std::vector<std::pair<int, FileMetaData>> new_files_;
};
```

**使用场景：**

```cpp
// Minor Compaction
VersionEdit edit;
edit.AddFile(0, file_number, file_size, smallest, largest);
edit.SetLogNumber(new_log_number);
versions_->LogAndApply(&edit, &mutex_);

// Major Compaction  
VersionEdit edit;
// 删除输入文件
for (auto* f : compaction->inputs_[0]) {
  edit.RemoveFile(level, f->number);
}
for (auto* f : compaction->inputs_[1]) {
  edit.RemoveFile(level + 1, f->number);
}
// 添加输出文件
for (auto& output : compact->outputs) {
  edit.AddFile(level + 1, output.number, output.file_size,
               output.smallest, output.largest);
}
versions_->LogAndApply(&edit, &mutex_);
```

### 2.2 VersionEdit编码深度解析

#### 2.2.1 编码格式详解

```cpp
// db/version_edit.cc
void VersionEdit::EncodeTo(std::string* dst) const {
  // ========== 元数据字段 ==========
  if (has_comparator_) {
    PutVarint32(dst, kComparator);  // Tag=1
    PutLengthPrefixedSlice(dst, comparator_);
  }

  if (has_log_number_) {
    PutVarint32(dst, kLogNumber);  // Tag=2
    PutVarint64(dst, log_number_);
  }

  if (has_prev_log_number_) {
    PutVarint32(dst, kPrevLogNumber);  // Tag=9
    PutVarint64(dst, prev_log_number_);
  }

  if (has_next_file_number_) {
    PutVarint32(dst, kNextFileNumber);  // Tag=3
    PutVarint64(dst, next_file_number_);
  }

  if (has_last_sequence_) {
    PutVarint32(dst, kLastSequence);  // Tag=4
    PutVarint64(dst, last_sequence_);
  }

  // ========== Compaction指针 ==========
  for (size_t i = 0; i < compact_pointers_.size(); i++) {
    PutVarint32(dst, kCompactPointer);  // Tag=5
    PutVarint32(dst, compact_pointers_[i].first);  // level
    PutLengthPrefixedSlice(dst, compact_pointers_[i].second.Encode());
  }

  // ========== 删除的文件 ==========
  for (const auto& deleted_file_kvp : deleted_files_) {
    PutVarint32(dst, kDeletedFile);  // Tag=6
    PutVarint32(dst, deleted_file_kvp.first);   // level
    PutVarint64(dst, deleted_file_kvp.second);  // file number
  }

  // ========== 新增的文件 ==========
  for (size_t i = 0; i < new_files_.size(); i++) {
    const FileMetaData& f = new_files_[i].second;
    PutVarint32(dst, kNewFile);  // Tag=7
    PutVarint32(dst, new_files_[i].first);  // level
    PutVarint64(dst, f.number);
    PutVarint64(dst, f.file_size);
    PutLengthPrefixedSlice(dst, f.smallest.Encode());
    PutLengthPrefixedSlice(dst, f.largest.Encode());
  }
}
```

#### 2.2.2 编码示例

```
示例1：Minor Compaction
-----------------------
操作：
- 添加文件123到Level-0
- 更新log_number

编码：
  [Tag=kNewFile][Level=0][FileNum=123][FileSize=2097152]
    [Smallest="user:001@100"][Largest="user:999@100"]
  [Tag=kLogNumber][LogNumber=124]

字节表示（hex）：
  07 00 7B 00 20 00 00 ...  (NewFile, level=0, num=123, size=2MB)
  02 7C                      (LogNumber=124)

---

示例2：Major Compaction (Level-1 → Level-2)
-------------------------------------------
操作：
- 删除Level-1的文件45, 46
- 删除Level-2的文件100, 101, 102
- 添加Level-2的文件200, 201, 202

编码：
  [Tag=kDeletedFile][Level=1][FileNum=45]
  [Tag=kDeletedFile][Level=1][FileNum=46]
  [Tag=kDeletedFile][Level=2][FileNum=100]
  [Tag=kDeletedFile][Level=2][FileNum=101]
  [Tag=kDeletedFile][Level=2][FileNum=102]
  [Tag=kNewFile][Level=2][FileNum=200][FileSize=4194304]
    [Smallest="a@200"][Largest="m@200"]
  [Tag=kNewFile][Level=2][FileNum=201][FileSize=4194304]
    [Smallest="n@200"][Largest="z@200"]

---

示例3：数据库初始化
-------------------
编码（第一个MANIFEST记录）：
  [Tag=kComparator][Name="leveldb.BytewiseComparator"]
  [Tag=kLogNumber][LogNumber=0]
  [Tag=kNextFileNumber][NextFile=2]
  [Tag=kLastSequence][Seq=0]

字节表示：
  01 1C 6C 65 76 65 6C 64 62 2E ...  (Comparator)
  02 00                              (LogNumber=0)
  03 02                              (NextFile=2)
  04 00                              (LastSeq=0)
```

#### 2.2.3 解码过程

```cpp
// db/version_edit.cc, lines 106-207
Status VersionEdit::DecodeFrom(const Slice& src) {
  Clear();
  Slice input = src;
  const char* msg = nullptr;
  uint32_t tag;

  // 临时存储
  int level;
  uint64_t number;
  FileMetaData f;
  Slice str;
  InternalKey key;

  // 循环解析每个Tag
  while (msg == nullptr && GetVarint32(&input, &tag)) {
    switch (tag) {
      case kComparator:
        // 解析比较器名称
        if (GetLengthPrefixedSlice(&input, &str)) {
          comparator_ = str.ToString();
          has_comparator_ = true;
        } else {
          msg = "comparator name";
        }
        break;

      case kLogNumber:
        // 解析log_number
        if (GetVarint64(&input, &log_number_)) {
          has_log_number_ = true;
        } else {
          msg = "log number";
        }
        break;

      case kPrevLogNumber:
        // 解析prev_log_number
        if (GetVarint64(&input, &prev_log_number_)) {
          has_prev_log_number_ = true;
        } else {
          msg = "previous log number";
        }
        break;

      case kNextFileNumber:
        // 解析next_file_number
        if (GetVarint64(&input, &next_file_number_)) {
          has_next_file_number_ = true;
        } else {
          msg = "next file number";
        }
        break;

      case kLastSequence:
        // 解析last_sequence
        if (GetVarint64(&input, &last_sequence_)) {
          has_last_sequence_ = true;
        } else {
          msg = "last sequence number";
        }
        break;

      case kCompactPointer:
        // 解析compaction指针
        if (GetLevel(&input, &level) &&
            GetInternalKey(&input, &key)) {
          compact_pointers_.push_back(std::make_pair(level, key));
        } else {
          msg = "compaction pointer";
        }
        break;

      case kDeletedFile:
        // 解析删除的文件
        if (GetLevel(&input, &level) &&
            GetVarint64(&input, &number)) {
          deleted_files_.insert(std::make_pair(level, number));
        } else {
          msg = "deleted file";
        }
        break;

      case kNewFile:
        // 解析新增的文件
        if (GetLevel(&input, &level) &&
            GetVarint64(&input, &f.number) &&
            GetVarint64(&input, &f.file_size) &&
            GetInternalKey(&input, &f.smallest) &&
            GetInternalKey(&input, &f.largest)) {
          new_files_.push_back(std::make_pair(level, f));
        } else {
          msg = "new-file entry";
        }
        break;

      default:
        // 未知Tag
        msg = "unknown tag";
        break;
    }
  }

  // 检查是否有错误
  if (msg != nullptr) {
    return Status::Corruption(std::string(msg) + " error in VersionEdit");
  } else if (!input.empty()) {
    // 有未解析的数据
    return Status::Corruption("extra data in VersionEdit");
  } else {
    return Status::OK();
  }
}
```

**解析流程图：**

```
输入字节流：
  [07 00 7B 00 20 00 00 ...] [02 7C]

解析过程：
  ┌─────────────────────────────────────┐
  │ GetVarint32(&input, &tag)           │
  │ tag = 7 (kNewFile)                  │
  └────────────────┬────────────────────┘
                   │
  ┌────────────────▼────────────────────┐
  │ case kNewFile:                       │
  │   GetLevel(&input, &level)          │
  │   level = 0                         │
  │   GetVarint64(&input, &number)      │
  │   number = 123                      │
  │   GetVarint64(&input, &file_size)   │
  │   file_size = 2097152               │
  │   GetInternalKey(&input, &smallest) │
  │   GetInternalKey(&input, &largest)  │
  │   new_files_.push_back(...)         │
  └────────────────┬────────────────────┘
                   │
  ┌────────────────▼────────────────────┐
  │ GetVarint32(&input, &tag)           │
  │ tag = 2 (kLogNumber)                │
  └────────────────┬────────────────────┘
                   │
  ┌────────────────▼────────────────────┐
  │ case kLogNumber:                    │
  │   GetVarint64(&input, &log_number)  │
  │   log_number = 124                  │
  │   has_log_number_ = true            │
  └────────────────┬────────────────────┘
                   │
  ┌────────────────▼────────────────────┐
  │ GetVarint32(&input, &tag)           │
  │ 返回false (到达末尾)                │
  │ 循环结束                            │
  └─────────────────────────────────────┘

结果：
  new_files_ = [(0, FileMetaData{123, 2MB, ...})]
  log_number_ = 124
  has_log_number_ = true
```

#### 2.2.4 Varint编码原理

```cpp
// util/coding.h
// Varint32: 变长整数编码
// 小数字用更少的字节

uint32_t v = 300;
编码结果: 0xAC 0x02 (2字节)
编码过程:
  300 = 0b100101100
  分成7位一组: 10 0101100
  第一组: 0101100 (最高位设为1) → 10101100 (0xAC)
  第二组: 10 (最高位设为0) → 00000010 (0x02)

uint32_t v = 127;
编码结果: 0x7F (1字节)
编码过程:
  127 = 0b1111111
  只有7位: 1111111 (最高位设为0) → 0x7F

// 解码
bool GetVarint32(Slice* input, uint32_t* value) {
  uint32_t result = 0;
  for (uint32_t shift = 0; shift <= 28; shift += 7) {
    uint32_t byte;
    if (!input->empty()) {
      byte = *(const unsigned char*)(input->data());
      input->remove_prefix(1);
    } else {
      return false;
    }
    if (byte & 128) {
      // 还有后续字节
      result |= ((byte & 127) << shift);
    } else {
      // 最后一个字节
      result |= (byte << shift);
      *value = result;
      return true;
    }
  }
  return false;
}
```

**为什么使用Varint？**

```
固定长度编码 vs Varint编码：

固定长度（4字节）：
  1      → 0x00 0x00 0x00 0x01 (4字节)
  1000   → 0x00 0x00 0x03 0xE8 (4字节)
  100000 → 0x00 0x01 0x86 0xA0 (4字节)

Varint编码：
  1      → 0x01 (1字节)  ✓ 节省75%
  1000   → 0xE8 0x07 (2字节) ✓ 节省50%
  100000 → 0xA0 0x93 0x06 (3字节) ✓ 节省25%

LevelDB文件编号通常较小：
  1-10000 → 大部分只需1-2字节
  编码效率高！
```

## 3. VersionSet

### 3.1 版本链管理

```cpp
// db/version_set.h
class VersionSet {
 public:
  Status LogAndApply(VersionEdit* edit, port::Mutex* mu);
  Status Recover(bool* save_manifest);
  
  Version* current() const { return current_; }
  uint64_t LastSequence() const { return last_sequence_; }
  uint64_t NewFileNumber() { return next_file_number_++; }

 private:
  Env* const env_;
  const std::string dbname_;
  const Options* const options_;
  TableCache* const table_cache_;
  const InternalKeyComparator icmp_;
  
  uint64_t next_file_number_;
  uint64_t manifest_file_number_;
  uint64_t last_sequence_;
  uint64_t log_number_;
  uint64_t prev_log_number_;

  WritableFile* descriptor_file_;
  log::Writer* descriptor_log_;
  Version dummy_versions_;  // 链表头
  Version* current_;         // 当前版本
};
```

**版本链示例：**

```
dummy_versions_ ←→ v1 ←→ v2 ←→ v3 (current_)
                   ↑
                refs_=0 (可删除)
                   
                   v2: refs_=2 (2个读操作)
                   v3: refs_=1 (current)
```

### 3.2 LogAndApply实现

```cpp
// db/version_set.cc
Status VersionSet::LogAndApply(VersionEdit* edit, port::Mutex* mu) {
  if (edit->has_log_number_) {
    assert(edit->log_number_ >= log_number_);
    assert(edit->log_number_ < next_file_number_);
  } else {
    edit->SetLogNumber(log_number_);
  }

  if (!edit->has_prev_log_number_) {
    edit->SetPrevLogNumber(prev_log_number_);
  }

  edit->SetNextFile(next_file_number_);
  edit->SetLastSequence(last_sequence_);

  Version* v = new Version(this);
  {
    Builder builder(this, current_);
    builder.Apply(edit);
    builder.SaveTo(v);
  }
  Finalize(v);

  std::string new_manifest_file;
  Status s;
  if (descriptor_log_ == nullptr) {
    assert(descriptor_file_ == nullptr);
    new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
    s = env_->NewWritableFile(new_manifest_file, &descriptor_file_);
    if (s.ok()) {
      descriptor_log_ = new log::Writer(descriptor_file_);
      s = WriteSnapshot(descriptor_log_);
    }
  }

  {
    mu->Unlock();
    if (s.ok()) {
      std::string record;
      edit->EncodeTo(&record);
      s = descriptor_log_->AddRecord(record);
      if (s.ok()) {
        s = descriptor_file_->Sync();
      }
      if (!s.ok()) {
        Log(options_->info_log, "MANIFEST write: %s\n", s.ToString().c_str());
      }
    }

    if (s.ok() && !new_manifest_file.empty()) {
      s = SetCurrentFile(env_, dbname_, manifest_file_number_);
    }
    mu->Lock();
  }

  if (s.ok()) {
    AppendVersion(v);
    log_number_ = edit->log_number_;
    prev_log_number_ = edit->prev_log_number_;
  } else {
    delete v;
    if (!new_manifest_file.empty()) {
      delete descriptor_log_;
      delete descriptor_file_;
      descriptor_log_ = nullptr;
      descriptor_file_ = nullptr;
      env_->RemoveFile(new_manifest_file);
    }
  }

  return s;
}
```

**流程：**

```
1. 创建新Version
   Builder::Apply(edit)  // 应用变更
   Builder::SaveTo(v)    // 生成新版本

2. 写入MANIFEST
   edit->EncodeTo(&record)
   descriptor_log_->AddRecord(record)
   descriptor_file_->Sync()

3. 更新CURRENT
   SetCurrentFile(manifest_file_number_)

4. 链入版本链
   AppendVersion(v)
   current_ = v
```

## 4. MANIFEST文件

### 4.1 文件结构

```
MANIFEST-000001:
  [Snapshot]  ← 完整状态
  [Edit 1]    ← 增量变更
  [Edit 2]
  [Edit 3]
  ...

CURRENT:
  MANIFEST-000001  ← 指向当前MANIFEST
```

**Snapshot内容：**

```
- Comparator name
- Log number
- Next file number
- Last sequence
- Compact pointers
- All files (每层的所有SSTable)
```

### 4.2 Recover恢复深度分析

#### 4.2.1 完整Recovery流程

```cpp
// db/version_set.cc, lines 861-992
Status VersionSet::Recover(bool* save_manifest) {
  // ========== 阶段1：读取CURRENT文件 ==========
  std::string current;
  Status s = ReadFileToString(env_, CurrentFileName(dbname_), &current);

  // CURRENT文件内容示例："MANIFEST-000001\n"
  if (!s.ok()) {
    return s;
  }

  // 验证格式
  if (current.empty() || current[current.size() - 1] != '\n') {
    return Status::Corruption("CURRENT file does not end with newline");
  }
  current.resize(current.size() - 1);  // 去掉换行符

  // ========== 阶段2：打开MANIFEST文件 ==========
  std::string dscname = dbname_ + "/" + current;  // 完整路径
  SequentialFile* file;
  s = env_->NewSequentialFile(dscname, &file);
  if (!s.ok()) {
    if (s.IsNotFound()) {
      return Status::Corruption("CURRENT points to a non-existent file",
                                current);
    }
    return s;
  }

  // ========== 阶段3：读取并解析MANIFEST记录 ==========
  bool have_log_number = false;
  bool have_prev_log_number = false;
  bool have_next_file = false;
  bool have_last_sequence = false;
  uint64_t next_file = 0;
  uint64_t last_sequence = 0;
  uint64_t log_number = 0;
  uint64_t prev_log_number = 0;

  Builder builder(this, current_);  // 从当前Version构建
  int read_records = 0;

  {
    // 创建log reader
    LogReporter reporter;
    reporter.status = &s;
    log::Reader reader(file, &reporter, true /*checksum*/, 0 /*initial_offset*/);

    Slice record;
    std::string scratch;

    // 逐条读取记录
    while (reader.ReadRecord(&record, &scratch) && s.ok()) {
      ++read_records;

      // 解析VersionEdit
      VersionEdit edit;
      s = edit.DecodeFrom(record);

      if (s.ok()) {
        // 验证比较器名称
        if (edit.has_comparator_ &&
            edit.comparator_ != icmp_.user_comparator()->Name()) {
          s = Status::InvalidArgument(
              edit.comparator_ + " does not match existing comparator ",
              icmp_.user_comparator()->Name());
        }
      }

      if (s.ok()) {
        // 应用变更到Builder
        builder.Apply(&edit);
      }

      // 提取元数据
      if (edit.has_log_number_) {
        log_number = edit.log_number_;
        have_log_number = true;
      }

      if (edit.has_prev_log_number_) {
        prev_log_number = edit.prev_log_number_;
        have_prev_log_number = true;
      }

      if (edit.has_next_file_number_) {
        next_file = edit.next_file_number_;
        have_next_file = true;
      }

      if (edit.has_last_sequence_) {
        last_sequence = edit.last_sequence_;
        have_last_sequence = true;
      }
    }
  }

  delete file;
  file = nullptr;

  // ========== 阶段4：验证完整性 ==========
  if (s.ok()) {
    if (!have_next_file) {
      s = Status::Corruption("no meta-nextfile entry in descriptor");
    } else if (!have_log_number) {
      s = Status::Corruption("no meta-lognumber entry in descriptor");
    } else if (!have_last_sequence) {
      s = Status::Corruption("no last-sequence-number entry in descriptor");
    }

    if (!have_prev_log_number) {
      prev_log_number = 0;  // 兼容旧版本
    }

    // 确保文件编号被使用
    MarkFileNumberUsed(prev_log_number);
    MarkFileNumberUsed(log_number);
  }

  // ========== 阶段5：重建Version ==========
  if (s.ok()) {
    // 创建新的Version对象
    Version* v = new Version(this);

    // 将Builder的内容保存到Version
    builder.SaveTo(v);

    // 计算Compaction分数
    Finalize(v);

    // 添加到版本链
    AppendVersion(v);

    // 更新VersionSet状态
    manifest_file_number_ = next_file;
    next_file_number_ = next_file + 1;
    last_sequence_ = last_sequence;
    log_number_ = log_number;
    prev_log_number_ = prev_log_number;

    // ========== 阶段6：决定是否重写MANIFEST ==========
    if (ReuseManifest(dscname, current)) {
      // MANIFEST足够小，可以重用
    } else {
      // MANIFEST太大，需要重写
      *save_manifest = true;
    }
  } else {
    std::string error = s.ToString();
    Log(options_->info_log, "Error recovering version set with %d records: %s",
        read_records, error.c_str());
  }

  return s;
}
```

#### 4.2.2 Recovery流程图

```
Recovery完整流程：

┌─────────────────────────────────────────────┐
│ 1. 读取CURRENT文件                         │
│    /tmp/db/CURRENT                         │
│    内容: "MANIFEST-000001\n"               │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 2. 打开MANIFEST文件                        │
│    /tmp/db/MANIFEST-000001                 │
│    └─> SequentialFile                      │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 3. 读取log记录                             │
│    Record 1: [Comparator][LogNumber][...]  │
│    Record 2: [NewFile Level0 #123]        │
│    Record 3: [NewFile Level1 #200]        │
│    Record 4: [NewFile Level2 #300]        │
│    ...                                    │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 4. 解析每个VersionEdit                     │
│    for each record:                        │
│      DecodeFrom(record) → VersionEdit      │
│      builder.Apply(&edit)                  │
│      提取元数据                            │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 5. 验证完整性                              │
│    ✓ have_next_file                       │
│    ✓ have_log_number                      │
│    ✓ have_last_sequence                   │
│    MarkFileNumberUsed(...)                │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 6. 重建Version                             │
│    builder.SaveTo(v)                       │
│    Finalize(v)                             │
│    AppendVersion(v)                        │
│    current_ = v                            │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ 7. 决定是否重写MANIFEST                     │
│    if (ReuseManifest(...)):                │
│      重用现有MANIFEST                      │
│    else:                                   │
│      save_manifest = true                  │
│      → 后续会写新的MANIFEST                │
└─────────────────────────────────────────────┘
```

#### 4.2.3 Builder::Apply详解

```cpp
// db/version_set.cc
void VersionSet::Builder::Apply(const VersionEdit* edit) {
  // ========== 更新Compaction指针 ==========
  for (const auto& p : edit->compact_pointers_) {
    vset_->compact_pointer_[p.first] = p.second.Encode().ToString();
  }

  // ========== 标记删除的文件 ==========
  for (const auto& p : edit->deleted_files_) {
    const int level = p.first;
    const uint64_t number = p.second;
    levels_[level].deleted_files.insert(number);
  }

  // ========== 添加新文件 ==========
  for (size_t i = 0; i < edit->new_files_.size(); i++) {
    const int level = edit->new_files_[i].first;
    FileMetaData* f = new FileMetaData(edit->new_files_[i].second);
    f->refs = 1;  // 初始引用计数

    // 计算允许的seek次数
    // 如果文件很小，允许更多seek
    f->allowed_seeks = static_cast<int>((f->file_size / 16384U));
    if (f->allowed_seeks < 100) f->allowed_seeks = 100;

    // 从删除集合中移除（防止误删）
    levels_[level].deleted_files.erase(f->number);

    // 添加到新增文件集合（有序集合）
    levels_[level].added_files->insert(f);
  }
}
```

**数据结构：**

```
LevelState结构：
┌─────────────────────────────────────┐
│ LevelState (每个level)              │
├─────────────────────────────────────┤
│ deleted_files: std::set<uint64_t>   │
│   └─ {45, 46, 100}  ← 要删除的文件  │
├─────────────────────────────────────┤
│ added_files: FileSet* (有序)        │
│   └─ {FileMetaData(200),            │
│        FileMetaData(201), ...}      │
└─────────────────────────────────────┘

Apply后的状态：
Level-1:
  deleted_files: {45, 46}  ← 将被删除
  added_files: {200, 201}  ← 新增的文件
```

#### 4.2.4 Builder::SaveTo详解

```cpp
// db/version_set.cc
void VersionSet::Builder::SaveTo(Version* v) {
  BySmallestKey cmp;
  cmp.internal_comparator = &vset_->icmp_;

  // 遍历每一层
  for (int level = 0; level < config::kNumLevels; level++) {
    // 基础Version的文件列表
    const std::vector<FileMetaData*>& base_files = base_->files_[level];

    // 迭代器
    std::vector<FileMetaData*>::const_iterator base_iter = base_files.begin();
    std::vector<FileMetaData*>::const_iterator base_end = base_files.end();

    // 新增的文件（有序集合）
    const FileSet* added_files = levels_[level].added_files;

    // 预留空间
    v->files_[level].reserve(base_files.size() + added_files->size());

    // 合并base和added
    for (const auto* added_file : *added_files) {
      // 跳过base中小于added_file的文件
      for (std::vector<FileMetaData*>::const_iterator bpos =
               std::upper_bound(base_iter, base_end, added_file, cmp);
           base_iter != bpos; ++base_iter) {
        MaybeAddFile(v, level, *base_iter);
      }

      // 添加新文件
      MaybeAddFile(v, level, added_file);
    }

    // 添加剩余的base文件
    for (; base_iter != base_end; ++base_iter) {
      MaybeAddFile(v, level, *base_iter);
    }
  }
}

void VersionSet::Builder::MaybeAddFile(Version* v, int level, FileMetaData* f) {
  if (levels_[level].deleted_files.count(f->number) > 0) {
    // 文件在删除列表中，跳过
  } else {
    // 添加到Version
    std::vector<FileMetaData*>* files = &v->files_[level];

    // Level>0的文件必须不重叠且有序
    if (level > 0 && !files->empty()) {
      assert(vset_->icmp_.Compare((*files)[files->size() - 1]->largest,
                                   f->smallest) < 0);
    }

    // 增加引用计数
    f->refs++;
    files->push_back(f);
  }
}
```

**合并算法示意图：**

```
合并base和added：

base:  [f001, f003, f005, f007, f009]
added: [f002, f004, f006]  (有序)

合并过程：
  added_file = f002
  └─> base跳过f001 (< f002)
  └─> 添加f002 ✓
  └─> base停在f003

  added_file = f004
  └─> base跳过f003 (< f004)
  └─> 添加f004 ✓
  └─> base停在f005

  added_file = f006
  └─> base跳过f005 (< f006)
  └─> 添加f006 ✓
  └─> base停在f007

  遍历剩余base:
  └─> 添加f007 ✓
  └─> 添加f009 ✓

结果：
  [f001, f002, f003, f004, f005, f006, f007, f009]

如果有deleted_files:
  deleted: {f003, f007}

  合并时：
    f001: 不在deleted, 添加 ✓
    f002: 不在deleted, 添加 ✓
    f003: 在deleted, 跳过 ✗
    f004: 不在deleted, 添加 ✓
    ...
```

#### 4.2.5 错误处理

```cpp
// Recovery错误类型

// 1. CURRENT文件缺失或损坏
if (!ReadFileToString(...)) {
  return Status::Corruption("CURRENT file missing");
}

// 2. CURRENT指向的MANIFEST不存在
if (s.IsNotFound()) {
  return Status::Corruption("CURRENT points to non-existent file");
}

// 3. MANIFEST记录损坏
if (!edit.DecodeFrom(record).ok()) {
  reporter.Corruption(pos, Status::Corruption("corrupted record"));
  // 继续尝试读取下一条记录
}

// 4. 缺少必需字段
if (!have_next_file) {
  return Status::Corruption("no meta-nextfile entry");
}
if (!have_log_number) {
  return Status::Corruption("no meta-lognumber entry");
}

// 5. 比较器不匹配
if (edit.comparator_ != icmp_.user_comparator()->Name()) {
  return Status::InvalidArgument("comparator mismatch");
}
```

**容错机制：**

```cpp
// LogReporter处理错误
struct LogReporter : public log::Reader::Reporter {
  Status* status;
  void Corruption(size_t bytes, const Status& s) override {
    // 只记录第一个错误
    if (this->status->ok()) {
      *this->status = s;
    }
  }
};

// 使用
LogReporter reporter;
reporter.status = &s;
log::Reader reader(file, &reporter, true, 0);

while (reader.ReadRecord(&record, &scratch) && s.ok()) {
  // 如果之前有错误，s.ok()返回false，循环停止
  // ...
}
```

## 5. Builder辅助类

```cpp
// db/version_set.cc (内部类)
class VersionSet::Builder {
 private:
  struct BySmallestKey {
    const InternalKeyComparator* internal_comparator;
    bool operator()(FileMetaData* f1, FileMetaData* f2) const {
      int r = internal_comparator->Compare(f1->smallest, f2->smallest);
      if (r != 0) {
        return (r < 0);
      } else {
        return (f1->number < f2->number);
      }
    }
  };

  typedef std::set<FileMetaData*, BySmallestKey> FileSet;
  struct LevelState {
    std::set<uint64_t> deleted_files;
    FileSet* added_files;
  };

  VersionSet* vset_;
  Version* base_;
  LevelState levels_[config::kNumLevels];

 public:
  Builder(VersionSet* vset, Version* base) : vset_(vset), base_(base) {
    base_->Ref();
    BySmallestKey cmp;
    cmp.internal_comparator = &vset_->icmp_;
    for (int level = 0; level < config::kNumLevels; level++) {
      levels_[level].added_files = new FileSet(cmp);
    }
  }

  ~Builder() {
    for (int level = 0; level < config::kNumLevels; level++) {
      const FileSet* added = levels_[level].added_files;
      std::vector<FileMetaData*> to_unref;
      to_unref.reserve(added->size());
      for (FileSet::const_iterator it = added->begin(); it != added->end();
           ++it) {
        to_unref.push_back(*it);
      }
      delete added;
      for (uint32_t i = 0; i < to_unref.size(); i++) {
        FileMetaData* f = to_unref[i];
        f->refs--;
        if (f->refs <= 0) {
          delete f;
        }
      }
    }
    base_->Unref();
  }

  void Apply(const VersionEdit* edit) {
    for (const auto& p : edit->compact_pointers_) {
      vset_->compact_pointer_[p.first] = p.second.Encode().ToString();
    }

    for (const auto& p : edit->deleted_files_) {
      const int level = p.first;
      const uint64_t number = p.second;
      levels_[level].deleted_files.insert(number);
    }

    for (size_t i = 0; i < edit->new_files_.size(); i++) {
      const int level = edit->new_files_[i].first;
      FileMetaData* f = new FileMetaData(edit->new_files_[i].second);
      f->refs = 1;

      f->allowed_seeks = static_cast<int>((f->file_size / 16384U));
      if (f->allowed_seeks < 100) f->allowed_seeks = 100;

      levels_[level].deleted_files.erase(f->number);
      levels_[level].added_files->insert(f);
    }
  }

  void SaveTo(Version* v) {
    BySmallestKey cmp;
    cmp.internal_comparator = &vset_->icmp_;
    for (int level = 0; level < config::kNumLevels; level++) {
      const std::vector<FileMetaData*>& base_files = base_->files_[level];
      std::vector<FileMetaData*>::const_iterator base_iter = base_files.begin();
      std::vector<FileMetaData*>::const_iterator base_end = base_files.end();
      const FileSet* added_files = levels_[level].added_files;
      v->files_[level].reserve(base_files.size() + added_files->size());
      for (const auto* added_file : *added_files) {
        for (std::vector<FileMetaData*>::const_iterator bpos =
                 std::upper_bound(base_iter, base_end, added_file, cmp);
             base_iter != bpos; ++base_iter) {
          MaybeAddFile(v, level, *base_iter);
        }

        MaybeAddFile(v, level, added_file);
      }

      for (; base_iter != base_end; ++base_iter) {
        MaybeAddFile(v, level, *base_iter);
      }
    }
  }

  void MaybeAddFile(Version* v, int level, FileMetaData* f) {
    if (levels_[level].deleted_files.count(f->number) > 0) {
      // 文件已删除
    } else {
      std::vector<FileMetaData*>* files = &v->files_[level];
      if (level > 0 && !files->empty()) {
        assert(vset_->icmp_.Compare((*files)[files->size() - 1]->largest,
                                     f->smallest) < 0);
      }
      f->refs++;
      files->push_back(f);
    }
  }
};
```

## 6. 性能影响

### 6.1 Version切换开销

```
LogAndApply操作：
1. 创建新Version：O(文件数量)
2. 写入MANIFEST：O(变更数量)  
3. Sync MANIFEST：~10ms
4. 更新CURRENT：~1ms

总开销：~15ms
频率：每次Compaction完成

影响：
- 后台操作，不阻塞读写
- MANIFEST是顺序写，性能好
```

### 6.2 内存使用

```
VersionSet内存开销：
- 每个Version：~100KB (取决于文件数量)
- 版本链：通常1-3个Version
- FileMetaData：每个文件~100字节

示例（10000个文件）：
- Version: 10000 × 100B = 1MB
- 3个Version: 3MB
- 可接受的内存开销
```

## 总结

今天我们学习了：
1. ✅ **Version**：数据库状态快照，引用计数管理
2. ✅ **VersionEdit**：记录增量变更
3. ✅ **VersionSet**：版本链管理，LogAndApply
4. ✅ **MANIFEST**：持久化版本信息，崩溃恢复

**关键要点：**
- Version解决Compaction与读取的并发问题
- VersionEdit记录增量，避免重写全量
- MANIFEST保证元数据持久性
- Builder辅助类简化Version构建

**思考题：**
1. 为什么需要版本链而不是单个Version?
2. MANIFEST损坏如何恢复?
3. Version的引用计数何时为0?
4. 如何优化MANIFEST的大小?

**明天预告：Day 12 - Cache与Bloom Filter优化**
我们将学习LevelDB的关键性能优化组件。
