# Day 5: SSTable文件格式与Block结构

## 学习目标
- 理解SSTable的文件格式和整体结构
- 掌握Block的前缀压缩编码
- 学习BlockHandle和Footer的作用
- 理解两级索引结构如何加速查找

## 1. SSTable概述

### 1.1 什么是SSTable?

**SSTable = Sorted String Table（有序字符串表）**

```
特性：
1. 不可变（Immutable）：一旦写入，永不修改
2. 有序（Sorted）：键按字典序排序
3. 持久化（Persistent）：存储在磁盘上
4. 压缩（Compressed）：使用Snappy/Zstd压缩
```

**在LevelDB中的作用：**

```
写入流程：
MemTable (4MB) → 满
  ↓
Immutable MemTable
  ↓
SSTable文件 (Level-0)
  ↓
Compaction → Level-1, Level-2...
```

### 1.2 为什么需要SSTable?

**问题1：内存有限**
```
解决：将数据持久化到磁盘
- MemTable只保留最近写入
- 历史数据存储在SSTable
```

**问题2：随机写入慢**
```
解决：批量顺序写入
- 累积多个写入到MemTable
- 一次性排序写入SSTable
```

**问题3：如何快速查找？**
```
解决：索引 + 过滤器
- 索引块：快速定位数据块
- Bloom过滤器：避免无效读取
```

## 2. SSTable文件格式

### 2.1 整体结构

```
SSTable文件布局（从前到后）:

┌────────────────────────────────────┐
│  Data Block 1                      │  ← 存储实际键值对
│  (键值对, 前缀压缩)                 │
├────────────────────────────────────┤
│  Data Block 2                      │
│  ...                               │
├────────────────────────────────────┤
│  Data Block N                      │
├────────────────────────────────────┤
│  Meta Block                        │  ← 元数据（如统计信息）
│  (可选)                            │
├────────────────────────────────────┤
│  Meta Index Block                  │  ← Meta块的索引
│  (指向Meta块的BlockHandle)         │
├────────────────────────────────────┤
│  Index Block                       │  ← 数据块索引
│  (每个Data Block的最大键+Handle)    │
├────────────────────────────────────┤
│  Footer (48字节，固定大小)          │  ← 文件元信息
│  - MetaIndex Handle (可变长)        │
│  - Index Handle (可变长)            │
│  - Padding (填充到40字节)           │
│  - Magic Number (8字节)             │
└────────────────────────────────────┘
```

**关键特性：**
- Footer固定大小：48字节（位于文件末尾）
- 从Footer开始反向解析文件
- 索引块和元索引块不压缩（便于快速访问）

### 2.2 读取流程

```cpp
// table/table.cc, lines 38-80
Status Table::Open(const Options& options, RandomAccessFile* file,
                   uint64_t size, Table** table) {
  if (size < Footer::kEncodedLength) {
    return Status::Corruption("file is too short");
  }

  // 1. 从文件末尾读取Footer
  char footer_space[Footer::kEncodedLength];
  Slice footer_input;
  Status s = file->Read(size - Footer::kEncodedLength,
                        Footer::kEncodedLength,
                        &footer_input, footer_space);

  // 2. 解析Footer
  Footer footer;
  s = footer.DecodeFrom(&footer_input);
  if (!s.ok()) return s;

  // 3. 读取Index Block
  BlockContents index_block_contents;
  ReadOptions opt;
  s = ReadBlock(file, opt, footer.index_handle(), &index_block_contents);

  // 4. 创建Table对象
  Block* index_block = new Block(index_block_contents);
  *table = new Table(rep);
  // ...
}
```

**查找流程：**

```
1. 读取Footer → 获取Index Block位置
2. 读取Index Block → 找到目标键所在的Data Block
3. 读取Data Block → 在块内查找键
4. 返回值
```

## 3. BlockHandle和Footer

### 3.1 BlockHandle定义

```cpp
// table/format.h, lines 21-44
class BlockHandle {
 public:
  enum { kMaxEncodedLength = 10 + 10 };  // 最大20字节

  uint64_t offset() const { return offset_; }
  void set_offset(uint64_t offset) { offset_ = offset; }

  uint64_t size() const { return size_; }
  void set_size(uint64_t size) { size_ = size; }

  void EncodeTo(std::string* dst) const;
  Status DecodeFrom(Slice* input);

 private:
  uint64_t offset_;  // 块在文件中的偏移（字节）
  uint64_t size_;    // 块的大小（字节，不含trailer）
};
```

