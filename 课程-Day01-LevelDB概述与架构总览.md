# Day 1: LevelDB概述与架构总览

## 学习目标
- 理解LevelDB的设计目标和应用场景
- 掌握LSM-Tree的基本概念
- 了解LevelDB的整体架构
- 认识关键组件和数据流

## 1. LevelDB简介

### 1.1 什么是LevelDB？

LevelDB是Google开发的一个快速的键值存储库，提供从字符串键到字符串值的有序映射。它是一个**嵌入式数据库**，而不是客户端-服务器架构的数据库。

**核心特性：**
- 键和值可以是任意字节数组
- 数据按键排序存储
- 支持前向和后向迭代
- 支持批量原子写入
- 使用Snappy或Zstd压缩
- 不是SQL数据库，不支持关系模型

**典型应用场景：**
- Chrome浏览器（IndexedDB实现）
- Bitcoin Core（区块链索引）
- Minecraft Pocket Edition（世界存储）
- Riak（分布式数据库后端）

### 1.2 为什么需要LevelDB？

传统B-Tree数据库的问题：
```
问题1：随机写入性能差
- 每次写入需要磁盘随机寻道
- SSD也有写入放大问题

问题2：写入放大严重
- 修改一个字节可能需要重写整个页（4KB）
- 页分裂会导致更多写入

LevelDB的解决方案：LSM-Tree
- 所有写入都是顺序追加
- 后台异步整理数据
- 写入吞吐量提升10-100倍
```

## 2. LSM-Tree核心思想

### 2.1 什么是LSM-Tree？

LSM (Log-Structured Merge-Tree) = 日志结构合并树

**核心思想：将随机写转换为顺序写**

```
传统B-Tree：
  写入 → 查找位置 → 原地更新 → 刷盘
  特点：随机I/O，慢

LSM-Tree：
  写入 → 追加到日志 → 写入内存表 → 异步刷盘
  特点：顺序I/O，快
```

### 2.2 LSM-Tree的分层结构

```
Level 0:  [SSTable 1] [SSTable 2] [SSTable 3] [SSTable 4]
          ↑ 可能有重叠的键范围

Level 1:  [SSTable 5] [SSTable 6] [SSTable 7]
          ↑ 键范围不重叠，有序排列

Level 2:  [SSTable 8] [SSTable 9] ... [SSTable N]
          ↑ 键范围不重叠，总大小 ~10x Level 1

Level 3+: 指数增长（每层大小是上一层的10倍）
```

**关键规则：**
- Level 0: 最多4个文件，允许键范围重叠
- Level 1+: 键范围严格不重叠，二分查找
- Level N 大小上限: 10^N MB

### 2.3 Compaction（压缩整理）

当某一层文件过多时，会触发**Compaction**：

```
Level i 文件:    [100-200] [150-250]
                    ↓ 合并
Level i+1 文件:  [50-120] [220-300]
                    ↓ 产生新文件
结果:            [50-120] [100-250] [220-300]
                 （移除重复键，删除过期数据）
```

## 3. LevelDB整体架构

### 3.1 核心组件层次图

```
┌─────────────────────────────────────────┐
│         Public API (include/leveldb/)    │
│  DB, WriteBatch, Iterator, Snapshot...   │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│          DB Layer (db/)                  │
│                                          │
│  ┌──────────────┐    ┌───────────────┐  │
│  │   MemTable   │    │  Immutable    │  │
│  │  (SkipList)  │→   │   MemTable    │  │
│  └──────────────┘    └───────────────┘  │
│         ↓                    ↓           │
│  ┌──────────────────────────────────┐   │
│  │      VersionSet                  │   │
│  │  (管理所有SSTable的版本)          │   │
│  └──────────────────────────────────┘   │
│         ↓                                │
│  ┌──────────────────────────────────┐   │
│  │      Compaction Engine           │   │
│  │  (后台压缩整理)                   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│       Table Layer (table/)               │
│                                          │
│  ┌──────────────┐    ┌───────────────┐  │
│  │  SSTable     │    │  Block        │  │
│  │  (Sorted     │→   │  (Data +      │  │
│  │   String     │    │   Index)      │  │
│  │   Table)     │    └───────────────┘  │
│  └──────────────┘                       │
│         ↓                                │
│  ┌──────────────────────────────────┐   │
│  │   Filter Block (Bloom Filter)    │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│      Utilities (util/)                   │
│  Cache, Arena, Coding, CRC32C, Env...   │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│      OS Abstraction (port/)              │
│  Mutex, CondVar, Thread, File I/O...    │
└─────────────────────────────────────────┘
```

### 3.2 关键数据结构定位

