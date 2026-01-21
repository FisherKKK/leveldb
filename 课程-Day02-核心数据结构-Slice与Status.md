# Day 2: 核心数据结构 - Slice与Status

## 学习目标
- 理解Slice的零拷贝设计
- 掌握Status错误处理机制
- 学习Options配置系统
- 理解无异常的错误处理模式

## 1. Slice：高效的字节数组引用

### 1.1 为什么需要Slice？

传统C++处理字符串的问题：

```cpp
// 问题1：std::string 拷贝开销大
void Process(std::string key) {  // 拷贝整个字符串！
    // ...
}

// 问题2：const char* 不知道长度
void Process(const char* data) {  // 需要strlen()或终止符
    size_t len = strlen(data);    // O(N) 开销
}

// 问题3：子字符串需要分配内存
std::string full = "user:1001:name";
std::string prefix = full.substr(0, 9);  // 分配新内存！
```

**Slice的解决方案：**

```cpp
// include/leveldb/slice.h
class Slice {
 private:
  const char* data_;  // 指向数据（不拥有）
  size_t size_;       // 长度

 public:
  // 零拷贝构造
  Slice(const char* d, size_t n) : data_(d), size_(n) {}
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) {}
  Slice(const char* s) : data_(s), size_(strlen(s)) {}
};
```

### 1.2 Slice的核心特性

#### 特性1：零拷贝（Non-owning Reference）

```cpp
std::string buffer = "Hello, LevelDB!";

Slice s1(buffer);              // 只复制指针和长度
Slice s2(buffer.data(), 5);    // "Hello"，无内存分配
Slice s3 = Slice(buffer).substr(7, 7);  // "LevelDB"，无内存分配

// 危险：Slice不拥有内存！
Slice dangerous() {
    std::string temp = "temporary";
    return Slice(temp);  // ❌ 悬空指针！temp销毁后
}
```

**关键点：**
- Slice类似`std::string_view`（C++17）
- 只存储指针和长度（16字节）
- 不管理内存生命周期
- 必须确保底层数据有效

#### 特性2：高效的子串操作

```cpp
// include/leveldb/slice.h, lines 58-68
void remove_prefix(size_t n) {
  assert(n <= size());
  data_ += n;      // 指针移动
  size_ -= n;      // 长度减少
}

// 使用示例
Slice key = "user:1001:name";
key.remove_prefix(5);  // 现在是 "1001:name"，无拷贝！
```

#### 特性3：方便的比较操作

```cpp
// include/leveldb/slice.h, lines 43-56
int compare(const Slice& b) const {
  const size_t min_len = (size_ < b.size_) ? size_ : b.size_;
  int r = memcmp(data_, b.data_, min_len);
  if (r == 0) {
    if (size_ < b.size_) r = -1;
    else if (size_ > b.size_) r = +1;
  }
  return r;
}

// 使用
Slice a = "apple";
Slice b = "banana";
if (a < b) {  // 使用operator<
    // 字典序比较
}
```

### 1.3 Slice在LevelDB中的应用

#### 应用1：API接口

```cpp
// include/leveldb/db.h
class DB {
 public:
  // 所有键值参数都用Slice，避免拷贝
  virtual Status Put(const WriteOptions& options,
                     const Slice& key,
                     const Slice& value) = 0;

  virtual Status Get(const ReadOptions& options,
                     const Slice& key,
                     std::string* value) = 0;
};

// 使用示例
db->Put(WriteOptions(), "key", "value");  // 从字面量构造Slice
db->Put(WriteOptions(),
        Slice("binary\0data", 11),        // 支持二进制数据
        my_string_value);                 // 从string构造Slice
```

#### 应用2：内部键格式

```cpp
// db/dbformat.h
// Internal Key = User Key + Sequence Number (7 bytes) + Type (1 byte)
class InternalKey {
  std::string rep_;  // 拥有内存

 public:
  Slice user_key() const {
    // 返回前N-8字节，零拷贝！
    return Slice(rep_.data(), rep_.size() - 8);
  }
};
```

#### 应用3：SSTable查找

