# Day 6: Write-Ahead Log (WAL)

## 学习目标
- 理解WAL在LevelDB中的作用
- 掌握log文件的格式和结构
- 学习log_writer和log_reader的实现
- 理解崩溃恢复流程

## 1. WAL概述

### 1.1 为什么需要WAL？

**持久性问题：**

```
场景：写入操作的生命周期
┌─────────────────────────────────────┐
│ Client: db->Put("key", "value")     │
└──────────┬──────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 写入MemTable（内存）                 │  ← 断电丢失！
└─────────────────────────────────────┘
           ↓ (几秒后)
┌─────────────────────────────────────┐
│ 刷盘到SSTable                       │
└─────────────────────────────────────┘

问题：MemTable在刷盘前断电，数据丢失
```

**WAL解决方案：**

```
写入流程：
Client
  ↓
┌─────────────────────────────────────┐
│ 1. 先写WAL (*.log)                  │  ← 顺序追加，极快
│    fsync保证持久化                  │  ← 可选
└──────────┬──────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 2. 再写MemTable                     │  ← 内存操作
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 3. 后台刷盘（异步）                  │
└─────────────────────────────────────┘

保证：即使MemTable丢失，也能从WAL恢复
```

### 1.2 WAL的核心特性

**特性1：顺序写入**
```
WAL文件结构：
000001.log:  [Record1][Record2][Record3]...
             ↑ 只追加，永不修改

性能对比：
- 随机写入：~100 IOPS (HDD)
- 顺序写入：~1000 IOPS (HDD)
- SSD：顺序写也有优势（减少写放大）
```

**特性2：可选的fsync**
```cpp
WriteOptions options;
options.sync = false;  // 快速模式（可能丢失）
options.sync = true;   // 安全模式（保证持久化）

// 性能差异
sync=false: ~100,000 ops/sec
sync=true:  ~100 ops/sec  (慢1000倍)
```

**特性3：崩溃恢复**
```
数据库启动流程：
1. 检查MANIFEST，找到已刷盘的MemTable
2. 扫描WAL文件，找到未刷盘的记录
3. 重放WAL，恢复MemTable
4. 继续服务
```

## 2. WAL文件格式

### 2.1 整体结构

```
*.log 文件布局：
┌─────────────────────────────────────────────┐
│ Block 0 (32KB)                              │
│ ┌─────────────────────────────────────────┐ │
│ │ Record 1 | Record 2 | Record 3 | ...    │ │
│ └─────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│ Block 1 (32KB)                              │
│ ┌─────────────────────────────────────────┐ │
│ │ Record N | Record N+1 | ...             │ │
│ └─────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│ Block 2 (32KB)                              │
│ ...                                         │
└─────────────────────────────────────────────┘

关键参数：
- Block大小：32KB (kBlockSize)
- Header大小：7 bytes
- 最小Record：Header(7) + Data(1) = 8 bytes
```

**为什么分块？**
1. **对齐读取**：32KB是常见的I/O块大小
2. **限制错误传播**：块损坏不影响其他块
3. **快速定位**：可以跳过完整的块

### 2.2 Record格式

```cpp
// db/log_format.h, lines 15-32

Record Header (7 bytes):
┌──────────┬──────────┬───────┬──────┐
│ Checksum │ Length   │ Type  │ Data │
│ (4 bytes)│ (2 bytes)│(1 byte)│ ...  │
└──────────┴──────────┴───────┴──────┘

字段说明：
- Checksum: CRC32C(type + data)
- Length: 数据长度（不包括header）
- Type: Record类型（FULL/FIRST/MIDDLE/LAST）
```

**Record类型：**

```cpp
enum RecordType {
  kZeroType = 0,     // 预分配的空间
  kFullType = 1,     // 完整记录
  kFirstType = 2,    // 第一个片段
  kMiddleType = 3,   // 中间片段
  kLastType = 4      // 最后片段
};
```

**分片示例：**

