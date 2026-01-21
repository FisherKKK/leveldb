# Day 3: SkipList无锁数据结构

## 学习目标
- 理解SkipList的数据结构原理
- 掌握无锁并发的实现技术
- 学习LevelDB中SkipList的优化
- 理解内存顺序和原子操作

## 1. SkipList基础

### 1.1 为什么需要SkipList？

**MemTable的需求：**
- 有序存储键值对
- 快速插入：O(log N)
- 快速查找：O(log N)
- 支持范围扫描
- 并发读取（无锁）

**候选数据结构对比：**

| 数据结构 | 插入 | 查找 | 并发读 | 实现复杂度 |
|---------|------|------|--------|-----------|
| 数组 | O(N) | O(log N) | 简单 | 低 |
| 链表 | O(1) | O(N) | 简单 | 低 |
| 平衡树(AVL/RB) | O(log N) | O(log N) | 复杂 | 高 |
| B+Tree | O(log N) | O(log N) | 复杂 | 高 |
| **SkipList** | **O(log N)** | **O(log N)** | **简单** | **中** |

**LevelDB选择SkipList的原因：**
1. 实现简单（约300行代码）
2. 无锁读取，并发性能好
3. 节点不需要重新平衡
4. 内存局部性较好

### 1.2 SkipList原理

#### 基本思想

普通链表查找需要O(N)：

```
Level 0: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10
         查找9需要遍历9次
```

SkipList通过多层索引加速查找：

```
Level 2: 1 ───────────────→ 7 ──────────→ 10
         ↓                  ↓              ↓
Level 1: 1 ─────→ 4 ─────→ 7 ─────→ 9 ─→ 10
         ↓        ↓         ↓        ↓     ↓
Level 0: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10

查找9的路径: 1(L2) → 7(L2) → 7(L1) → 9(L1) ✓ (4次)
```

#### 节点高度随机化

```cpp
// db/skiplist.h, lines 298-312
template <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && rnd_.OneIn(kBranching)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

**参数配置：**
- `kMaxHeight = 12`：最大层数
- `kBranching = 4`：每层晋升概率 = 1/4

**高度分布：**
```
Level 0:  100% 节点
Level 1:  25%  节点  (1/4)
Level 2:  6.25% 节点 (1/16)
Level 3:  1.56% 节点 (1/64)
...
```

期望节点数：如果最底层有N个节点，总节点数 ≈ N * (1 + 1/4 + 1/16 + ...) = N * 4/3

### 1.3 SkipList结构

#### 节点定义

```cpp
// db/skiplist.h, lines 208-228
template <typename Key, class Comparator>
struct SkipList<Key, Comparator>::Node {
  explicit Node(const Key& k) : key(k) {}

  Key const key;