```cpp
// table/table.cc
Iterator* Table::BlockReader(void* arg, const ReadOptions& options,
                              const Slice& index_value) {
  // index_value 直接引用索引块中的数据
  // 解析BlockHandle，无需拷贝
  BlockHandle handle;
  Slice input = index_value;
  Status s = handle.DecodeFrom(&input);  // input会移动指针
  // ...
}
```

### 1.4 Slice的最佳实践

**✅ 正确用法：**

```cpp
// 1. 栈上临时变量
void ProcessKey(const Slice& key) {
  if (key.starts_with("user:")) {
    Slice user_id = Slice(key.data() + 5, key.size() - 5);
    // 使用 user_id，离开作用域前key必须有效
  }
}

// 2. 立即使用
std::string buffer = GetData();
db->Put(WriteOptions(), "key", Slice(buffer));  // buffer立即使用

// 3. 确保生命周期
class Request {
  std::string key_buffer_;   // 拥有内存
  Slice key_;                // 引用key_buffer_

 public:
  Request(const std::string& k)
    : key_buffer_(k), key_(key_buffer_) {}
};
```

**❌ 错误用法：**

```cpp
// 1. 返回临时变量的Slice
Slice GetKey() {
  std::string temp = "key";
  return Slice(temp);  // ❌ temp销毁，返回悬空指针
}

// 2. 保存临时string的Slice
Slice saved_key;
{
  std::string temp = "key";
  saved_key = Slice(temp);  // ❌ temp销毁后saved_key无效
}
std::cout << saved_key.ToString();  // 未定义行为

// 3. 多线程不安全
std::string shared_buffer;
// 线程1修改shared_buffer
// 线程2使用Slice(shared_buffer)  // ❌ 数据竞争
```

## 2. Status：无异常的错误处理

### 2.1 为什么不用异常？

LevelDB禁用C++异常（`-fno-exceptions`）：

**原因：**
1. **性能开销**：异常会增加二进制大小和运行时开销
2. **可预测性**：数据库代码需要明确的错误路径
3. **C兼容性**：需要提供C语言API
4. **Google风格**：Google C++规范禁止异常

**Status模式 vs 异常：**

```cpp
// 异常方式（LevelDB不使用）
try {
  db->Put("key", "value");
  db->Get("key");
} catch (const IOException& e) {
  // 处理错误
}

// Status方式（LevelDB使用）
Status s = db->Put(WriteOptions(), "key", "value");
if (!s.ok()) {
  std::cerr << s.ToString() << std::endl;
  return;
}
```

### 2.2 Status的实现

#### 核心结构

```cpp
// include/leveldb/status.h
class Status {
 public:
  Status() noexcept : state_(nullptr) {}  // OK状态
  ~Status() { delete[] state_; }

  // 错误类型判断
  bool ok() const { return (state_ == nullptr); }
  bool IsNotFound() const { return code() == kNotFound; }
  bool IsCorruption() const { return code() == kCorruption; }
  bool IsIOError() const { return code() == kIOError; }
  bool IsInvalidArgument() const { return code() == kInvalidArgument; }

  // 错误消息
  std::string ToString() const;

 private:
  // state_ == nullptr: OK
  // state_ != nullptr: [0] = length of message, [1] = code, [2..] = message
  const char* state_;

  enum Code {
    kOk = 0,
    kNotFound = 1,
    kCorruption = 2,
    kNotSupported = 3,
    kInvalidArgument = 4,
    kIOError = 5
  };
};
```

#### 内存布局

```cpp
// util/status.cc, lines 18-30
// state_内存布局:
// state_[0..3]  = 消息长度 (uint32)
// state_[4]     = 错误码 (Code)
// state_[5..]   = 错误消息 (可选)

Status::Status(Code code, const Slice& msg, const Slice& msg2) {
  assert(code != kOk);
  const uint32_t len1 = static_cast<uint32_t>(msg.size());
  const uint32_t len2 = static_cast<uint32_t>(msg2.size());
  const uint32_t size = len1 + (len2 ? (2 + len2) : 0);
  char* result = new char[size + 5];
  std::memcpy(result, &size, sizeof(size));
  result[4] = static_cast<char>(code);
  std::memcpy(result + 5, msg.data(), len1);
  if (len2) {
    result[5 + len1] = ':';
    result[6 + len1] = ' ';
    std::memcpy(result + 7 + len1, msg2.data(), len2);
  }
  state_ = result;
}
```