**编码格式：**

```
BlockHandle编码（Varint64）：
┌────────────────┬────────────────┐
│  offset        │  size          │
│  (varint64)    │  (varint64)    │
└────────────────┴────────────────┘
   最多10字节       最多10字节

示例：
offset=1000, size=4096
编码：\xe8\x07 (1000的varint) + \x80\x20 (4096的varint)
实际长度：2 + 2 = 4字节
```

**实现：**

```cpp
// table/format.cc, lines 16-30
void BlockHandle::EncodeTo(std::string* dst) const {
  PutVarint64(dst, offset_);
  PutVarint64(dst, size_);
}

Status BlockHandle::DecodeFrom(Slice* input) {
  if (GetVarint64(input, &offset_) &&
      GetVarint64(input, &size_)) {
    return Status::OK();
  } else {
    return Status::Corruption("bad block handle");
  }
}
```

### 3.2 Footer结构

```cpp
// table/format.h, lines 46-71
class Footer {
 public:
  enum { kEncodedLength = 2 * BlockHandle::kMaxEncodedLength + 8 };

  const BlockHandle& metaindex_handle() const { return metaindex_handle_; }
  void set_metaindex_handle(const BlockHandle& h) { metaindex_handle_ = h; }

  const BlockHandle& index_handle() const { return index_handle_; }
  void set_index_handle(const BlockHandle& h) { index_handle_ = h; }

  void EncodeTo(std::string* dst) const;
  Status DecodeFrom(Slice* input);

 private:
  BlockHandle metaindex_handle_;
  BlockHandle index_handle_;
};

static const uint64_t kTableMagicNumber = 0xdb4775248b80fb57ull;
```

**Footer内存布局（48字节）：**

```
┌─────────────────────────────┐
│  MetaIndex Handle           │  可变长（最多20字节）
│  (offset + size)            │
├─────────────────────────────┤
│  Index Handle               │  可变长（最多20字节）
│  (offset + size)            │
├─────────────────────────────┤
│  Padding                    │  填充到40字节
│  (零填充)                   │
├─────────────────────────────┤
│  Magic Number               │  固定8字节
│  0xdb4775248b80fb57         │  (小端序)
└─────────────────────────────┘
总计：48字节
```

**编码实现：**

```cpp
// table/format.cc, lines 32-67
void Footer::EncodeTo(std::string* dst) const {
  const size_t original_size = dst->size();

  // 1. 编码两个BlockHandle
  metaindex_handle_.EncodeTo(dst);
  index_handle_.EncodeTo(dst);

  // 2. 填充到40字节
  dst->resize(original_size + 2 * BlockHandle::kMaxEncodedLength);

  // 3. 追加Magic Number
  PutFixed32(dst, static_cast<uint32_t>(kTableMagicNumber & 0xffffffffu));
  PutFixed32(dst, static_cast<uint32_t>(kTableMagicNumber >> 32));

  assert(dst->size() == original_size + kEncodedLength);
}

Status Footer::DecodeFrom(Slice* input) {
  if (input->size() < kEncodedLength) {
    return Status::Corruption("not an sstable (footer too short)");
  }

  // 1. 验证Magic Number
  const char* magic_ptr = input->data() + kEncodedLength - 8;
  const uint32_t magic_lo = DecodeFixed32(magic_ptr);
  const uint32_t magic_hi = DecodeFixed32(magic_ptr + 4);
  const uint64_t magic = ((static_cast<uint64_t>(magic_hi) << 32) |
                          (static_cast<uint64_t>(magic_lo)));
  if (magic != kTableMagicNumber) {
    return Status::Corruption("not an sstable (bad magic number)");
  }

  // 2. 解码BlockHandle
  Status result = metaindex_handle_.DecodeFrom(input);
  if (result.ok()) {
    result = index_handle_.DecodeFrom(input);
  }
  return result;
}
```

**为什么需要Magic Number？**

```
作用：
1. 文件完整性验证：确认是有效的SSTable文件
2. 防止误读：避免将其他文件当作SSTable
3. 版本标识：不同版本可用不同魔数

Magic Number选择：
- 随机值：0xdb4775248b80fb57
- 不太可能在普通数据中出现
- 大小端敏感（用于检测字节序）
```

## 4. Data Block结构

### 4.1 前缀压缩原理