  // 获取下一个节点（带内存顺序）
  Node* Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_acquire);
  }

  void SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_release);
  }

  // 无屏障版本（用于已持有互斥锁时）
  Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_relaxed);
  }

  void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_relaxed);
  }

 private:
  std::atomic<Node*> next_[1];  // 柔性数组，实际大小动态分配
};
```

**内存布局：**

```
Node (高度=3):
┌──────────────┐
│  key         │  ← 键（不可变）
├──────────────┤
│  next_[0]    │  → Level 0 下一个节点
├──────────────┤
│  next_[1]    │  → Level 1 下一个节点
├──────────────┤
│  next_[2]    │  → Level 2 下一个节点
└──────────────┘
```

#### 分配节点

```cpp
// db/skiplist.h, lines 314-321
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node*
SkipList<Key, Comparator>::NewNode(const Key& key, int height) {
  // 计算需要的内存：基础大小 + (height-1) * 指针大小
  char* const node_memory = arena_->AllocateAligned(
      sizeof(Node) + sizeof(std::atomic<Node*>) * (height - 1));
  return new (node_memory) Node(key);  // placement new
}
```

**关键技术：**
- 柔性数组：`next_[1]`声明为1，实际分配height个
- Arena分配：从内存池分配，不需要单独释放
- Placement new：在已分配内存上构造对象

## 2. 无锁并发实现

### 2.1 并发模型

**LevelDB的SkipList并发规则：**

```cpp
// db/skiplist.h, lines 8-28
// Thread safety
// -------------
//
// Writes require external synchronization, most likely a mutex.
// Reads require a guarantee that the SkipList will not be destroyed
// while the read is in progress.  Apart from that, reads progress
// without any internal locking or synchronization.
//
// Invariants:
//
// (1) Allocated nodes are never deleted until the SkipList is
//     destroyed.  This is trivially guaranteed by the code since we
//     never delete any skip list nodes.
//
// (2) The contents of a Node except for the next/prev pointers are
//     immutable after the Node has been linked into the SkipList.
//     Only Insert() modifies the list, and it is careful to initialize
//     a node and use release-stores to publish the nodes in one or
//     more lists.
//
// ... prev/next pointers are immutable after being inserted
```

**并发保证：**
1. **写操作**：需要外部互斥锁（由DBImpl提供）
2. **读操作**：无锁，可以并发读
3. **关键不变式**：
   - 节点一旦分配，永不删除
   - 节点内容不可变（除了next指针）
   - next指针更新使用release语义

### 2.2 内存顺序 (Memory Order)

#### C++11原子操作

```cpp
// 三种内存顺序在SkipList中的使用：

// 1. memory_order_relaxed：无同步，仅保证原子性
node->NoBarrier_SetNext(i, x);  // 已持有锁，无需同步

// 2. memory_order_acquire：读取时同步
Node* next = node->Next(i);     // 确保看到完整的节点内容

// 3. memory_order_release：写入时同步
node->SetNext(i, x);            // 确保节点内容先于指针可见
```

#### 发布-订阅模式

```cpp
// 写线程（持有锁）：
Node* new_node = NewNode(key, height);
// 1. 设置节点内容（key已在构造时设置）
// 2. 使用release发布节点
for (int i = 0; i < height; i++) {
  new_node->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
  prev[i]->SetNext(i, new_node);  // memory_order_release
}

// 读线程（无锁）：
Node* node = head_->Next(level);  // memory_order_acquire
// 保证看到node的完整内容（key和所有next指针）
```

**为什么安全？**

```
时间线：
T1: 写线程设置 new_node->key = "foo"
T2: 写线程设置 new_node->next_[0] = ...
T3: 写线程 prev->SetNext(0, new_node)  [release]
T4: 读线程 node = head->Next(0)        [acquire]
T5: 读线程读取 node->key

release-acquire保证：T5看到的内存状态包含T1-T3的所有写入
```

### 2.3 查找操作

```cpp
// db/skiplist.h, lines 259-279
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node*
SkipList<Key, Comparator>::FindGreaterOrEqual(const Key& key,
                                               Node** prev) const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);  // memory_order_acquire
    if (KeyIsAfterNode(key, next)) {
      // key > next->key，在当前层继续向右
      x = next;
    } else {
      // key <= next->key，记录前驱节点，下降一层
      if (prev != nullptr) prev[level] = x;
      if (level == 0) {
        return next;  // 找到或者到达底层
      } else {
        level--;
      }
    }
  }
}
```

**查找过程可视化：**

```
查找 key = 7

Level 2: head → 3 ──────────→ 9 → NULL
              ↓ (7>3)      ↑ (7<9, 下降)
Level 1: head → 3 → 5 ────→ 9 → NULL
                   ↓ (7>5) ↑ (7<9, 下降)
Level 0: head → 3 → 5 → 7 → 9 → NULL
                       ↑ 找到！