```
场景：写入50KB数据（超过单个Block）

Block 0 (32KB):
┌────────────────────────────────────┐
│ Header: Type=FIRST, Length=32KB-7  │
│ Data: [0..32KB-7]                  │
└────────────────────────────────────┘

Block 1 (32KB):
┌────────────────────────────────────┐
│ Header: Type=MIDDLE, Length=32KB   │
│ Data: [32KB-7..64KB-7]             │
└────────────────────────────────────┘

Block 2 (32KB):
┌────────────────────────────────────┐
│ Header: Type=LAST, Length=~18KB    │
│ Data: [64KB-7..50KB]               │
└────────────────────────────────────┘
```

### 2.3 对齐和填充

```cpp
// db/log_writer.cc, lines 53-68

// 当前Block剩余空间
int leftover = kBlockSize - block_offset_;

// 情况1：剩余空间 < Header大小
if (leftover < kHeaderSize) {
  // 填充零，跳到下一个Block
  if (leftover > 0) {
    dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
  }
  block_offset_ = 0;
}

// 情况2：剩余空间 >= Header，判断是否分片
const int avail = kBlockSize - block_offset_ - kHeaderSize;
if (left < avail) {
  // 可以放下，写入FULL
} else {
  // 需要分片，写入FIRST
}
```

**填充示例：**

```
Block末尾只剩5字节：
┌─────────────────────────────┬──────┐
│ ... [Record N]              │\x00  │
│                             │\x00  │
│                             │\x00  │
│                             │\x00  │
│                             │\x00  │
└─────────────────────────────┴──────┘
                              ↑ 填充零
                              （剩余空间 < 7字节）
```

## 3. LogWriter实现

### 3.1 Writer类定义

```cpp
// db/log_writer.h, lines 18-42
namespace log {

class Writer {
 public:
  explicit Writer(WritableFile* dest);
  Writer(WritableFile* dest, uint64_t dest_length);
  ~Writer();

  Status AddRecord(const Slice& slice);

 private:
  WritableFile* dest_;       // 底层文件
  int block_offset_;         // 当前Block内偏移
  uint32_t type_crc_[kMaxRecordType + 1];  // 预计算的CRC

  Status EmitPhysicalRecord(RecordType type, const char* ptr, size_t length);

  // 禁止拷贝
  Writer(const Writer&);
  void operator=(const Writer&);
};

}  // namespace log
```

### 3.2 添加记录

```cpp
// db/log_writer.cc, lines 36-107
Status Writer::AddRecord(const Slice& slice) {
  const char* ptr = slice.data();
  size_t left = slice.size();

  Status s;
  bool begin = true;
  do {
    const int leftover = kBlockSize - block_offset_;
    assert(leftover >= 0);

    if (leftover < kHeaderSize) {
      // 剩余空间不足，填充并切换到下一个Block
      if (leftover > 0) {
        static_assert(kHeaderSize == 7, "");
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
      }
      block_offset_ = 0;
    }

    // 可用空间（减去Header）
    const int avail = kBlockSize - block_offset_ - kHeaderSize;
    const size_t fragment_length = (left < avail) ? left : avail;

    RecordType type;
    const bool end = (left == fragment_length);
    if (begin && end) {
      type = kFullType;
    } else if (begin) {
      type = kFirstType;
    } else if (end) {
      type = kLastType;
    } else {
      type = kMiddleType;
    }

    s = EmitPhysicalRecord(type, ptr, fragment_length);
    ptr += fragment_length;
    left -= fragment_length;
    begin = false;
  } while (s.ok() && left > 0);

  return s;
}
```

**逻辑流程：**

```
AddRecord("50KB data")
  ↓
┌─────────────────────────────┐
│ 1. 计算Block剩余空间        │
│    leftover = 32KB - offset │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ 2. 判断是否需要填充         │
│    if leftover < 7: 填充    │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ 3. 确定Record类型           │
│    FULL/FIRST/MIDDLE/LAST   │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ 4. 写入物理Record           │
│    EmitPhysicalRecord()     │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ 5. 移动指针，继续循环       │
└─────────────────────────────┘
```

