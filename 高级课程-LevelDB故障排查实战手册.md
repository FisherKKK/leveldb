# LevelDB故障排查实战手册

## 🎯 课程目标

本手册将教你成为故障排查专家，掌握：
- 使用Valgrind/ASan检测内存问题
- 使用GDB分析崩溃和coredump
- 使用TSan检测数据竞争和死锁
- 诊断性能突降和数据损坏
- 使用strace/ltrace追踪系统调用

---

## 目录

1. [内存问题诊断](#1-内存问题诊断)
2. [崩溃调试](#2-崩溃调试)
3. [死锁分析](#3-死锁分析)
4. [数据损坏排查](#4-数据损坏排查)
5. [性能突降诊断](#5-性能突降诊断)
6. [系统调用追踪](#6-系统调用追踪)

---

## 1. 内存问题诊断

### 1.1 Valgrind - 内存错误检测

**安装Valgrind：**
```bash
sudo apt-get install valgrind
```

**基本用法：**
```bash
# 编译调试版本（带符号信息）
cd /home/dev/leveldb
mkdir -p build-debug && cd build-debug
cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_FLAGS="-g -O0" ..
cmake --build .

# 运行Valgrind
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         --verbose \
         --log-file=valgrind-out.txt \
         ./db_bench --benchmarks=fillrandom --num=10000

# 查看结果
cat valgrind-out.txt
```

**典型内存问题案例：**

**案例1：内存泄漏**
```bash
# valgrind-out.txt
==12345== LEAK SUMMARY:
==12345==    definitely lost: 4,096 bytes in 1 blocks
==12345==    indirectly lost: 0 bytes in 0 blocks
==12345==      possibly lost: 0 bytes in 0 blocks
==12345==
==12345== 4,096 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x4C2E0EF: operator new(unsigned long) (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==12345==    by 0x5B8C29: leveldb::DBImpl::Open(leveldb::Options const&, std::string const&, leveldb::DB**) (db_impl.cc:1254)
==12345==    by 0x40123A: main (db_bench.cc:456)
```

问题定位：
```cpp
// db/db_impl.cc:1254
Status DBImpl::Open(const Options& options, const std::string& dbname, DB** dbptr) {
  // ...
  VersionSet* versions = new VersionSet(dbname, &options, table_cache, &internal_comparator);
  // 错误：如果后续失败，versions没有被delete

  if (!s.ok()) {
    return s;  // 泄漏！
  }
  // ...
}

// 修复：
Status DBImpl::Open(...) {
  // ...
  std::unique_ptr<VersionSet> versions(new VersionSet(...));

  if (!s.ok()) {
    return s;  // 自动释放
  }
  // ...
  impl->versions_ = versions.release();  // 转移所有权
}
```

**案例2：Use-After-Free**
```bash
# valgrind-out.txt
==12345== Invalid read of size 8
==12345==    at 0x5B8C45: leveldb::MemTable::Get(leveldb::LookupKey const&, std::string*, leveldb::Status*) (memtable.cc:89)
==12345==    by 0x5B7A23: leveldb::DBImpl::Get(leveldb::ReadOptions const&, leveldb::Slice const&, std::string*) (db_impl.cc:1156)
==12345==  Address 0x6d2e040 is 0 bytes inside a block of size 64 free'd
==12345==    at 0x4C2F24B: operator delete(void*) (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==12345==    by 0x5B9234: leveldb::MemTable::Unref() (memtable.cc:52)
==12345==    by 0x5B7A12: leveldb::DBImpl::Get(...) (db_impl.cc:1155)
```

问题代码：
```cpp
// 错误代码
Status DBImpl::Get(const ReadOptions& options, const Slice& key, std::string* value) {
  MutexLock l(&mutex_);
  MemTable* mem = mem_;
  mem->Ref();

  mutex_.Unlock();

  // 在这里，另一个线程可能delete了mem
  bool found = mem->Get(lkey, value, &s);  // Use-After-Free!

  mutex_.Lock();
  mem->Unref();
  return s;
}

// 正确做法：在Unref之前保持引用
Status DBImpl::Get(...) {
  MutexLock l(&mutex_);
  MemTable* mem = mem_;
  mem->Ref();

  {
    mutex_.Unlock();
    bool found = mem->Get(lkey, value, &s);  // 安全：仍持有引用
    mutex_.Lock();
  }

  mem->Unref();  // 在这里才释放
  return s;
}
```

### 1.2 AddressSanitizer (ASan) - 快速内存检测

ASan比Valgrind快10-100倍，适合频繁测试。

**编译启用ASan：**
```bash
cd /home/dev/leveldb
mkdir -p build-asan && cd build-asan
cmake -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_CXX_FLAGS="-g -O1 -fsanitize=address -fno-omit-frame-pointer" \
      -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address" \
      ..
cmake --build .

# 运行
./db_bench --benchmarks=fillrandom --num=10000
```

**ASan检测到的问题示例：**

**案例：堆缓冲区溢出**
```bash
# ASan输出
=================================================================
==23456==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x603000000018 at pc 0x00000051a2b4
WRITE of size 4 at 0x603000000018 thread T0
    #0 0x51a2b3 in leveldb::WriteBatchInternal::SetSequence(leveldb::WriteBatch*, unsigned long) db/write_batch.cc:45
    #1 0x4f23a1 in leveldb::DBImpl::Write(leveldb::WriteOptions const&, leveldb::WriteBatch*) db/db_impl.cc:1234
    #2 0x401234 in main db_bench.cc:567

0x603000000018 is located 0 bytes to the right of 8-byte region [0x603000000010,0x603000000018)
allocated by thread T0 here:
    #0 0x7f8b2d4e5b40 in __interceptor_malloc (/usr/lib/x86_64-linux-gnu/libasan.so.4+0xdeb40)
    #1 0x4e12c3 in leveldb::WriteBatch::WriteBatch() db/write_batch.cc:28
```

问题代码：
```cpp
// db/write_batch.cc
void WriteBatchInternal::SetSequence(WriteBatch* b, SequenceNumber seq) {
  // WriteBatch内部结构：
  // [0-7]: sequence number (8 bytes)
  // [8-11]: count (4 bytes)
  // [12+]: records

  // 错误：假设rep_至少有12字节，但初始化时可能只有8字节
  EncodeFixed64(&b->rep_[0], seq);
  // 如果rep_.size() == 8，这里写入会越界4字节
}

// 修复：
void WriteBatchInternal::SetSequence(WriteBatch* batch, SequenceNumber seq) {
  assert(batch->rep_.size() >= kHeader);  // kHeader = 12
  EncodeFixed64(&batch->rep_[0], seq);
}

// WriteBatch构造时确保大小
WriteBatch::WriteBatch() {
  Clear();
}

void WriteBatch::Clear() {
  rep_.clear();
  rep_.resize(kHeader);  // 确保至少12字节
}
```

### 1.3 常见内存问题模式

**模式1：迭代器失效**
```cpp
// 错误：迭代器失效
std::vector<int> vec = {1, 2, 3, 4, 5};
for (auto it = vec.begin(); it != vec.end(); ++it) {
  if (*it == 3) {
    vec.erase(it);  // it失效！
  }
}

// 正确：
for (auto it = vec.begin(); it != vec.end(); ) {
  if (*it == 3) {
    it = vec.erase(it);  // 返回下一个有效迭代器
  } else {
    ++it;
  }
}
```

**模式2：双重释放**
```cpp
// 错误：双重释放
class MyClass {
 public:
  ~MyClass() {
    delete[] data_;
  }

  MyClass(const MyClass& other) {
    data_ = other.data_;  // 浅拷贝
    size_ = other.size_;
  }

 private:
  char* data_;
  size_t size_;
};

void Foo() {
  MyClass a;
  MyClass b = a;  // 拷贝
}  // a和b的析构函数都会delete同一个data_！

// 正确：实现深拷贝或禁用拷贝
class MyClass {
 public:
  MyClass(const MyClass&) = delete;
  MyClass& operator=(const MyClass&) = delete;

  // 或者实现深拷贝
  MyClass(const MyClass& other) {
    size_ = other.size_;
    data_ = new char[size_];
    memcpy(data_, other.data_, size_);
  }
};
```

**模式3：栈缓冲区溢出**
```cpp
// 错误：栈缓冲区溢出
void ParseKey(const char* input) {
  char buffer[16];
  strcpy(buffer, input);  // 如果input > 16字节，溢出！
}

// 正确：检查边界
void ParseKey(const char* input) {
  char buffer[16];
  strncpy(buffer, input, sizeof(buffer) - 1);
  buffer[sizeof(buffer) - 1] = '\0';
}

// 更好：使用std::string
void ParseKey(const char* input) {
  std::string buffer(input);
}
```

---

## 2. 崩溃调试

### 2.1 生成Coredump

**启用Coredump：**
```bash
# 检查当前限制
ulimit -c

# 设置无限制
ulimit -c unlimited

# 设置coredump文件位置和命名
sudo sysctl -w kernel.core_pattern=/tmp/core-%e-%p-%t

# 永久生效（添加到/etc/sysctl.conf）
echo "kernel.core_pattern=/tmp/core-%e-%p-%t" | sudo tee -a /etc/sysctl.conf
```

**触发崩溃并生成coredump：**
```cpp
// crash_test.cc - 模拟崩溃
#include <iostream>

void CrashFunction() {
  int* ptr = nullptr;
  *ptr = 42;  // 段错误
}

int main() {
  std::cout << "About to crash...\n";
  CrashFunction();
  return 0;
}
```

编译并运行：
```bash
g++ -g -O0 crash_test.cc -o crash_test
./crash_test

# 查看生成的coredump
ls -lh /tmp/core-*
```

### 2.2 使用GDB分析Coredump

**加载coredump：**
```bash
gdb ./crash_test /tmp/core-crash_test-12345-1234567890

# GDB中查看崩溃位置
(gdb) bt
#0  0x0000000000401156 in CrashFunction () at crash_test.cc:5
#1  0x0000000000401178 in main () at crash_test.cc:10

# 查看源码
(gdb) list CrashFunction
1	#include <iostream>
2
3	void CrashFunction() {
4	  int* ptr = nullptr;
5	  *ptr = 42;  // 段错误
6	}
7
8	int main() {
9	  std::cout << "About to crash...\n";
10	  CrashFunction();

# 查看变量
(gdb) frame 0
(gdb) print ptr
$1 = (int *) 0x0
```

### 2.3 真实案例：LevelDB崩溃调试

**案例：SkipList插入崩溃**

崩溃日志：
```
Segmentation fault (core dumped)
```

GDB分析：
```gdb
$ gdb ./db_bench /tmp/core-db_bench-23456-1234567890

(gdb) bt
#0  0x00007f8b2d3e4567 in leveldb::SkipList<leveldb::MemTable::KeyComparator>::Insert(char const*) () at db/skiplist.h:356
#1  0x00007f8b2d3e3a23 in leveldb::MemTable::Add(unsigned long, leveldb::ValueType, leveldb::Slice const&, leveldb::Slice const&) () at db/memtable.cc:89
#2  0x00007f8b2d3f1234 in leveldb::WriteBatchInternal::InsertInto(leveldb::WriteBatch const*, leveldb::MemTable*) () at db/write_batch.cc:123
#3  0x00007f8b2d3e8abc in leveldb::DBImpl::Write(leveldb::WriteOptions const&, leveldb::WriteBatch*) () at db/db_impl.cc:1234

(gdb) frame 0
#0  0x00007f8b2d3e4567 in leveldb::SkipList<...>::Insert(const Key& key) at db/skiplist.h:356
356	    prev[i]->SetNext(i, x);

(gdb) print i
$1 = 8

(gdb) print GetMaxHeight()
$2 = 7

(gdb) print prev[8]
$3 = (Node *) 0x6f6f6f6f6f6f6f6f  # 未初始化指针！

(gdb) print prev[7]
$4 = (Node *) 0x7f8b2c001230
```

**问题分析：**
```cpp
// db/skiplist.h
template <typename Key, class Comparator>
void SkipList<Key, Comparator>::Insert(const Key& key) {
  Node* prev[kMaxHeight];  // 未初始化！

  Node* x = FindGreaterOrEqual(key, prev);

  int height = RandomHeight();
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;  // 初始化新层
    }
    max_height_.store(height, std::memory_order_relaxed);
  }

  x = NewNode(key, height);
  for (int i = 0; i < height; i++) {
    x->SetNext(i, prev[i]->Next(i));
    prev[i]->SetNext(i, x);  // 崩溃：如果prev[i]未初始化
  }
}
```

**问题根源：**
- `prev`数组未初始化
- `FindGreaterOrEqual`只填充`[0, GetMaxHeight())`范围
- 如果`RandomHeight()`返回的值在首次插入时就很大，`prev`高索引未初始化

**修复：**
```cpp
void SkipList<Key, Comparator>::Insert(const Key& key) {
  Node* prev[kMaxHeight];
  Node* x = FindGreaterOrEqual(key, prev);

  int height = RandomHeight();
  int max_height = GetMaxHeight();

  if (height > max_height) {
    for (int i = max_height; i < height; i++) {
      prev[i] = head_;  // 初始化新层
    }
    max_height_.store(height, std::memory_order_relaxed);
  }

  x = NewNode(key, height);
  for (int i = 0; i < height; i++) {
    // 现在prev[i]一定被初始化了
    x->SetNext(i, prev[i]->Next(i));
    prev[i]->SetNext(i, x);
  }
}
```

### 2.4 常见崩溃模式

**模式1：空指针解引用**
```gdb
(gdb) bt
#0  0x0000000000401234 in Foo::DoSomething() at foo.cc:45
#1  0x0000000000401567 in main() at main.cc:12

(gdb) frame 0
(gdb) print this
$1 = (Foo *) 0x0

# 诊断：this为空，说明在nullptr上调用成员函数
```

**模式2：栈溢出**
```gdb
(gdb) bt
#0  RecursiveFunction() at foo.cc:10
#1  RecursiveFunction() at foo.cc:15
#2  RecursiveFunction() at foo.cc:15
... (重复数千次)
#9999  RecursiveFunction() at foo.cc:15

# 诊断：无限递归导致栈溢出
```

**模式3：数据竞争导致的崩溃**
```gdb
(gdb) bt
#0  std::vector::operator[](size_t) at stl_vector.h:1234
#1  ProcessData() at worker.cc:56

(gdb) print vec_.size()
$1 = 0

(gdb) info threads
  Id   Target Id         Frame
* 1    Thread 0x7f... ProcessData() at worker.cc:56
  2    Thread 0x7f... ClearData() at worker.cc:89

# 诊断：线程1正在访问vec_，线程2同时调用clear()
```

---

## 3. 死锁分析

### 3.1 使用GDB检测死锁

**模拟死锁：**
```cpp
// deadlock_test.cc
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::mutex mutex1;
std::mutex mutex2;

void Thread1() {
  std::cout << "Thread1: locking mutex1\n";
  std::lock_guard<std::mutex> lock1(mutex1);
  std::this_thread::sleep_for(std::chrono::milliseconds(100));

  std::cout << "Thread1: locking mutex2\n";
  std::lock_guard<std::mutex> lock2(mutex2);  // 死锁！
  std::cout << "Thread1: done\n";
}

void Thread2() {
  std::cout << "Thread2: locking mutex2\n";
  std::lock_guard<std::mutex> lock2(mutex2);
  std::this_thread::sleep_for(std::chrono::milliseconds(100));

  std::cout << "Thread2: locking mutex1\n";
  std::lock_guard<std::mutex> lock1(mutex1);  // 死锁！
  std::cout << "Thread2: done\n";
}

int main() {
  std::thread t1(Thread1);
  std::thread t2(Thread2);

  t1.join();
  t2.join();

  return 0;
}
```

编译并运行：
```bash
g++ -std=c++11 -g -pthread deadlock_test.cc -o deadlock_test
./deadlock_test &
PID=$!

# 程序挂起，检测死锁
ps aux | grep deadlock_test

# 使用GDB attach
sudo gdb -p $PID
```

GDB调试：
```gdb
# 查看所有线程
(gdb) info threads
  Id   Target Id         Frame
  1    Thread 0x7f8b... (LWP 12345) __lll_lock_wait () at lowlevellock.S:135
  2    Thread 0x7f8a... (LWP 12346) __lll_lock_wait () at lowlevellock.S:135
* 3    Thread 0x7f89... (LWP 12347) main () at deadlock_test.cc:35

# 切换到线程1
(gdb) thread 1
(gdb) bt
#0  __lll_lock_wait () at lowlevellock.S:135
#1  0x00007f8b2d... in __pthread_mutex_lock () at pthread_mutex_lock.c:80
#2  0x0000000000401567 in __gthread_mutex_lock (__mutex=0x603140 <mutex2>) at gthr-default.h:748
#3  0x0000000000401678 in std::mutex::lock() at mutex:135
#4  0x00000000004017ab in Thread1() at deadlock_test.cc:14

# 切换到线程2
(gdb) thread 2
(gdb) bt
#0  __lll_lock_wait () at lowlevellock.S:135
#1  0x00007f8b2d... in __pthread_mutex_lock () at pthread_mutex_lock.c:80
#2  0x0000000000401567 in __gthread_mutex_lock (__mutex=0x603120 <mutex1>) at gthr-default.h:748
#3  0x0000000000401678 in std::mutex::lock() at mutex:135
#4  0x00000000004018cd in Thread2() at deadlock_test.cc:23

# 分析：
# 线程1持有mutex1，等待mutex2
# 线程2持有mutex2，等待mutex1
# 经典死锁！
```

### 3.2 ThreadSanitizer (TSan) - 数据竞争检测

**编译启用TSan：**
```bash
cd /home/dev/leveldb
mkdir -p build-tsan && cd build-tsan
cmake -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_CXX_FLAGS="-g -O1 -fsanitize=thread -fno-omit-frame-pointer" \
      -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=thread" \
      ..
cmake --build .

./db_bench --benchmarks=fillrandom --num=10000 --threads=4
```

**TSan检测到的问题：**

**案例：数据竞争**
```bash
==================
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 8 at 0x7f8b2c001234 by thread T2:
    #0 leveldb::MemTable::Unref() db/memtable.cc:52
    #1 leveldb::DBImpl::CompactMemTable() db/db_impl.cc:789

  Previous read of size 8 at 0x7f8b2c001234 by thread T1:
    #0 leveldb::MemTable::Get(...) db/memtable.cc:89
    #1 leveldb::DBImpl::Get(...) db/db_impl.cc:1156

  Location is heap block of size 64 at 0x7f8b2c001230 allocated by main thread:
    #0 operator new(unsigned long) <null>:0
    #1 leveldb::DBImpl::Open(...) db/db_impl.cc:1234
```

问题代码：
```cpp
// 错误：未加锁访问共享状态
void DBImpl::CompactMemTable() {
  // 没有持有mutex_
  if (imm_ != nullptr) {
    imm_->Unref();  // 数据竞争！
  }
}

Status DBImpl::Get(...) {
  // 没有持有mutex_
  if (imm_ != nullptr) {
    imm_->Get(...);  // 数据竞争！
  }
}

// 正确：使用互斥锁
void DBImpl::CompactMemTable() {
  MutexLock l(&mutex_);
  if (imm_ != nullptr) {
    imm_->Unref();  // 安全
  }
}
```

### 3.3 死锁预防策略

**策略1：锁顺序**
```cpp
// 定义全局锁顺序
// mutex_A < mutex_B < mutex_C

// 正确：按顺序获取锁
void Foo() {
  std::lock_guard<std::mutex> lock_a(mutex_a_);
  std::lock_guard<std::mutex> lock_b(mutex_b_);  // 顺序正确
  // ...
}

// 错误：逆序获取锁
void Bar() {
  std::lock_guard<std::mutex> lock_b(mutex_b_);
  std::lock_guard<std::mutex> lock_a(mutex_a_);  // 可能死锁！
}
```

**策略2：使用std::lock**
```cpp
// 同时获取多个锁，避免死锁
void Transfer(Account& from, Account& to, int amount) {
  // 同时锁定两个账户，避免死锁
  std::lock(from.mutex_, to.mutex_);

  // 采用锁
  std::lock_guard<std::mutex> lock1(from.mutex_, std::adopt_lock);
  std::lock_guard<std::mutex> lock2(to.mutex_, std::adopt_lock);

  from.balance -= amount;
  to.balance += amount;
}
```

**策略3：超时机制**
```cpp
// 使用try_lock_for避免永久阻塞
void Foo() {
  std::unique_lock<std::timed_mutex> lock(mutex_, std::chrono::seconds(1));

  if (!lock.owns_lock()) {
    // 超时，可能死锁
    LOG(ERROR) << "Failed to acquire lock, possible deadlock";
    return;
  }

  // ...
}
```

---

## 4. 数据损坏排查

### 4.1 Checksum校验失败

**错误日志：**
```
leveldb: Corruption: block checksum mismatch at offset 12345
leveldb: Corruption: bad magic number in footer
```

**诊断步骤：**

1. **检查文件完整性：**
```bash
# 查看SSTable文件
ls -lh /tmp/testdb/*.ldb

# 检查文件大小（Footer应该是48字节）
file_size=$(stat -c%s /tmp/testdb/000123.ldb)
echo "File size: $file_size"

# 读取Footer (最后48字节)
tail -c 48 /tmp/testdb/000123.ldb | hexdump -C
```

2. **使用LevelDB工具检查：**
```cpp
// repair_db.cc - 数据库修复工具
#include "leveldb/db.h"
#include <iostream>

int main(int argc, char** argv) {
  if (argc < 2) {
    std::cerr << "Usage: repair_db <dbpath>\n";
    return 1;
  }

  std::string dbpath = argv[1];
  leveldb::Options options;
  options.paranoid_checks = true;

  // 尝试打开（会检测损坏）
  leveldb::DB* db;
  leveldb::Status s = leveldb::DB::Open(options, dbpath, &db);

  if (!s.ok()) {
    std::cout << "Database corrupted: " << s.ToString() << "\n";
    std::cout << "Attempting repair...\n";

    // 修复数据库
    s = leveldb::RepairDB(dbpath, options);
    if (s.ok()) {
      std::cout << "Repair successful\n";

      // 重新打开
      s = leveldb::DB::Open(options, dbpath, &db);
      if (s.ok()) {
        std::cout << "Database reopened successfully\n";
        delete db;
        return 0;
      }
    }

    std::cout << "Repair failed: " << s.ToString() << "\n";
    return 1;
  }

  std::cout << "Database is healthy\n";
  delete db;
  return 0;
}
```

编译并运行：
```bash
g++ -std=c++11 -I../include repair_db.cc -L../build -lleveldb -pthread -o repair_db
./repair_db /tmp/corrupted_db
```

3. **Hexdump分析：**
```bash
# 查看SSTable Footer
file=/tmp/testdb/000123.ldb
size=$(stat -c%s "$file")
dd if="$file" bs=1 skip=$((size - 48)) count=48 | hexdump -C

# 正常Footer格式：
# 00000000  xx xx ... xx xx  |metaindex handle (varint)|
# 00000014  xx xx ... xx xx  |index handle (varint)    |
# 00000028  00 00 00 00 00 00 00 00  |padding                  |
# 00000030  db fe 03 f9 2f 4d 3f b6  |magic number             |

# 损坏的Footer：
# 00000030  00 00 00 00 00 00 00 00  |magic number错误！       |
```

### 4.2 日志回放失败

**错误日志：**
```
leveldb: Recovering log #123
leveldb: Corruption: log record too short at offset 4567
```

**分析WAL文件：**
```cpp
// inspect_log.cc - WAL日志检查工具
#include "leveldb/env.h"
#include "db/log_reader.h"
#include "db/log_writer.h"
#include <iostream>

int main(int argc, char** argv) {
  if (argc < 2) {
    std::cerr << "Usage: inspect_log <logfile>\n";
    return 1;
  }

  leveldb::Env* env = leveldb::Env::Default();
  leveldb::SequentialFile* file;
  leveldb::Status s = env->NewSequentialFile(argv[1], &file);

  if (!s.ok()) {
    std::cerr << "Failed to open: " << s.ToString() << "\n";
    return 1;
  }

  class Reporter : public leveldb::log::Reader::Reporter {
   public:
    void Corruption(size_t bytes, const leveldb::Status& status) override {
      std::cout << "Corruption at offset " << bytes << ": "
                << status.ToString() << "\n";
      corruption_count_++;
    }
    int corruption_count_ = 0;
  };

  Reporter reporter;
  leveldb::log::Reader reader(file, &reporter, true, 0);

  std::string record;
  int record_count = 0;

  while (reader.ReadRecord(&record, nullptr)) {
    record_count++;
    std::cout << "Record #" << record_count
              << ", size=" << record.size() << "\n";
  }

  std::cout << "\nTotal records: " << record_count << "\n";
  std::cout << "Corruptions: " << reporter.corruption_count_ << "\n";

  delete file;
  return 0;
}
```

### 4.3 数据不一致

**问题：写入的数据读不到**

诊断步骤：
```cpp
// consistency_check.cc
#include "leveldb/db.h"
#include <iostream>
#include <random>

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;
  options.paranoid_checks = true;

  leveldb::Status s = leveldb::DB::Open(options, "/tmp/testdb", &db);
  if (!s.ok()) {
    std::cerr << "Open failed: " << s.ToString() << "\n";
    return 1;
  }

  const int kNumKeys = 10000;
  std::mt19937 rng(12345);

  // 写入数据
  std::cout << "Writing " << kNumKeys << " keys...\n";
  leveldb::WriteOptions write_options;
  write_options.sync = true;  // 强制刷盘

  for (int i = 0; i < kNumKeys; i++) {
    std::string key = "key" + std::to_string(i);
    std::string value = "value" + std::to_string(i) + "_" + std::to_string(rng());

    s = db->Put(write_options, key, value);
    if (!s.ok()) {
      std::cerr << "Put failed: " << s.ToString() << "\n";
      return 1;
    }
  }

  // 验证数据
  std::cout << "Verifying...\n";
  leveldb::ReadOptions read_options;
  rng.seed(12345);  // 重置随机数
  int missing = 0;
  int mismatch = 0;

  for (int i = 0; i < kNumKeys; i++) {
    std::string key = "key" + std::to_string(i);
    std::string expected = "value" + std::to_string(i) + "_" + std::to_string(rng());
    std::string actual;

    s = db->Get(read_options, key, &actual);
    if (s.IsNotFound()) {
      std::cout << "Missing: " << key << "\n";
      missing++;
    } else if (!s.ok()) {
      std::cerr << "Get failed: " << s.ToString() << "\n";
      return 1;
    } else if (actual != expected) {
      std::cout << "Mismatch: " << key << "\n";
      std::cout << "  Expected: " << expected << "\n";
      std::cout << "  Actual: " << actual << "\n";
      mismatch++;
    }
  }

  std::cout << "\nResults:\n";
  std::cout << "  Missing keys: " << missing << "\n";
  std::cout << "  Mismatched values: " << mismatch << "\n";
  std::cout << "  Verified: " << (kNumKeys - missing - mismatch) << "\n";

  delete db;
  return 0;
}
```

---

## 5. 性能突降诊断

### 5.1 识别性能突降

**监控工具：**
```cpp
// performance_monitor.cc
#include "leveldb/db.h"
#include <iostream>
#include <chrono>
#include <thread>

class PerformanceMonitor {
 public:
  PerformanceMonitor(leveldb::DB* db) : db_(db), running_(true) {
    monitor_thread_ = std::thread(&PerformanceMonitor::Monitor, this);
  }

  ~PerformanceMonitor() {
    running_ = false;
    if (monitor_thread_.joinable()) {
      monitor_thread_.join();
    }
  }

 private:
  void Monitor() {
    while (running_) {
      std::string stats;
      db_->GetProperty("leveldb.stats", &stats);

      auto now = std::chrono::system_clock::now();
      std::time_t time = std::chrono::system_clock::to_time_t(now);

      std::cout << "\n=== " << std::ctime(&time);
      std::cout << stats << "\n";

      std::this_thread::sleep_for(std::chrono::seconds(10));
    }
  }

  leveldb::DB* db_;
  std::atomic<bool> running_;
  std::thread monitor_thread_;
};

int main() {
  leveldb::DB* db;
  leveldb::Options options;
  options.create_if_missing = true;
  options.write_buffer_size = 4 * 1024 * 1024;  // 4MB

  leveldb::Status s = leveldb::DB::Open(options, "/tmp/monitor_db", &db);
  if (!s.ok()) {
    std::cerr << "Open failed\n";
    return 1;
  }

  PerformanceMonitor monitor(db);

  // 模拟工作负载
  leveldb::WriteOptions write_options;
  for (int i = 0; i < 1000000; i++) {
    std::string key = "key" + std::to_string(i);
    std::string value(1000, 'x');

    db->Put(write_options, key, value);

    if (i % 10000 == 0) {
      std::cout << "Written " << i << " keys\n";
    }
  }

  delete db;
  return 0;
}
```

**输出分析：**
```
=== Mon Jan 19 10:00:00 2026
                               Compactions
Level  Files Size(MB) Time(sec) Read(MB) Write(MB)
--------------------------------------------------
  0        4      16         5       0       16
  1       10      80        12      20       20
  2       45     360        30     120      120

# 关键指标：
# - Level-0文件数：4 (正常，< 8)
# - Compaction时间：正常
# - 写放大：1.25x (20MB read, 16MB write at L0->L1)

=== Mon Jan 19 10:01:00 2026
                               Compactions
Level  Files Size(MB) Time(sec) Read(MB) Write(MB)
--------------------------------------------------
  0       12      48        45      16       48   ← Level-0堆积！
  1       10      80        12      20       20
  2       45     360        30     120      120

# 问题：Level-0文件数激增 (12个)
# 原因：Compaction跟不上写入速度
```

### 5.2 常见性能问题

**问题1：Level-0堆积**
```bash
# 症状
leveldb.stats显示Level-0文件数 > 8

# 原因
- 写入速度过快
- Compaction线程阻塞
- 磁盘I/O瓶颈

# 解决方案
# 1. 增大write_buffer_size
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB

# 2. 禁用压缩（临时）
options.compression = leveldb::kNoCompression;

# 3. 手动触发Compaction
db->CompactRange(nullptr, nullptr);
```

**问题2：读取延迟高**
```bash
# 使用perf分析
perf record -g ./db_bench --benchmarks=readrandom --num=100000
perf report

# 热点函数：
# 45% leveldb::TableCache::Get  ← Cache miss严重
# 20% leveldb::Block::Iter::Next
# 15% snappy::Uncompress

# 解决方案
# 1. 增大Cache
options.block_cache = leveldb::NewLRUCache(512 * 1024 * 1024);  // 512MB

# 2. 开启Bloom Filter
options.filter_policy = leveldb::NewBloomFilterPolicy(10);

# 3. 预热Cache
for (auto it = db->NewIterator(read_options); it->Valid(); it->Next()) {
  // 遍历预热
}
```

**问题3：写入突然变慢**
```bash
# 检查磁盘I/O
iostat -x 1

# 输出：
Device  r/s   w/s  rMB/s  wMB/s  %util
sda     100   500   10.0   50.0   95%   ← 磁盘已满负载

# 检查Compaction状态
db->GetProperty("leveldb.num-files-at-level0", &value);
# value = "12"  ← 触发写停顿

# 解决方案：等待Compaction完成
```

---

## 6. 系统调用追踪

### 6.1 使用strace追踪系统调用

**基本用法：**
```bash
# 追踪db_bench的系统调用
strace -o strace.log ./db_bench --benchmarks=fillseq --num=10000

# 查看日志
less strace.log
```

**常见系统调用分析：**
```bash
# 统计系统调用
strace -c ./db_bench --benchmarks=fillseq --num=10000

# 输出：
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.23    0.123456          12     10234           write
 23.45    0.064123           8      8012           read
 12.34    0.033712          15      2247           pwrite64
  8.91    0.024345          10      2434           fsync
  5.67    0.015489          23       673           open
  2.34    0.006391          19       336           close
  1.23    0.003367          11       306           mmap
  0.83    0.002267          14       162           munmap
------ ----------- ----------- --------- --------- ----------------
100.00    0.273150                 24404           total
```

**案例：诊断慢写入**
```bash
# 追踪写操作
strace -e trace=write,pwrite64,fsync -T ./db_bench --benchmarks=fillrandom --num=1000

# 输出：
pwrite64(5, "...", 4096, 0) = 4096 <0.000123>
pwrite64(5, "...", 4096, 4096) = 4096 <0.000145>
...
fsync(5) = 0 <0.015234>  ← 15ms，正常
...
pwrite64(5, "...", 4096, 819200) = 4096 <0.000134>
fsync(5) = 0 <0.234567>  ← 234ms！太慢

# 问题：fsync耗时突然增加
# 可能原因：
# - 磁盘写缓存满
# - I/O调度问题
# - 磁盘硬件问题
```

### 6.2 使用ltrace追踪库函数调用

**追踪malloc/free：**
```bash
ltrace -e malloc,free -c ./db_bench --benchmarks=fillseq --num=1000

# 输出：
% time     seconds  usecs/call     calls      function
------ ----------- ----------- --------- --------------------
 56.78    0.012345          12      1029 malloc
 43.22    0.009391           9      1042 free
------ ----------- ----------- --------- --------------------
100.00    0.021736                  2071 total
```

**追踪特定函数：**
```bash
# 追踪snappy压缩
ltrace -e '*Compress*' ./db_bench --benchmarks=fillrandom --num=1000

# 输出：
snappy::Compress(...) = 0 <0.001234>
snappy::Compress(...) = 0 <0.001456>
...
```

### 6.3 综合诊断案例

**问题：写入性能下降10倍**

诊断流程：

1. **检查系统调用：**
```bash
strace -c -T ./db_bench --benchmarks=fillrandom --num=10000 2>&1 | tail -20

#  % time     seconds  usecs/call     calls
#  89.23    5.234567      50000           104 fsync  ← 异常！

# 发现：fsync平均耗时50ms
```

2. **检查磁盘I/O：**
```bash
iostat -x 1 10

# Device  r/s   w/s  await  %util
# sda      20   200  250ms   100%  ← await太高！

# 发现：磁盘等待时间250ms
```

3. **检查I/O调度器：**
```bash
cat /sys/block/sda/queue/scheduler
# [cfq] deadline noop

# 切换到deadline
echo deadline > /sys/block/sda/queue/scheduler

# 重新测试
./db_bench --benchmarks=fillrandom --num=10000
# fillrandom: 85000 ops/sec (恢复正常)
```

4. **根因分析：**
```
问题：CFQ调度器在高并发写入时性能差
解决：切换到deadline调度器
效果：性能恢复10倍
```

---

## 总结

今天我们学习了：

1. ✅ **内存问题诊断**：Valgrind, ASan检测内存泄漏、use-after-free、缓冲区溢出
2. ✅ **崩溃调试**：coredump生成、GDB分析、真实崩溃案例
3. ✅ **死锁分析**：GDB检测死锁、TSan检测数据竞争、死锁预防策略
4. ✅ **数据损坏排查**：checksum验证、WAL日志分析、一致性检查
5. ✅ **性能突降诊断**：性能监控、Level-0堆积、读写延迟分析
6. ✅ **系统调用追踪**：strace, ltrace, I/O性能诊断

**关键要点：**
- 使用正确的工具诊断不同类型的问题
- Valgrind慢但准确，ASan快但需要重新编译
- GDB是崩溃调试的核心工具
- TSan能检测微妙的并发问题
- 性能问题往往是I/O瓶颈

**工具箱：**
- `valgrind --leak-check=full`: 内存泄漏检测
- `gdb -p PID`: attach到进程
- `strace -c`: 系统调用统计
- `iostat -x`: 磁盘I/O监控
- `perf record/report`: 性能分析

**下一步学习：**
1. 实战练习：在测试环境中模拟各种故障
2. 构建自动化故障检测系统
3. 学习分布式系统的故障诊断
4. 研究生产环境的监控最佳实践

恭喜你完成了故障排查手册！🎉