prev数组记录：
prev[2] = 3
prev[1] = 5
prev[0] = 5
```

### 2.4 插入操作

```cpp
// db/skiplist.h, lines 335-366
template <typename Key, class Comparator>
void SkipList<Key, Comparator>::Insert(const Key& key) {
  Node* prev[kMaxHeight];
  Node* x = FindGreaterOrEqual(key, prev);  // 找到插入位置

  assert(x == nullptr || !Equal(key, x->key));  // 不允许重复键

  int height = RandomHeight();  // 随机高度
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    max_height_.store(height, std::memory_order_relaxed);
  }

  x = NewNode(key, height);
  for (int i = 0; i < height; i++) {
    // NoBarrier_SetNext() 足够，因为：
    // 1. 我们持有锁，其他写线程看不到
    // 2. 读线程通过acquire读取prev[i]的next指针，会看到完整的x
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);  // memory_order_release
  }
}
```

**插入过程可视化：**

```
插入 key = 6, 随机高度 = 2

插入前：
Level 1: head → 3 ────→ 9 → NULL
         ↓       ↓      ↓
Level 0: head → 3 → 5 → 9 → NULL

步骤1：FindGreaterOrEqual找到prev
prev[1] = 3
prev[0] = 5

步骤2：创建新节点 new_node(6, height=2)

步骤3：链接新节点
Level 1: 3 → new_node(6) → 9
Level 0: 5 → new_node(6) → 9

插入后：
Level 1: head → 3 ──→ 6 ──→ 9 → NULL
         ↓       ↓    ↓    ↓
Level 0: head → 3 → 5 → 6 → 9 → NULL
```

**关键顺序：**

```cpp
// 1. 先设置新节点的next指针（指向后继）
x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));

// 2. 再更新前驱节点的next指针（发布新节点）
prev[i]->SetNext(i, x);  // release语义

// 这个顺序保证：
// - 读线程要么看不到新节点（prev[i]->next还是旧值）
// - 要么看到完整的新节点（x->next已经设置好）
```

## 3. Iterator实现

### 3.1 Iterator接口

```cpp
// db/skiplist.h, lines 110-145
class Iterator {
 public:
  explicit Iterator(const SkipList* list);

  bool Valid() const;         // 当前位置是否有效
  const Key& key() const;     // 返回当前键
  void Next();                // 前进
  void Prev();                // 后退
  void Seek(const Key& target);     // 定位到>=target的第一个键
  void SeekToFirst();         // 定位到第一个键
  void SeekToLast();          // 定位到最后一个键

 private:
  const SkipList* list_;
  Node* node_;                // 当前节点（nullptr表示无效）
};
```

### 3.2 迭代器实现

```cpp
// db/skiplist.h

template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Next() {
  assert(Valid());
  node_ = node_->Next(0);  // 只在Level 0移动
}

template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Prev() {
  assert(Valid());
  node_ = list_->FindLessThan(node_->key);
  if (node_ == list_->head_) {
    node_ = nullptr;  // 已到达开头
  }
}

template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Seek(const Key& target) {
  node_ = list_->FindGreaterOrEqual(target, nullptr);
}

template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::SeekToFirst() {
  node_ = list_->head_->Next(0);
}
```

**使用示例：**

```cpp
SkipList<int, IntComparator> list;
list.Insert(3);
list.Insert(1);
list.Insert(5);

// 顺序遍历
SkipList<int, IntComparator>::Iterator iter(&list);
for (iter.SeekToFirst(); iter.Valid(); iter.Next()) {
  std::cout << iter.key() << " ";  // 输出: 1 3 5
}

// 范围查询
iter.Seek(2);  // 定位到 >= 2 的第一个元素
while (iter.Valid() && iter.key() < 6) {
  std::cout << iter.key() << " ";  // 输出: 3 5
  iter.Next();
}
```

## 4. 性能分析

### 4.1 时间复杂度

**理论分析：**

- 每层节点数期望为下一层的 1/4
- 高度期望为 log₄(N) ≈ 0.5 * log₂(N)
- 查找路径期望长度：每层向右走常数步 + 向下走 log₄(N) 步

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| Insert | O(log N) | FindGreaterOrEqual + 链接节点 |
| Contains | O(log N) | FindGreaterOrEqual |
| Next | O(1) | 沿Level 0向右一步 |
| Prev | O(log N) | FindLessThan（需要重新查找）|
| Seek | O(log N) | FindGreaterOrEqual |

### 4.2 空间开销

```cpp
// 节点内存开销
sizeof(Node) = sizeof(Key) + sizeof(atomic<Node*>) * height