**为什么需要压缩？**

```
场景：存储相似的键
user:1001:name
user:1001:age
user:1001:email
user:1002:name
user:1002:age

未压缩存储：5个键 * 平均14字节 = 70字节

前缀压缩：
user:1001:name    (完整，14字节)
         :age     (共享10字节，只存储4字节)
         :email   (共享10字节，只存储6字节)
user:1002:name    (共享5字节，存储9字节)
         :age     (共享14字节，只存储4字节)

压缩后：14+4+6+9+4 = 37字节，节省47%！
```

**前缀压缩格式：**

```
每个条目格式：
┌──────────────┬────────────────┬──────────────┬────────────┬─────────┐
│ shared       │ non_shared     │ value_len    │ key_delta  │ value   │
│ (varint32)   │ (varint32)     │ (varint32)   │ [non_shared]│[value_len]│
└──────────────┴────────────────┴──────────────┴────────────┴─────────┘

shared: 与前一个键共享的字节数
non_shared: 键的非共享部分长度
value_len: 值的长度
key_delta: 键的差异部分
value: 值数据
```

**示例：**

```
条目1: key="apple", value="fruit"
编码: [0][5]["apple"][5]["fruit"]
      ↑ 第一个键，shared=0

条目2: key="application", value="software"
编码: [3][8]["lication"][8]["software"]
      ↑ 共享"app"

条目3: key="apply", value="verb"
编码: [3][2]["ly"][4]["verb"]
      ↑ 共享"app"
```

### 4.2 重启点（Restart Points）

**问题：前缀压缩的缺点**

```
查找"application"：
1. 从头开始：apple → application（需要解码前面的键）
2. 无法跳过中间键
3. 查找效率降低到O(N)

解决方案：重启点
- 每K个键（默认16）创建一个重启点
- 重启点处的键完整存储（shared=0）
- 重启点数组支持二分查找
```

**Block尾部结构：**

```
┌─────────────────────────────────────┐
│ Entry 0 (restart point)             │  shared=0
│ Entry 1                             │  shared>0
│ Entry 2                             │  shared>0
│ ...                                 │
│ Entry 15                            │  shared>0
├─────────────────────────────────────┤
│ Entry 16 (restart point)            │  shared=0
│ Entry 17                            │  shared>0
│ ...                                 │
├─────────────────────────────────────┤
│ Restart[0] = 0 (uint32)             │  第0个重启点偏移
│ Restart[1] = offset1 (uint32)       │  第16个条目偏移
│ Restart[2] = offset2 (uint32)       │  第32个条目偏移
│ ...                                 │
│ Restart[N-1] = offsetN (uint32)     │
├─────────────────────────────────────┤
│ Num Restarts = N (uint32)           │  重启点数量
└─────────────────────────────────────┘
```

**查找算法：**

```cpp
// table/block.cc, lines 164-227
void Seek(const Slice& target) {
  // 1. 二分查找重启点数组
  uint32_t left = 0;
  uint32_t right = num_restarts_ - 1;
  while (left < right) {
    uint32_t mid = (left + right + 1) / 2;
    uint32_t region_offset = GetRestartPoint(mid);

    // 解码mid处的键
    const char* key_ptr;
    uint32_t shared, non_shared, value_length;
    ParseKey(region_offset, &shared, &non_shared, &value_length);
    Slice mid_key(key_ptr, non_shared);  // shared必定为0

    if (Compare(mid_key, target) < 0) {
      left = mid;
    } else {
      right = mid - 1;
    }
  }

  // 2. 从重启点开始线性扫描
  SeekToRestartPoint(left);
  while (true) {
    if (!ParseNextKey()) return;
    if (Compare(key_, target) >= 0) return;
  }
}
```

**时间复杂度：**

```
二分查找重启点：O(log(N/16))
线性扫描：O(16)
总计：O(log N)

对比：
- 无重启点：O(N)
- 重启点间隔16：O(log N + 16) ≈ O(log N)
```

## 5. BlockBuilder实现

### 5.1 类定义

```cpp
// table/block_builder.h
class BlockBuilder {
 public:
  explicit BlockBuilder(const Options* options);

  void Reset();
  void Add(const Slice& key, const Slice& value);
  Slice Finish();
  size_t CurrentSizeEstimate() const;
  bool empty() const { return buffer_.empty(); }

 private:
  const Options* options_;
  std::string buffer_;               // 数据缓冲区
  std::vector<uint32_t> restarts_;   // 重启点偏移数组
  int counter_;                      // 自上次重启以来的条目数
  bool finished_;                    // Finish()是否已调用
  std::string last_key_;             // 最后添加的键
};
```