### 3.3 写入物理Record

```cpp
// db/log_writer.cc, lines 109-137
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr,
                                   size_t length) {
  assert(length <= 0xffff);  // 2字节长度限制
  assert(block_offset_ + kHeaderSize + length <= kBlockSize);

  // 格式化Header
  char buf[kHeaderSize];
  buf[4] = static_cast<char>(length & 0xff);         // Length低字节
  buf[5] = static_cast<char>(length >> 8);           // Length高字节
  buf[6] = static_cast<char>(t);                     // Type

  // 计算CRC: Type + Data
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, length);
  crc = crc32c::Mask(crc);  // 调整到安全范围
  EncodeFixed32(buf, crc);  // CRC写入buf[0..3]

  // 写入Header
  Status s = dest_->Append(Slice(buf, kHeaderSize));
  if (s.ok()) {
    // 写入Data
    s = dest_->Append(Slice(ptr, length));
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  block_offset_ += kHeaderSize + length;
  return s;
}
```

**CRC校验：**

```cpp
// util/crc32c.h
// CRC32C: Castagnoli多项式
// 硬件加速（SSE4.2指令）

uint32_t Value(const char* data, size_t n);
uint32_t Extend(uint32_t crc, const char* data, size_t n);
uint32_t Mask(uint32_t crc);  // 避免全0和全1
```

## 4. LogReader实现

### 4.1 Reader类定义

```cpp
// db/log_reader.h, lines 30-90
namespace log {

class Reader {
 public:
  class Reporter {
   public:
    virtual ~Reporter();
    virtual void Corruption(size_t bytes, const Status& status) = 0;
  };

  Reader(SequentialFile* file, Reporter* reporter, bool checksum,
         uint64_t initial_offset);
  ~Reader();

  bool ReadRecord(Slice* record, std::string* scratch);
  uint64_t LastRecordOffset();

 private:
  SequentialFile* const file_;
  Reporter* const reporter_;
  bool const checksum_;          // 是否验证CRC
  char* const backing_store_;    // Buffer (kBlockSize)
  Slice buffer_;                 // 当前Buffer数据
  bool eof_;                     // 是否到达文件尾

  uint64_t last_record_offset_;
  uint64_t end_of_buffer_offset_;

  enum {
    kEof = kMaxRecordType + 1,
    kBadRecord = kMaxRecordType + 2
  };

  bool SkipToInitialBlock();
  unsigned int ReadPhysicalRecord(Slice* result);
  void ReportCorruption(uint64_t bytes, const char* reason);
  void ReportDrop(uint64_t bytes, const Status& reason);
};

}  // namespace log
```

### 4.2 读取记录

```cpp
// db/log_reader.cc, lines 82-165
bool Reader::ReadRecord(Slice* record, std::string* scratch) {
  if (last_record_offset_ < initial_offset_) {
    if (!SkipToInitialBlock()) {
      return false;
    }
  }

  scratch->clear();
  record->clear();
  bool in_fragmented_record = false;
  uint64_t prospective_record_offset = 0;

  Slice fragment;
  while (true) {
    const unsigned int record_type = ReadPhysicalRecord(&fragment);

    switch (record_type) {
      case kFullType:
        if (in_fragmented_record) {
          ReportCorruption(scratch->size(), "partial record without end");
        }
        prospective_record_offset = last_record_offset_;
        scratch->clear();
        *record = fragment;
        last_record_offset_ = prospective_record_offset;
        return true;

      case kFirstType:
        if (in_fragmented_record) {
          ReportCorruption(scratch->size(), "partial record without end");
        }
        prospective_record_offset = last_record_offset_;
        scratch->assign(fragment.data(), fragment.size());
        in_fragmented_record = true;
        break;

      case kMiddleType:
        if (!in_fragmented_record) {
          ReportCorruption(fragment.size(),
                           "missing start of fragmented record");
        } else {
          scratch->append(fragment.data(), fragment.size());
        }
        break;

      case kLastType:
        if (!in_fragmented_record) {
          ReportCorruption(fragment.size(),
                           "missing start of fragmented record");
        } else {
          scratch->append(fragment.data(), fragment.size());
          *record = Slice(*scratch);
          last_record_offset_ = prospective_record_offset;
          return true;
        }
        break;

      case kEof:
        if (in_fragmented_record) {
          scratch->clear();
        }
        return false;

      case kBadRecord:
        if (in_fragmented_record) {
          ReportCorruption(scratch->size(), "error in middle of record");
          in_fragmented_record = false;
          scratch->clear();
        }
        break;

      default:
        char buf[40];
        std::snprintf(buf, sizeof(buf), "unknown record type %u", record_type);
        ReportCorruption((fragment.size() + (in_fragmented_record ? scratch->size() : 0)),
                         buf);
        in_fragmented_record = false;
        scratch->clear();
        break;
    }
  }
  return false;
}
```

