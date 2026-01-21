# IO优化课程-03: 磁盘IO深度优化

## 🎯 课程目标

本课程将深入讲解磁盘IO优化的所有技术,掌握:
- 磁盘硬件特性 (HDD vs SSD vs NVMe)
- Linux IO栈深度剖析
- 文件系统优化策略
- IO调度器选择与调优
- Direct IO与Buffered IO
- 异步IO (io_uring, libaio)
- 写入策略 (Write-Through, Write-Back, WAL)
- 数据库存储引擎优化

---

## 目录

1. [磁盘硬件特性](#1-磁盘硬件特性)
2. [Linux IO栈详解](#2-linux-io栈详解)
3. [文件系统优化](#3-文件系统优化)
4. [磁盘读取优化策略](#4-磁盘读取优化策略)
5. [IO模式选择](#5-io模式选择)
6. [异步IO技术](#6-异步io技术)
7. [写入优化策略](#7-写入优化策略)
8. [性能测量与调优](#8-性能测量与调优)
9. [实战案例](#9-实战案例)

---

## 1. 磁盘硬件特性

### 1.1 HDD (机械硬盘) 工作原理

```
HDD结构:

┌────────────────────────────────────────┐
│        读写头 (Read/Write Head)        │
│            ↓                           │
│    ┌───────────────────────┐          │
│    │      磁盘盘片         │          │
│    │   (Platters)          │          │
│    │                       │          │
│    │  ┌─────────────┐      │          │
│    │  │   扇区      │      │          │
│    │  │ (Sector)    │      │          │
│    │  │  512B/4KB   │      │          │
│    │  └─────────────┘      │          │
│    │                       │          │
│    │      磁道 (Track)      │          │
│    └───────────────────────┘          │
│           ↑                            │
│      主轴电机 (Spindle Motor)          │
│      7200 RPM / 15000 RPM             │
└────────────────────────────────────────┘

访问延迟分解:
- Seek Time (寻道时间): 4-10 ms
  - 磁头移动到目标磁道
- Rotational Latency (旋转延迟): 2-4 ms
  - 等待扇区旋转到磁头下方
  - 平均 = 60s / (2 * RPM)
  - 7200 RPM: 60/(2*7200) = 4.17 ms
- Transfer Time (传输时间): 0.1-1 ms
  - 实际读取数据

总延迟: 6-15 ms (随机访问)
```

**HDD性能特点:**
```
┌──────────────────┬──────────────┬──────────────┐
│ 操作类型         │ 性能         │ 原因         │
├──────────────────┼──────────────┼──────────────┤
│ 顺序读           │ 150-200 MB/s │ 无需寻道     │
│ 随机读 (4KB)     │ 100-200 IOPS │ 寻道开销大   │
│ 顺序写           │ 150-200 MB/s │ 写缓存       │
│ 随机写 (4KB)     │ 80-150 IOPS  │ 寻道+写入    │
└──────────────────┴──────────────┴──────────────┘

关键洞察:
- 顺序 vs 随机: 1000倍差距 (150 MB/s vs 0.4 MB/s)
- 优化核心: 减少随机访问,增加顺序访问
```

### 1.2 SSD (固态硬盘) 工作原理

```
SSD架构:

┌─────────────────────────────────────────────┐
│ Host Interface (PCIe/SATA)                  │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────┴───────────────────────────┐
│ SSD Controller (控制器)                     │
│ - FTL (Flash Translation Layer)            │
│ - Wear Leveling (磨损均衡)                  │
│ - Garbage Collection (垃圾回收)            │
│ - ECC (错误校正)                            │
└─────────────────┬───────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    ↓             ↓             ↓
┌────────┐   ┌────────┐   ┌────────┐
│ NAND   │   │ NAND   │   │ NAND   │
│ Flash  │   │ Flash  │   │ Flash  │
│ Chip 0 │   │ Chip 1 │   │ Chip N │
└────────┘   └────────┘   └────────┘

NAND Flash单元:
┌──────────────────┬─────────┬──────────┬────────┐
│ 类型             │ 每单元  │ 寿命     │ 速度   │
├──────────────────┼─────────┼──────────┼────────┤
│ SLC (Single)     │ 1 bit   │ 100K写入 │ 最快   │
│ MLC (Multi)      │ 2 bits  │ 10K写入  │ 快     │
│ TLC (Triple)     │ 3 bits  │ 3K写入   │ 中等   │
│ QLC (Quad)       │ 4 bits  │ 1K写入   │ 较慢   │
└──────────────────┴─────────┴──────────┴────────┘

关键特性:
- 读写单位: 页 (Page, 4KB-16KB)
- 擦除单位: 块 (Block, 256KB-4MB)
- 必须先擦除才能写入
```

**SSD性能特点:**
```
┌──────────────────┬──────────────┬──────────────┐
│ 操作类型         │ SATA SSD     │ NVMe SSD     │
├──────────────────┼──────────────┼──────────────┤
│ 顺序读           │ 550 MB/s     │ 3500 MB/s    │
│ 顺序写           │ 520 MB/s     │ 3000 MB/s    │
│ 随机读 (4KB)     │ 90K IOPS     │ 600K IOPS    │
│ 随机写 (4KB)     │ 80K IOPS     │ 550K IOPS    │
│ 延迟             │ 100 μs       │ 10-20 μs     │
└──────────────────┴──────────────┴──────────────┘

优势:
- 随机访问快: 无寻道时间
- 延迟低: 10-100 μs vs 10 ms
- 高IOPS: 100K+ vs 200
```

### 1.3 Write Amplification (写放大)

```cpp
// SSD的写放大问题示例

/*
原始写入: 4KB
实际写入: 可能是 256KB+

原因:
1. Read-Modify-Write (读-改-写)
   - 无法原地更新
   - 必须读取整个块,修改,擦除,重新写入

2. Garbage Collection (垃圾回收)
   - 回收无效页时,需要移动有效页
   - 额外的写入

示例:
┌─────────────────────────────────────────┐
│ 块 (Block) - 256KB                      │
├────┬────┬────┬────┬────┬────┬────┬────┤
│ P0 │ P1 │ P2 │ P3 │ P4 │ P5 │ P6 │ P7 │
│Valid│Inv│Valid│Inv│Valid│Inv│Valid│Inv│
└────┴────┴────┴────┴────┴────┴────┴────┘

写入新数据到P1位置:
1. 分配新页P8 (写入新数据)
2. 标记P1为Invalid
3. 当块满时,GC触发:
   - 读取P0,P2,P4,P6 (有效页)
   - 写入到新块
   - 擦除旧块

写放大 = 实际写入 / 用户写入
       = (4KB + 4*4KB) / 4KB
       = 5x
*/

// 减少写放大的策略
void MinimizeWriteAmplification() {
  // 1. 批量写入
  const int BATCH_SIZE = 1024 * 1024;  // 1MB
  char buffer[BATCH_SIZE];

  // 收集数据
  int offset = 0;
  for (auto& item : items) {
    memcpy(buffer + offset, &item, sizeof(item));
    offset += sizeof(item);

    if (offset >= BATCH_SIZE) {
      write(fd, buffer, offset);
      offset = 0;
    }
  }

  // 2. 对齐写入 (页边界)
  posix_memalign(&buffer, 4096, size);

  // 3. 使用TRIM/DISCARD
  // 通知SSD哪些块可以擦除
  #ifdef __linux__
  ioctl(fd, BLKDISCARD, &range);
  #endif
}
```

---

## 2. Linux IO栈详解

### 2.1 完整IO路径

```
应用程序写入数据的完整路径:

┌─────────────────────────────────────────────┐
│ 用户空间 (User Space)                       │
│ ┌─────────────────────────────────────────┐ │
│ │ Application: write(fd, buf, size)       │ │
│ └───────────────┬─────────────────────────┘ │
└─────────────────┼───────────────────────────┘
                  │ System Call
┌─────────────────┼───────────────────────────┐
│ 内核空间        ↓                            │
│ ┌─────────────────────────────────────────┐ │
│ │ VFS层: sys_write()                      │ │
│ │ - file descriptor → inode               │ │
│ │ - 权限检查                               │ │
│ └───────────────┬─────────────────────────┘ │
│                 ↓                            │
│ ┌─────────────────────────────────────────┐ │
│ │ Page Cache (页缓存)                     │ │
│ │ - find_or_create_page()                 │ │
│ │ - copy_from_user(page, buf, size)       │ │
│ │ - mark_page_dirty()                     │ │
│ └───────────────┬─────────────────────────┘ │
│                 │ (异步写回)                 │
│                 ↓                            │
│ ┌─────────────────────────────────────────┐ │
│ │ 文件系统层 (ext4/xfs/btrfs)             │ │
│ │ - 日志 (Journal)                        │ │
│ │ - 元数据更新                             │ │
│ │ - 数据块分配                             │ │
│ └───────────────┬─────────────────────────┘ │
│                 ↓                            │
│ ┌─────────────────────────────────────────┐ │
│ │ Block Layer (块层)                      │ │
│ │ - bio (block IO) 请求                   │ │
│ │ - IO合并 (merge)                        │ │
│ │ - IO调度器 (scheduler)                  │ │
│ └───────────────┬─────────────────────────┘ │
│                 ↓                            │
│ ┌─────────────────────────────────────────┐ │
│ │ 设备驱动 (SCSI/NVMe/SATA)               │ │
│ │ - DMA设置                               │ │
│ │ - 命令队列                               │ │
│ └───────────────┬─────────────────────────┘ │
└─────────────────┼───────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 硬件层                                       │
│ ┌─────────────────────────────────────────┐ │
│ │ 磁盘控制器                               │ │
│ │ - 命令处理                               │ │
│ │ - 数据传输                               │ │
│ └───────────────┬─────────────────────────┘ │
│                 ↓                            │
│            物理磁盘                          │
└─────────────────────────────────────────────┘
```

### 2.2 Page Cache 详解

```cpp
/*
Page Cache工作机制:

1. 读取路径:
   read() → 检查Page Cache
            ├─ Hit: 直接返回 (快!)
            └─ Miss:
                ├─ 同步读取磁盘
                ├─ 填充Page Cache
                ├─ 预读后续页 (readahead)
                └─ 返回数据

2. 写入路径:
   write() → 写入Page Cache (快速返回)
             └─ 标记页为"脏" (dirty)
                 └─ 后台线程 (pdflush/flush) 异步写回
                     ├─ 每30秒
                     ├─ 脏页超过阈值
                     └─ sync/fsync调用
*/

// 查看Page Cache统计
void PrintPageCacheStats() {
  std::ifstream meminfo("/proc/meminfo");
  std::string line;

  while (std::getline(meminfo, line)) {
    if (line.find("Cached:") != std::string::npos ||
        line.find("Dirty:") != std::string::npos ||
        line.find("Writeback:") != std::string::npos) {
      std::cout << line << "\n";
    }
  }
}

// 输出示例:
// Cached:         16384000 kB  ← Page Cache大小
// Dirty:            102400 kB  ← 脏页 (待写回)
// Writeback:          2048 kB  ← 正在写回

// 控制Page Cache行为
void ControlPageCache() {
  // 1. 立即刷新脏页
  sync();  // 刷新所有文件系统

  int fd = open("file.dat", O_WRONLY);
  fsync(fd);      // 刷新特定文件
  fdatasync(fd);  // 刷新数据 (不含元数据,更快)

  // 2. 清空Page Cache (需要root)
  // echo 3 > /proc/sys/vm/drop_caches

  // 3. 建议内核丢弃缓存
  posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);

  // 4. 禁用Page Cache (Direct IO)
  fd = open("file.dat", O_RDWR | O_DIRECT);
}
```

### 2.3 IO调度器

```
Linux IO调度器对比:

┌──────────────┬────────────────┬────────────────┬────────────┐
│ 调度器       │ 策略           │ 适用场景       │ 延迟       │
├──────────────┼────────────────┼────────────────┼────────────┤
│ noop         │ 先进先出 (FIFO)│ SSD/NVMe       │ 最低       │
│ (none)       │ 无合并无排序   │ 随机访问       │            │
├──────────────┼────────────────┼────────────────┼────────────┤
│ deadline     │ 防止饥饿       │ 数据库         │ 低         │
│              │ 读优先         │ 延迟敏感       │            │
├──────────────┼────────────────┼────────────────┼────────────┤
│ cfq          │ 公平队列       │ 桌面系统       │ 中等       │
│ (Complete    │ 每进程队列     │ 多任务         │            │
│  Fair Queue) │ 时间片轮转     │                │            │
├──────────────┼────────────────┼────────────────┼────────────┤
│ bfq          │ 预算公平       │ 桌面/多媒体    │ 低(交互)   │
│ (Budget Fair │ IO带宽公平     │ 交互式应用     │            │
│  Queueing)   │                │                │            │
├──────────────┼────────────────┼────────────────┼────────────┤
│ kyber        │ 多队列         │ 现代SSD        │ 极低       │
│              │ 延迟感知       │ NVMe           │            │
├──────────────┼────────────────┼────────────────┼────────────┤
│ mq-deadline  │ 多队列deadline │ 现代SSD        │ 低         │
│              │ (blk-mq)       │ 高并发         │            │
└──────────────┴────────────────┴────────────────┴────────────┘

查看和设置调度器:
```bash
# 查看当前调度器
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# 修改调度器
echo none > /sys/block/sda/queue/scheduler  # SSD推荐
echo mq-deadline > /sys/block/sda/queue/scheduler

# 永久修改 (systemd)
# /etc/udev/rules.d/60-scheduler.rules:
# ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/scheduler}="mq-deadline"
```

**调度器选择指南:**
```
HDD:
- 推荐: mq-deadline 或 bfq
- 原因: 需要排序以减少寻道

SSD (SATA):
- 推荐: mq-deadline 或 kyber
- 原因: 低延迟,但仍需适度调度

NVMe SSD:
- 推荐: none (无调度器)
- 原因: 设备已足够快,调度开销>收益

数据库:
- 推荐: mq-deadline
- 原因: 读优先,防止写入阻塞读取
```

---

## 3. 文件系统优化

### 3.1 文件系统对比

```
┌──────────┬───────────┬───────────┬───────────┬───────────┐
│ 特性     │ ext4      │ XFS       │ Btrfs     │ F2FS      │
├──────────┼───────────┼───────────┼───────────┼───────────┤
│ 适用场景 │ 通用      │ 大文件    │ 高级特性  │ SSD/Flash │
│          │           │ 高性能    │ 快照/压缩 │           │
├──────────┼───────────┼───────────┼───────────┼───────────┤
│ 大文件   │ 好        │ 最佳      │ 好        │ 中等      │
│ 性能     │           │           │           │           │
├──────────┼───────────┼───────────┼───────────┼───────────┤
│ 小文件   │ 最佳      │ 中等      │ 好        │ 最佳      │
│ 性能     │           │           │           │           │
├──────────┼───────────┼───────────┼───────────┼───────────┤
│ 元数据   │ 日志      │ 日志      │ COW       │ 日志      │
│ 一致性   │           │           │           │           │
├──────────┼───────────┼───────────┼───────────┼───────────┤
│ 在线扩展 │ 是        │ 是        │ 是        │ 否        │
├──────────┼───────────┼───────────┼───────────┼───────────┤
│ 最大文件 │ 16TB      │ 8EB       │ 16EB      │ 3.94TB    │
│ 大小     │           │           │           │           │
└──────────┴───────────┴───────────┴───────────┴───────────┘
```

### 3.2 ext4优化

```bash
# 创建优化的ext4文件系统
mkfs.ext4 \
  -O ^has_journal \      # 禁用日志 (SSD,风险自负)
  -O extent \            # 使用extent (默认)
  -O flex_bg \           # 弹性块组
  -E lazy_itable_init=0,lazy_journal_init=0 \  # 初始化表
  -b 4096 \              # 4KB块大小
  -i 16384 \             # inode比率 (每16KB一个)
  -m 1 \                 # 保留空间 1% (默认5%)
  /dev/sdb1

# 挂载选项优化
mount -t ext4 \
  -o noatime \           # 不更新访问时间 (重要!)
  -o nodiratime \        # 不更新目录访问时间
  -o data=writeback \    # 数据不经日志 (最快,最危险)
  # -o data=ordered \    # 数据在元数据前 (默认,推荐)
  # -o data=journal \    # 所有数据经日志 (最慢,最安全)
  -o commit=60 \         # 60秒提交一次 (默认5秒)
  -o barrier=0 \         # 禁用写屏障 (UPS环境)
  /dev/sdb1 /mnt/data

# noatime的影响
# - 每次读取文件都会更新inode的访问时间
# - 造成额外的写入
# - 禁用后性能提升: 5-30% (取决于工作负载)
```

### 3.3 XFS优化

```bash
# 创建优化的XFS文件系统
mkfs.xfs \
  -b size=4096 \         # 4KB块
  -d agcount=32 \        # 分配组数量 (并发性)
  -d sunit=128 \         # 条带单元 (RAID)
  -d swidth=1024 \       # 条带宽度
  -l size=128m \         # 日志大小
  -l lazy-count=1 \      # 延迟计数
  /dev/sdb1

# 挂载选项
mount -t xfs \
  -o noatime \
  -o nodiratime \
  -o logbufs=8 \         # 日志缓冲区数量
  -o logbsize=256k \     # 日志缓冲区大小
  -o nobarrier \         # 禁用写屏障
  -o inode64 \           # 64位inode (大文件系统)
  -o largeio \           # 优化大IO
  -o swalloc \           # 条带化分配
  /dev/sdb1 /mnt/data

# XFS的优势
# - Allocation Groups: 并发分配
# - Delayed Allocation: 延迟分配,减少碎片
# - 大文件性能卓越
```

---
## 4. 磁盘读取优化策略

### 4.1 预读（Readahead）机制

**Linux内核预读原理：**

```
预读工作流程：

1. 应用发起read()调用
   ↓
2. 内核检测顺序访问模式
   ↓
3. 触发预读：提前读取后续数据到Page Cache
   ↓
4. 应用后续读取直接从Page Cache获取（快！）

预读窗口大小：
- 初始窗口：128KB
- 顺序访问时，窗口指数增长：256KB → 512KB → 1MB → ...
- 最大窗口：可配置（默认128KB）

查看/设置预读大小：
```bash
# 查看当前预读大小（单位：512字节扇区）
blockdev --getra /dev/sda
# 输出：256（= 128KB）

# 设置预读大小为8MB
blockdev --setra 16384 /dev/sda  # 16384 * 512 = 8MB

# 永久设置
echo 'blockdev --setra 16384 /dev/sda' >> /etc/rc.local
```

**应用层控制预读：**

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>

// 方法1：posix_fadvise 建议内核预读
void OptimizedSequentialRead(const char* filename) {
  int fd = open(filename, O_RDONLY);
  if (fd < 0) return;

  // 建议1：顺序访问（触发激进预读）
  posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);

  // 建议2：预读特定范围
  off_t offset = 0;
  size_t length = 10 * 1024 * 1024;  // 预读10MB
  posix_fadvise(fd, offset, length, POSIX_FADV_WILLNEED);

  // 建议3：读完即丢弃（减少cache污染）
  posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);

  char buffer[4096];
  while (read(fd, buffer, sizeof(buffer)) > 0) {
    process(buffer);
  }

  close(fd);
}

// 方法2：madvise（用于mmap）
void OptimizedMmapRead(const char* filename, size_t filesize) {
  int fd = open(filename, O_RDONLY);
  void* mapped = mmap(nullptr, filesize, PROT_READ, MAP_PRIVATE, fd, 0);

  // 建议顺序访问
  madvise(mapped, filesize, MADV_SEQUENTIAL);

  // 预读全部内容
  madvise(mapped, filesize, MADV_WILLNEED);

  // 处理数据
  process_mmap_data(mapped, filesize);

  munmap(mapped, filesize);
  close(fd);
}

// 方法3：readahead系统调用（显式预读）
#define _GNU_SOURCE
#include <fcntl.h>

void ExplicitReadahead(int fd, off_t offset, size_t count) {
  // 显式请求预读
  readahead(fd, offset, count);

  // 稍后读取时，数据已在cache
  lseek(fd, offset, SEEK_SET);
  char buffer[4096];
  read(fd, buffer, count);  // 从cache读取，快！
}
```

**预读策略对比：**

```cpp
#include <iostream>
#include <chrono>
#include <fcntl.h>
#include <unistd.h>

// 无预读：每次读取都触发磁盘IO
void ReadWithoutPrefetch(const char* filename, size_t filesize) {
  int fd = open(filename, O_RDONLY | O_DIRECT);  // Direct IO，绕过cache
  char* buffer = aligned_alloc(512, 4096);

  auto start = std::chrono::high_resolution_clock::now();

  for (off_t offset = 0; offset < filesize; offset += 4096) {
    pread(fd, buffer, 4096, offset);
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "Without prefetch: " << duration.count() << " ms" << std::endl;

  free(buffer);
  close(fd);
}

// 有预读：内核自动预读后续数据
void ReadWithPrefetch(const char* filename, size_t filesize) {
  int fd = open(filename, O_RDONLY);  // Buffered IO，自动预读
  posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);

  char buffer[4096];
  auto start = std::chrono::high_resolution_clock::now();

  while (read(fd, buffer, sizeof(buffer)) > 0) {
    // 处理数据
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "With prefetch: " << duration.count() << " ms" << std::endl;

  close(fd);
}

// 测试结果（1GB文件，HDD）：
// Without prefetch: 15000 ms（随机读，大量寻道）
// With prefetch:     8000 ms（顺序读，最小化寻道）
// 性能提升：1.9倍
```

### 4.2 索引与元数据优化

**B+树索引读取优化：**

```cpp
// 优化1：索引块缓存
class BTreeIndex {
  struct Node {
    static constexpr int FANOUT = 256;  // 扇出大小
    int num_keys;
    int64_t keys[FANOUT];
    union {
      Node* children[FANOUT + 1];  // 内部节点
      int64_t offsets[FANOUT];     // 叶子节点：数据文件偏移
    };
  };

  LRUCache<int64_t, Node*> node_cache_;  // 缓存热点节点

public:
  int64_t Lookup(int64_t key) {
    Node* node = root_;

    // 自顶向下遍历
    while (!node->is_leaf) {
      // 检查cache
      if (auto* cached = node_cache_.Get(node->id)) {
        node = cached;
      } else {
        // Cache miss：从磁盘读取
        node = LoadNodeFromDisk(node->id);
        node_cache_.Put(node->id, node);
      }

      // 二分查找
      int pos = std::lower_bound(node->keys, node->keys + node->num_keys, key)
                - node->keys;
      node = node->children[pos];
    }

    // 叶子节点：返回数据偏移
    int pos = std::lower_bound(node->keys, node->keys + node->num_keys, key)
              - node->keys;
    if (pos < node->num_keys && node->keys[pos] == key) {
      return node->offsets[pos];
    }

    return -1;  // Not found
  }

  // 优化2：批量范围查询（顺序读）
  std::vector<int64_t> RangeLookup(int64_t start, int64_t end) {
    std::vector<int64_t> results;

    // 1. 找到起始叶子节点
    Node* leaf = FindLeaf(start);

    // 2. 顺序遍历叶子节点链表
    while (leaf != nullptr) {
      for (int i = 0; i < leaf->num_keys; i++) {
        if (leaf->keys[i] >= start && leaf->keys[i] < end) {
          results.push_back(leaf->offsets[i]);
        }
        if (leaf->keys[i] >= end) {
          return results;
        }
      }

      // 下一个叶子节点（顺序读，cache友好）
      leaf = leaf->next;
    }

    return results;
  }
};
```

**布隆过滤器加速查找：**

```cpp
#include <vector>
#include <string>

// 布隆过滤器：快速判断key可能存在
class BloomFilter {
  std::vector<uint8_t> bits_;
  int num_hashes_;

public:
  BloomFilter(size_t num_entries, double false_positive_rate = 0.01) {
    // 计算bit数组大小
    double ln2 = 0.693147180559945309417;
    size_t num_bits = -num_entries * log(false_positive_rate) / (ln2 * ln2);

    // 计算hash函数个数
    num_hashes_ = (num_bits / num_entries) * ln2;

    bits_.resize((num_bits + 7) / 8, 0);
  }

  void Insert(const std::string& key) {
    for (int i = 0; i < num_hashes_; i++) {
      uint64_t hash = Hash(key, i);
      size_t bit_pos = hash % (bits_.size() * 8);
      bits_[bit_pos / 8] |= (1 << (bit_pos % 8));
    }
  }

  bool MayContain(const std::string& key) const {
    for (int i = 0; i < num_hashes_; i++) {
      uint64_t hash = Hash(key, i);
      size_t bit_pos = hash % (bits_.size() * 8);
      if ((bits_[bit_pos / 8] & (1 << (bit_pos % 8))) == 0) {
        return false;  // 确定不存在
      }
    }
    return true;  // 可能存在（假阳性）
  }

private:
  uint64_t Hash(const std::string& key, int seed) const {
    // MurmurHash3或xxHash
    uint64_t h = seed;
    for (char c : key) {
      h = h * 31 + c;
    }
    return h;
  }
};

// 使用Bloom Filter优化磁盘读取
class SSTableReader {
  BloomFilter bloom_filter_;
  int data_fd_;

public:
  std::string Get(const std::string& key) {
    // 1. 先查bloom filter（内存操作，极快）
    if (!bloom_filter_.MayContain(key)) {
      return "";  // 确定不存在，避免磁盘读取！
    }

    // 2. 可能存在，查询索引
    int64_t offset = index_->Lookup(key);
    if (offset < 0) {
      return "";  // 假阳性
    }

    // 3. 从磁盘读取数据
    return ReadFromDisk(data_fd_, offset);
  }
};

// 效果：
// - 假阴性率：0%（如果说不存在，就一定不存在）
// - 假阳性率：1%（如果说存在，有1%概率不存在）
// - 节省磁盘IO：99%的"不存在"查询被过滤
```

### 4.3 读缓存分层策略

**多级缓存架构：**

```cpp
// L1: 热点记录缓存（LRU，内存）
// L2: 块缓存（LRU，内存）
// L3: Page Cache（OS管理）
// L4: SSD缓存（可选）
// L5: HDD存储

class TieredCache {
  // L1: 记录级缓存（最热数据）
  LRUCache<std::string, std::string> record_cache_{1000};  // 1000条记录

  // L2: 块级缓存（热数据块）
  LRUCache<int64_t, Block*> block_cache_{1000};  // 1000个块

  int data_fd_;

public:
  std::string Get(const std::string& key) {
    // 1. 查L1缓存
    if (auto* value = record_cache_.Get(key)) {
      return *value;  // 命中，最快！
    }

    // 2. 查L2块缓存
    int64_t block_offset = GetBlockOffset(key);
    Block* block = nullptr;

    if (auto* cached_block = block_cache_.Get(block_offset)) {
      block = cached_block;  // 命中L2
    } else {
      // 3. L2 miss：从磁盘读取（会经过Page Cache）
      block = ReadBlockFromDisk(block_offset);
      block_cache_.Put(block_offset, block);
    }

    // 4. 在块内查找
    std::string value = block->Get(key);

    // 5. 插入L1缓存
    if (!value.empty()) {
      record_cache_.Put(key, value);
    }

    return value;
  }

  // 预热缓存：启动时加载热数据
  void WarmupCache(const std::vector<std::string>& hot_keys) {
    for (const auto& key : hot_keys) {
      std::string value = Get(key);  // 加载到缓存
    }
  }
};
```

**缓存替换策略对比：**

```cpp
// LRU (Least Recently Used) - 传统策略
class LRUCache {
  std::list<std::pair<Key, Value>> items_;
  std::unordered_map<Key, decltype(items_)::iterator> index_;
  size_t capacity_;

public:
  Value* Get(const Key& key) {
    auto it = index_.find(key);
    if (it == index_.end()) {
      return nullptr;  // Miss
    }

    // 移到链表头（最近使用）
    items_.splice(items_.begin(), items_, it->second);
    return &it->second->second;
  }

  void Put(const Key& key, const Value& value) {
    // 驱逐最久未使用的项
    if (items_.size() >= capacity_) {
      auto& evict = items_.back();
      index_.erase(evict.first);
      items_.pop_back();
    }

    items_.push_front({key, value});
    index_[key] = items_.begin();
  }
};

// LFU (Least Frequently Used) - 频率优先
class LFUCache {
  struct Entry {
    Value value;
    int freq;
  };

  std::unordered_map<Key, Entry> cache_;
  size_t capacity_;

public:
  Value* Get(const Key& key) {
    auto it = cache_.find(key);
    if (it == cache_.end()) {
      return nullptr;
    }

    it->second.freq++;  // 增加访问频率
    return &it->second.value;
  }

  void Put(const Key& key, const Value& value) {
    if (cache_.size() >= capacity_) {
      // 驱逐频率最低的项
      auto min_it = std::min_element(
          cache_.begin(), cache_.end(),
          [](const auto& a, const auto& b) {
            return a.second.freq < b.second.freq;
          });
      cache_.erase(min_it);
    }

    cache_[key] = {value, 1};
  }
};

// ARC (Adaptive Replacement Cache) - 自适应
// 结合LRU和LFU优点，动态调整策略
// PostgreSQL使用的缓存算法
```

### 4.4 批量读取与合并

**问题：小随机读的低效：**

```cpp
// 不好：1000次小随机读
void RandomReads_Bad(const std::vector<int64_t>& offsets) {
  int fd = open("datafile.dat", O_RDONLY);

  for (int64_t offset : offsets) {
    char buffer[4096];
    pread(fd, buffer, 4096, offset);  // 1000次系统调用，1000次磁盘IO
    process(buffer);
  }

  close(fd);
  // HDD: 1000 * 10ms = 10秒
}

// 好：排序+顺序读
void RandomReads_Good(std::vector<int64_t> offsets) {
  int fd = open("datafile.dat", O_RDONLY);

  // 1. 排序偏移量
  std::sort(offsets.begin(), offsets.end());

  // 2. 顺序读取
  for (int64_t offset : offsets) {
    char buffer[4096];
    pread(fd, buffer, 4096, offset);  // 变为准顺序读
    process(buffer);
  }

  close(fd);
  // HDD: 大部分是顺序读，约2秒（5倍提升）
}

// 更好：批量读取+合并
void RandomReads_Best(std::vector<int64_t> offsets) {
  int fd = open("datafile.dat", O_RDONLY);

  // 1. 排序偏移量
  std::sort(offsets.begin(), offsets.end());

  // 2. 合并相邻读取
  std::vector<std::pair<int64_t, size_t>> merged_reads;
  int64_t start = offsets[0];
  size_t length = 4096;

  for (size_t i = 1; i < offsets.size(); i++) {
    if (offsets[i] - offsets[i-1] <= 64 * 1024) {  // 间隔<64KB，合并
      length = offsets[i] + 4096 - start;
    } else {
      merged_reads.push_back({start, length});
      start = offsets[i];
      length = 4096;
    }
  }
  merged_reads.push_back({start, length});

  // 3. 批量读取
  for (auto [offset, len] : merged_reads) {
    char* buffer = new char[len];
    pread(fd, buffer, len, offset);  // 大块读取

    // 提取需要的数据
    for (int64_t off : offsets) {
      if (off >= offset && off < offset + len) {
        process(buffer + (off - offset));
      }
    }

    delete[] buffer;
  }

  close(fd);
  // 性能：10-20倍提升（减少IO次数）
}
```

**Read-ahead聚合优化：**

```cpp
// 智能预读：根据访问模式动态调整
class SmartReadahead {
  struct AccessPattern {
    std::vector<int64_t> recent_offsets;
    int sequential_count = 0;
    int random_count = 0;
  };

  AccessPattern pattern_;
  int fd_;

public:
  void Read(int64_t offset, char* buffer, size_t size) {
    // 1. 记录访问模式
    pattern_.recent_offsets.push_back(offset);
    if (pattern_.recent_offsets.size() > 10) {
      pattern_.recent_offsets.erase(pattern_.recent_offsets.begin());
    }

    // 2. 检测访问模式
    bool is_sequential = CheckSequential();

    if (is_sequential) {
      // 3a. 顺序访问：激进预读
      size_t readahead_size = 1024 * 1024;  // 预读1MB
      posix_fadvise(fd_, offset, readahead_size, POSIX_FADV_WILLNEED);
      pattern_.sequential_count++;
    } else {
      // 3b. 随机访问：禁用预读（避免污染cache）
      posix_fadvise(fd_, offset, size, POSIX_FADV_RANDOM);
      pattern_.random_count++;
    }

    // 4. 实际读取
    pread(fd_, buffer, size, offset);
  }

private:
  bool CheckSequential() {
    if (pattern_.recent_offsets.size() < 3) {
      return false;
    }

    // 检查最近3次访问是否连续
    for (size_t i = 1; i < pattern_.recent_offsets.size(); i++) {
      int64_t delta = pattern_.recent_offsets[i] - pattern_.recent_offsets[i-1];
      if (delta < 0 || delta > 1024 * 1024) {  // 非顺序
        return false;
      }
    }

    return true;
  }
};
```

### 4.5 并发读取优化

**多线程并行读取：**

```cpp
#include <thread>
#include <vector>
#include <future>

// 单线程顺序读取
void SingleThreadRead(const char* filename, size_t filesize) {
  int fd = open(filename, O_RDONLY);
  char* buffer = new char[filesize];

  auto start = std::chrono::high_resolution_clock::now();

  read(fd, buffer, filesize);

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "Single thread: " << duration.count() << " ms" << std::endl;

  delete[] buffer;
  close(fd);
}

// 多线程并行读取（适合SSD/NVMe）
void MultiThreadRead(const char* filename, size_t filesize, int num_threads) {
  std::vector<std::thread> threads;
  std::vector<char*> buffers(num_threads);

  size_t chunk_size = filesize / num_threads;

  auto start = std::chrono::high_resolution_clock::now();

  for (int i = 0; i < num_threads; i++) {
    threads.emplace_back([&, i]() {
      int fd = open(filename, O_RDONLY);
      buffers[i] = new char[chunk_size];

      int64_t offset = i * chunk_size;
      size_t size = (i == num_threads - 1) ? (filesize - offset) : chunk_size;

      pread(fd, buffers[i], size, offset);

      close(fd);
    });
  }

  for (auto& t : threads) {
    t.join();
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

  std::cout << "Multi thread (" << num_threads << "): " << duration.count() << " ms"
            << std::endl;

  for (auto* buf : buffers) {
    delete[] buf;
  }
}

// 测试结果（1GB文件）：
// HDD:
//   Single thread: 8000 ms
//   Multi thread (4): 10000 ms（更慢！寻道开销）
//
// NVMe SSD:
//   Single thread: 1000 ms
//   Multi thread (4): 300 ms（3.3倍提升）
//
// 结论：SSD受益于并发读取，HDD不适合
```

---


## 5. IO模式选择

### 5.1 Buffered IO vs Direct IO

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <iostream>

// Buffered IO (默认模式)
void BufferedIOExample() {
  int fd = open("test.dat", O_RDWR | O_CREAT, 0644);

  char buf[4096];
  memset(buf, 'A', sizeof(buf));

  // 写入 → Page Cache → 异步写回磁盘
  write(fd, buf, sizeof(buf));
  // 函数立即返回 (数据在内存)

  // 读取 → Page Cache命中 → 立即返回
  lseek(fd, 0, SEEK_SET);
  read(fd, buf, sizeof(buf));
  // 非常快 (内存访问)

  close(fd);
}

// Direct IO (绕过Page Cache)
void DirectIOExample() {
  int fd = open("test.dat", O_RDWR | O_CREAT | O_DIRECT, 0644);

  // 必须对齐!
  void* buf;
  posix_memalign(&buf, 4096, 4096);  // 4KB对齐
  memset(buf, 'A', 4096);

  // 写入 → 直接到磁盘 → 阻塞直到完成
  ssize_t written = write(fd, buf, 4096);
  // 慢,但数据已持久化

  // 读取 → 直接从磁盘
  lseek(fd, 0, SEEK_SET);
  read(fd, buf, 4096);
  // 慢,无缓存

  free(buf);
  close(fd);
}

// 性能对比
void BenchmarkIOMode() {
  const size_t SIZE = 100 * 1024 * 1024;  // 100MB
  void* buf;
  posix_memalign(&buf, 4096, SIZE);
  memset(buf, 'A', SIZE);

  // Buffered IO
  auto start = std::chrono::high_resolution_clock::now();
  int fd1 = open("buffered.dat", O_RDWR | O_CREAT | O_TRUNC, 0644);
  write(fd1, buf, SIZE);
  fsync(fd1);  // 强制刷新
  close(fd1);
  auto end = std::chrono::high_resolution_clock::now();
  auto buffered_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  // Direct IO
  start = std::chrono::high_resolution_clock::now();
  int fd2 = open("direct.dat", O_RDWR | O_CREAT | O_TRUNC | O_DIRECT, 0644);
  write(fd2, buf, SIZE);
  close(fd2);
  end = std::chrono::high_resolution_clock::now();
  auto direct_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  std::cout << "Buffered IO: " << buffered_time.count() << " ms\n";
  std::cout << "Direct IO:   " << direct_time.count() << " ms\n";

  free(buf);

  // 预期输出 (SSD):
  // Buffered IO: 250 ms
  // Direct IO:   280 ms
  // (Direct IO稍慢,但更可控)
}
```

**Direct IO使用指南:**
```cpp
// 何时使用Direct IO?

// ✓ 适合场景:
// 1. 数据库 (InnoDB, PostgreSQL)
//    - 自己管理缓存
//    - 避免双重缓存
// 2. 大文件流式处理
//    - 视频编码
//    - 数据管道
// 3. 需要精确控制IO时机
//    - WAL (Write-Ahead Log)
//    - 检查点

// ✗ 不适合场景:
// 1. 小文件随机访问
// 2. 读多写少的场景
// 3. 通用应用程序

// Direct IO要求:
// - 缓冲区地址: 512字节或4KB对齐
// - 偏移量: 512字节或4KB对齐
// - 大小: 512字节或4KB的倍数

// 示例: 正确的Direct IO
void CorrectDirectIO() {
  int fd = open("file.dat", O_RDWR | O_DIRECT);

  // 对齐检查
  void* buf;
  size_t alignment = 4096;
  size_t size = 1024 * 1024;  // 1MB

  // 对齐分配
  if (posix_memalign(&buf, alignment, size) != 0) {
    perror("posix_memalign failed");
    return;
  }

  // 对齐的偏移量
  off_t offset = 4096 * 100;  // 400KB (对齐)

  // 对齐的大小
  size_t write_size = 4096 * 10;  // 40KB (对齐)

  pwrite(fd, buf, write_size, offset);

  free(buf);
  close(fd);
}
```

### 5.2 同步IO vs 异步IO

```cpp
// 同步IO (阻塞)
void SyncIOExample() {
  int fd = open("file.dat", O_RDONLY);
  char buf[4096];

  // 阻塞直到读取完成
  ssize_t n = read(fd, buf, sizeof(buf));
  // 线程在这里等待...

  close(fd);
}

// 异步IO (非阻塞)
#include <aio.h>

void AsyncIOExample() {
  int fd = open("file.dat", O_RDONLY);

  // 准备异步请求
  struct aiocb cb;
  memset(&cb, 0, sizeof(cb));
  cb.aio_fildes = fd;
  cb.aio_buf = malloc(4096);
  cb.aio_nbytes = 4096;
  cb.aio_offset = 0;

  // 发起异步读取 (立即返回)
  aio_read(&cb);

  // 做其他工作...
  DoOtherWork();

  // 等待完成
  while (aio_error(&cb) == EINPROGRESS) {
    usleep(1000);
  }

  // 获取结果
  ssize_t n = aio_return(&cb);

  free((void*)cb.aio_buf);
  close(fd);
}
```


## 扩展：磁盘数据布局与对齐

### 5.3 磁盘扇区对齐

**问题：未对齐的IO效率低**

```cpp
// 扇区大小：
// - 传统HDD：512字节
// - 高级格式HDD：4096字节（4KB）
// - SSD：4096字节（4KB物理扇区）

// 不好：未对齐的写入
void UnalignedDiskWrite(int fd) {
    char buffer[5000];  // 5KB
    memset(buffer, 'A', sizeof(buffer));

    // 写入到偏移量1000（未对齐）
    lseek(fd, 1000, SEEK_SET);
    write(fd, buffer, sizeof(buffer));

    // 问题：
    // - 跨越2个4KB扇区
    // - 需要读-修改-写（RMW）操作
    // - 性能降低50%
}

// 好：对齐的写入
void AlignedDiskWrite(int fd) {
    constexpr size_t SECTOR_SIZE = 4096;

    // 分配对齐的缓冲区
    void* buffer = nullptr;
    posix_memalign(&buffer, SECTOR_SIZE, SECTOR_SIZE);
    memset(buffer, 'A', SECTOR_SIZE);

    // 写入到对齐的偏移量
    lseek(fd, 0, SEEK_SET);  // 偏移量0（对齐）
    write(fd, buffer, SECTOR_SIZE);

    free(buffer);

    // 优势：
    // - 直接写入完整扇区
    // - 无需RMW
    // - 最优性能
}
```

**Direct IO的对齐要求：**

```cpp
#include <fcntl.h>
#include <unistd.h>

// Direct IO要求三重对齐
void DirectIOAlignment() {
    constexpr size_t ALIGNMENT = 4096;  // 4KB

    // 1. 地址对齐
    void* buffer = nullptr;
    posix_memalign(&buffer, ALIGNMENT, ALIGNMENT);

    // 2. 大小对齐
    size_t size = ALIGNMENT;  // 必须是4KB的整数倍

    // 3. 偏移对齐
    off_t offset = 0;  // 必须是4KB的整数倍

    int fd = open("test.dat", O_RDWR | O_DIRECT);

    // 所有参数都对齐
    pwrite(fd, buffer, size, offset);  // 成功

    // 任何一个未对齐都会失败
    // pwrite(fd, buffer + 1, size, offset);  // 失败：地址未对齐
    // pwrite(fd, buffer, size + 1, offset);  // 失败：大小未对齐
    // pwrite(fd, buffer, size, 1);           // 失败：偏移未对齐

    free(buffer);
    close(fd);
}

// 通用对齐辅助函数
size_t AlignUp(size_t value, size_t alignment) {
    return (value + alignment - 1) & ~(alignment - 1);
}

size_t AlignDown(size_t value, size_t alignment) {
    return value & ~(alignment - 1);
}

bool IsAligned(void* ptr, size_t alignment) {
    return ((uintptr_t)ptr & (alignment - 1)) == 0;
}
```

### 5.4 文件数据布局优化

**行式存储 vs 列式存储：**

```cpp
// 场景：存储用户数据到文件

// 方式1：行式存储（Row-oriented）
struct UserRowFormat {
    struct User {
        uint32_t id;
        char name[32];
        uint32_t age;
        uint64_t salary;
    };  // 48 bytes per user

    static void WriteRowFormat(const std::vector<User>& users, int fd) {
        // 连续写入每个用户的所有字段
        for (const auto& user : users) {
            write(fd, &user, sizeof(User));
        }
        // 布局：[user1][user2][user3]...
    }

    static std::vector<User> ReadAllUsers(int fd, size_t count) {
        std::vector<User> users(count);
        // 读取所有用户
        read(fd, users.data(), count * sizeof(User));
        return users;
    }

    static std::vector<uint32_t> ReadOnlyAges(int fd, size_t count) {
        std::vector<uint32_t> ages;
        User user;

        for (size_t i = 0; i < count; i++) {
            read(fd, &user, sizeof(User));  // 读取整个user（48字节）
            ages.push_back(user.age);       // 只需要age（4字节）
        }

        // 问题：读取了48字节，只用了4字节
        // IO放大：12倍
        return ages;
    }
};

// 方式2：列式存储（Column-oriented）
struct UserColumnFormat {
    struct UserTable {
        std::vector<uint32_t> ids;
        std::vector<std::string> names;
        std::vector<uint32_t> ages;
        std::vector<uint64_t> salaries;
    };

    static void WriteColumnFormat(const UserTable& table, int fd) {
        // 写入每一列
        size_t count = table.ids.size();

        // ID列
        write(fd, table.ids.data(), count * sizeof(uint32_t));

        // Name列
        for (const auto& name : table.names) {
            write(fd, name.c_str(), 32);
        }

        // Age列
        write(fd, table.ages.data(), count * sizeof(uint32_t));

        // Salary列
        write(fd, table.salaries.data(), count * sizeof(uint64_t));

        // 布局：[所有IDs][所有Names][所有Ages][所有Salaries]
    }

    static std::vector<uint32_t> ReadOnlyAges(int fd, size_t count) {
        // 计算age列的偏移
        off_t age_offset = count * sizeof(uint32_t)  // IDs
                         + count * 32;                 // Names

        // 直接seek到age列
        lseek(fd, age_offset, SEEK_SET);

        // 只读取age列
        std::vector<uint32_t> ages(count);
        read(fd, ages.data(), count * sizeof(uint32_t));

        // 只读取需要的数据，无IO放大！
        return ages;
    }
};

// 性能对比
void BenchmarkFileLayout() {
    constexpr size_t COUNT = 1000000;

    // 生成测试数据
    std::vector<UserRowFormat::User> users(COUNT);
    for (size_t i = 0; i < COUNT; i++) {
        users[i].id = i;
        users[i].age = rand() % 100;
        users[i].salary = 50000 + rand() % 50000;
    }

    // 行存储
    int fd_row = open("users_row.dat", O_RDWR | O_CREAT | O_TRUNC, 0644);
    UserRowFormat::WriteRowFormat(users, fd_row);
    close(fd_row);

    fd_row = open("users_row.dat", O_RDONLY);
    auto start = std::chrono::high_resolution_clock::now();
    auto ages_row = UserRowFormat::ReadOnlyAges(fd_row, COUNT);
    auto end = std::chrono::high_resolution_clock::now();
    auto row_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();
    close(fd_row);

    // 列存储
    UserColumnFormat::UserTable table;
    for (const auto& user : users) {
        table.ids.push_back(user.id);
        table.ages.push_back(user.age);
        table.salaries.push_back(user.salary);
    }

    int fd_col = open("users_col.dat", O_RDWR | O_CREAT | O_TRUNC, 0644);
    UserColumnFormat::WriteColumnFormat(table, fd_col);
    close(fd_col);

    fd_col = open("users_col.dat", O_RDONLY);
    start = std::chrono::high_resolution_clock::now();
    auto ages_col = UserColumnFormat::ReadOnlyAges(fd_col, COUNT);
    end = std::chrono::high_resolution_clock::now();
    auto col_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();
    close(fd_col);

    std::cout << "Row Format: " << row_time << " ms\n";
    std::cout << "Column Format: " << col_time << " ms\n";
    std::cout << "Speedup: " << (double)row_time / col_time << "x\n";
}

// 输出示例：
// Row Format: 500 ms (读取48MB)
// Column Format: 40 ms (读取4MB)
// Speedup: 12.5x
```

### 5.5 SSTable数据布局（LevelDB/RocksDB）

**SSTable文件格式：**

```
┌─────────────────────────────────────────────────┐
│ Data Block 1 (4KB)                              │
│ - Key-Value对（排序存储）                        │
│ - 压缩（Snappy/LZ4）                            │
├─────────────────────────────────────────────────┤
│ Data Block 2 (4KB)                              │
├─────────────────────────────────────────────────┤
│ Data Block 3 (4KB)                              │
├─────────────────────────────────────────────────┤
│ ...                                             │
├─────────────────────────────────────────────────┤
│ Filter Block (Bloom Filter)                     │
│ - 快速判断key是否可能存在                        │
├─────────────────────────────────────────────────┤
│ Meta Index Block                                │
│ - 指向Filter Block                              │
├─────────────────────────────────────────────────┤
│ Index Block                                     │
│ - 每个Data Block的offset + size                │
│ - 每个Block的最大key                            │
├─────────────────────────────────────────────────┤
│ Footer (48 bytes, 固定大小)                     │
│ - Meta Index Block offset/size                 │
│ - Index Block offset/size                      │
│ - Magic Number                                  │
└─────────────────────────────────────────────────┘

优化点：
1. 块对齐：每个block 4KB对齐（SSD友好）
2. 索引在尾部：可以mmap整个文件，index立即可用
3. Bloom Filter：避免不必要的读取
4. 压缩：减少IO量
```

**实现示例：**

```cpp
class SSTable {
    struct BlockHandle {
        uint64_t offset;
        uint64_t size;
    };

public:
    std::string Get(const std::string& key) {
        // 1. 查询Bloom Filter（内存）
        if (!bloom_filter_.MayContain(key)) {
            return "";  // 确定不存在，避免IO
        }

        // 2. 二分查找Index Block（内存）
        BlockHandle data_block_handle = index_block_.FindBlockHandle(key);

        // 3. 读取Data Block（磁盘IO）
        // 检查Block Cache
        std::string cache_key = MakeCacheKey(file_number_, data_block_handle.offset);
        Block* block = block_cache_.Lookup(cache_key);

        if (block == nullptr) {
            // Cache miss：从磁盘读取
            // 4KB对齐读取，充分利用SSD特性
            block = ReadBlock(data_block_handle.offset, data_block_handle.size);
            block_cache_.Insert(cache_key, block);
        }

        // 4. 在Block内二分查找
        return block->Get(key);
    }

private:
    Block* ReadBlock(uint64_t offset, uint64_t size) {
        // 确保4KB对齐
        assert(offset % 4096 == 0);

        // 分配对齐的缓冲区
        void* buffer = nullptr;
        posix_memalign(&buffer, 4096, AlignUp(size, 4096));

        // Direct IO读取
        pread(fd_, buffer, AlignUp(size, 4096), offset);

        // 解压缩
        std::string uncompressed;
        Snappy::Uncompress((char*)buffer, size, &uncompressed);

        free(buffer);

        return new Block(uncompressed);
    }
};
```

### 5.6 日志文件布局优化（WAL）

**问题：小写入的性能问题**

```cpp
// 不好：每次写入都fsync
void BadWALWrite(int fd, const std::string& log_entry) {
    write(fd, log_entry.c_str(), log_entry.size());
    fsync(fd);  // 每次都刷盘，慢！

    // 性能：~10ms per write（HDD）
    // 吞吐量：100 writes/sec
}

// 好：批量写入 + Group Commit
class WALWriter {
    int fd_;
    std::string buffer_;
    std::mutex mutex_;
    std::condition_variable cv_;
    std::atomic<bool> flush_pending_{false};
    std::thread flush_thread_;

public:
    WALWriter(int fd) : fd_(fd) {
        // 后台flush线程
        flush_thread_ = std::thread([this]() {
            while (true) {
                std::this_thread::sleep_for(std::chrono::milliseconds(10));
                Flush();
            }
        });
    }

    void Append(const std::string& log_entry) {
        std::unique_lock<std::mutex> lock(mutex_);

        // 添加到缓冲区
        buffer_.append(log_entry);

        // 缓冲区满，触发flush
        if (buffer_.size() >= 4096) {
            flush_pending_ = true;
            cv_.notify_one();
        }

        // 等待flush完成
        cv_.wait(lock, [this]() { return !flush_pending_.load(); });
    }

    void Flush() {
        std::unique_lock<std::mutex> lock(mutex_);

        if (buffer_.empty()) {
            return;
        }

        // 批量写入
        write(fd_, buffer_.data(), buffer_.size());
        fsync(fd_);  // 一次fsync，刷多个log entry

        buffer_.clear();
        flush_pending_ = false;
        cv_.notify_all();
    }
};

// 性能：
// - 10个log entry批量写入
// - 1次fsync
// - 吞吐量：1000 writes/sec（10倍提升）
```

**对齐的WAL布局：**

```
WAL文件格式（LevelDB风格）：

┌────────────────────────────────────────────┐
│ Block 0 (32KB)                             │
│ ┌────────────────────────────────────────┐ │
│ │ Record 1: [CRC][Length][Type][Data]   │ │
│ ├────────────────────────────────────────┤ │
│ │ Record 2: [CRC][Length][Type][Data]   │ │
│ ├────────────────────────────────────────┤ │
│ │ Record 3: [CRC][Length][Type][Data]   │ │
│ └────────────────────────────────────────┘ │
│ Padding to 32KB                            │
├────────────────────────────────────────────┤
│ Block 1 (32KB)                             │
│ ...                                        │
└────────────────────────────────────────────┘

优化点：
1. 32KB块对齐（文件系统友好）
2. CRC校验（数据完整性）
3. Record类型：Full/First/Middle/Last（支持大记录）
4. Padding填充（确保下个block对齐）
```

### 5.7 索引文件布局优化

**B+树磁盘布局：**

```cpp
class DiskBPlusTree {
    static constexpr size_t PAGE_SIZE = 4096;  // 4KB页

    struct Node {
        bool is_leaf;
        uint16_t num_keys;
        uint16_t parent_offset;  // 父节点在文件中的偏移（页号）

        // Internal node
        uint64_t keys[255];          // 255个key
        uint32_t children[256];      // 256个child指针（页号）

        // Leaf node
        uint64_t leaf_keys[255];
        char values[255][32];        // 255个value
        uint32_t next_leaf;          // 下一个叶子节点（链表）

        // 填充到4KB
        char padding[...];
    };

    static_assert(sizeof(Node) == PAGE_SIZE, "Node must be 4KB");

public:
    std::string Search(uint64_t key) {
        // 1. 读取root节点（第0页）
        Node root = ReadPage(0);

        // 2. 向下遍历（每次读取1页，4KB对齐）
        uint32_t page_id = 0;
        while (!root.is_leaf) {
            // 二分查找
            int pos = BinarySearch(root.keys, root.num_keys, key);
            page_id = root.children[pos];

            // 读取子节点
            root = ReadPage(page_id);
        }

        // 3. 叶子节点查找
        int pos = BinarySearch(root.leaf_keys, root.num_keys, key);
        if (pos < root.num_keys && root.leaf_keys[pos] == key) {
            return std::string(root.values[pos], 32);
        }

        return "";
    }

private:
    Node ReadPage(uint32_t page_id) {
        Node node;

        // 4KB对齐读取
        off_t offset = page_id * PAGE_SIZE;
        pread(fd_, &node, PAGE_SIZE, offset);

        return node;
    }
};

// 优化点：
// 1. 节点大小 = 页大小（4KB）：每次读取完整节点
// 2. 对齐访问：充分利用OS page cache
// 3. 高扇出：255-way tree，减少树高度
// 4. 叶子链表：范围查询高效
```

---

## 磁盘数据布局优化检查清单

### ✅ 对齐检查

- [ ] Direct IO使用4KB对齐（地址、大小、偏移）
- [ ] 数据块大小匹配扇区大小（512B/4KB）
- [ ] WAL记录对齐到块边界
- [ ] 索引节点对齐到页大小

### ✅ 布局检查

- [ ] 分析查询选择行存储或列存储
- [ ] 索引在文件尾部（快速加载）
- [ ] 使用Bloom Filter减少IO
- [ ] 热数据和冷数据分离存储

### ✅ 性能检查

- [ ] 避免读-修改-写（RMW）操作
- [ ] 批量写入减少fsync次数
- [ ] 压缩减少IO量
- [ ] 使用Block Cache减少磁盘访问

---


---

## 6. 异步IO技术

### 6.1 io_uring (现代异步IO)

```cpp
#include <liburing.h>
#include <iostream>
#include <vector>

// io_uring基础用法
class IOUringExample {
 public:
  IOUringExample() {
    // 初始化io_uring (队列深度32)
    io_uring_queue_init(32, &ring_, 0);
  }

  ~IOUringExample() {
    io_uring_queue_exit(&ring_);
  }

  void ReadFile(const char* path) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) {
      perror("open");
      return;
    }

    // 获取文件大小
    struct stat st;
    fstat(fd, &st);
    size_t file_size = st.st_size;

    // 分配缓冲区
    void* buffer = aligned_alloc(4096, file_size);

    // 获取SQE (Submission Queue Entry)
    struct io_uring_sqe* sqe = io_uring_get_sqe(&ring_);

    // 准备读取请求
    io_uring_prep_read(sqe, fd, buffer, file_size, 0);

    // 设置用户数据 (用于识别完成事件)
    io_uring_sqe_set_data(sqe, buffer);

    // 提交请求
    io_uring_submit(&ring_);

    // 等待完成
    struct io_uring_cqe* cqe;
    io_uring_wait_cqe(&ring_, &cqe);

    // 处理结果
    if (cqe->res < 0) {
      std::cerr << "Read failed: " << strerror(-cqe->res) << "\n";
    } else {
      std::cout << "Read " << cqe->res << " bytes\n";
      // 使用buffer中的数据...
    }

    // 标记完成事件已处理
    io_uring_cqe_seen(&ring_, cqe);

    free(buffer);
    close(fd);
  }

  // 批量读取 (展示高性能用法)
  void BatchRead(const std::vector<std::string>& files) {
    std::vector<int> fds;
    std::vector<void*> buffers;

    // 提交所有读取请求
    for (const auto& file : files) {
      int fd = open(file.c_str(), O_RDONLY);
      fds.push_back(fd);

      struct stat st;
      fstat(fd, &st);

      void* buf = aligned_alloc(4096, st.st_size);
      buffers.push_back(buf);

      struct io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
      io_uring_prep_read(sqe, fd, buf, st.st_size, 0);
      io_uring_sqe_set_data(sqe, buf);
    }

    // 一次性提交所有请求
    io_uring_submit(&ring_);

    // 等待所有完成
    for (size_t i = 0; i < files.size(); i++) {
      struct io_uring_cqe* cqe;
      io_uring_wait_cqe(&ring_, &cqe);

      if (cqe->res < 0) {
        std::cerr << "Read failed\n";
      } else {
        std::cout << "Read " << cqe->res << " bytes\n";
      }

      io_uring_cqe_seen(&ring_, cqe);
    }

    // 清理
    for (auto buf : buffers) free(buf);
    for (auto fd : fds) close(fd);
  }

 private:
  struct io_uring ring_;
};

// io_uring vs 传统IO性能对比
void BenchmarkIOUring() {
  const int NUM_FILES = 100;
  std::vector<std::string> files;

  // 创建测试文件
  for (int i = 0; i < NUM_FILES; i++) {
    char filename[32];
    snprintf(filename, sizeof(filename), "test_%d.dat", i);
    files.push_back(filename);

    int fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    char buf[4096];
    memset(buf, 'A', sizeof(buf));
    write(fd, buf, sizeof(buf));
    close(fd);
  }

  // 同步读取
  auto start = std::chrono::high_resolution_clock::now();
  for (const auto& file : files) {
    int fd = open(file.c_str(), O_RDONLY);
    char buf[4096];
    read(fd, buf, sizeof(buf));
    close(fd);
  }
  auto end = std::chrono::high_resolution_clock::now();
  auto sync_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  // io_uring批量读取
  IOUringExample uring;
  start = std::chrono::high_resolution_clock::now();
  uring.BatchRead(files);
  end = std::chrono::high_resolution_clock::now();
  auto uring_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  std::cout << "Sync IO:    " << sync_time.count() << " ms\n";
  std::cout << "io_uring:   " << uring_time.count() << " ms\n";
  std::cout << "Speedup:    "
            << (double)sync_time.count() / uring_time.count() << "x\n";

  // 清理
  for (const auto& file : files) {
    unlink(file.c_str());
  }

  // 预期输出:
  // Sync IO:    850 ms
  // io_uring:   120 ms
  // Speedup:    7.1x
}
```

### 6.2 libaio (Linux AIO)

```cpp
#include <libaio.h>
#include <fcntl.h>
#include <string.h>

// libaio示例
class LibAIOExample {
 public:
  LibAIOExample(int max_events = 128) {
    memset(&ctx_, 0, sizeof(ctx_));
    io_setup(max_events, &ctx_);
  }

  ~LibAIOExample() {
    io_destroy(ctx_);
  }

  void AsyncRead(int fd, void* buf, size_t size, off_t offset) {
    struct iocb cb;
    struct iocb* cbs[1] = { &cb };

    // 准备读取请求
    io_prep_pread(&cb, fd, buf, size, offset);

    // 提交
    int ret = io_submit(ctx_, 1, cbs);
    if (ret != 1) {
      perror("io_submit");
      return;
    }

    // 等待完成
    struct io_event events[1];
    ret = io_getevents(ctx_, 1, 1, events, nullptr);

    if (ret == 1) {
      std::cout << "Read " << events[0].res << " bytes\n";
    }
  }

  // 批量异步IO
  void BatchAsyncIO(int fd, size_t block_size, size_t num_blocks) {
    std::vector<struct iocb> cbs(num_blocks);
    std::vector<struct iocb*> cb_ptrs(num_blocks);
    std::vector<void*> buffers(num_blocks);

    // 准备所有请求
    for (size_t i = 0; i < num_blocks; i++) {
      buffers[i] = aligned_alloc(4096, block_size);
      io_prep_pread(&cbs[i], fd, buffers[i], block_size, i * block_size);
      cb_ptrs[i] = &cbs[i];
    }

    // 提交所有请求
    int submitted = io_submit(ctx_, num_blocks, cb_ptrs.data());
    std::cout << "Submitted " << submitted << " requests\n";

    // 等待所有完成
    std::vector<struct io_event> events(num_blocks);
    int completed = io_getevents(ctx_, num_blocks, num_blocks,
                                  events.data(), nullptr);
    std::cout << "Completed " << completed << " requests\n";

    // 清理
    for (auto buf : buffers) free(buf);
  }

 private:
  io_context_t ctx_;
};
```

---

## 7. 写入优化策略

### 7.1 Write-Ahead Log (WAL)

```cpp
// WAL实现示例
class WriteAheadLog {
 public:
  WriteAheadLog(const char* log_path) {
    // Direct IO + O_SYNC确保持久化
    log_fd_ = open(log_path, O_WRONLY | O_CREAT | O_APPEND | O_DIRECT | O_SYNC,
                   0644);

    posix_memalign(&log_buffer_, 4096, LOG_BUFFER_SIZE);
    log_offset_ = 0;
  }

  ~WriteAheadLog() {
    Flush();
    close(log_fd_);
    free(log_buffer_);
  }

  // 记录操作 (先写日志)
  void LogOperation(const char* op, const void* data, size_t size) {
    // 构造日志条目
    struct LogEntry {
      uint32_t checksum;
      uint32_t size;
      char operation[16];
      char data[1];  // 可变长度
    };

    size_t entry_size = sizeof(LogEntry) + size - 1;

    // 检查缓冲区空间
    if (log_offset_ + entry_size > LOG_BUFFER_SIZE) {
      Flush();
    }

    // 写入日志条目
    LogEntry* entry = (LogEntry*)((char*)log_buffer_ + log_offset_);
    entry->size = size;
    strncpy(entry->operation, op, sizeof(entry->operation));
    memcpy(entry->data, data, size);
    entry->checksum = CalculateChecksum(entry, entry_size);

    log_offset_ += entry_size;

    // 批量写入 (性能优化)
    if (log_offset_ >= LOG_BUFFER_SIZE / 2) {
      Flush();
    }
  }

  void Flush() {
    if (log_offset_ == 0) return;

    // 对齐到4KB
    size_t aligned_size = (log_offset_ + 4095) & ~4095;

    // 写入磁盘 (同步)
    ssize_t written = write(log_fd_, log_buffer_, aligned_size);
    if (written != (ssize_t)aligned_size) {
      perror("write failed");
    }

    // O_SYNC确保数据已到达磁盘

    log_offset_ = 0;
  }

 private:
  int log_fd_;
  void* log_buffer_;
  size_t log_offset_;
  static constexpr size_t LOG_BUFFER_SIZE = 1024 * 1024;  // 1MB

  uint32_t CalculateChecksum(const void* data, size_t size) {
    // CRC32或其他校验和
    return 0;  // 简化示例
  }
};

// 使用WAL的数据库更新
void DatabaseUpdate() {
  WriteAheadLog wal("db.wal");

  // 1. 先写WAL (持久化)
  wal.LogOperation("UPDATE", "row_id=100, value=200", 24);

  // 2. 更新内存数据结构
  UpdateInMemoryTable(100, 200);

  // 3. 异步刷新到磁盘
  // (崩溃时可从WAL恢复)
}
```

### 7.2 Group Commit (组提交)

```cpp
// Group Commit优化
class GroupCommit {
 public:
  GroupCommit(int fd) : fd_(fd), pending_writes_(0) {
    commit_thread_ = std::thread(&GroupCommit::CommitThread, this);
  }

  ~GroupCommit() {
    running_ = false;
    cv_.notify_one();
    commit_thread_.join();
  }

  // 异步写入 (加入批次)
  void AsyncWrite(const void* data, size_t size) {
    std::unique_lock<std::mutex> lock(mutex_);

    // 添加到待写入列表
    pending_data_.push_back({data, size});
    pending_writes_++;

    // 通知提交线程
    cv_.notify_one();

    // 等待提交完成
    cv_.wait(lock, [this, writes = pending_writes_] {
      return committed_writes_ >= writes;
    });
  }

 private:
  void CommitThread() {
    while (running_) {
      std::unique_lock<std::mutex> lock(mutex_);

      // 等待数据或超时
      cv_.wait_for(lock, std::chrono::milliseconds(10),
                   [this] { return !pending_data_.empty() || !running_; });

      if (pending_data_.empty()) continue;

      // 收集所有待写入数据
      std::vector<struct iovec> iovecs;
      for (const auto& item : pending_data_) {
        iovecs.push_back({const_cast<void*>(item.data), item.size});
      }

      // 一次性写入 (writev)
      lock.unlock();
      ssize_t written = writev(fd_, iovecs.data(), iovecs.size());
      fsync(fd_);  // 确保持久化
      lock.lock();

      // 更新已提交计数
      committed_writes_ = pending_writes_;
      pending_data_.clear();

      // 通知所有等待线程
      cv_.notify_all();
    }
  }

  int fd_;
  std::mutex mutex_;
  std::condition_variable cv_;
  std::thread commit_thread_;
  bool running_ = true;

  struct WriteItem {
    const void* data;
    size_t size;
  };
  std::vector<WriteItem> pending_data_;

  int pending_writes_ = 0;
  int committed_writes_ = 0;
};

// 效果:
// - 多个写入合并为一次fsync
// - fsync开销从O(N)降到O(1)
// - 吞吐量提升10-100倍
```

---

## 8. 性能测量与调优

### 8.1 iostat - IO统计

```bash
# 基础用法
iostat -x 1

# 输出示例:
# Device    r/s   w/s   rkB/s   wkB/s  await  %util
# sda      120.0  80.0  4800.0  3200.0  12.5   85.0

# 关键指标解读:
# r/s, w/s: 每秒读写次数 (IOPS)
# rkB/s, wkB/s: 每秒读写KB数 (带宽)
# await: 平均等待时间 (ms)
#   - HDD: <15ms 正常
#   - SSD: <5ms 正常
# %util: 设备利用率
#   - >80%: 接近饱和
#   - 100%: 饱和

# 详细统计
iostat -x -d -m 1

# 分析瓶颈:
# 1. await高 + %util高 → 磁盘瓶颈
# 2. await高 + %util低 → 应用问题或锁竞争
# 3. r/s高 + rkB/s低 → 小随机读
# 4. w/s高 + wkB/s低 → 小随机写
```

### 8.2 blktrace - IO追踪

```bash
# 开启追踪
blktrace -d /dev/sda -o trace

# 运行工作负载...

# 停止追踪 (Ctrl+C)

# 解析追踪
blkparse -i trace

# 输出示例:
#   8,0    1        1     0.000000000  1234  Q  WS 0 + 8 [process]
#   8,0    1        2     0.000001234  1234  G  WS 0 + 8 [process]
#   8,0    1        3     0.000002345  1234  I  WS 0 + 8 [process]
#   8,0    1        4     0.000003456     0  D  WS 0 + 8 [swapper]
#   8,0    1        5     0.012345678     0  C  WS 0 + 8 [0]

# 事件类型:
# Q: 排队 (Queued)
# G: 获取请求 (Get Request)
# I: 插入 (Inserted)
# D: 派发 (Dispatched)
# C: 完成 (Completed)

# 计算延迟:
# Q→D: 排队时间
# D→C: 设备处理时间
# Q→C: 总延迟

# 生成IO模式报告
btt -i trace
```

### 8.3 fio - 磁盘基准测试

```bash
# 随机读测试 (4KB)
fio --name=random-read \
    --ioengine=libaio \
    --iodepth=32 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --size=1G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting

# 顺序写测试 (1MB)
fio --name=seq-write \
    --ioengine=libaio \
    --iodepth=32 \
    --rw=write \
    --bs=1m \
    --direct=1 \
    --size=10G \
    --numjobs=1 \
    --runtime=60

# 混合读写测试 (70%读, 30%写)
fio --name=mixed \
    --ioengine=libaio \
    --iodepth=32 \
    --rw=randrw \
    --rwmixread=70 \
    --bs=4k \
    --direct=1 \
    --size=1G \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting

# 数据库模拟 (8KB, fsync)
fio --name=database \
    --ioengine=sync \
    --iodepth=1 \
    --rw=randwrite \
    --bs=8k \
    --fsync=1 \
    --size=1G \
    --numjobs=16 \
    --runtime=60 \
    --group_reporting
```

---

## 9. 实战案例

### 案例1: LevelDB的优化

```cpp
// LevelDB使用的IO优化技术

// 1. Append-Only写入 (顺序IO)
class LogWriter {
 public:
  Status AddRecord(const Slice& data) {
    // 总是追加到文件末尾
    // 避免随机写入
    return AppendToFile(file_, data);
  }
};

// 2. Manifest使用mmap
class VersionSet {
  void LogAndApply(VersionEdit* edit) {
    // 使用mmap避免系统调用开销
    void* mapped = mmap(nullptr, size, PROT_WRITE,
                        MAP_SHARED, manifest_fd_, 0);
    memcpy(mapped, data, size);
    msync(mapped, size, MS_SYNC);
    munmap(mapped, size);
  }
};

// 3. SSTable使用mmap + madvise
class Table {
  static Status Open(const Options& options,
                     RandomAccessFile* file,
                     uint64_t file_size,
                     Table** table) {
    // mmap整个文件
    void* mapped = mmap(nullptr, file_size, PROT_READ,
                        MAP_SHARED, fd, 0);

    // 建议内核:顺序访问
    madvise(mapped, file_size, MADV_SEQUENTIAL);

    // 或者:随机访问 (索引)
    // madvise(mapped, file_size, MADV_RANDOM);

    return Status::OK();
  }
};

// 4. Compaction时的批量写入
void DoCompactionWork() {
  // 使用大缓冲区减少系统调用
  WritableFile* outfile;
  env_->NewWritableFile(output_path, &outfile);

  // 设置64KB缓冲区
  outfile->SetWriteBufferSize(65536);

  // 批量写入
  for (auto& entry : entries) {
    outfile->Append(entry);
  }

  // 一次性刷新
  outfile->Sync();
}
```

### 案例2: 日志系统优化

```cpp
// 高性能日志系统
class HighPerfLogger {
 public:
  HighPerfLogger(const char* path) {
    // Direct IO + 大缓冲区
    fd_ = open(path, O_WRONLY | O_CREAT | O_APPEND | O_DIRECT, 0644);

    posix_memalign(&buffer_, 4096, BUFFER_SIZE);
    buffer_offset_ = 0;

    // 后台刷新线程
    flush_thread_ = std::thread(&HighPerfLogger::FlushThread, this);
  }

  ~HighPerfLogger() {
    running_ = false;
    cv_.notify_one();
    flush_thread_.join();

    Flush();
    close(fd_);
    free(buffer_);
  }

  void Log(const char* message) {
    size_t len = strlen(message);

    std::lock_guard<std::mutex> lock(mutex_);

    // 检查空间
    if (buffer_offset_ + len + 1 > BUFFER_SIZE) {
      // 异步刷新
      cv_.notify_one();

      // 等待空间
      while (buffer_offset_ + len + 1 > BUFFER_SIZE) {
        std::this_thread::sleep_for(std::chrono::microseconds(100));
      }
    }

    // 写入缓冲区
    memcpy((char*)buffer_ + buffer_offset_, message, len);
    buffer_offset_ += len;
    ((char*)buffer_)[buffer_offset_++] = '\n';
  }

 private:
  void FlushThread() {
    while (running_) {
      std::unique_lock<std::mutex> lock(mutex_);

      // 等待数据或超时
      cv_.wait_for(lock, std::chrono::milliseconds(100),
                   [this] { return buffer_offset_ > 0 || !running_; });

      if (buffer_offset_ == 0) continue;

      // 复制数据
      void* temp_buffer;
      posix_memalign(&temp_buffer, 4096, BUFFER_SIZE);
      size_t size = buffer_offset_;
      memcpy(temp_buffer, buffer_, size);
      buffer_offset_ = 0;

      lock.unlock();

      // 对齐写入
      size_t aligned_size = (size + 4095) & ~4095;
      write(fd_, temp_buffer, aligned_size);

      free(temp_buffer);
    }
  }

  void Flush() {
    if (buffer_offset_ == 0) return;

    size_t aligned_size = (buffer_offset_ + 4095) & ~4095;
    write(fd_, buffer_, aligned_size);
    buffer_offset_ = 0;
  }

  int fd_;
  void* buffer_;
  size_t buffer_offset_;
  static constexpr size_t BUFFER_SIZE = 1024 * 1024;  // 1MB

  std::mutex mutex_;
  std::condition_variable cv_;
  std::thread flush_thread_;
  bool running_ = true;
};

// 性能:
// - 批量写入: 减少系统调用
// - Direct IO: 避免Page Cache污染
// - 异步刷新: 不阻塞日志调用
// - 吞吐量: ~1GB/s (SSD)
```

---

## 总结

今天我们深入学习了磁盘IO优化:

1. ✅ **磁盘硬件**: HDD vs SSD vs NVMe特性对比
2. ✅ **Linux IO栈**: Page Cache, IO调度器, 块层
3. ✅ **文件系统**: ext4, XFS优化, 挂载选项
4. ✅ **IO模式**: Buffered vs Direct, 同步vs异步
5. ✅ **异步IO**: io_uring, libaio高性能用法
6. ✅ **写入策略**: WAL, Group Commit
7. ✅ **性能工具**: iostat, blktrace, fio
8. ✅ **实战案例**: LevelDB, 高性能日志

**关键要点:**
- HDD: 顺序vs随机 = 1000倍差距
- SSD: Write Amplification是性能杀手
- Page Cache: 读性能利器,写入需谨慎
- Direct IO: 数据库必备,通用应用慎用
- io_uring: 现代异步IO,性能提升5-10倍
- WAL: 持久化的最佳实践

**性能提升总结:**
| 优化技术 | 典型提升 |
|---------|---------|
| 顺序访问 vs 随机访问 (HDD) | 100-1000x |
| io_uring vs 同步IO | 5-10x |
| Group Commit | 10-100x |
| Buffered IO (cache hit) | 1000x |
| Direct IO + 批量 | 2-5x |

**工具箱:**
```bash
# IO监控
iostat -x 1
iotop -o

# IO追踪
blktrace -d /dev/sda
btt -i trace

# 基准测试
fio [配置]
dd if=/dev/zero of=test bs=1M count=1024 oflag=direct

# 文件系统调优
tune2fs -l /dev/sda1
xfs_info /mnt/data
```

**下一步:**
- 课程04: 网络IO深度优化
- 课程05: 缓存策略与预取优化

恭喜你完成了磁盘IO深度优化课程! 🎉
