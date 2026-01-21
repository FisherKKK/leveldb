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

### 2.2 编码格式

```
VersionEdit编码（写入MANIFEST）：

[Tag: varint32][Data...]

Tag类型：
  kComparator = 1
  kLogNumber = 2
  kNextFileNumber = 3
  kLastSequence = 4
  kCompactPointer = 5
  kDeletedFile = 6
  kNewFile = 7
  kPrevLogNumber = 9

示例：
  Tag=kNewFile, Data=[Level:varint][FileNum:varint][FileSize:varint]
                     [SmallestKey:LengthPrefixed][LargestKey:LengthPrefixed]
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

### 4.2 Recover恢复

```cpp
// db/version_set.cc
Status VersionSet::Recover(bool* save_manifest) {
  struct LogReporter : public log::Reader::Reporter {
    Status* status;
    void Corruption(size_t bytes, const Status& s) override {
      if (this->status->ok()) *this->status = s;
    }
  };

  std::string current;
  Status s = ReadFileToString(env_, CurrentFileName(dbname_), &current);
  if (!s.ok()) {
    return s;
  }
  if (current.empty() || current[current.size() - 1] != '\n') {
    return Status::Corruption("CURRENT file does not end with newline");
  }
  current.resize(current.size() - 1);

  std::string dscname = dbname_ + "/" + current;
  SequentialFile* file;
  s = env_->NewSequentialFile(dscname, &file);
  if (!s.ok()) {
    if (s.IsNotFound()) {
      return Status::Corruption("CURRENT points to a non-existent file",
                                current);
    }
    return s;
  }

  bool have_log_number = false;
  bool have_prev_log_number = false;
  bool have_next_file = false;
  bool have_last_sequence = false;
  uint64_t next_file = 0;
  uint64_t last_sequence = 0;
  uint64_t log_number = 0;
  uint64_t prev_log_number = 0;
  Builder builder(this, current_);
  int read_records = 0;

  {
    LogReporter reporter;
    reporter.status = &s;
    log::Reader reader(file, &reporter, true, 0);
    Slice record;
    std::string scratch;
    while (reader.ReadRecord(&record, &scratch) && s.ok()) {
      ++read_records;
      VersionEdit edit;
      s = edit.DecodeFrom(record);
      if (s.ok()) {
        if (edit.has_comparator_ &&
            edit.comparator_ != icmp_.user_comparator()->Name()) {
          s = Status::InvalidArgument(
              edit.comparator_ + " does not match existing comparator ",
              icmp_.user_comparator()->Name());
        }
      }

      if (s.ok()) {
        builder.Apply(&edit);
      }

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

  if (s.ok()) {
    if (!have_next_file) {
      s = Status::Corruption("no meta-nextfile entry in descriptor");
    } else if (!have_log_number) {
      s = Status::Corruption("no meta-lognumber entry in descriptor");
    } else if (!have_last_sequence) {
      s = Status::Corruption("no last-sequence-number entry in descriptor");
    }

    if (!have_prev_log_number) {
      prev_log_number = 0;
    }

    MarkFileNumberUsed(prev_log_number);
    MarkFileNumberUsed(log_number);
  }

  if (s.ok()) {
    Version* v = new Version(this);
    builder.SaveTo(v);
    Finalize(v);
    AppendVersion(v);
    manifest_file_number_ = next_file;
    next_file_number_ = next_file + 1;
    last_sequence_ = last_sequence;
    log_number_ = log_number;
    prev_log_number_ = prev_log_number;

    if (ReuseManifest(dscname, current)) {
      // 重用现有MANIFEST
    } else {
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