### 5.2 Add()实现

```cpp
// table/block_builder.cc, lines 71-105
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);
  assert(!finished_);
  assert(counter_ <= options_->block_restart_interval);
  assert(buffer_.empty() || options_->comparator->Compare(key, last_key_piece) > 0);

  size_t shared = 0;

  // 1. 判断是否需要重启
  if (counter_ < options_->block_restart_interval) {
    // 计算与上一键的公共前缀
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
    // 达到重启间隔，记录重启点
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }

  const size_t non_shared = key.size() - shared;

  // 2. 编码：shared, non_shared, value_length
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

  // 3. 追加key的非共享部分和value
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());

  // 4. 更新状态
  last_key_.resize(shared);
  last_key_.append(key.data() + shared, non_shared);
  assert(Slice(last_key_) == key);
  counter_++;
}
```

**Add()可视化：**

```
初始状态：
buffer_ = ""
last_key_ = ""
counter_ = 0

Add("apple", "fruit"):
  counter_ = 0 → 重启点
  restarts_ = [0]
  编码: [0][5][5]["apple"]["fruit"]
  last_key_ = "apple"
  counter_ = 1

Add("application", "software"):
  counter_ = 1 < 16 → 不重启
  shared = 3 ("app")
  编码: [3][8][8]["lication"]["software"]
  last_key_ = "application"
  counter_ = 2

Add("banana", "fruit"):
  counter_ = 2 < 16 → 不重启
  shared = 0 (无公共前缀)
  编码: [0][6][5]["banana"]["fruit"]
  last_key_ = "banana"
  counter_ = 3
```

### 5.3 Finish()实现

```cpp
// table/block_builder.cc, lines 107-128
Slice BlockBuilder::Finish() {
  // 1. 追加重启点数组
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }

  // 2. 追加重启点数量
  PutFixed32(&buffer_, restarts_.size());

  finished_ = true;
  return Slice(buffer_);
}
```

**最终Block格式：**

```
┌────────────────────────────────────┐
│  Entry 0                           │
│  Entry 1                           │
│  ...                               │
│  Entry N-1                         │
├────────────────────────────────────┤
│  Restart[0]    (uint32, 小端序)     │
│  Restart[1]    (uint32, 小端序)     │
│  ...                               │
│  Restart[M-1]  (uint32, 小端序)     │
├────────────────────────────────────┤
│  Num Restarts  (uint32, 小端序)     │
└────────────────────────────────────┘
```

### 5.4 大小估算

```cpp
// table/block_builder.cc, lines 55-59
size_t BlockBuilder::CurrentSizeEstimate() const {
  return (buffer_.size() +                      // 原始数据
          restarts_.size() * sizeof(uint32_t) + // 重启数组
          sizeof(uint32_t));                    // 数量字段
}
```

## 6. Block读取

### 6.1 Block类

```cpp
// table/block.h
class Block {
 public:
  explicit Block(const BlockContents& contents);
  ~Block();

  size_t size() const { return size_; }
  Iterator* NewIterator(const Comparator* comparator);

 private:
  uint32_t NumRestarts() const;

  const char* data_;           // 块数据
  size_t size_;                // 块大小
  uint32_t restart_offset_;    // 重启数组在data_中的偏移
  bool owned_;                 // 是否拥有data_内存
};
```

**NumRestarts()实现：**

```cpp
// table/block.cc, lines 20-23
uint32_t Block::NumRestarts() const {
  assert(size_ >= sizeof(uint32_t));
  return DecodeFixed32(data_ + size_ - sizeof(uint32_t));
}
```

### 6.2 Block迭代器

```cpp
// table/block.cc, lines 77-290
class Block::Iter : public Iterator {
 private:
  const Comparator* const comparator_;
  const char* const data_;       // 块数据
  uint32_t const restarts_;      // 重启数组偏移
  uint32_t const num_restarts_;  // 重启点数量

  uint32_t current_;             // 当前条目偏移（data_内）
  uint32_t restart_index_;       // 当前重启点索引
  std::string key_;              // 当前完整键
  Slice value_;                  // 当前值
  Status status_;

 public:
  bool Valid() const override { return current_ < restarts_; }

  void Seek(const Slice& target) override {
    // 二分查找 + 线性扫描
  }

  void Next() override {
    ParseNextKey();
  }

  Slice key() const override { return key_; }
  Slice value() const override { return value_; }
};
```