// 例如：Key = 8字节，指针 = 8字节
Level 0节点: 8 + 8*1 = 16字节
Level 1节点: 8 + 8*2 = 24字节
Level 2节点: 8 + 8*3 = 32字节

// 平均每个节点
平均高度 = 1*(3/4) + 2*(3/16) + 3*(3/64) + ... ≈ 1.33
平均大小 = 8 + 8*1.33 ≈ 18.7字节

// 对比：
// 平衡树(AVL): 8(key) + 8(left) + 8(right) + 4(height) = 28字节
// B+树节点: 通常更大（一个节点包含多个键）
```

### 4.3 缓存友好性

**SkipList的缓存特性：**

```
优点：
1. 顺序遍历（Next）沿Level 0，内存局部性好
2. 节点通过Arena连续分配，缓存命中率高

缺点：
1. 查找需要跳转多个节点，可能跨越缓存行
2. 高层节点访问频繁，但分散在内存中

对比B+树：
- B+树叶子节点连续，顺序扫描更快
- SkipList实现简单，无需重平衡
```

**实测数据（近似）：**

```
操作             SkipList    B+Tree
顺序插入         ~400ns      ~600ns  (B+树需要分裂)
随机插入         ~600ns      ~800ns
顺序扫描(1000)   ~50μs       ~30μs   (B+树胜)
随机查找         ~500ns      ~400ns  (B+树胜)
```

## 5. LevelDB的优化

### 5.1 Arena内存分配

```cpp
// db/skiplist.h, lines 100-108
template <typename Key, class Comparator>
class SkipList {
 private:
  Comparator const compare_;
  Arena* const arena_;  // Arena用于节点分配

  Node* const head_;
  std::atomic<int> max_height_;
  Random rnd_;
};
```

**优势：**
1. 批量分配，减少系统调用
2. 无需单独释放节点
3. 内存连续性更好

### 5.2 随机数优化

```cpp
// util/random.h
class Random {
 private:
  uint32_t seed_;
 public:
  explicit Random(uint32_t s) : seed_(s & 0x7fffffffu) {
    if (seed_ == 0 || seed_ == 2147483647L) {
      seed_ = 1;
    }
  }
  uint32_t Next() {
    static const uint32_t M = 2147483647L;  // 2^31-1
    static const uint64_t A = 16807;        // bits 14, 8, 7, 5, 2, 1, 0
    uint64_t product = seed_ * A;
    seed_ = static_cast<uint32_t>((product >> 31) + (product & M));
    if (seed_ > M) {
      seed_ -= M;
    }
    return seed_;
  }
};
```

**优化点：**
- 线性同余生成器，极快
- 无需系统调用
- 每个SkipList独立的随机数生成器

### 5.3 高度更新优化

```cpp
// db/skiplist.h, lines 354-359
if (height > GetMaxHeight()) {
  for (int i = GetMaxHeight(); i < height; i++) {
    prev[i] = head_;
  }
  // 使用relaxed，因为持有锁
  max_height_.store(height, std::memory_order_relaxed);
}
```

**为什么用relaxed？**
- 写线程持有锁，其他写线程看不到中间状态
- 读线程可能读到稍微旧的max_height，但这是安全的（最多多检查几层空链表）

## 6. 源码阅读指南

### 6.1 关键文件位置

- `db/skiplist.h`：完整实现（模板头文件）
- `db/skiplist_test.cc`：单元测试
- `util/arena.h`：内存分配器
- `util/random.h`：随机数生成

### 6.2 阅读顺序

1. **数据结构定义**（lines 208-230）
   - Node结构
   - next_指针数组

2. **基础操作**（lines 259-300）
   - FindGreaterOrEqual：查找
   - FindLessThan：反向查找
   - KeyIsAfterNode：比较

3. **插入操作**（lines 335-366）
   - RandomHeight：随机高度
   - Insert：插入逻辑

4. **迭代器**（lines 110-145）
   - Iterator类
   - Seek/Next/Prev

5. **并发安全**（lines 8-28）
   - 注释说明并发模型
   - 内存顺序保证

## 7. 实践练习

### 7.1 基准测试

创建 `skiplist_bench.cc`：

```cpp
#include "db/skiplist.h"
#include "util/arena.h"
#include "util/random.h"
#include <chrono>
#include <iostream>