**设计亮点：**
- OK状态零成本（`state_ = nullptr`）
- 拷贝构造只复制指针（浅拷贝，引用计数）
- 错误消息按需分配

### 2.3 Status的使用模式

#### 模式1：立即检查

```cpp
Status s = db->Put(WriteOptions(), key, value);
if (!s.ok()) {
  std::cerr << "Put failed: " << s.ToString() << std::endl;
  return s;  // 向上传播错误
}
```

#### 模式2：批量操作后检查

```cpp
Status s;
WriteBatch batch;
batch.Put("key1", "value1");
batch.Put("key2", "value2");
s = db->Write(WriteOptions(), &batch);
if (!s.ok()) {
  // 整个批次失败
}
```

#### 模式3：多种错误处理

```cpp
Status s = db->Get(ReadOptions(), key, &value);
if (s.ok()) {
  // 成功获取值
} else if (s.IsNotFound()) {
  // 键不存在，正常情况
  value = default_value;
} else if (s.IsCorruption()) {
  // 数据损坏，严重错误
  LOG(FATAL) << "Corruption: " << s.ToString();
} else {
  // 其他错误
  LOG(ERROR) << "Error: " << s.ToString();
}
```

#### 模式4：后台任务错误存储

```cpp
// db/db_impl.h, lines 174-206
class DBImpl : public DB {
 private:
  Status bg_error_;  // 后台任务遇到的错误

  void RecordBackgroundError(const Status& s) {
    mutex_.AssertHeld();
    if (bg_error_.ok()) {
      bg_error_ = s;
      background_work_finished_signal_.SignalAll();
    }
  }

  // 每次操作前检查后台错误
  Status Write(const WriteOptions& options, WriteBatch* updates) {
    MutexLock l(&mutex_);
    Status status = bg_error_;  // 检查后台错误
    if (!status.ok()) {
      return status;
    }
    // ...
  }
};
```

### 2.4 错误传播示例

```cpp
// 完整的错误处理链
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();

  Iterator* iter = mem->NewIterator();
  Status s;
  {
    mutex_.Unlock();
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }
  delete iter;

  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
      int level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
      edit->AddFile(level, meta.number, meta.file_size,
                    meta.smallest, meta.largest);
    }
  }

  return s;  // 返回错误或成功
}
```

## 3. Options：配置系统

### 3.1 Options结构

```cpp
// include/leveldb/options.h
struct Options {
  // Comparator
  const Comparator* comparator;  // 键比较器

  // 创建和错误处理
  bool create_if_missing = false;   // 不存在时创建
  bool error_if_exists = false;     // 存在时报错
  bool paranoid_checks = false;     // 严格检查

  // 环境和日志
  Env* env;                         // 平台抽象
  Logger* info_log = nullptr;       // 日志输出

  // 性能调优
  size_t write_buffer_size = 4 * 1024 * 1024;   // 4MB MemTable
  int max_open_files = 1000;                     // 文件句柄限制
  Cache* block_cache = nullptr;                  // 块缓存（8MB默认）
  size_t block_size = 4 * 1024;                  // 4KB 块大小
  int block_restart_interval = 16;               // 重启点间隔

  // 压缩
  CompressionType compression = kSnappyCompression;  // Snappy压缩

  // 过滤器
  const FilterPolicy* filter_policy = nullptr;  // 布隆过滤器

  // 高级选项
  int max_file_size = 2 * 1024 * 1024;  // 2MB SSTable大小
};
```

### 3.2 ReadOptions和WriteOptions

```cpp
// include/leveldb/options.h
struct ReadOptions {
  bool verify_checksums = false;  // 验证校验和
  bool fill_cache = true;         // 填充块缓存
  const Snapshot* snapshot = nullptr;  // 快照读
};

struct WriteOptions {
  bool sync = false;  // fsync保证持久化
};
```

**性能影响：**