**状态机：**

```
初始状态
  ↓
┌─────────────────────────────────┐
│ ReadPhysicalRecord()            │
└──────┬──────────────────────────┘
       ↓
  ┌────────┐
  │ Type?  │
  └───┬────┘
      ├─→ FULL  ─→ 返回完整记录
      ├─→ FIRST ─→ 进入分片模式，继续读MIDDLE/LAST
      ├─→ MIDDLE─→ 追加到scratch
      ├─→ LAST  ─→ 追加到scratch，返回完整记录
      ├─→ EOF   ─→ 返回false
      └─→ BAD   ─→ 报告错误，继续读
```

### 4.3 读取物理Record

```cpp
// db/log_reader.cc, lines 167-260
unsigned int Reader::ReadPhysicalRecord(Slice* result) {
  while (true) {
    if (buffer_.size() < kHeaderSize) {
      if (!eof_) {
        // 读取下一个Block
        buffer_.clear();
        Status status = file_->Read(kBlockSize, &buffer_, backing_store_);
        end_of_buffer_offset_ += buffer_.size();
        if (!status.ok()) {
          buffer_.clear();
          ReportDrop(kBlockSize, status);
          eof_ = true;
          return kEof;
        } else if (buffer_.size() < kBlockSize) {
          eof_ = true;
        }
        continue;
      } else {
        return kEof;
      }
    }

    // 解析Header
    const char* header = buffer_.data();
    const uint32_t a = static_cast<uint32_t>(header[4]) & 0xff;
    const uint32_t b = static_cast<uint32_t>(header[5]) & 0xff;
    const unsigned int type = header[6];
    const uint32_t length = a | (b << 8);

    if (kHeaderSize + length > buffer_.size()) {
      size_t drop_size = buffer_.size();
      buffer_.clear();
      ReportCorruption(drop_size, "bad record length");
      return kBadRecord;
    }

    if (type == kZeroType && length == 0) {
      // 跳过填充
      buffer_.clear();
      return kBadRecord;
    }

    // 验证CRC
    if (checksum_) {
      uint32_t expected_crc = crc32c::Unmask(DecodeFixed32(header));
      uint32_t actual_crc = crc32c::Value(header + 6, 1 + length);
      if (actual_crc != expected_crc) {
        size_t drop_size = buffer_.size();
        buffer_.clear();
        ReportCorruption(drop_size, "checksum mismatch");
        return kBadRecord;
      }
    }

    buffer_.remove_prefix(kHeaderSize + length);

    *result = Slice(header + kHeaderSize, length);
    return type;
  }
}
```

## 5. 崩溃恢复

### 5.1 恢复流程