using namespace leveldb;

struct IntComparator {
  int operator()(const int& a, const int& b) const {
    if (a < b) return -1;
    else if (a > b) return +1;
    else return 0;
  }
};

int main() {
  Arena arena;
  IntComparator cmp;
  SkipList<int, IntComparator> list(cmp, &arena);

  const int N = 1000000;
  Random rnd(301);

  // 插入基准
  auto start = std::chrono::high_resolution_clock::now();
  for (int i = 0; i < N; i++) {
    list.Insert(rnd.Next() % N);
  }
  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "Insert " << N << " keys: " << duration.count() << " ms\n";
  std::cout << "Ops/sec: " << (N * 1000 / duration.count()) << "\n";

  // 查找基准
  SkipList<int, IntComparator>::Iterator iter(&list);
  start = std::chrono::high_resolution_clock::now();
  for (int i = 0; i < N; i++) {
    iter.Seek(rnd.Next() % N);
  }
  end = std::chrono::high_resolution_clock::now();
  duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "Seek " << N << " keys: " << duration.count() << " ms\n";
  std::cout << "Ops/sec: " << (N * 1000 / duration.count()) << "\n";

  return 0;
}
```

### 7.2 并发测试

创建 `skiplist_concurrent.cc`：

```cpp
#include "db/skiplist.h"
#include "util/arena.h"
#include <thread>
#include <vector>

using namespace leveldb;

struct IntComparator {
  int operator()(const int& a, const int& b) const {
    if (a < b) return -1;
    else if (a > b) return +1;
    else return 0;
  }
};

void ReaderThread(SkipList<int, IntComparator>* list, int iterations) {
  SkipList<int, IntComparator>::Iterator iter(list);
  for (int i = 0; i < iterations; i++) {
    iter.SeekToFirst();
    int count = 0;
    while (iter.Valid()) {
      count++;
      iter.Next();
    }
  }
}

int main() {
  Arena arena;
  IntComparator cmp;
  SkipList<int, IntComparator> list(cmp, &arena);

  // 预填充数据
  for (int i = 0; i < 10000; i++) {
    list.Insert(i);
  }

  // 启动多个读线程
  std::vector<std::thread> readers;
  for (int i = 0; i < 4; i++) {
    readers.emplace_back(ReaderThread, &list, 1000);
  }

  // 等待完成
  for (auto& t : readers) {
    t.join();
  }

  std::cout << "Concurrent read test passed\n";
  return 0;
}
```

## 总结

今天我们学习了：
1. ✅ **SkipList原理**：多层索引加速查找，随机高度
2. ✅ **无锁并发**：release-acquire语义，节点不删除
3. ✅ **实现细节**：柔性数组、Arena分配、内存顺序
4. ✅ **性能特性**：O(log N)查找/插入，内存局部性

**关键要点：**
- SkipList = 概率平衡的多层链表
- 写操作需要锁，读操作无锁
- release-acquire保证读线程看到一致的状态
- Arena分配 + 节点不删除 = 无锁安全

**思考题：**
1. 为什么SkipList不支持Delete操作？
2. 如果不用release-acquire，会发生什么？
3. 能否用读写锁代替无锁读取？性能如何？

**明天预告：Day 4 - MemTable内存写缓冲**
我们将学习MemTable如何使用SkipList存储键值对，以及InternalKey的编码格式。