```cpp
// 最快写入（可能丢失数据）
WriteOptions fast;
fast.sync = false;
db->Put(fast, key, value);  // ~100,000 ops/sec

// 安全写入（保证持久化）
WriteOptions safe;
safe.sync = true;
db->Put(safe, key, value);  // ~100 ops/sec (慢1000倍！)
```

### 3.3 配置示例

```cpp
// 生产环境配置
leveldb::Options options;
options.create_if_missing = true;
options.error_if_exists = false;
options.paranoid_checks = true;  // 严格检查

// 调大写缓冲以提高吞吐量
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB

// 设置块缓存
options.block_cache = leveldb::NewLRUCache(512 * 1024 * 1024);  // 512MB

// 启用布隆过滤器
options.filter_policy = leveldb::NewBloomFilterPolicy(10);  // 10 bits/key

// 打开数据库
leveldb::DB* db;
leveldb::Status status = leveldb::DB::Open(options, "/data/mydb", &db);
if (!status.ok()) {
  std::cerr << "Open failed: " << status.ToString() << std::endl;
  exit(1);
}

// 使用后清理
delete db;
delete options.block_cache;
delete options.filter_policy;
```

## 4. 代码实践

### 4.1 Slice性能测试

创建文件 `slice_benchmark.cc`:

```cpp
#include <chrono>
#include <iostream>
#include <string>
#include "leveldb/slice.h"

using namespace std;
using namespace leveldb;

void BenchmarkStringCopy() {
  const int N = 1000000;
  string source = "This is a test string for benchmarking";

  auto start = chrono::high_resolution_clock::now();
  for (int i = 0; i < N; i++) {
    string copy = source;  // 完整拷贝
    (void)copy;
  }
  auto end = chrono::high_resolution_clock::now();

  auto duration = chrono::duration_cast<chrono::milliseconds>(end - start);
  cout << "String copy: " << duration.count() << " ms\n";
}

void BenchmarkSliceRef() {
  const int N = 1000000;
  string source = "This is a test string for benchmarking";

  auto start = chrono::high_resolution_clock::now();
  for (int i = 0; i < N; i++) {
    Slice ref = source;  // 只复制指针和长度
    (void)ref;
  }
  auto end = chrono::high_resolution_clock::now();

  auto duration = chrono::duration_cast<chrono::milliseconds>(end - start);
  cout << "Slice reference: " << duration.count() << " ms\n";
}

int main() {
  BenchmarkStringCopy();
  BenchmarkSliceRef();
  return 0;
}
```

### 4.2 Status错误处理

创建文件 `status_example.cc`:

```cpp
#include <iostream>
#include "leveldb/db.h"
#include "leveldb/status.h"

using namespace leveldb;

Status ProcessKey(DB* db, const Slice& key) {
  std::string value;
  Status s = db->Get(ReadOptions(), key, &value);

  if (s.ok()) {
    std::cout << "Value: " << value << std::endl;
  } else if (s.IsNotFound()) {
    std::cout << "Key not found\n";
    return Status::OK();  // 不存在不算错误
  } else {
    return s;  // 传播其他错误
  }

  return Status::OK();
}

int main() {
  DB* db;
  Options options;
  options.create_if_missing = true;

  Status status = DB::Open(options, "/tmp/testdb", &db);
  if (!status.ok()) {
    std::cerr << "Failed to open database: "
              << status.ToString() << std::endl;
    return 1;
  }

  // 写入数据
  status = db->Put(WriteOptions(), "key1", "value1");
  if (!status.ok()) {
    std::cerr << "Put failed: " << status.ToString() << std::endl;
    delete db;
    return 1;
  }

  // 读取存在的键
  ProcessKey(db, "key1");

  // 读取不存在的键
  ProcessKey(db, "nonexistent");

  delete db;
  return 0;
}
```

## 5. 深入源码

### 5.1 Slice实现细节

位置：`include/leveldb/slice.h`