```cpp
// db/db_impl.cc, lines 168-238
Status DBImpl::Recover(VersionEdit* edit, bool* save_manifest) {
  mutex_.AssertHeld();

  // 创建目录
  env_->CreateDir(dbname_);
  assert(db_lock_ == nullptr);
  Status s = env_->LockFile(LockFileName(dbname_), &db_lock_);
  if (!s.ok()) {
    return s;
  }

  if (!env_->FileExists(CurrentFileName(dbname_))) {
    if (options_.create_if_missing) {
      s = NewDB();
      if (!s.ok()) {
        return s;
      }
    } else {
      return Status::InvalidArgument(
          dbname_, "does not exist (create_if_missing is false)");
    }
  } else {
    if (options_.error_if_exists) {
      return Status::InvalidArgument(dbname_,
                                      "exists (error_if_exists is true)");
    }
  }

  s = versions_->Recover(save_manifest);
  if (!s.ok()) {
    return s;
  }

  // 恢复WAL日志
  SequenceNumber max_sequence(0);
  const uint64_t min_log = versions_->LogNumber();
  const uint64_t prev_log = versions_->PrevLogNumber();
  std::vector<std::string> filenames;
  s = env_->GetChildren(dbname_, &filenames);
  if (!s.ok()) {
    return s;
  }

  std::set<uint64_t> expected;
  versions_->AddLiveFiles(&expected);
  uint64_t number;
  FileType type;
  std::vector<uint64_t> logs;
  for (size_t i = 0; i < filenames.size(); i++) {
    if (ParseFileName(filenames[i], &number, &type)) {
      expected.erase(number);
      if (type == kLogFile && ((number >= min_log) || (number == prev_log)))
        logs.push_back(number);
    }
  }

  // 按顺序恢复日志文件
  std::sort(logs.begin(), logs.end());
  for (size_t i = 0; i < logs.size(); i++) {
    s = RecoverLogFile(logs[i], (i == logs.size() - 1), save_manifest, edit,
                       &max_sequence);
    if (!s.ok()) {
      return s;
    }
  }

  if (versions_->LastSequence() < max_sequence) {
    versions_->SetLastSequence(max_sequence);
  }

  return Status::OK();
}
```

### 5.2 恢复单个日志文件

```cpp
// db/db_impl.cc, lines 345-448
Status DBImpl::RecoverLogFile(uint64_t log_number, bool last_log,
                               bool* save_manifest, VersionEdit* edit,
                               SequenceNumber* max_sequence) {
  struct LogReporter : public log::Reader::Reporter {
    Env* env;
    Logger* info_log;
    const char* fname;
    Status* status;
    void Corruption(size_t bytes, const Status& s) override {
      Log(info_log, "%s%s: dropping %d bytes; %s",
          (this->status == nullptr ? "(ignoring error) " : ""), fname,
          static_cast<int>(bytes), s.ToString().c_str());
      if (this->status != nullptr && this->status->ok()) *this->status = s;
    }
  };

  mutex_.AssertHeld();

  // 打开日志文件
  std::string fname = LogFileName(dbname_, log_number);
  SequentialFile* file;
  Status status = env_->NewSequentialFile(fname, &file);
  if (!status.ok()) {
    MaybeIgnoreError(&status);
    return status;
  }

  LogReporter reporter;
  reporter.env = env_;
  reporter.info_log = options_.info_log;
  reporter.fname = fname.c_str();
  reporter.status = (options_.paranoid_checks ? &status : nullptr);

  log::Reader reader(file, &reporter, true, 0);
  Log(options_.info_log, "Recovering log #%llu",
      (unsigned long long)log_number);

  // 读取所有记录
  std::string scratch;
  Slice record;
  WriteBatch batch;
  int compactions = 0;
  MemTable* mem = nullptr;
  while (reader.ReadRecord(&record, &scratch) && status.ok()) {
    if (record.size() < 12) {
      reporter.Corruption(record.size(),
                          Status::Corruption("log record too small"));
      continue;
    }
    WriteBatchInternal::SetContents(&batch, record);

    if (mem == nullptr) {
      mem = new MemTable(internal_comparator_);
      mem->Ref();
    }
    status = WriteBatchInternal::InsertInto(&batch, mem);
    MaybeIgnoreError(&status);
    if (!status.ok()) {
      break;
    }
    const SequenceNumber last_seq = WriteBatchInternal::Sequence(&batch) +
                                     WriteBatchInternal::Count(&batch) - 1;
    if (last_seq > *max_sequence) {
      *max_sequence = last_seq;
    }

    if (mem->ApproximateMemoryUsage() > options_.write_buffer_size) {
      compactions++;
      *save_manifest = true;
      status = WriteLevel0Table(mem, edit, nullptr);
      mem->Unref();
      mem = nullptr;
      if (!status.ok()) {
        break;
      }
    }
  }

  delete file;

  // 处理最后的MemTable
  if (status.ok() && mem != nullptr) {
    if (last_log && mem->ApproximateMemoryUsage() < options_.write_buffer_size) {
      // 重用这个MemTable
      assert(memtable_ == nullptr);
      memtable_ = mem;
      mem = nullptr;
    } else {
      *save_manifest = true;
      status = WriteLevel0Table(mem, edit, nullptr);
    }
  }

  if (mem != nullptr) {
    mem->Unref();
  }

  return status;
}
```