| 组件 | 文件位置 | 作用 |
|------|---------|------|
| DBImpl | `db/db_impl.h` | 数据库主实现类 |
| MemTable | `db/memtable.h` | 内存写缓冲 |
| SkipList | `db/skiplist.h` | 有序跳表结构 |
| WriteBatch | `db/write_batch.h` | 批量写操作 |
| VersionSet | `db/version_set.h` | LSM-Tree版本管理 |
| Table | `table/table.h` | SSTable读取接口 |
| TableBuilder | `table/table_builder.h` | SSTable构建器 |
| BlockCache | `util/cache.h` | LRU块缓存 |
| Arena | `util/arena.h` | 内存池分配器 |

## 4. 数据流动路径

### 4.1 写入路径 (Write Path)

```
┌─────────────┐
│ Client Put  │
└──────┬──────┘
       ↓
┌──────────────────────────────────┐
│ 1. 写WAL日志 (Write-Ahead Log)    │  ← 持久化保证
│    文件: *.log                    │
└──────┬───────────────────────────┘
       ↓
┌──────────────────────────────────┐
│ 2. 插入MemTable (内存跳表)        │  ← O(log N) 插入
│    最大大小: ~4MB                 │
└──────┬───────────────────────────┘
       ↓
┌──────────────────────────────────┐
│ 3. MemTable满 → Immutable        │
└──────┬───────────────────────────┘
       ↓
┌──────────────────────────────────┐
│ 4. 后台线程: Minor Compaction     │
│    Immutable MemTable → Level-0   │
│    生成SSTable文件 (*.ldb)        │
└──────┬───────────────────────────┘
       ↓
┌──────────────────────────────────┐
│ 5. Level-0文件过多 → Major        │
│    Compaction                     │
│    合并到Level-1及更高层          │
└──────────────────────────────────┘
```

**关键点：**
- 写入先到WAL保证持久性（fsync可选）
- MemTable写入是内存操作，极快
- 后台异步刷盘，不阻塞写入

### 4.2 读取路径 (Read Path)

```
┌─────────────┐
│ Client Get  │
└──────┬──────┘
       ↓
┌──────────────────────────────────┐
│ 1. 查找MemTable                   │  ← 最新数据
└──────┬───────────────────────────┘
       ↓ 未找到
┌──────────────────────────────────┐
│ 2. 查找Immutable MemTable         │  ← 正在刷盘的数据
└──────┬───────────────────────────┘
       ↓ 未找到
┌──────────────────────────────────┐
│ 3. 查找Level-0 SSTables           │  ← 需要检查所有文件
│    (可能有重叠，都要查)            │     (因为键范围重叠)
└──────┬───────────────────────────┘
       ↓ 未找到
┌──────────────────────────────────┐
│ 4. 查找Level-1 SSTables           │  ← 二分查找定位文件
│    (键范围不重叠)                 │
└──────┬───────────────────────────┘
       ↓ 未找到
┌──────────────────────────────────┐
│ 5. 查找Level-2+ SSTables          │  ← 二分查找定位文件
└──────┬───────────────────────────┘
       ↓ 未找到
┌──────────────────────────────────┐
│ 返回 NotFound                     │
└──────────────────────────────────┘
```

**性能优化：**
- Bloom Filter: 快速判断键是否存在
- Block Cache: 缓存最近读取的数据块
- Table Cache: 缓存打开的SSTable文件

## 5. 文件类型

LevelDB数据库目录包含以下文件：

```
/path/to/database/
├── 000003.log          # Write-Ahead Log (WAL)
├── 000005.ldb          # SSTable 数据文件
├── 000007.ldb
├── CURRENT             # 指向当前MANIFEST文件
├── MANIFEST-000004     # 元数据：所有SSTable的信息
├── LOG                 # 当前日志（info/error）
├── LOG.old             # 旧日志
└── LOCK                # 文件锁（防止多进程同时打开）
```

**文件说明：**

| 文件类型 | 扩展名 | 作用 | 大小 |
|---------|--------|------|------|
| WAL | `.log` | 写前日志，保证持久性 | 最大4MB |
| SSTable | `.ldb` | 排序的键值对存储 | 最大2MB |
| MANIFEST | `MANIFEST-*` | 记录数据库版本信息 | 小文件 |
| CURRENT | 无 | 指向当前MANIFEST | 几十字节 |
| LOG | 无 | 运行日志 | 可配置 |
| LOCK | 无 | 进程锁 | 0字节 |

## 6. 性能特点对比

### 6.1 LevelDB vs 传统B-Tree数据库

| 操作 | LevelDB (LSM-Tree) | B-Tree | 说明 |
|------|-------------------|--------|------|
| 顺序写入 | ★★★★★ (极快) | ★★★☆☆ | LSM顺序追加 |
| 随机写入 | ★★★★☆ (快) | ★★☆☆☆ | B-Tree随机寻道 |
| 顺序读取 | ★★★★★ (极快) | ★★★★☆ | 都支持范围查询 |
| 随机读取 | ★★★☆☆ (中等) | ★★★★☆ | 需要查多层 |
| 空间效率 | ★★★★☆ (好) | ★★★☆☆ | 压缩效果好 |
| 写放大 | ★★★☆☆ (有) | ★★☆☆☆ (严重) | Compaction代价 |