**ParseNextKey()实现：**

```cpp
// table/block.cc, lines 118-152
bool ParseNextKey() {
  current_ = NextEntryOffset();
  const char* p = data_ + current_;
  const char* limit = data_ + restarts_;

  if (p >= limit) {
    current_ = restarts_;
    restart_index_ = num_restarts_;
    return false;
  }

  // 解码shared, non_shared, value_length
  uint32_t shared, non_shared, value_length;
  p = DecodeEntry(p, limit, &shared, &non_shared, &value_length);

  if (p == nullptr || key_.size() < shared) {
    CorruptionError();
    return false;
  } else {
    // 重建完整键
    key_.resize(shared);
    key_.append(p, non_shared);
    value_ = Slice(p + non_shared, value_length);

    // 更新restart_index_
    while (restart_index_ + 1 < num_restarts_ &&
           GetRestartPoint(restart_index_ + 1) < current_) {
      ++restart_index_;
    }
    return true;
  }
}
```

**解码条目：**

```cpp
// table/block.cc, lines 97-116
static inline const char* DecodeEntry(const char* p, const char* limit,
                                      uint32_t* shared,
                                      uint32_t* non_shared,
                                      uint32_t* value_length) {
  if (limit - p < 3) return nullptr;  // 至少需要3字节

  *shared = reinterpret_cast<const uint8_t*>(p)[0];
  *non_shared = reinterpret_cast<const uint8_t*>(p)[1];
  *value_length = reinterpret_cast<const uint8_t*>(p)[2];

  if ((*shared | *non_shared | *value_length) < 128) {
    // 快速路径：所有值都小于128（1字节varint）
    p += 3;
  } else {
    // 慢速路径：使用完整varint解码
    if ((p = GetVarint32Ptr(p, limit, shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, non_shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, value_length)) == nullptr) return nullptr;
  }

  if (static_cast<uint32_t>(limit - p) < (*non_shared + *value_length)) {
    return nullptr;
  }
  return p;
}
```

## 7. Block Trailer

### 7.1 Trailer结构

```cpp
// table/format.h, line 79
static const size_t kBlockTrailerSize = 5;  // 1字节类型 + 4字节CRC
```

**格式：**

```
Block in file:
┌────────────────────────────────────┐
│  Block Content                     │  size字节
│  (压缩后的数据)                     │
├────────────────────────────────────┤
│  Type (1 byte)                     │  压缩类型
│  - 0x0: kNoCompression             │
│  - 0x1: kSnappyCompression         │
│  - 0x2: kZstdCompression           │
├────────────────────────────────────┤
│  CRC32C (4 bytes)                  │  校验和（掩码处理）
│  (对content + type计算)             │
└────────────────────────────────────┘
总计：size + 5字节
```

### 7.2 写入Trailer

```cpp
// table/table_builder.cc, lines 192-209
void WriteRawBlock(const Slice& block_contents,
                   CompressionType type,
                   BlockHandle* handle) {
  handle->set_offset(r->offset);
  handle->set_size(block_contents.size());

  // 1. 写入块内容
  r->status = r->file->Append(block_contents);

  if (r->status.ok()) {
    // 2. 构造trailer
    char trailer[kBlockTrailerSize];
    trailer[0] = type;

    uint32_t crc = crc32c::Value(block_contents.data(), block_contents.size());
    crc = crc32c::Extend(crc, trailer, 1);  // 包含type字节
    crc = crc32c::Mask(crc);
    EncodeFixed32(trailer + 1, crc);

    // 3. 写入trailer
    r->status = r->file->Append(Slice(trailer, kBlockTrailerSize));
    if (r->status.ok()) {
      r->offset += block_contents.size() + kBlockTrailerSize;
    }
  }
}
```

**CRC32C掩码：**

```cpp
// util/crc32c.h
inline uint32_t Mask(uint32_t crc) {
  // 旋转CRC并添加常量，防止零CRC
  return ((crc >> 15) | (crc << 17)) + 0xa282ead8ul;
}

inline uint32_t Unmask(uint32_t masked_crc) {
  uint32_t rot = masked_crc - 0xa282ead8ul;
  return ((rot >> 17) | (rot << 15));
}
```