**恢复逻辑：**

```
1. 找到所有WAL文件
   ├─ 000001.log
   ├─ 000003.log
   └─ 000005.log (当前)

2. 按顺序重放每个文件
   ├─ 读取Record
   ├─ 解析为WriteBatch
   ├─ 插入到MemTable
   └─ MemTable满时刷盘

3. 处理最后的MemTable
   ├─ 如果小于阈值：保留为当前MemTable
   └─ 如果大于阈值：刷盘为SSTable

4. 更新SequenceNumber
```

## 6. 性能分析

### 6.1 写入性能

```cpp
// 测试：100万次Put操作

// 场景1：sync=false (默认)
WriteOptions options;
options.sync = false;
// 吞吐量：~100,000 ops/sec
// WAL写入：缓冲到OS缓存
// 持久性：可能丢失最后几秒数据

// 场景2：sync=true (安全)
WriteOptions options;
options.sync = true;
// 吞吐量：~100 ops/sec (慢1000倍！)
// WAL写入：每次fsync()
// 持久性：保证不丢失

// 场景3：批量写入 + sync
WriteBatch batch;
for (int i = 0; i < 1000; i++) {
  batch.Put(key, value);
}
WriteOptions options;
options.sync = true;
db->Write(options, &batch);
// 吞吐量：~100,000 ops/sec
// 持久性：保证不丢失
// 优势：批量fsync
```

### 6.2 磁盘使用

```
WAL文件生命周期：
000001.log  ← 当前写入
  ↓ MemTable满
000003.log  ← 切换新文件
  ↓ 000001.log对应的MemTable刷盘完成
删除000001.log  ← 空间回收

磁盘占用：
- 正常情况：1-2个WAL文件
- 极端情况：多个WAL文件（后台刷盘慢）
- 单个WAL大小：~4MB（MemTable大小）
```

### 6.3 恢复时间

```
恢复时间 = WAL文件数 × 单文件恢复时间

单文件恢复时间：
- I/O时间：~10ms (读取4MB)
- 重放时间：~100ms (重建MemTable)
- 总计：~110ms

最坏情况：
- 10个WAL文件（异常情况）
- 恢复时间：~1秒

优化：
- 及时刷盘，减少WAL积累
- 限制WAL文件数量
```

## 7. 代码实践

### 7.1 WAL写入示例

```cpp
#include "db/log_writer.h"
#include "leveldb/env.h"
#include <iostream>

using namespace leveldb;

int main() {
  Env* env = Env::Default();

  // 创建WAL文件
  WritableFile* file;
  Status s = env->NewWritableFile("test.log", &file);
  if (!s.ok()) {
    std::cerr << "Failed to create file: " << s.ToString() << std::endl;
    return 1;
  }

  log::Writer writer(file);

  // 写入记录
  for (int i = 0; i < 100; i++) {
    std::string record = "Record " + std::to_string(i);
    s = writer.AddRecord(Slice(record));
    if (!s.ok()) {
      std::cerr << "Failed to add record: " << s.ToString() << std::endl;
      break;
    }
  }

  delete file;
  std::cout << "Wrote 100 records to test.log\n";
  return 0;
}
```