```cpp
class LEVELDB_EXPORT Slice {
 public:
  // 构造函数
  Slice() : data_(""), size_(0) {}
  Slice(const char* d, size_t n) : data_(d), size_(n) {}
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) {}
  Slice(const char* s) : data_(s), size_(strlen(s)) {}

  // 拷贝构造（默认，浅拷贝）
  Slice(const Slice&) = default;
  Slice& operator=(const Slice&) = default;

  // 访问器
  const char* data() const { return data_; }
  size_t size() const { return size_; }
  bool empty() const { return size_ == 0; }
  char operator[](size_t n) const {
    assert(n < size());
    return data_[n];
  }

  // 清空
  void clear() {
    data_ = "";
    size_ = 0;
  }

  // 移除前缀
  void remove_prefix(size_t n) {
    assert(n <= size());
    data_ += n;
    size_ -= n;
  }

  // 转换为string（拷贝）
  std::string ToString() const { return std::string(data_, size_); }

  // 比较
  int compare(const Slice& b) const;

  // 前缀匹配
  bool starts_with(const Slice& x) const {
    return ((size_ >= x.size_) && (memcmp(data_, x.data_, x.size_) == 0));
  }

 private:
  const char* data_;
  size_t size_;
};

// 全局比较运算符
inline bool operator==(const Slice& x, const Slice& y) {
  return ((x.size() == y.size()) &&
          (memcmp(x.data(), y.data(), x.size()) == 0));
}

inline bool operator!=(const Slice& x, const Slice& y) { return !(x == y); }
```

### 5.2 Status的ToString实现

位置：`util/status.cc`

```cpp
std::string Status::ToString() const {
  if (state_ == nullptr) {
    return "OK";
  } else {
    char tmp[30];
    const char* type;
    switch (code()) {
      case kOk:
        type = "OK";
        break;
      case kNotFound:
        type = "NotFound: ";
        break;
      case kCorruption:
        type = "Corruption: ";
        break;
      case kNotSupported:
        type = "Not implemented: ";
        break;
      case kInvalidArgument:
        type = "Invalid argument: ";
        break;
      case kIOError:
        type = "IO error: ";
        break;
      default:
        std::snprintf(tmp, sizeof(tmp),
                      "Unknown code(%d): ", static_cast<int>(code()));
        type = tmp;
        break;
    }
    std::string result(type);
    uint32_t length;
    std::memcpy(&length, state_, sizeof(length));
    result.append(state_ + 5, length);
    return result;
  }
}
```

## 6. 设计模式与最佳实践

### 6.1 零拷贝模式

**适用场景：**
- 数据只读访问
- 临时引用
- 性能敏感代码

**实现要点：**
```cpp
class ZeroCopyProcessor {
  std::string buffer_;  // 拥有内存

 public:
  Slice Process(const Slice& input) {
    // 1. 处理输入（不拷贝）
    if (input.starts_with("prefix:")) {
      // 2. 需要修改时才拷贝
      buffer_ = input.ToString();
      buffer_.erase(0, 7);  // 移除前缀
      return Slice(buffer_);
    }
    // 3. 无需修改直接返回
    return input;
  }
};
```

### 6.2 错误处理模式

**级联检查：**

```cpp
Status OpenDatabase(const std::string& path, DB** dbptr) {
  Options options;
  options.create_if_missing = true;

  Status s = DB::Open(options, path, dbptr);
  if (!s.ok()) return s;  // 立即返回错误

  // 执行初始化
  s = (*dbptr)->Put(WriteOptions(), "version", "1.0");
  if (!s.ok()) {
    delete *dbptr;
    *dbptr = nullptr;
    return s;
  }

  return Status::OK();
}
```

## 总结

今天我们学习了：
1. ✅ **Slice**：零拷贝字节数组引用，避免string拷贝开销
2. ✅ **Status**：无异常错误处理，OK状态零成本
3. ✅ **Options**：灵活的配置系统
4. ✅ 设计模式：零拷贝、错误传播、生命周期管理

**关键要点：**
- Slice只存储指针和长度（16字节），不管理内存
- Status通过nullptr表示OK，错误信息按需分配
- 所有LevelDB API都返回Status，必须检查

**思考题：**
1. Slice能否存储二进制数据（包含'\0'）？
2. 如果Status拷贝代价很低，为什么还要用引用传递？
3. 如何安全地在多线程中使用Slice？

**明天预告：Day 3 - SkipList无锁数据结构**
我们将学习LevelDB最核心的内存数据结构——SkipList的实现原理。