**为什么需要掩码？**

```
原因：
1. 防止零CRC：空数据的CRC=0，掩码后非零
2. 检测实现错误：如果代码忘记计算CRC，不会碰巧得到正确的掩码值
3. 增加随机性：相似的数据有不同的掩码值
```

### 7.3 读取和校验

```cpp
// table/format.cc, lines 69-162
Status ReadBlock(RandomAccessFile* file,
                 const ReadOptions& options,
                 const BlockHandle& handle,
                 BlockContents* result) {
  // 1. 读取块内容 + trailer
  size_t n = static_cast<size_t>(handle.size());
  char* buf = new char[n + kBlockTrailerSize];
  Slice contents;
  Status s = file->Read(handle.offset(), n + kBlockTrailerSize,
                        &contents, buf);

  if (s.ok()) {
    // 2. 提取type和CRC
    const char* data = contents.data();
    if (contents.size() != n + kBlockTrailerSize) {
      delete[] buf;
      return Status::Corruption("truncated block read");
    }

    // 3. 验证CRC（如果需要）
    if (options.verify_checksums) {
      const uint32_t crc = crc32c::Unmask(DecodeFixed32(data + n + 1));
      const uint32_t actual = crc32c::Value(data, n + 1);
      if (actual != crc) {
        delete[] buf;
        return Status::Corruption("block checksum mismatch");
      }
    }

    // 4. 根据类型解压缩
    switch (data[n]) {
      case kNoCompression:
        // 直接使用
        result->data = Slice(data, n);
        result->heap_allocated = true;
        result->cachable = false;
        break;

      case kSnappyCompression: {
        size_t ulength = 0;
        if (!port::Snappy_GetUncompressedLength(data, n, &ulength)) {
          delete[] buf;
          return Status::Corruption("corrupted compressed block");
        }
        char* ubuf = new char[ulength];
        if (!port::Snappy_Uncompress(data, n, ubuf)) {
          delete[] buf;
          delete[] ubuf;
          return Status::Corruption("corrupted compressed block");
        }
        delete[] buf;
        result->data = Slice(ubuf, ulength);
        result->heap_allocated = true;
        result->cachable = true;
        break;
      }

      // Zstd类似...
    }
  }

  return s;
}
```

## 8. Index Block

### 8.1 索引条目格式

```cpp
// table/table_builder.cc, lines 94-123

Index Block条目：
key: 分隔键（>= 当前Data Block的最大键，< 下一个Data Block的最小键）
value: BlockHandle（编码为varint64）

示例：
Data Block 1: keys = [apple, banana, cherry]
Data Block 2: keys = [dog, elephant, fox]

Index Block:
Entry 1: key="cherry", value=BlockHandle(offset=0, size=4096)
Entry 2: key="fox", value=BlockHandle(offset=4096, size=3584)

优化：使用FindShortestSeparator找到最短的分隔键
- "cherry" 和 "dog" 之间 → "d" （更短！）
- 减少索引块大小
```

**FindShortestSeparator优化：**

```cpp
// util/comparator.cc, lines 55-77
void BytewiseComparator::FindShortestSeparator(
    std::string* start,
    const Slice& limit) const {
  // 找到第一个不同的字节
  size_t min_length = std::min(start->size(), limit.size());
  size_t diff_index = 0;
  while ((diff_index < min_length) &&
         ((*start)[diff_index] == limit[diff_index])) {
    diff_index++;
  }

  if (diff_index >= min_length) {
    // start是limit的前缀，无法缩短
  } else {
    uint8_t diff_byte = static_cast<uint8_t>((*start)[diff_index]);
    if (diff_byte < static_cast<uint8_t>(0xff) &&
        diff_byte + 1 < static_cast<uint8_t>(limit[diff_index])) {
      // 可以缩短
      (*start)[diff_index]++;
      start->resize(diff_index + 1);
    }
  }
}

示例：
start="hello", limit="world"
  diff_index=0, 'h'!=('w'), 'h'+1='i' < 'w'
  结果："i"

start="apple", limit="banana"
  diff_index=0, 'a'!='b', 'a'+1='b' = 'b'（不满足<）
  结果："apple"（不变）
```

### 8.2 两级迭代器