### 7.2 WAL读取示例

```cpp
#include "db/log_reader.h"
#include "leveldb/env.h"
#include <iostream>

using namespace leveldb;

class SimpleReporter : public log::Reader::Reporter {
 public:
  void Corruption(size_t bytes, const Status& status) override {
    std::cerr << "Corruption: " << bytes << " bytes, "
              << status.ToString() << std::endl;
  }
};

int main() {
  Env* env = Env::Default();

  // 打开WAL文件
  SequentialFile* file;
  Status s = env->NewSequentialFile("test.log", &file);
  if (!s.ok()) {
    std::cerr << "Failed to open file: " << s.ToString() << std::endl;
    return 1;
  }

  SimpleReporter reporter;
  log::Reader reader(file, &reporter, true, 0);

  // 读取记录
  std::string scratch;
  Slice record;
  int count = 0;
  while (reader.ReadRecord(&record, &scratch)) {
    std::cout << "Record " << count++ << ": "
              << record.ToString() << std::endl;
  }

  delete file;
  std::cout << "Read " << count << " records from test.log\n";
  return 0;
}
```

### 7.3 崩溃恢复测试

```cpp
#include "leveldb/db.h"
#include <iostream>
#include <cstdlib>

using namespace leveldb;

void WriteData(DB* db, int start, int count) {
  for (int i = start; i < start + count; i++) {
    std::string key = "key" + std::to_string(i);
    std::string value = "value" + std::to_string(i);
    Status s = db->Put(WriteOptions(), key, value);
    if (!s.ok()) {
      std::cerr << "Put failed: " << s.ToString() << std::endl;
    }
  }
}

int main() {
  DB* db;
  Options options;
  options.create_if_missing = true;

  // 场景1：正常写入
  Status s = DB::Open(options, "/tmp/crash_test", &db);
  if (!s.ok()) {
    std::cerr << "Open failed: " << s.ToString() << std::endl;
    return 1;
  }

  WriteData(db, 0, 1000);
  std::cout << "Wrote 1000 records\n";

  // 模拟崩溃（不正常关闭）
  delete db;
  std::cout << "Simulated crash (no clean shutdown)\n";

  // 场景2：恢复
  s = DB::Open(options, "/tmp/crash_test", &db);
  if (!s.ok()) {
    std::cerr << "Recovery failed: " << s.ToString() << std::endl;
    return 1;
  }
  std::cout << "Recovery successful\n";

  // 验证数据
  std::string value;
  for (int i = 0; i < 1000; i++) {
    std::string key = "key" + std::to_string(i);
    s = db->Get(ReadOptions(), key, &value);
    if (!s.ok()) {
      std::cerr << "Missing key after recovery: " << key << std::endl;
    }
  }
  std::cout << "All data recovered successfully\n";

  delete db;
  return 0;
}
```

## 总结

今天我们学习了：
1. ✅ **WAL作用**：保证持久性，支持崩溃恢复
2. ✅ **文件格式**：32KB Block + 7字节Header + CRC校验
3. ✅ **Record类型**：FULL/FIRST/MIDDLE/LAST支持大记录分片
4. ✅ **Writer/Reader**：高效的追加写入和顺序读取
5. ✅ **崩溃恢复**：按序重放WAL，重建MemTable

**关键要点：**
- WAL是LSM-Tree持久性的保证
- 顺序追加写入，性能极高
- CRC校验保证数据完整性
- 分Block存储，限制错误传播
- sync选项在性能和持久性之间权衡

**思考题：**
1. 为什么Block大小选择32KB？
2. 如果WAL损坏，LevelDB如何处理？
3. 能否并发写入WAL？为什么？
4. WAL和传统数据库的Redo Log有什么区别？

**明天预告：Day 7 - 读取路径与迭代器**
我们将学习LevelDB的完整读取路径，包括多层迭代器的合并和快照机制。