### 6.2 典型性能数据

```
硬件环境:
- CPU: Intel Core i7
- 存储: SSD
- 内存: 16GB

LevelDB性能:
- 顺序写入:  ~400,000 ops/sec
- 随机写入:  ~100,000 ops/sec
- 顺序读取:  ~1,000,000 ops/sec
- 随机读取:  ~200,000 ops/sec

对比SQLite (B-Tree):
- 随机写入: ~1,000 ops/sec  ← 慢100倍！
```

## 7. 代码组织结构

```
leveldb/
├── include/leveldb/     # 公共API头文件
│   ├── db.h            # 主要接口
│   ├── options.h       # 配置选项
│   ├── slice.h         # 字节数组引用
│   └── ...
├── db/                 # 数据库核心实现
│   ├── db_impl.cc      # DBImpl主实现
│   ├── memtable.cc     # 内存表
│   ├── skiplist.h      # 跳表
│   ├── version_set.cc  # 版本管理
│   └── ...
├── table/              # SSTable实现
│   ├── table.cc        # 读取
│   ├── table_builder.cc# 构建
│   ├── block.cc        # 数据块
│   └── ...
├── util/               # 工具类
│   ├── cache.cc        # LRU缓存
│   ├── arena.cc        # 内存分配器
│   ├── bloom.cc        # 布隆过滤器
│   └── ...
├── port/               # 平台抽象层
│   ├── port_stdcxx.h   # 标准C++实现
│   └── ...
└── doc/                # 文档
    ├── index.md
    ├── impl.md
    └── ...
```

## 8. 设计哲学

### 8.1 核心设计原则

1. **简单性优先**
   - 代码库只有约1万行C++
   - 没有复杂的SQL解析器
   - 专注于核心键值存储

2. **性能导向**
   - 零拷贝设计 (Slice)
   - 无锁读取 (SkipList)
   - 批量写优化

3. **可靠性**
   - 无异常处理 (Status返回值)
   - 严格的错误检查
   - WAL保证持久性

4. **可移植性**
   - 抽象平台层 (port/)
   - 支持POSIX和Windows
   - C++11标准

### 8.2 权衡与取舍

**LevelDB的选择：**
```
✓ 优化写入性能 → 牺牲部分读性能
✓ 数据压缩 → CPU开销增加
✓ 后台整理 → 写放大存在
✓ 简单API → 功能有限（无SQL）
✓ 单机嵌入 → 无网络开销
```

## 9. 下一步学习路线

在接下来的课程中，我们将深入学习：

- **Day 2-4**: 基础数据结构（Slice, Status, SkipList, MemTable）
- **Day 5-7**: 存储引擎（WriteBatch, WAL, SSTable, 索引）
- **Day 8-10**: 核心机制（LSM-Tree, 读写路径, 缓存）
- **Day 11-12**: 高级主题（Compaction, 并发控制）
- **Day 13-14**: 性能优化与生产实践

## 10. 动手实践

### 10.1 编译LevelDB

```bash
cd /home/dev/leveldb
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
```

### 10.2 运行基准测试

```bash
# 写入100万条记录
./db_bench --benchmarks=fillseq --num=1000000

# 随机读取
./db_bench --benchmarks=readrandom --num=1000000 --use_existing_db=1

# 查看结果
./db_bench --benchmarks=fillseq,readseq,readrandom --num=100000
```

### 10.3 阅读示例代码

打开 `doc/index.md` 查看官方示例：
```cpp
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);

// 写入
db->Put(leveldb::WriteOptions(), "key1", "value1");

// 读取
std::string value;
db->Get(leveldb::ReadOptions(), "key1", &value);

delete db;
```

## 总结

今天我们学习了：
1. ✅ LevelDB是基于LSM-Tree的键值存储库
2. ✅ LSM-Tree通过分层结构将随机写转换为顺序写
3. ✅ 核心组件包括MemTable、SSTable、VersionSet、Compaction
4. ✅ 写入路径：WAL → MemTable → SSTable → Compaction
5. ✅ 读取路径：MemTable → Immutable → Level-0 → Level-1+
6. ✅ 优化写入性能，牺牲部分读性能

**思考题：**
1. 为什么LSM-Tree的写入比B-Tree快？
2. Level-0为什么允许文件重叠，而Level-1+不允许？
3. 如果只有写入没有读取，LevelDB会一直积累文件吗？

**明天预告：Day 2 - 核心数据结构 (Slice, Status, Options)**
我们将学习LevelDB的基础类型和接口设计。