```cpp
// table/table.cc, lines 223-308
Iterator* Table::NewIterator(const ReadOptions& options) const {
  return NewTwoLevelIterator(
      rep_->index_block->NewIterator(rep_->options.comparator),
      &Table::BlockReader,  // 读取Data Block的函数
      const_cast<Table*>(this),
      options);
}

// 两级迭代器逻辑
class TwoLevelIterator : public Iterator {
  Iterator* index_iter_;   // 索引块迭代器（第一级）
  Iterator* data_iter_;    // 数据块迭代器（第二级）

  void Seek(const Slice& target) {
    // 1. 在索引块中查找
    index_iter_->Seek(target);

    // 2. 加载对应的数据块
    InitDataBlock();

    // 3. 在数据块中查找
    if (data_iter_ != nullptr) {
      data_iter_->Seek(target);
    }
  }
};
```

**查找流程可视化：**

```
假设查找key="dog"

1. Index Block:
   ["cherry", BlockHandle(0, 4096)]
   ["fox", BlockHandle(4096, 3584)]  ← Seek("dog")定位到这里

2. 读取Data Block 2 (offset=4096, size=3584)

3. Data Block 2迭代器:
   ["dog", "canine"]  ← 找到！
   ["elephant", "mammal"]
   ["fox", "animal"]
```

## 9. 压缩

### 9.1 压缩策略

```cpp
// table/table_builder.cc, lines 141-190
void TableBuilder::WriteBlock(BlockBuilder* block, BlockHandle* handle) {
  Slice raw = block->Finish();
  Slice block_contents;
  CompressionType type = r->options.compression;

  switch (type) {
    case kNoCompression:
      block_contents = raw;
      break;

    case kSnappyCompression: {
      std::string* compressed = &r->compressed_output;
      if (port::Snappy_Compress(raw.data(), raw.size(), compressed) &&
          compressed->size() < raw.size() - (raw.size() / 8u)) {
        // 压缩率 > 12.5%，使用压缩版本
        block_contents = *compressed;
      } else {
        // 压缩效果不好，使用原始数据
        block_contents = raw;
        type = kNoCompression;
      }
      break;
    }

    // Zstd类似...
  }

  WriteRawBlock(block_contents, type, handle);
  r->compressed_output.clear();
  block->Reset();
}
```

**压缩决策：**

```
压缩条件：
compressed_size < raw_size - (raw_size / 8)
即：压缩后大小 < 原始大小 * 87.5%

原因：
1. 避免负优化：压缩效果不好时反而增加CPU开销
2. 解压开销：压缩率太低不值得
3. 经验值：12.5%是合理的阈值
```

### 9.2 Snappy特性

```
Snappy压缩特点：
1. 速度极快：~250 MB/s压缩，~500 MB/s解压
2. 压缩率适中：典型2-4倍
3. 无需额外内存：流式处理
4. 开销固定：适合小块压缩

对比：
Snappy: 快速，压缩率中等，LevelDB默认
Zstd: 较慢，压缩率高，可配置级别
Gzip: 慢，压缩率高，不适合LevelDB

选择Snappy的原因：
- 读取路径敏感：解压必须快
- 块很小（4KB）：不需要高压缩率
- CPU友好：减少阻塞时间
```

## 10. 完整示例

### 10.1 构建SSTable

```cpp
#include "leveldb/table_builder.h"
#include "leveldb/env.h"

void BuildSSTable() {
  leveldb::Options options;
  options.compression = leveldb::kSnappyCompression;

  // 1. 创建文件
  leveldb::WritableFile* file;
  leveldb::Env::Default()->NewWritableFile("/tmp/test.sst", &file);

  // 2. 创建TableBuilder
  leveldb::TableBuilder builder(options, file);

  // 3. 添加键值对（必须有序！）
  builder.Add("apple", "fruit");
  builder.Add("banana", "fruit");
  builder.Add("carrot", "vegetable");
  builder.Add("dog", "animal");

  // 4. 完成构建
  leveldb::Status s = builder.Finish();
  if (!s.ok()) {
    std::cerr << "Build failed: " << s.ToString() << std::endl;
  }

  // 5. 关闭文件
  file->Close();
  delete file;

  std::cout << "SSTable created:\n";
  std::cout << "  Entries: " << builder.NumEntries() << "\n";
  std::cout << "  Size: " << builder.FileSize() << " bytes\n";
}
```

### 10.2 读取SSTable

```cpp
#include "leveldb/table.h"
#include "leveldb/env.h"

void ReadSSTable() {
  leveldb::Options options;

  // 1. 打开文件
  leveldb::RandomAccessFile* file;
  leveldb::Env::Default()->NewRandomAccessFile("/tmp/test.sst", &file);

  uint64_t file_size;
  leveldb::Env::Default()->GetFileSize("/tmp/test.sst", &file_size);

  // 2. 打开Table
  leveldb::Table* table;
  leveldb::Status s = leveldb::Table::Open(options, file, file_size, &table);
  if (!s.ok()) {
    std::cerr << "Open failed: " << s.ToString() << std::endl;
    return;
  }

  // 3. 创建迭代器
  leveldb::ReadOptions read_opts;
  leveldb::Iterator* iter = table->NewIterator(read_opts);

  // 4. 遍历所有条目
  for (iter->SeekToFirst(); iter->Valid(); iter->Next()) {
    std::cout << iter->key().ToString() << " : "
              << iter->value().ToString() << std::endl;
  }

  // 5. 查找特定键
  iter->Seek("dog");
  if (iter->Valid()) {
    std::cout << "\nFound: " << iter->key().ToString()
              << " = " << iter->value().ToString() << std::endl;
  }

  delete iter;
  delete table;
  delete file;
}
```

## 11. 性能分析

### 11.1 空间效率

```
示例数据：
- 1000个键值对
- 平均键长：20字节
- 平均值长：100字节
- 总数据：120KB

未压缩SSTable：
- 数据块：120KB
- 前缀压缩：节省~30% → 84KB
- 索引块：1KB（每块4KB，需要~21个索引条目）
- Footer：48字节
- 总计：~85KB

Snappy压缩SSTable：
- 数据块压缩：84KB → ~30KB（压缩率~2.8x）
- 索引块不压缩：1KB
- 总计：~31KB

对比原始数据（120KB）：
- 压缩SSTable节省74%空间
```

### 11.2 读取性能

```
随机读取一个键：

1. 读取Footer：1次I/O（48字节）
2. 读取Index Block：1次I/O（~1KB）
3. 二分查找索引：O(log N)，N=索引条目数
4. 读取Data Block：1次I/O（~4KB压缩）
5. 解压Data Block：~1ms
6. 二分查找重启点：O(log M)，M=16
7. 线性扫描：O(16)

总计：
- I/O：3次（~5KB）
- CPU：解压1ms + 查找<1ms
- 总延迟：~2-5ms（取决于磁盘）

优化后（使用缓存）：
- Index Block缓存：省1次I/O
- Data Block缓存：省1次I/O + 解压
- 最快：<1ms（纯内存操作）
```

### 11.3 Block大小权衡

```
Block Size = 4KB（默认）：

优点：
- 缓存友好：L2 cache通常256KB，可缓存60+块
- 压缩快：4KB数据Snappy压缩<0.1ms
- 索引小：1000个键 → 21个索引条目（~1KB索引）

缺点：
- 索引开销：索引块占总大小~3%
- 小文件开销：对小SSTable浪费

Block Size = 64KB：

优点：
- 索引更小：索引块占总大小<1%
- 压缩率更高：更多上下文

缺点：
- 缓存命中率低：每个块占用更多缓存
- 解压慢：64KB解压~1-2ms
- 读放大：读1个键需解压64KB

结论：
- 4KB适合随机读多的场景（LevelDB默认）
- 64KB适合顺序扫描多的场景
```

## 总结

今天我们学习了：
1. ✅ **SSTable结构**：Data Blocks + Index Block + Footer
2. ✅ **前缀压缩**：节省空间，每16个键设重启点
3. ✅ **BlockHandle**：指向块的offset+size，varint编码
4. ✅ **Footer**：48字节固定大小，包含Index Handle和Magic Number
5. ✅ **两级索引**：Index Block → Data Block，O(log N)查找
6. ✅ **压缩**：Snappy压缩，压缩率>12.5%才使用

**关键要点：**
- SSTable不可变，一次写入永不修改
- 前缀压缩+重启点平衡空间和性能
- 两级索引支持高效查找
- Footer固定在文件末尾，作为解析入口

**思考题：**
1. 为什么Index Block不压缩？
2. 如果重启点间隔改为1，会怎样？改为1000呢？
3. Magic Number放在文件开头还是结尾更好？为什么？

**明天预告：Day 6 - Write-Ahead Log (WAL)**
我们将学习LevelDB如何通过WAL保证数据持久性和崩溃恢复。
