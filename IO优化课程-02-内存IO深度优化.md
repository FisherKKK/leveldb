# IO优化课程-02: 内存IO深度优化

## 🎯 课程目标

本课程将深入讲解内存IO优化的所有技术，掌握：
- CPU Cache架构与优化策略
- 内存访问模式与数据布局优化
- NUMA架构与亲和性优化
- 预取技术 (Prefetching)
- 内存对齐与False Sharing避免
- 内存带宽优化与测量
- TLB优化与大页 (Huge Pages)

---

## 目录

1. [CPU Cache深度剖析](#1-cpu-cache深度剖析)
2. [内存访问模式优化](#2-内存访问模式优化)
3. [内存读取优化策略](#3-内存读取优化策略)
4. [NUMA优化](#4-numa优化)
5. [预取技术](#5-预取技术)
6. [内存对齐与False Sharing](#6-内存对齐与false-sharing)
7. [内存带宽优化](#7-内存带宽优化)
8. [TLB与大页优化](#8-tlb与大页优化)
9. [实战案例](#9-实战案例)

---

## 1. CPU Cache深度剖析

### 1.1 Cache层次结构详解

```
现代CPU Cache架构 (Intel/AMD)：

Core 0:
┌─────────────────────────────────────────┐
│ L1 Instruction Cache (32KB)             │
│ - 8-way set associative                 │
│ - 64-byte cache line                    │
│ - Latency: 4 cycles                     │
├─────────────────────────────────────────┤
│ L1 Data Cache (32KB)                    │
│ - 8-way set associative                 │
│ - 64-byte cache line                    │
│ - Latency: 4 cycles                     │
│ - Write-back policy                     │
├─────────────────────────────────────────┤
│ L2 Cache (256KB - 512KB)                │
│ - 4-way to 8-way associative            │
│ - 64-byte cache line                    │
│ - Latency: 12 cycles                    │
│ - Unified (指令+数据)                    │
└──────────────┬──────────────────────────┘
               │
┌──────────────┴──────────────────────────┐
│ L3 Cache (LLC - Last Level Cache)       │
│ - 8MB - 64MB (shared across all cores)  │
│ - 16-way associative                    │
│ - 64-byte cache line                    │
│ - Latency: 40-50 cycles                 │
│ - Inclusive or Non-inclusive            │
└──────────────┬──────────────────────────┘
               ↓
         Main Memory (DRAM)
         Latency: 200+ cycles

关键概念：
- Cache Line: 64字节的传输单位
- Set Associative: N-way表示每个set有N个cache line
- Write-back: 写入先到cache，稍后flush到内存
- Write-through: 写入直接到内存
```

### 1.2 Cache工作原理

**Cache查找过程：**
```
虚拟地址: 0x12345678
         ↓
┌────────────────────────────────────┐
│ 64位地址分解 (64-byte cache line)  │
├────────┬──────────┬────────────────┤
│ Tag    │ Index    │ Offset (6 bits)│
│ (高位) │ (中间)   │ (低6位)        │
└────────┴──────────┴────────────────┘
           ↓           ↓
      查找Set     Cache Line内偏移

查找步骤：
1. 使用Index找到对应的Set
2. 在Set内比较所有Tag (8-way = 比较8个tag)
3. 如果Tag匹配 → Cache Hit
4. 使用Offset定位到cache line内的字节

示例 (L1 Data Cache: 32KB, 8-way):
- Cache Line: 64 bytes
- Total Lines: 32KB / 64 = 512 lines
- Sets: 512 / 8 = 64 sets
- Index: log2(64) = 6 bits
- Offset: log2(64) = 6 bits
- Tag: 64 - 6 - 6 = 52 bits
```

**Cache Miss类型：**
```
1. Compulsory Miss (强制性缺失)
   - 首次访问数据，必然miss
   - 无法避免
   - 解决：预取

2. Capacity Miss (容量缺失)
   - Cache容量不足，数据被驱逐
   - 工作集 > Cache大小
   - 解决：减少工作集、增大Cache

3. Conflict Miss (冲突缺失)
   - 多个地址映射到同一Set
   - 即使Cache有空间，也会驱逐
   - 解决：改变数据布局、使用更高的associativity

示例代码：演示Conflict Miss
```cpp
#include <iostream>
#include <chrono>

// 演示Conflict Miss
void DemonstrateConflictMiss() {
  const size_t cache_size = 32 * 1024;  // 32KB L1
  const size_t line_size = 64;
  const size_t num_lines = cache_size / line_size;  // 512

  // 分配恰好填满cache的数组
  char* arr1 = new char[cache_size];
  char* arr2 = new char[cache_size];

  // 初始化
  for (size_t i = 0; i < cache_size; i++) {
    arr1[i] = i & 0xFF;
    arr2[i] = i & 0xFF;
  }

  // 测试1：顺序访问单个数组 (无conflict)
  auto start = std::chrono::high_resolution_clock::now();
  volatile int sum1 = 0;
  for (int iter = 0; iter < 1000; iter++) {
    for (size_t i = 0; i < cache_size; i++) {
      sum1 += arr1[i];
    }
  }
  auto end = std::chrono::high_resolution_clock::now();
  auto duration1 = std::chrono::duration_cast<std::chrono::nanoseconds>(
      end - start);

  // 测试2：交替访问两个数组 (conflict!)
  start = std::chrono::high_resolution_clock::now();
  volatile int sum2 = 0;
  for (int iter = 0; iter < 1000; iter++) {
    for (size_t i = 0; i < cache_size; i++) {
      sum2 += arr1[i];
      sum2 += arr2[i];  // 驱逐arr1[i]的cache line
    }
  }
  end = std::chrono::high_resolution_clock::now();
  auto duration2 = std::chrono::duration_cast<std::chrono::nanoseconds>(
      end - start);

  std::cout << "Single array:  " << duration1.count() << " ns\n";
  std::cout << "Two arrays:    " << duration2.count() << " ns\n";
  std::cout << "Slowdown:      " << (double)duration2.count() / duration1.count()
            << "x\n";

  delete[] arr1;
  delete[] arr2;
}

// 预期输出：
// Single array:  5000000 ns
// Two arrays:    15000000 ns (3x slower!)
// Slowdown:      3.0x
```

### 1.3 Cache优化策略

**策略1：提高空间局部性 (Spatial Locality)**
```cpp
// 不好：跳跃访问
void BadSpatialLocality(int matrix[1024][1024]) {
  for (int col = 0; col < 1024; col++) {
    for (int row = 0; row < 1024; row++) {
      matrix[row][col] = 0;  // 每次跳过1024个int
    }
  }
  // 每个cache line (64 bytes = 16 ints) 只用了1个int
  // Cache利用率: 1/16 = 6.25%
}

// 好：连续访问
void GoodSpatialLocality(int matrix[1024][1024]) {
  for (int row = 0; row < 1024; row++) {
    for (int col = 0; col < 1024; col++) {
      matrix[row][col] = 0;  // 连续访问
    }
  }
  // 每个cache line装载16个int，全部使用
  // Cache利用率: 100%
  // 性能提升: 16x!
}
```

**策略2：提高时间局部性 (Temporal Locality)**
```cpp
// 不好：数据使用后立即驱逐
void BadTemporalLocality(int* a, int* b, int* c, int n) {
  for (int i = 0; i < n; i++) {
    c[i] = a[i] + b[i];
  }
  for (int i = 0; i < n; i++) {
    c[i] = c[i] * 2;  // c[i]可能已被驱逐
  }
}

// 好：立即重用数据
void GoodTemporalLocality(int* a, int* b, int* c, int n) {
  for (int i = 0; i < n; i++) {
    c[i] = a[i] + b[i];
    c[i] = c[i] * 2;  // c[i]仍在cache
  }
}
```

**策略3：Cache Blocking (分块优化)**
```cpp
// 矩阵乘法：C = A × B
// 基础版本：cache miss严重
void MatMulBasic(double** A, double** B, double** C, int N) {
  for (int i = 0; i < N; i++) {
    for (int j = 0; j < N; j++) {
      double sum = 0;
      for (int k = 0; k < N; k++) {
        sum += A[i][k] * B[k][j];  // B列访问：cache miss!
      }
      C[i][j] = sum;
    }
  }
}

// 优化版本：Cache Blocking
void MatMulBlocked(double** A, double** B, double** C, int N, int BLOCK) {
  // BLOCK = 32 (适合L1 cache)
  for (int ii = 0; ii < N; ii += BLOCK) {
    for (int jj = 0; jj < N; jj += BLOCK) {
      for (int kk = 0; kk < N; kk += BLOCK) {
        // 小块乘法（完全在cache中）
        for (int i = ii; i < ii + BLOCK && i < N; i++) {
          for (int j = jj; j < jj + BLOCK && j < N; j++) {
            double sum = C[i][j];
            for (int k = kk; k < kk + BLOCK && k < N; k++) {
              sum += A[i][k] * B[k][j];
            }
            C[i][j] = sum;
          }
        }
      }
    }
  }
  // 性能提升：2-10x（取决于N）
}

// 块大小选择指南：
// - L1 cache (32KB): BLOCK = 32 (3个32x32 double矩阵 = 24KB)
// - L2 cache (256KB): BLOCK = 128
// - L3 cache (8MB): BLOCK = 512
```

### 1.4 Cache性能测量

**测量Cache Miss率：**
```bash
# 使用perf测量cache miss
perf stat -e L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./program

# 输出示例：
#  1,234,567,890  L1-dcache-loads
#     62,345,678  L1-dcache-load-misses   # 5.05% miss rate
#     12,345,678  LLC-loads
#      1,234,567  LLC-load-misses         # 10% miss rate

# 分析：
# - L1 miss rate: 62M / 1234M = 5.05% (可接受)
# - LLC miss rate: 1.2M / 12.3M = 10% (较高，考虑优化)
```

**自定义Cache性能测试：**
```cpp
#include <iostream>
#include <chrono>
#include <x86intrin.h>  // __rdtsc()

// 测量不同数组大小的访问延迟（揭示cache层次）
void MeasureCacheLatency() {
  std::cout << "Array Size\tLatency (cycles)\n";

  for (size_t size = 1024; size <= 64 * 1024 * 1024; size *= 2) {
    size_t num_accesses = 100000000 / (size / sizeof(int));
    int* arr = new int[size / sizeof(int)];

    // 初始化（确保分配）
    for (size_t i = 0; i < size / sizeof(int); i++) {
      arr[i] = i;
    }

    // 测量随机访问
    uint64_t start = __rdtsc();
    volatile int sum = 0;
    for (size_t access = 0; access < num_accesses; access++) {
      size_t index = (access * 2654435761u) % (size / sizeof(int));
      sum += arr[index];
    }
    uint64_t end = __rdtsc();

    double cycles_per_access = (double)(end - start) / num_accesses;

    std::cout << size / 1024 << " KB\t\t" << cycles_per_access << "\n";

    delete[] arr;
  }
}

// 预期输出：
// Array Size	Latency (cycles)
// 1 KB		4.2     ← L1 cache
// 2 KB		4.3
// 8 KB		4.5
// 32 KB		5.1
// 64 KB		12.8    ← L2 cache
// 256 KB		13.5
// 512 KB		14.2
// 1024 KB		42.3    ← L3 cache
// 8 MB		45.8
// 16 MB		203.5   ← DRAM!
```

---

## 2. 内存访问模式优化

### 2.1 数据布局优化 (Data Layout)

**Array of Structures (AoS) vs Structure of Arrays (SoA)**

```cpp
// AoS (Array of Structures) - 传统布局
struct Particle_AoS {
  float x, y, z;     // 位置
  float vx, vy, vz;  // 速度
  float mass;        // 质量
  float charge;      // 电荷
};

void UpdateParticles_AoS(Particle_AoS* particles, int n) {
  for (int i = 0; i < n; i++) {
    // 更新位置
    particles[i].x += particles[i].vx;
    particles[i].y += particles[i].vy;
    particles[i].z += particles[i].vz;
  }
  // 问题：每个cache line (64 bytes) 只装2个particle
  // 加载了mass和charge，但不使用 → 浪费带宽
}

// SoA (Structure of Arrays) - 优化布局
struct Particles_SoA {
  float* x;      // 所有x坐标
  float* y;      // 所有y坐标
  float* z;      // 所有z坐标
  float* vx;     // 所有vx
  float* vy;     // 所有vy
  float* vz;     // 所有vz
  float* mass;   // 所有mass
  float* charge; // 所有charge
  int count;
};

void UpdateParticles_SoA(Particles_SoA& particles) {
  for (int i = 0; i < particles.count; i++) {
    particles.x[i] += particles.vx[i];
    particles.y[i] += particles.vy[i];
    particles.z[i] += particles.vz[i];
  }
  // 优点：
  // - 每个cache line装16个float
  // - 只加载需要的数据
  // - SIMD友好
  // 性能提升：2-4x
}

// 基准测试
void BenchmarkDataLayout() {
  const int N = 1000000;

  // AoS测试
  Particle_AoS* aos = new Particle_AoS[N];
  auto start = std::chrono::high_resolution_clock::now();
  for (int iter = 0; iter < 100; iter++) {
    UpdateParticles_AoS(aos, N);
  }
  auto end = std::chrono::high_resolution_clock::now();
  auto aos_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  // SoA测试
  Particles_SoA soa;
  soa.count = N;
  soa.x = new float[N];
  soa.vx = new float[N];
  // ... 分配其他数组

  start = std::chrono::high_resolution_clock::now();
  for (int iter = 0; iter < 100; iter++) {
    UpdateParticles_SoA(soa);
  }
  end = std::chrono::high_resolution_clock::now();
  auto soa_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  std::cout << "AoS time: " << aos_time.count() << " ms\n";
  std::cout << "SoA time: " << soa_time.count() << " ms\n";
  std::cout << "Speedup:  " << (double)aos_time.count() / soa_time.count()
            << "x\n";

  // 预期输出：
  // AoS time: 350 ms
  // SoA time: 120 ms
  // Speedup:  2.9x
}
```

**Hot/Cold Data Separation**
```cpp
// 不好：混合热数据和冷数据
struct DatabaseRow {
  uint64_t id;           // 热：经常访问
  uint64_t timestamp;    // 热：经常访问
  char name[256];        // 冷：很少访问
  char description[1024];// 冷：很少访问
  double value;          // 热：经常访问
};
// 问题：访问id时，加载整个1KB+ struct → 浪费cache

// 好：分离热数据和冷数据
struct DatabaseRow_Hot {
  uint64_t id;
  uint64_t timestamp;
  double value;
  uint32_t cold_data_index;  // 指向冷数据的索引
};  // 24 bytes

struct DatabaseRow_Cold {
  char name[256];
  char description[1024];
};  // 1280 bytes

// 使用：
DatabaseRow_Hot* hot_data = new DatabaseRow_Hot[N];
DatabaseRow_Cold* cold_data = new DatabaseRow_Cold[N];

// 热路径：只访问hot_data
for (int i = 0; i < N; i++) {
  if (hot_data[i].timestamp > threshold) {
    ProcessValue(hot_data[i].value);
  }
}
// Cache利用率：64 bytes / 24 bytes ≈ 2.6个struct per cache line
// vs 原来：64 bytes / 1296 bytes < 1个struct
```

### 2.2 循环优化

**Loop Unrolling (循环展开)**
```cpp
// 原始循环
void SumBasic(int* arr, int n) {
  int sum = 0;
  for (int i = 0; i < n; i++) {
    sum += arr[i];
  }
  // 每次迭代：1次加法，1次跳转
}

// 4路展开
void SumUnrolled4(int* arr, int n) {
  int sum1 = 0, sum2 = 0, sum3 = 0, sum4 = 0;
  int i = 0;

  for (; i + 4 <= n; i += 4) {
    sum1 += arr[i];
    sum2 += arr[i + 1];
    sum3 += arr[i + 2];
    sum4 += arr[i + 3];
  }

  // 处理剩余元素
  int sum = sum1 + sum2 + sum3 + sum4;
  for (; i < n; i++) {
    sum += arr[i];
  }
  // 优点：
  // - 减少75%的跳转
  // - 4路并行（ILP - Instruction Level Parallelism）
  // - 预取效果更好
}

// 性能对比
void BenchmarkUnrolling() {
  const int N = 10000000;
  int* arr = new int[N];

  // 初始化
  for (int i = 0; i < N; i++) arr[i] = i;

  // 基础版本
  auto start = std::chrono::high_resolution_clock::now();
  for (int iter = 0; iter < 100; iter++) {
    SumBasic(arr, N);
  }
  auto end = std::chrono::high_resolution_clock::now();
  auto basic_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  // 展开版本
  start = std::chrono::high_resolution_clock::now();
  for (int iter = 0; iter < 100; iter++) {
    SumUnrolled4(arr, N);
  }
  end = std::chrono::high_resolution_clock::now();
  auto unrolled_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  std::cout << "Basic:    " << basic_time.count() << " ms\n";
  std::cout << "Unrolled: " << unrolled_time.count() << " ms\n";
  std::cout << "Speedup:  " << (double)basic_time.count() / unrolled_time.count()
            << "x\n";

  delete[] arr;

  // 预期输出：
  // Basic:    450 ms
  // Unrolled: 180 ms
  // Speedup:  2.5x
}
```

**Loop Tiling (循环分块)**
```cpp
// 2D数组求和
void Sum2D_NoTiling(int** arr, int N) {
  int sum = 0;
  for (int i = 0; i < N; i++) {
    for (int j = 0; j < N; j++) {
      sum += arr[i][j];
    }
  }
  // 如果N很大，每行访问时，前面的行已被驱逐
}

// Loop Tiling优化
void Sum2D_Tiling(int** arr, int N, int TILE) {
  int sum = 0;

  for (int ii = 0; ii < N; ii += TILE) {
    for (int jj = 0; jj < N; jj += TILE) {
      // 小块完全在cache中
      for (int i = ii; i < ii + TILE && i < N; i++) {
        for (int j = jj; j < jj + TILE && j < N; j++) {
          sum += arr[i][j];
        }
      }
    }
  }
  // TILE = 32: 32x32 int = 4KB，适合L1 cache
}
```

---

## 3. 内存读取优化策略

### 3.1 读取路径分析

```
内存读取的完整路径：

Application Code
      ↓
┌──────────────────────────────┐
│ 1. 虚拟地址转换 (TLB查找)    │ ← 10-20 cycles (TLB miss)
└──────────────────────────────┘
      ↓
┌──────────────────────────────┐
│ 2. L1 Cache查找               │ ← 4 cycles (hit)
│    - Tag比较                  │
│    - Data返回                 │
└──────────────────────────────┘
      ↓ (miss)
┌──────────────────────────────┐
│ 3. L2 Cache查找               │ ← 12 cycles (hit)
└──────────────────────────────┘
      ↓ (miss)
┌──────────────────────────────┐
│ 4. L3 Cache查找               │ ← 40-50 cycles (hit)
└──────────────────────────────┘
      ↓ (miss)
┌──────────────────────────────┐
│ 5. 主内存读取                 │ ← 200+ cycles
│    - 内存控制器               │
│    - DRAM访问                 │
│    - 填充cache line (64字节) │
└──────────────────────────────┘

优化目标：尽可能在L1/L2命中，避免访问主内存
```

### 3.2 顺序读 vs 随机读优化

**顺序读优化：**

```cpp
#include <iostream>
#include <chrono>
#include <vector>

// 顺序读：充分利用Cache Line和硬件预取
void SequentialRead(const std::vector<int>& data) {
  volatile long long sum = 0;

  auto start = std::chrono::high_resolution_clock::now();

  // 顺序访问
  for (size_t i = 0; i < data.size(); i++) {
    sum += data[i];  // 预取自动触发，下一个cache line提前加载
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);

  std::cout << "Sequential Read: " << duration.count() << " ns" << std::endl;
  std::cout << "Throughput: " << (data.size() * sizeof(int) * 1000.0) / duration.count()
            << " GB/s" << std::endl;
}

// 随机读：Cache Miss严重
void RandomRead(const std::vector<int>& data, const std::vector<size_t>& indices) {
  volatile long long sum = 0;

  auto start = std::chrono::high_resolution_clock::now();

  // 随机访问
  for (size_t idx : indices) {
    sum += data[idx];  // 每次可能Cache Miss
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);

  std::cout << "Random Read: " << duration.count() << " ns" << std::endl;
  std::cout << "Throughput: " << (indices.size() * sizeof(int) * 1000.0) / duration.count()
            << " GB/s" << std::endl;
}

// 测试结果（1GB数据）：
// Sequential Read: 250,000,000 ns = 4 GB/s
// Random Read:   2,500,000,000 ns = 0.4 GB/s
// 差距：10倍！
```

**随机读优化策略：**

```cpp
// 策略1：分组排序（Group and Sort）
// 将随机访问的索引排序，转换为准顺序访问
void OptimizedRandomRead(const std::vector<int>& data, std::vector<size_t> indices) {
  volatile long long sum = 0;

  // 1. 排序索引
  std::sort(indices.begin(), indices.end());

  auto start = std::chrono::high_resolution_clock::now();

  // 2. 按排序后的顺序访问
  for (size_t idx : indices) {
    sum += data[idx];
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);

  std::cout << "Optimized Random Read: " << duration.count() << " ns" << std::endl;
  // 性能提升：2-5倍（取决于数据分布）
}

// 策略2：批量预取
void BatchPrefetchRead(const std::vector<int>& data, const std::vector<size_t>& indices) {
  volatile long long sum = 0;
  const int PREFETCH_DISTANCE = 8;  // 提前预取8个元素

  auto start = std::chrono::high_resolution_clock::now();

  // 预取前几个元素
  for (int i = 0; i < PREFETCH_DISTANCE && i < indices.size(); i++) {
    __builtin_prefetch(&data[indices[i]], 0, 3);  // 0=read, 3=high temporal locality
  }

  for (size_t i = 0; i < indices.size(); i++) {
    // 预取未来的元素
    if (i + PREFETCH_DISTANCE < indices.size()) {
      __builtin_prefetch(&data[indices[i + PREFETCH_DISTANCE]], 0, 3);
    }

    // 读取当前元素（此时应该已在cache）
    sum += data[indices[i]];
  }

  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);

  std::cout << "Batch Prefetch Read: " << duration.count() << " ns" << std::endl;
  // 性能提升：1.5-3倍
}
```

### 3.3 批量读取优化

**问题：多次小读取的开销**

```cpp
// 不好：多次小读取
void MultipleSmallReads(const char* filename) {
  int fd = open(filename, O_RDONLY);

  for (int i = 0; i < 1000; i++) {
    char buffer[64];
    lseek(fd, i * 1024, SEEK_SET);  // 系统调用开销
    read(fd, buffer, 64);            // 系统调用开销
    process(buffer);
  }

  close(fd);
  // 问题：2000次系统调用！
}

// 好：批量读取到内存
void BatchRead(const char* filename) {
  int fd = open(filename, O_RDONLY);

  // 1. 一次性读取到内存
  size_t filesize = 1000 * 1024;
  char* buffer = new char[filesize];
  read(fd, buffer, filesize);  // 1次系统调用
  close(fd);

  // 2. 内存中处理
  for (int i = 0; i < 1000; i++) {
    char* data = buffer + i * 1024;
    process(data);  // 纯内存操作
  }

  delete[] buffer;
  // 性能提升：10-100倍
}

// 更好：使用mmap
void MmapRead(const char* filename) {
  int fd = open(filename, O_RDONLY);
  size_t filesize = 1000 * 1024;

  // 映射文件到内存
  char* mapped = (char*)mmap(nullptr, filesize, PROT_READ, MAP_PRIVATE, fd, 0);

  // 建议内核预读
  madvise(mapped, filesize, MADV_SEQUENTIAL);

  for (int i = 0; i < 1000; i++) {
    char* data = mapped + i * 1024;
    process(data);  // 自动page fault，按需加载
  }

  munmap(mapped, filesize);
  close(fd);
}
```

### 3.4 Cache-Aware数据结构

**B+Tree vs Hash Table的读取性能**

```cpp
// B+Tree：Cache友好的读取
template<typename K, typename V, int B = 256>
class BPlusTree {
  struct Node {
    int num_keys;
    K keys[B];           // 连续存储，cache友好
    union {
      Node* children[B+1];  // 内部节点
      V values[B];          // 叶子节点
    };
    Node* next;          // 叶子节点链表
  };

  Node* root;

public:
  // 查找：O(log_B N)次cache miss
  V* Find(const K& key) {
    Node* node = root;

    // 自顶向下：每层1次cache miss
    while (!node->is_leaf) {
      // 二分查找keys数组（全在同一cache line）
      int pos = std::lower_bound(node->keys, node->keys + node->num_keys, key)
                - node->keys;
      node = node->children[pos];  // 下一层：cache miss
    }

    // 叶子节点内查找
    int pos = std::lower_bound(node->keys, node->keys + node->num_keys, key)
              - node->keys;
    if (pos < node->num_keys && node->keys[pos] == key) {
      return &node->values[pos];
    }
    return nullptr;
  }

  // 范围查询：极其高效（顺序读）
  std::vector<V> RangeQuery(const K& start, const K& end) {
    std::vector<V> result;

    // 1. 找到起始叶子节点
    Node* node = FindLeaf(start);

    // 2. 顺序遍历叶子节点链表
    while (node != nullptr) {
      for (int i = 0; i < node->num_keys; i++) {
        if (node->keys[i] >= start && node->keys[i] < end) {
          result.push_back(node->values[i]);  // 顺序读，cache命中
        }
        if (node->keys[i] >= end) {
          return result;
        }
      }
      node = node->next;  // 下一个cache line
    }

    return result;
  }
};

// Hash Table：随机读，cache不友好
template<typename K, typename V>
class HashTable {
  struct Entry {
    K key;
    V value;
    Entry* next;  // 链表法解决冲突
  };

  std::vector<Entry*> buckets;

public:
  // 查找：可能多次cache miss
  V* Find(const K& key) {
    size_t bucket_idx = hash(key) % buckets.size();
    Entry* entry = buckets[bucket_idx];  // 随机访问：cache miss

    // 遍历链表：每个节点都可能cache miss
    while (entry != nullptr) {
      if (entry->key == key) {
        return &entry->value;
      }
      entry = entry->next;  // 指针追踪：cache miss
    }

    return nullptr;
  }

  // 范围查询：性能差（需要扫描所有bucket）
  std::vector<V> RangeQuery(const K& start, const K& end) {
    std::vector<V> result;

    // 必须扫描所有bucket
    for (Entry* entry : buckets) {
      while (entry != nullptr) {
        if (entry->key >= start && entry->key < end) {
          result.push_back(entry->value);  // 随机跳跃，大量cache miss
        }
        entry = entry->next;
      }
    }

    return result;  // 性能：10-100倍慢于B+Tree
  }
};

// 性能对比（100万条数据）：
// B+Tree:
//   - 单点查询：~50ns (3-4次cache miss)
//   - 范围查询：~1μs/100条 (顺序读)
// Hash Table:
//   - 单点查询：~30ns (1-2次cache miss) ← 单点查询更快
//   - 范围查询：~100μs/100条 (随机读) ← 范围查询慢100倍
```

### 3.5 读放大问题与优化

**问题：LSM-Tree的读放大**

```cpp
// LSM-Tree读取流程
class LSMTree {
public:
  std::string Get(const std::string& key) {
    // 1. 查找MemTable（内存，快）
    auto* result = memtable_->Get(key);
    if (result != nullptr) {
      return *result;
    }

    // 2. 查找Immutable MemTable（内存，快）
    result = imm_memtable_->Get(key);
    if (result != nullptr) {
      return *result;
    }

    // 3. 查找Level-0 SSTable（磁盘，慢）
    // Level-0文件可能重叠，需要查所有文件！
    for (auto* file : level0_files_) {
      result = file->Get(key);  // 磁盘读取
      if (result != nullptr) {
        return *result;
      }
    }
    // 读放大：Level-0有10个文件，需要读10次！

    // 4. 查找Level-1+ SSTable（二分查找）
    for (int level = 1; level < max_level_; level++) {
      auto* file = FindFile(level, key);  // 二分查找，只读1个文件
      if (file != nullptr) {
        result = file->Get(key);
        if (result != nullptr) {
          return *result;
        }
      }
    }

    return "";  // Not found
  }
};

// 优化1：Bloom Filter快速过滤
class SSTable {
  BloomFilter bloom_filter_;

public:
  std::string* Get(const std::string& key) {
    // 先查bloom filter（内存操作，极快）
    if (!bloom_filter_.MayContain(key)) {
      return nullptr;  // 确定不存在，避免磁盘读取！
    }

    // 可能存在，进行磁盘读取
    return ReadFromDisk(key);
  }
};

// 优化2：Block Cache缓存热数据
class SSTable {
  LRUCache<std::string, Block*> block_cache_;

public:
  std::string* Get(const std::string& key) {
    // 1. 读取index block（查cache）
    BlockHandle handle = index_block_->FindBlockHandle(key);

    // 2. 读取data block（查cache）
    std::string cache_key = MakeCacheKey(file_number_, handle.offset);
    Block* block = block_cache_.Lookup(cache_key);

    if (block == nullptr) {
      // Cache miss：从磁盘读取
      block = ReadBlock(handle);
      block_cache_.Insert(cache_key, block);
    } else {
      // Cache hit：无需磁盘读取！
    }

    return block->Get(key);
  }
};

// 优化3：Compaction减少文件数
// 定期合并Level-0文件到Level-1，减少读放大
// Level-0: 10个文件 → 读10次
// Compaction后 Level-1: 1个文件 → 读1次
// 读放大降低：10x → 1x
```

### 3.6 读取并发优化

**多线程读取优化**

```cpp
#include <thread>
#include <vector>
#include <atomic>

// 问题：多线程随机读，Cache颠簸
void MultiThreadRandomRead_Bad(const std::vector<int>& data, int num_threads) {
  std::atomic<long long> sum{0};
  std::vector<std::thread> threads;

  auto worker = [&](int thread_id) {
    long long local_sum = 0;

    // 所有线程访问相同的随机位置
    for (int i = 0; i < 1000000; i++) {
      size_t idx = rand() % data.size();
      local_sum += data[idx];  // Cache竞争！
    }

    sum += local_sum;
  };

  for (int i = 0; i < num_threads; i++) {
    threads.emplace_back(worker, i);
  }

  for (auto& t : threads) {
    t.join();
  }

  // 问题：多个线程竞争同一cache line，导致cache颠簸（false sharing）
  // 性能：反而比单线程慢！
}

// 优化：分区读取，减少Cache竞争
void MultiThreadRandomRead_Good(const std::vector<int>& data, int num_threads) {
  std::vector<long long> thread_sums(num_threads, 0);  // 每个线程独立sum
  std::vector<std::thread> threads;

  auto worker = [&](int thread_id) {
    // 每个线程负责一个分区
    size_t partition_size = data.size() / num_threads;
    size_t start = thread_id * partition_size;
    size_t end = (thread_id == num_threads - 1) ? data.size() : start + partition_size;

    long long local_sum = 0;

    // 只访问自己的分区
    for (size_t i = start; i < end; i++) {
      local_sum += data[i];  // 顺序读，cache友好
    }

    thread_sums[thread_id] = local_sum;  // 写入独立位置，避免false sharing
  };

  for (int i = 0; i < num_threads; i++) {
    threads.emplace_back(worker, i);
  }

  for (auto& t : threads) {
    t.join();
  }

  // 汇总结果
  long long total_sum = 0;
  for (auto sum : thread_sums) {
    total_sum += sum;
  }

  // 性能提升：接近线性扩展（8线程 ≈ 8倍速度）
}

// 优化：使用Read-Copy-Update (RCU)模式
class RCUData {
  struct Data {
    std::vector<int> values;
    std::atomic<int> ref_count{0};
  };

  std::atomic<Data*> current_data_;

public:
  // 读取：无锁，极快
  class ReadGuard {
    Data* data_;
  public:
    ReadGuard(Data* d) : data_(d) {
      data_->ref_count.fetch_add(1);
    }
    ~ReadGuard() {
      data_->ref_count.fetch_sub(1);
    }
    const std::vector<int>& get() const { return data_->values; }
  };

  ReadGuard Read() {
    Data* data = current_data_.load(std::memory_order_acquire);
    return ReadGuard(data);
  }

  // 写入：复制数据，更新指针
  void Update(const std::vector<int>& new_values) {
    Data* new_data = new Data{new_values, 0};
    Data* old_data = current_data_.exchange(new_data, std::memory_order_release);

    // 等待所有读者完成
    while (old_data->ref_count.load() > 0) {
      std::this_thread::yield();
    }

    delete old_data;
  }
};

// 使用RCU：多个读者无阻塞并发
void ConcurrentReadWithRCU(RCUData& data, int num_readers) {
  std::vector<std::thread> readers;

  for (int i = 0; i < num_readers; i++) {
    readers.emplace_back([&]() {
      for (int iter = 0; iter < 1000000; iter++) {
        auto guard = data.Read();  // 无锁读取
        volatile int value = guard.get()[0];
      }
    });
  }

  for (auto& t : readers) {
    t.join();
  }

  // 性能：接近理想并发（无锁竞争）
}
```

---

## 4. NUMA优化

### 4.1 NUMA架构详解

```
NUMA (Non-Uniform Memory Access) 架构：

2-Socket服务器示例：

┌────────────────────────────────────────────────┐
│ Socket 0 (Node 0)                              │
│ ┌──────────────────────────────────────────┐  │
│ │ CPU 0-15 (16 cores)                      │  │
│ │ L1/L2 per core, L3 shared (20MB)        │  │
│ └──────────────────────────────────────────┘  │
│                    ↕                           │
│ ┌──────────────────────────────────────────┐  │
│ │ Local Memory (64GB)                      │  │
│ │ Latency: ~60 ns                          │  │
│ │ Bandwidth: 100 GB/s                      │  │
│ └──────────────────────────────────────────┘  │
└──────────────────┬─────────────────────────────┘
                   │
            QPI/UPI Interconnect
          (Latency: ~100 ns)
                   │
┌──────────────────┴─────────────────────────────┐
│ Socket 1 (Node 1)                              │
│ ┌──────────────────────────────────────────┐  │
│ │ Remote Memory (64GB)                     │  │
│ │ Latency: ~120 ns (2x slower!)            │  │
│ │ Bandwidth: 50 GB/s (2x slower!)          │  │
│ └──────────────────────────────────────────┘  │
│                    ↕                           │
│ ┌──────────────────────────────────────────┐  │
│ │ CPU 16-31 (16 cores)                     │  │
│ │ L1/L2 per core, L3 shared (20MB)        │  │
│ └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘

关键问题：
- CPU 0访问Node 0内存：60 ns
- CPU 0访问Node 1内存：120 ns (2x slower!)
- 跨NUMA带宽降低50%
```

### 3.2 NUMA检测与配置

**检测NUMA拓扑：**
```bash
# 查看NUMA配置
numactl --hardware

# 输出示例：
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
# node 0 size: 65536 MB
# node 0 free: 45000 MB
# node 1 cpus: 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
# node 1 size: 65536 MB
# node 1 free: 52000 MB
# node distances:
# node   0   1
#   0:  10  21   ← 本地访问距离10，远程访问距离21
#   1:  21  10

# 查看进程的NUMA统计
numastat -p <PID>

# 查看系统NUMA统计
numastat

# 输出示例：
#                            node0         node1
# numa_hit               12345678      23456789  ← 本地内存命中
# numa_miss               123456       234567   ← 远程内存访问
# numa_foreign            234567       123456
# local_node            12222222      23222222
# other_node             246912       468135
```

**NUMA性能测试：**
```cpp
#include <iostream>
#include <numa.h>
#include <chrono>
#include <thread>
#include <vector>

// 测试本地 vs 远程内存访问延迟
void MeasureNUMALatency() {
  if (numa_available() < 0) {
    std::cerr << "NUMA not available\n";
    return;
  }

  int num_nodes = numa_max_node() + 1;
  std::cout << "Number of NUMA nodes: " << num_nodes << "\n\n";

  const size_t array_size = 100 * 1024 * 1024;  // 100 MB
  const size_t num_accesses = 10000000;

  for (int node = 0; node < num_nodes; node++) {
    // 在node上分配内存
    char* mem = (char*)numa_alloc_onnode(array_size, node);
    memset(mem, 0, array_size);

    // 在每个CPU上测试访问该内存
    for (int cpu = 0; cpu < numa_num_configured_cpus(); cpu++) {
      // 绑定到特定CPU
      cpu_set_t cpuset;
      CPU_ZERO(&cpuset);
      CPU_SET(cpu, &cpuset);
      pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);

      // 测量访问延迟
      auto start = std::chrono::high_resolution_clock::now();
      volatile char sum = 0;
      for (size_t i = 0; i < num_accesses; i++) {
        size_t index = (i * 2654435761u) % array_size;
        sum += mem[index];
      }
      auto end = std::chrono::high_resolution_clock::now();

      auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(
          end - start);
      double ns_per_access = (double)duration.count() / num_accesses;

      int cpu_node = numa_node_of_cpu(cpu);
      std::cout << "CPU " << cpu << " (node " << cpu_node << ") "
                << "→ Memory node " << node << ": "
                << ns_per_access << " ns/access";

      if (cpu_node == node) {
        std::cout << " (LOCAL)\n";
      } else {
        std::cout << " (REMOTE)\n";
      }
    }

    numa_free(mem, array_size);
    std::cout << "\n";
  }
}

// 预期输出（2-socket系统）：
// CPU 0 (node 0) → Memory node 0: 65 ns/access (LOCAL)
// CPU 0 (node 0) → Memory node 1: 125 ns/access (REMOTE)
// CPU 16 (node 1) → Memory node 0: 130 ns/access (REMOTE)
// CPU 16 (node 1) → Memory node 1: 68 ns/access (LOCAL)
```

### 3.3 NUMA优化策略

**策略1：绑定CPU和内存到同一Node**
```cpp
#include <numa.h>
#include <numaif.h>

// 方法1：使用numactl命令行工具
// numactl --cpunodebind=0 --membind=0 ./myprogram

// 方法2：代码中绑定
void BindToNUMANode(int node) {
  // 绑定当前线程到node
  numa_run_on_node(node);

  // 设置内存分配策略
  numa_set_preferred(node);

  std::cout << "Thread bound to NUMA node " << node << "\n";
}

// 优化的并行处理
void ParallelProcessing_NUMA(std::vector<Task>& tasks) {
  int num_nodes = numa_max_node() + 1;
  std::vector<std::thread> threads;

  // 为每个NUMA node创建一个线程
  for (int node = 0; node < num_nodes; node++) {
    threads.emplace_back([node, &tasks]() {
      // 绑定到node
      BindToNUMANode(node);

      // 在本node分配数据
      size_t chunk_size = tasks.size() / num_nodes;
      size_t start = node * chunk_size;
      size_t end = (node == num_nodes - 1) ? tasks.size() : start + chunk_size;

      // 分配本地内存
      char* local_buffer = (char*)numa_alloc_onnode(BUFFER_SIZE, node);

      // 处理任务（全部在本地node）
      for (size_t i = start; i < end; i++) {
        ProcessTask(tasks[i], local_buffer);
      }

      numa_free(local_buffer, BUFFER_SIZE);
    });
  }

  for (auto& t : threads) t.join();
}
```

**策略2：Interleave内存分配**
```cpp
// 适用场景：多个线程访问共享数据

// 方法1：命令行
// numactl --interleave=all ./myprogram

// 方法2：代码中设置
void UseInterleavedMemory() {
  // 设置交错分配策略
  numa_set_interleave_mask(numa_all_nodes_ptr);

  // 分配内存（自动交错到所有node）
  size_t size = 1024 * 1024 * 1024;  // 1GB
  char* mem = (char*)malloc(size);

  // 使用内存
  // ...

  free(mem);
}

// 效果：
// - 避免单个node内存耗尽
// - 均衡内存带宽
// - 适合多线程随机访问
```

**策略3：First-Touch策略**
```cpp
// Linux的默认NUMA策略：First-Touch
// 内存分配在首次写入的CPU所在node

void OptimalFirstTouch(char* buffer, size_t size, int num_threads) {
  size_t chunk_size = size / num_threads;

  // 并行初始化（触发First-Touch）
  #pragma omp parallel for num_threads(num_threads)
  for (int i = 0; i < num_threads; i++) {
    // 每个线程绑定到一个node
    int node = i % numa_max_node();
    numa_run_on_node(node);

    // 初始化自己的chunk
    size_t start = i * chunk_size;
    size_t end = (i == num_threads - 1) ? size : start + chunk_size;

    memset(buffer + start, 0, end - start);
    // 内存分配在本地node
  }

  // 后续访问：每个线程访问自己的chunk → 本地访问
}
```

---

## 5. 预取技术 (Prefetching)

### 5.1 硬件预取

```
现代CPU的硬件预取器 (Hardware Prefetcher)：

类型1：Stream Prefetcher (流预取器)
- 检测顺序访问模式
- 自动预取后续cache line
- 例：访问0x1000, 0x1040, 0x1080 → 自动预取0x10C0, 0x1100...

类型2：Stride Prefetcher (步长预取器)
- 检测固定步长访问
- 例：访问0x1000, 0x1100, 0x1200 → 自动预取0x1300, 0x1400...

类型3：Adjacent Cache Line Prefetcher
- 访问一个cache line时，预取相邻line
- Intel: 自动预取下一个cache line

类型4：Data Cache Unit (DCU) Prefetcher
- L1 cache预取器
- 检测L2到L1的访问模式

控制硬件预取器：
```bash
# 禁用硬件预取（调试用）
echo 0 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Intel CPU：通过MSR寄存器控制
# wrmsr -a 0x1a4 0xf  # 禁用所有预取器
```

### 5.2 软件预取

**显式预取指令：**
```cpp
#include <xmmintrin.h>  // SSE
#include <emmintrin.h>  // SSE2

// GCC/Clang内建函数
void SoftwarePrefetch_Example() {
  int* arr = new int[1000000];

  for (int i = 0; i < 1000000; i++) {
    // 预取未来8个cache line的数据
    __builtin_prefetch(&arr[i + 512], 0, 3);
    //                      ↑        ↑  ↑
    //                    地址     读/写 locality

    // 处理当前数据
    ProcessData(arr[i]);
  }

  delete[] arr;
}

// 参数说明：
// __builtin_prefetch(addr, rw, locality)
//
// rw: 0 = read, 1 = write
// locality: 0 = no temporal locality (不保留在cache)
//           1 = low temporal locality (L3)
//           2 = moderate temporal locality (L2)
//           3 = high temporal locality (L1)
```

**预取策略对比：**
```cpp
// 测试不同预取距离的效果
void BenchmarkPrefetchDistance() {
  const size_t N = 10000000;
  int* arr = new int[N];

  for (size_t i = 0; i < N; i++) arr[i] = i;

  // 无预取
  auto start = std::chrono::high_resolution_clock::now();
  volatile long sum1 = 0;
  for (size_t i = 0; i < N; i++) {
    sum1 += arr[i] * 2;
  }
  auto end = std::chrono::high_resolution_clock::now();
  auto no_prefetch = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  // 预取距离：64 (1个cache line)
  start = std::chrono::high_resolution_clock::now();
  volatile long sum2 = 0;
  for (size_t i = 0; i < N; i++) {
    __builtin_prefetch(&arr[i + 64], 0, 1);
    sum2 += arr[i] * 2;
  }
  end = std::chrono::high_resolution_clock::now();
  auto prefetch_64 = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  // 预取距离：512 (8个cache line)
  start = std::chrono::high_resolution_clock::now();
  volatile long sum3 = 0;
  for (size_t i = 0; i < N; i++) {
    __builtin_prefetch(&arr[i + 512], 0, 1);
    sum3 += arr[i] * 2;
  }
  end = std::chrono::high_resolution_clock::now();
  auto prefetch_512 = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  std::cout << "No prefetch:        " << no_prefetch.count() << " ms\n";
  std::cout << "Prefetch +64:       " << prefetch_64.count() << " ms\n";
  std::cout << "Prefetch +512:      " << prefetch_512.count() << " ms\n";

  delete[] arr;

  // 预期输出：
  // No prefetch:        250 ms
  // Prefetch +64:       180 ms (1.4x faster)
  // Prefetch +512:      150 ms (1.7x faster)
}
```

**链表遍历预取：**
```cpp
struct Node {
  int data;
  Node* next;
};

// 不好：顺序遍历（cache miss严重）
int SumList_NoPrefetch(Node* head) {
  int sum = 0;
  for (Node* p = head; p != nullptr; p = p->next) {
    sum += p->data;  // 每次访问都可能miss
  }
  return sum;
}

// 好：提前预取
int SumList_Prefetch(Node* head) {
  int sum = 0;
  Node* p = head;
  Node* prefetch_ptr = head;

  // 提前预取8个节点
  for (int i = 0; i < 8 && prefetch_ptr; i++) {
    prefetch_ptr = prefetch_ptr->next;
  }

  while (p) {
    // 预取未来的节点
    if (prefetch_ptr) {
      __builtin_prefetch(prefetch_ptr, 0, 1);
      prefetch_ptr = prefetch_ptr->next;
    }

    // 处理当前节点（数据应该已在cache）
    sum += p->data;
    p = p->next;
  }

  return sum;
}

// 性能提升：2-3x（取决于链表长度和内存布局）
```

---

## 6. 内存对齐与False Sharing

### 6.1 内存对齐 (Memory Alignment)

**为什么需要对齐？**
```
未对齐访问的问题：

地址:   0x00  0x01  0x02  0x03  0x04  0x05  0x06  0x07
        ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
        │  a  │  b  │  c  │  d  │  e  │  f  │  g  │  h  │
        └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
Cache   ├─────────────────────────────┤├────────────────...
Line 0  │                             ││
        └─────────────────────────────┘└────────────────...

读取4字节int从0x01开始：
- 跨越2个cache line！
- 需要2次内存访问
- 某些CPU甚至会崩溃 (ARM)

对齐后从0x00开始：
- 单个cache line
- 1次内存访问
- 高效
```

**对齐规则：**
```cpp
#include <iostream>

struct Unaligned {
  char a;      // 1 byte, offset 0
  int b;       // 4 bytes, offset 1 (未对齐！)
  short c;     // 2 bytes, offset 5
};  // 总大小：7 bytes？

struct Aligned {
  char a;      // 1 byte, offset 0
  char pad1[3];// 3 bytes padding
  int b;       // 4 bytes, offset 4 (对齐到4字节边界)
  short c;     // 2 bytes, offset 8
  char pad2[2];// 2 bytes padding
};  // 总大小：12 bytes

int main() {
  std::cout << "Unaligned struct size: " << sizeof(Unaligned) << "\n";
  std::cout << "Aligned struct size:   " << sizeof(Aligned) << "\n";

  Unaligned u;
  std::cout << "Offset of u.a: " << offsetof(Unaligned, a) << "\n";
  std::cout << "Offset of u.b: " << offsetof(Unaligned, b) << "\n";
  std::cout << "Offset of u.c: " << offsetof(Unaligned, c) << "\n";

  // 输出：
  // Unaligned struct size: 12 (编译器自动填充!)
  // Aligned struct size:   12
  // Offset of u.a: 0
  // Offset of u.b: 4 (编译器自动对齐到4字节边界)
  // Offset of u.c: 8
}
```

**手动对齐：**
```cpp
// C++11: alignas
struct AlignedData {
  alignas(64) int data;  // 对齐到64字节 (cache line)
};

// GCC/Clang: __attribute__
struct AlignedData2 {
  int data __attribute__((aligned(64)));
};

// MSVC: __declspec
__declspec(align(64)) struct AlignedData3 {
  int data;
};

// 动态分配对齐内存
void* AllocAligned(size_t size, size_t alignment) {
  void* ptr = nullptr;
  posix_memalign(&ptr, alignment, size);
  return ptr;
}

// 使用示例
int* arr = (int*)AllocAligned(1024 * sizeof(int), 64);
// arr对齐到64字节边界
free(arr);
```

### 6.2 False Sharing (伪共享)

**什么是False Sharing？**
```
False Sharing示例：

2个CPU核心，各自修改不同变量：

内存布局：
┌─────────────────────────────────────────────┐
│ Cache Line 0 (64 bytes)                     │
│ ┌─────────┬─────────┬─────────┬─────────┐  │
│ │counter1 │counter2 │  pad    │  pad    │  │
│ │ (4B)    │ (4B)    │  (28B)  │  (28B)  │  │
│ └─────────┴─────────┴─────────┴─────────┘  │
└─────────────────────────────────────────────┘

CPU 0修改counter1:
1. 从内存加载整个cache line到CPU 0的cache
2. 修改counter1
3. Cache line标记为"Modified"

CPU 1修改counter2:
1. 从内存加载整个cache line到CPU 1的cache
2. CPU 0的cache line失效 (MESI协议)
3. 修改counter2
4. Cache line标记为"Modified"

CPU 0再次修改counter1:
5. CPU 1的cache line失效
6. CPU 0重新加载cache line
7. 修改counter1
... 循环往复

结果：虽然修改不同变量，但cache line在CPU间频繁传输
→ 性能下降10-100倍！
```

**False Sharing演示代码：**
```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <vector>

// 不好：False Sharing
struct SharedCounters {
  long counter1;  // CPU 0使用
  long counter2;  // CPU 1使用
};

void IncrementCounter1_Bad(SharedCounters* counters, long iterations) {
  for (long i = 0; i < iterations; i++) {
    counters->counter1++;
  }
}

void IncrementCounter2_Bad(SharedCounters* counters, long iterations) {
  for (long i = 0; i < iterations; i++) {
    counters->counter2++;
  }
}

// 好：避免False Sharing
struct PaddedCounters {
  alignas(64) long counter1;  // 对齐到cache line
  char pad[64 - sizeof(long)];
  alignas(64) long counter2;  // 另一个cache line
};

void IncrementCounter1_Good(PaddedCounters* counters, long iterations) {
  for (long i = 0; i < iterations; i++) {
    counters->counter1++;
  }
}

void IncrementCounter2_Good(PaddedCounters* counters, long iterations) {
  for (long i = 0; i < iterations; i++) {
    counters->counter2++;
  }
}

void BenchmarkFalseSharing() {
  const long iterations = 100000000;

  // 测试False Sharing版本
  SharedCounters bad_counters = {0, 0};
  auto start = std::chrono::high_resolution_clock::now();

  std::thread t1(IncrementCounter1_Bad, &bad_counters, iterations);
  std::thread t2(IncrementCounter2_Bad, &bad_counters, iterations);
  t1.join();
  t2.join();

  auto end = std::chrono::high_resolution_clock::now();
  auto bad_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  // 测试避免False Sharing版本
  PaddedCounters good_counters = {0, 0};
  start = std::chrono::high_resolution_clock::now();

  std::thread t3(IncrementCounter1_Good, &good_counters, iterations);
  std::thread t4(IncrementCounter2_Good, &good_counters, iterations);
  t3.join();
  t4.join();

  end = std::chrono::high_resolution_clock::now();
  auto good_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  std::cout << "With False Sharing:     " << bad_time.count() << " ms\n";
  std::cout << "Without False Sharing:  " << good_time.count() << " ms\n";
  std::cout << "Speedup:                "
            << (double)bad_time.count() / good_time.count() << "x\n";

  // 预期输出：
  // With False Sharing:     4500 ms
  // Without False Sharing:  350 ms
  // Speedup:                12.9x
}
```

**C++17: std::hardware_destructive_interference_size**
```cpp
#include <new>  // C++17

// 自动使用正确的cache line大小
struct OptimalCounters {
  alignas(std::hardware_destructive_interference_size) long counter1;
  alignas(std::hardware_destructive_interference_size) long counter2;
};

// 或者使用constexpr
constexpr size_t CACHE_LINE_SIZE =
    std::hardware_destructive_interference_size;  // 通常是64
```

---

# 数据布局与对齐深度优化补充内容

## 扩展：多层次数据对齐优化

### 6.3 对齐的多个层次

```cpp
// 层次1：基本类型对齐（编译器自动）
struct BasicAlignment {
    char c;       // 1 byte, offset 0
    // 3 bytes padding
    int i;        // 4 bytes, offset 4 (对齐到4字节)
    // 0 bytes padding
    double d;     // 8 bytes, offset 8 (对齐到8字节)
};
// sizeof = 16 bytes

// 层次2：Cache Line对齐（64字节）
struct alignas(64) CacheLineAligned {
    int data[16];  // 64 bytes
};
// 确保结构体起始地址对齐到64字节边界
// 避免跨cache line访问

// 层次3：页对齐（4096字节）
void* PageAlignedAlloc(size_t size) {
    void* ptr = nullptr;
    posix_memalign(&ptr, 4096, size);  // 对齐到4KB页边界
    return ptr;
}
// 用于Direct IO、DMA、大页等

// 层次4：扇区对齐（512字节或4096字节）
void* SectorAlignedAlloc(size_t size) {
    void* ptr = nullptr;
    posix_memalign(&ptr, 4096, size);  // 对齐到4KB扇区
    return ptr;
}
// Direct IO必须：地址、大小、偏移都要对齐

// 层次5：SIMD对齐（16/32字节）
struct alignas(32) AVXAligned {
    float data[8];  // 32 bytes = 8个float
};
// AVX指令要求32字节对齐
// SSE指令要求16字节对齐
```

**对齐层次图：**

```
┌─────────────────────────────────────────────────────────┐
│ SIMD对齐（16/32字节）                                    │
│ - SSE: 16字节                                           │
│ - AVX: 32字节                                           │
│ - AVX-512: 64字节                                       │
│ 目的：向量化指令高效执行                                │
└─────────────────────────────────────────────────────────┘
                        ↑ 包含
┌─────────────────────────────────────────────────────────┐
│ Cache Line对齐（64字节）                                 │
│ - 避免伪共享（False Sharing）                           │
│ - 一次加载完整数据结构                                   │
│ - 减少cache miss                                        │
└─────────────────────────────────────────────────────────┘
                        ↑ 包含
┌─────────────────────────────────────────────────────────┐
│ 扇区对齐（512字节/4096字节）                             │
│ - Direct IO要求                                         │
│ - DMA传输要求                                           │
│ - SSD优化（4KB扇区）                                    │
└─────────────────────────────────────────────────────────┘
                        ↑ 包含
┌─────────────────────────────────────────────────────────┐
│ 页对齐（4096字节/2MB/1GB）                               │
│ - mmap要求                                              │
│ - Huge Pages（2MB/1GB）                                 │
│ - 减少TLB miss                                          │
└─────────────────────────────────────────────────────────┘
```

### 6.4 结构体打包（Struct Packing）优化

**问题：内存浪费**

```cpp
// 不好的布局：浪费内存
struct BadLayout {
    char a;        // 1 byte
    // 7 bytes padding
    double b;      // 8 bytes
    char c;        // 1 byte
    // 7 bytes padding
    double d;      // 8 bytes
    char e;        // 1 byte
    // 7 bytes padding
};
// sizeof = 40 bytes（实际数据只有19字节，浪费21字节！）

// 好的布局：紧凑排列
struct GoodLayout {
    double b;      // 8 bytes, offset 0
    double d;      // 8 bytes, offset 8
    char a;        // 1 byte, offset 16
    char c;        // 1 byte, offset 17
    char e;        // 1 byte, offset 18
    // 5 bytes padding to align to 8
};
// sizeof = 24 bytes（节省40%内存！）

// 最优布局：使用#pragma pack
#pragma pack(push, 1)  // 强制1字节对齐（紧凑）
struct CompactLayout {
    double b;
    double d;
    char a;
    char c;
    char e;
};
#pragma pack(pop)
// sizeof = 19 bytes（完全紧凑，但访问可能慢）

// 性能测试
void TestStructLayout() {
    constexpr int N = 1000000;

    // 测试BadLayout
    BadLayout* bad = new BadLayout[N];
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; i++) {
        bad[i].a = i;
        bad[i].b = i;
        bad[i].c = i;
        bad[i].d = i;
        bad[i].e = i;
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto bad_time = std::chrono::duration_cast<std::chrono::microseconds>(
        end - start).count();

    // 测试GoodLayout
    GoodLayout* good = new GoodLayout[N];
    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; i++) {
        good[i].a = i;
        good[i].b = i;
        good[i].c = i;
        good[i].d = i;
        good[i].e = i;
    }
    end = std::chrono::high_resolution_clock::now();
    auto good_time = std::chrono::duration_cast<std::chrono::microseconds>(
        end - start).count();

    std::cout << "BadLayout:  " << bad_time << " μs, "
              << "Memory: " << N * sizeof(BadLayout) / 1024 / 1024 << " MB\n";
    std::cout << "GoodLayout: " << good_time << " μs, "
              << "Memory: " << N * sizeof(GoodLayout) / 1024 / 1024 << " MB\n";

    delete[] bad;
    delete[] good;
}

// 输出示例：
// BadLayout:  15000 μs, Memory: 38 MB
// GoodLayout: 9000 μs, Memory: 23 MB
// 性能提升：1.67倍
// 内存节省：40%
```

**自动优化工具：**

```cpp
// 使用C++17 hardware_constructive_interference_size
#include <new>

struct OptimallySpaced {
    alignas(std::hardware_constructive_interference_size) int data1;
    alignas(std::hardware_constructive_interference_size) int data2;
};
// 自动对齐到cache line大小（通常64字节）
// 避免false sharing

// 检查对齐
static_assert(alignof(OptimallySpaced) ==
              std::hardware_constructive_interference_size,
              "Alignment mismatch");
```

### 6.5 热路径数据布局优化

**原则：将频繁访问的数据放在一起**

```cpp
// 不好：冷热数据混合
struct BadDataLayout {
    // 热数据
    int id;                  // 频繁访问
    int status;              // 频繁访问

    // 冷数据（很少访问）
    char description[256];   // 256字节！
    time_t created_at;
    time_t updated_at;

    // 热数据
    int priority;            // 频繁访问
};
// 问题：访问热数据时，加载了大量冷数据到cache

// 好：热冷分离
struct HotData {
    int id;
    int status;
    int priority;
    ColdData* cold_ptr;      // 指向冷数据
};

struct ColdData {
    char description[256];
    time_t created_at;
    time_t updated_at;
};

// 热数据小（16字节），适合cache line
// 冷数据单独存储，按需加载

// 性能对比
void TestHotColdSeparation() {
    constexpr int N = 1000000;

    // BadDataLayout：热冷混合
    std::vector<BadDataLayout> bad_data(N);
    auto start = std::chrono::high_resolution_clock::now();

    int sum = 0;
    for (const auto& item : bad_data) {
        sum += item.id + item.status + item.priority;  // 只访问热数据
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto bad_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // HotData：热冷分离
    std::vector<HotData> hot_data(N);
    std::vector<ColdData> cold_data(N);
    for (int i = 0; i < N; i++) {
        hot_data[i].cold_ptr = &cold_data[i];
    }

    start = std::chrono::high_resolution_clock::now();

    sum = 0;
    for (const auto& item : hot_data) {
        sum += item.id + item.status + item.priority;
    }

    end = std::chrono::high_resolution_clock::now();
    auto good_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Mixed hot/cold: " << bad_time << " ms\n";
    std::cout << "Separated:      " << good_time << " ms\n";
    std::cout << "Speedup:        " << (double)bad_time / good_time << "x\n";
}

// 输出：
// Mixed hot/cold: 50 ms
// Separated:      8 ms
// Speedup:        6.25x
```

### 6.6 SIMD对齐与数据布局

**SIMD要求严格对齐：**

```cpp
#include <immintrin.h>  // AVX

// 不好：未对齐的数据
void UnalignedSIMD() {
    float data[8] = {1, 2, 3, 4, 5, 6, 7, 8};

    // 未对齐的加载（慢）
    __m256 vec = _mm256_loadu_ps(data);  // unaligned load
    // 性能损失：约10-20%
}

// 好：对齐的数据
void AlignedSIMD() {
    alignas(32) float data[8] = {1, 2, 3, 4, 5, 6, 7, 8};

    // 对齐的加载（快）
    __m256 vec = _mm256_load_ps(data);   // aligned load
    // 最优性能
}

// SIMD友好的数据布局：SoA（Structure of Arrays）
struct ParticlesSoA {
    alignas(32) float* x;      // 所有x坐标（连续）
    alignas(32) float* y;      // 所有y坐标（连续）
    alignas(32) float* z;      // 所有z坐标（连续）
    size_t count;
};

void UpdateParticlesSIMD(ParticlesSoA& particles) {
    // 一次处理8个粒子（AVX）
    for (size_t i = 0; i < particles.count; i += 8) {
        __m256 x = _mm256_load_ps(&particles.x[i]);
        __m256 y = _mm256_load_ps(&particles.y[i]);
        __m256 z = _mm256_load_ps(&particles.z[i]);

        // SIMD运算：8个粒子并行
        __m256 velocity = _mm256_set1_ps(0.1f);
        x = _mm256_add_ps(x, velocity);
        y = _mm256_add_ps(y, velocity);
        z = _mm256_add_ps(z, velocity);

        _mm256_store_ps(&particles.x[i], x);
        _mm256_store_ps(&particles.y[i], y);
        _mm256_store_ps(&particles.z[i], z);
    }
}

// 性能：比标量代码快8倍
```

### 6.7 NUMA感知的数据布局

**问题：跨NUMA节点访问慢**

```cpp
#include <numa.h>
#include <numaif.h>

// 不好：数据分配在远程节点
void BadNUMALayout() {
    // 在Node 0分配数据
    int* data = (int*)malloc(1024 * 1024 * sizeof(int));

    // 线程在Node 1运行
    #pragma omp parallel
    {
        int cpu = sched_getcpu();
        int node = numa_node_of_cpu(cpu);

        if (node == 1) {
            // 访问Node 0的数据（慢！）
            for (int i = 0; i < 1024 * 1024; i++) {
                data[i]++;  // 远程内存访问
            }
        }
    }

    free(data);
}

// 好：数据分配在本地节点
void GoodNUMALayout() {
    // 绑定到Node 1
    numa_run_on_node(1);
    numa_set_preferred(1);

    // 在Node 1分配数据
    int* data = (int*)numa_alloc_onnode(
        1024 * 1024 * sizeof(int), 1);

    // 线程在Node 1运行
    #pragma omp parallel
    {
        // 访问本地数据（快！）
        for (int i = 0; i < 1024 * 1024; i++) {
            data[i]++;
        }
    }

    numa_free(data, 1024 * 1024 * sizeof(int));
}

// 性能对比：
// 远程访问：100 ms（2倍延迟）
// 本地访问：50 ms
```

**分区数据布局：**

```cpp
// 为每个NUMA节点分配独立的数据分区
struct NUMAPartitionedData {
    struct Partition {
        int* data;
        size_t size;
        int node_id;
    };

    std::vector<Partition> partitions;
    int num_nodes;

    NUMAPartitionedData(size_t total_size) {
        num_nodes = numa_num_configured_nodes();
        size_t partition_size = total_size / num_nodes;

        for (int node = 0; node < num_nodes; node++) {
            Partition p;
            p.data = (int*)numa_alloc_onnode(
                partition_size * sizeof(int), node);
            p.size = partition_size;
            p.node_id = node;
            partitions.push_back(p);
        }
    }

    ~NUMAPartitionedData() {
        for (auto& p : partitions) {
            numa_free(p.data, p.size * sizeof(int));
        }
    }

    void Process() {
        // 每个线程处理自己节点的数据
        #pragma omp parallel
        {
            int cpu = sched_getcpu();
            int node = numa_node_of_cpu(cpu);

            // 只处理本地分区
            if (node < partitions.size()) {
                auto& partition = partitions[node];
                for (size_t i = 0; i < partition.size; i++) {
                    partition.data[i]++;  // 本地访问
                }
            }
        }
    }
};

// 性能提升：接近线性扩展（无跨节点访问）
```

### 6.8 实战：数据库行存储 vs 列存储布局

**行存储（Row-oriented）：**

```cpp
// 传统数据库：行存储
struct RowStore {
    struct Row {
        int id;
        char name[32];
        int age;
        double salary;
        char department[32];
    };  // 80 bytes per row

    std::vector<Row> rows;

    // 查询：SELECT age FROM employees WHERE age > 30
    std::vector<int> QueryAge(int min_age) {
        std::vector<int> results;

        for (const auto& row : rows) {
            // 加载整行（80字节）
            if (row.age > min_age) {
                results.push_back(row.age);  // 只需要4字节！
            }
        }

        return results;
    }
};

// 问题：
// - 查询age时，加载了name、salary、department（浪费）
// - Cache污染严重
// - IO放大：读取80字节，只用4字节
```

**列存储（Column-oriented）：**

```cpp
// 现代分析数据库：列存储
struct ColumnStore {
    std::vector<int> id_column;
    std::vector<std::string> name_column;
    std::vector<int> age_column;           // 连续存储
    std::vector<double> salary_column;
    std::vector<std::string> dept_column;

    // 查询：SELECT age FROM employees WHERE age > 30
    std::vector<int> QueryAge(int min_age) {
        std::vector<int> results;

        // 只扫描age列（连续内存）
        for (int age : age_column) {
            if (age > min_age) {
                results.push_back(age);
            }
        }

        return results;
    }
};

// 优势：
// - 只读取需要的列
// - 连续内存访问，cache友好
// - 压缩率高（相同类型数据）
// - SIMD友好（向量化扫描）
```

**性能对比：**

```cpp
void BenchmarkRowVsColumn() {
    constexpr int N = 10000000;  // 1000万行

    // 行存储
    RowStore row_store;
    row_store.rows.resize(N);
    for (int i = 0; i < N; i++) {
        row_store.rows[i].age = rand() % 100;
    }

    auto start = std::chrono::high_resolution_clock::now();
    auto results = row_store.QueryAge(30);
    auto end = std::chrono::high_resolution_clock::now();
    auto row_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // 列存储
    ColumnStore col_store;
    col_store.age_column.resize(N);
    for (int i = 0; i < N; i++) {
        col_store.age_column[i] = rand() % 100;
    }

    start = std::chrono::high_resolution_clock::now();
    results = col_store.QueryAge(30);
    end = std::chrono::high_resolution_clock::now();
    auto col_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Row Store:    " << row_time << " ms\n";
    std::cout << "Column Store: " << col_time << " ms\n";
    std::cout << "Speedup:      " << (double)row_time / col_time << "x\n";

    // 内存占用
    std::cout << "\nMemory Usage:\n";
    std::cout << "Row Store:    " << N * sizeof(RowStore::Row) / 1024 / 1024
              << " MB\n";
    std::cout << "Column Store: " << N * sizeof(int) / 1024 / 1024
              << " MB (只age列)\n";
}

// 输出示例：
// Row Store:    500 ms
// Column Store: 25 ms
// Speedup:      20x
//
// Memory Usage:
// Row Store:    762 MB
// Column Store: 38 MB (只age列)
```

---

## 数据布局优化检查清单

### ✅ 内存对齐检查

- [ ] 结构体字段按大小降序排列（减少padding）
- [ ] 使用`alignas`显式控制对齐
- [ ] Cache Line对齐（64字节）用于并发访问的数据
- [ ] SIMD对齐（16/32字节）用于向量化计算
- [ ] 页对齐（4KB）用于Direct IO和mmap

### ✅ 数据布局检查

- [ ] 热冷数据分离（频繁访问的数据在一起）
- [ ] 使用SoA布局用于SIMD和cache优化
- [ ] 考虑列式存储用于分析查询
- [ ] NUMA感知的数据分区
- [ ] 避免False Sharing（不同线程访问的数据分开）

### ✅ 性能验证

- [ ] 使用`sizeof`检查结构体大小
- [ ] 使用`alignof`检查对齐
- [ ] 使用perf检查cache miss率
- [ ] 基准测试对比不同布局的性能
- [ ] 测量内存带宽利用率

---

## 7. 内存带宽优化

### 7.1 测量内存带宽

**STREAM Benchmark - 内存带宽测试黄金标准：**
```cpp
#include <iostream>
#include <chrono>
#include <omp.h>

void MeasureMemoryBandwidth() {
  const size_t N = 100000000;  // 800MB (N * 8 bytes)
  double* a = new double[N];
  double* b = new double[N];
  double* c = new double[N];

  // 初始化
  #pragma omp parallel for
  for (size_t i = 0; i < N; i++) {
    a[i] = 1.0;
    b[i] = 2.0;
    c[i] = 0.0;
  }

  // Test 1: Copy (2个数组，2次访问)
  auto start = std::chrono::high_resolution_clock::now();
  #pragma omp parallel for
  for (size_t i = 0; i < N; i++) {
    c[i] = a[i];
  }
  auto end = std::chrono::high_resolution_clock::now();
  auto copy_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);
  double copy_bw = (2.0 * N * sizeof(double)) /
                   (copy_time.count() / 1000.0) / (1024*1024*1024);

  // Test 2: Scale (2个数组，1读1写)
  start = std::chrono::high_resolution_clock::now();
  double scalar = 3.0;
  #pragma omp parallel for
  for (size_t i = 0; i < N; i++) {
    b[i] = scalar * c[i];
  }
  end = std::chrono::high_resolution_clock::now();
  auto scale_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);
  double scale_bw = (2.0 * N * sizeof(double)) /
                    (scale_time.count() / 1000.0) / (1024*1024*1024);

  // Test 3: Add (3个数组，2读1写)
  start = std::chrono::high_resolution_clock::now();
  #pragma omp parallel for
  for (size_t i = 0; i < N; i++) {
    c[i] = a[i] + b[i];
  }
  end = std::chrono::high_resolution_clock::now();
  auto add_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);
  double add_bw = (3.0 * N * sizeof(double)) /
                  (add_time.count() / 1000.0) / (1024*1024*1024);

  // Test 4: Triad (3个数组，2读1写 + 乘法)
  start = std::chrono::high_resolution_clock::now();
  #pragma omp parallel for
  for (size_t i = 0; i < N; i++) {
    a[i] = b[i] + scalar * c[i];
  }
  end = std::chrono::high_resolution_clock::now();
  auto triad_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);
  double triad_bw = (3.0 * N * sizeof(double)) /
                    (triad_time.count() / 1000.0) / (1024*1024*1024);

  std::cout << "Memory Bandwidth (GB/s):\n";
  std::cout << "  Copy:  " << copy_bw << "\n";
  std::cout << "  Scale: " << scale_bw << "\n";
  std::cout << "  Add:   " << add_bw << "\n";
  std::cout << "  Triad: " << triad_bw << "\n";

  delete[] a;
  delete[] b;
  delete[] c;

  // 预期输出 (DDR4-3200):
  // Memory Bandwidth (GB/s):
  //   Copy:  45.2
  //   Scale: 43.8
  //   Add:   42.5
  //   Triad: 41.3
}
```

### 7.2 带宽优化技术

**技术1：Non-Temporal Stores (绕过Cache写入)**
```cpp
#include <emmintrin.h>  // SSE2

// 常规写入（通过cache）
void MemcpyNormal(void* dst, const void* src, size_t n) {
  memcpy(dst, src, n);
  // 1. 数据写入cache
  // 2. cache稍后写回内存
  // 问题：污染cache，且大块拷贝无需cache
}

// Non-Temporal写入（绕过cache）
void MemcpyNonTemporal(void* dst, const void* src, size_t n) {
  char* d = (char*)dst;
  const char* s = (const char*)src;

  // 使用128位对齐
  size_t i = 0;
  for (; i + 16 <= n; i += 16) {
    __m128i data = _mm_loadu_si128((__m128i*)(s + i));
    _mm_stream_si128((__m128i*)(d + i), data);  // Non-temporal store
  }

  // 处理剩余字节
  for (; i < n; i++) {
    d[i] = s[i];
  }

  _mm_sfence();  // 确保所有写入完成
}

// 性能对比
void BenchmarkNonTemporal() {
  const size_t size = 1024 * 1024 * 1024;  // 1GB
  char* src = new char[size];
  char* dst1 = new char[size];
  char* dst2 = new char[size];

  memset(src, 0xAA, size);

  // 测试常规memcpy
  auto start = std::chrono::high_resolution_clock::now();
  memcpy(dst1, src, size);
  auto end = std::chrono::high_resolution_clock::now();
  auto normal_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  // 测试non-temporal
  start = std::chrono::high_resolution_clock::now();
  MemcpyNonTemporal(dst2, src, size);
  end = std::chrono::high_resolution_clock::now();
  auto nt_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  std::cout << "Normal memcpy:      " << normal_time.count() << " ms\n";
  std::cout << "Non-temporal:       " << nt_time.count() << " ms\n";
  std::cout << "Speedup:            "
            << (double)normal_time.count() / nt_time.count() << "x\n";

  delete[] src;
  delete[] dst1;
  delete[] dst2;

  // 预期输出：
  // Normal memcpy:      450 ms
  // Non-temporal:       280 ms
  // Speedup:            1.6x
}
```

**技术2：Write-Combining (写合并)**
```cpp
// 小块写入合并为大块
void ScatterWrites_Bad(int* arr, int n) {
  // 每次写4字节，触发cache line加载
  for (int i = 0; i < n; i++) {
    arr[i * 100] = i;  // 跳跃写入
  }
  // 每次写入都触发cache miss
}

void CoalesceWrites_Good(int* arr, int n) {
  // 顺序写入，自动合并
  for (int i = 0; i < n; i++) {
    arr[i] = i;
  }
  // CPU自动将多个写入合并为cache line写入
  // 性能提升：10-100x
}
```

---

## 8. TLB与大页优化

### 8.1 TLB (Translation Lookaside Buffer)

```
虚拟内存地址转换：

虚拟地址 → TLB查找 → 物理地址
            ↓ Miss
         Page Table查找 (慢！~100 cycles)

TLB规格 (Intel Skylake):
┌────────────────┬─────────┬──────────────┬────────────┐
│ TLB类型        │ 条目数  │ 页大小       │ 覆盖范围   │
├────────────────┼─────────┼──────────────┼────────────┤
│ L1 DTLB        │ 64      │ 4KB          │ 256 KB     │
│ L1 DTLB        │ 32      │ 2MB/4MB      │ 64 MB      │
│ L2 STLB        │ 1536    │ 4KB/2MB      │ 6 GB       │
└────────────────┴─────────┴──────────────┴────────────┘

问题：大型应用（10GB+）会导致频繁TLB miss
```

### 8.2 Huge Pages (大页)

**配置Huge Pages：**
```bash
# 查看当前配置
cat /proc/meminfo | grep Huge

# 输出：
# HugePages_Total:       0
# HugePages_Free:        0
# Hugepagesize:       2048 kB

# 分配2048个2MB大页 (4GB)
echo 2048 > /proc/sys/vm/nr_hugepages

# 或永久配置 /etc/sysctl.conf:
# vm.nr_hugepages = 2048

# 挂载hugetlbfs
mkdir /mnt/huge
mount -t hugetlbfs none /mnt/huge

# Transparent Huge Pages (THP)
echo always > /sys/kernel/mm/transparent_hugepage/enabled
```

**使用Huge Pages：**
```cpp
#include <sys/mman.h>
#include <fcntl.h>

// 方法1：显式分配huge pages
void* AllocHugePage(size_t size) {
  // size必须是2MB的倍数
  void* ptr = mmap(nullptr, size,
                   PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                   -1, 0);

  if (ptr == MAP_FAILED) {
    perror("mmap huge pages failed");
    return nullptr;
  }

  return ptr;
}

// 方法2：使用hugetlbfs
void* AllocHugePageFS(size_t size) {
  int fd = open("/mnt/huge/myapp", O_CREAT | O_RDWR, 0755);
  if (fd < 0) {
    perror("open failed");
    return nullptr;
  }

  void* ptr = mmap(nullptr, size,
                   PROT_READ | PROT_WRITE,
                   MAP_SHARED, fd, 0);

  close(fd);
  unlink("/mnt/huge/myapp");

  return ptr;
}

// 性能测试
void BenchmarkHugePages() {
  const size_t size = 2ULL * 1024 * 1024 * 1024;  // 2GB

  // 测试4KB页
  void* normal_pages = mmap(nullptr, size,
                            PROT_READ | PROT_WRITE,
                            MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

  auto start = std::chrono::high_resolution_clock::now();
  volatile long sum1 = 0;
  for (size_t i = 0; i < size; i += 4096) {
    sum1 += ((char*)normal_pages)[i];
  }
  auto end = std::chrono::high_resolution_clock::now();
  auto normal_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  munmap(normal_pages, size);

  // 测试2MB大页
  void* huge_pages = AllocHugePage(size);

  start = std::chrono::high_resolution_clock::now();
  volatile long sum2 = 0;
  for (size_t i = 0; i < size; i += 4096) {
    sum2 += ((char*)huge_pages)[i];
  }
  end = std::chrono::high_resolution_clock::now();
  auto huge_time = std::chrono::duration_cast<std::chrono::milliseconds>(
      end - start);

  munmap(huge_pages, size);

  std::cout << "4KB pages:  " << normal_time.count() << " ms\n";
  std::cout << "2MB pages:  " << huge_time.count() << " ms\n";
  std::cout << "Speedup:    "
            << (double)normal_time.count() / huge_time.count() << "x\n";

  // 预期输出：
  // 4KB pages:  450 ms
  // 2MB pages:  180 ms
  // Speedup:    2.5x
}
```

---

## 9. 实战案例

### 9.1 案例1：数据库索引优化

```cpp
// 不好：B-Tree节点布局
struct BTreeNode_Bad {
  int keys[15];           // 60 bytes
  void* children[16];     // 128 bytes (指针)
  int num_keys;           // 4 bytes
  bool is_leaf;           // 1 byte
};  // 193 bytes

// 问题：
// - 不是cache line整数倍
// - children指针导致cache miss
// - 遍历需要多次间接访问

// 好：Cache-friendly B+Tree节点
struct BPlusTreeNode_Good {
  alignas(64) int num_keys;
  bool is_leaf;
  char pad1[63 - sizeof(int) - sizeof(bool)];

  // 将keys和values放一起（局部性）
  struct Entry {
    int key;
    union {
      int value;           // 叶子节点：数据
      BPlusTreeNode_Good* child;  // 内部节点：子节点
    };
  } entries[15];

  BPlusTreeNode_Good* next_leaf;  // 叶子节点链表
};  // 正好3个cache line (192 bytes)

// 优化效果：
// - 遍历keys时，全部在cache
// - 叶子节点遍历：顺序访问
// - 性能提升：2-3x
```

### 9.2 案例2：高性能哈希表

```cpp
// 开放寻址哈希表，优化cache局部性
template <typename K, typename V>
class CacheFriendlyHashMap {
 private:
  struct Bucket {
    K key;
    V value;
    uint8_t hash;      // 高8位hash（快速比较）
    uint8_t distance;  // Robin Hood hashing距离
  };

  static constexpr size_t BUCKET_SIZE = 16;  // 每组16个bucket

  struct BucketGroup {
    alignas(64) Bucket buckets[BUCKET_SIZE];  // 正好1个cache line
  };

  BucketGroup* table_;
  size_t capacity_;

 public:
  V* Find(const K& key) {
    size_t hash = Hash(key);
    uint8_t hash_high = hash >> 56;  // 高8位
    size_t group_idx = hash % capacity_;

    BucketGroup* group = &table_[group_idx];

    // 线性探测（同一cache line内）
    for (size_t i = 0; i < BUCKET_SIZE; i++) {
      Bucket& bucket = group->buckets[i];

      // 快速比较：先比较hash高8位
      if (bucket.hash == hash_high && bucket.key == key) {
        return &bucket.value;
      }

      // 空bucket：未找到
      if (bucket.hash == 0) {
        return nullptr;
      }
    }

    // 溢出到下一组（罕见）
    return FindSlow(key, hash, group_idx);
  }

  // 优点：
  // - 每次查找只访问1个cache line
  // - 高8位hash快速过滤
  // - Robin Hood hashing减少探测距离
  // 性能：比std::unordered_map快3-5x
};
```

### 9.3 案例3：SIMD向量化 + 内存优化

```cpp
#include <immintrin.h>  // AVX2

// 向量点积优化
float DotProduct_Optimized(const float* a, const float* b, size_t n) {
  // 对齐检查
  assert((uintptr_t)a % 32 == 0);
  assert((uintptr_t)b % 32 == 0);

  __m256 sum_vec = _mm256_setzero_ps();

  size_t i = 0;
  // AVX2: 每次处理8个float
  for (; i + 8 <= n; i += 8) {
    __m256 a_vec = _mm256_load_ps(&a[i]);  // 对齐加载
    __m256 b_vec = _mm256_load_ps(&b[i]);
    __m256 prod = _mm256_mul_ps(a_vec, b_vec);
    sum_vec = _mm256_add_ps(sum_vec, prod);
  }

  // 水平求和
  float sum_array[8];
  _mm256_store_ps(sum_array, sum_vec);
  float sum = sum_array[0] + sum_array[1] + sum_array[2] + sum_array[3] +
              sum_array[4] + sum_array[5] + sum_array[6] + sum_array[7];

  // 处理剩余元素
  for (; i < n; i++) {
    sum += a[i] * b[i];
  }

  return sum;
}

// 性能：比标量版本快8x（理论）、实际约6x（考虑水平求和开销）
```

---

## 总结

今天我们深入学习了内存IO优化：

1. ✅ **CPU Cache**：层次结构、工作原理、cache miss类型、优化策略
2. ✅ **内存访问模式**：AoS vs SoA、热冷分离、循环优化、Cache Blocking
3. ✅ **NUMA优化**：架构详解、检测配置、绑定策略、First-Touch
4. ✅ **预取技术**：硬件预取、软件预取、预取距离选择
5. ✅ **内存对齐**：对齐规则、手动对齐、False Sharing避免
6. ✅ **内存带宽**：测量方法、Non-Temporal Stores、Write-Combining
7. ✅ **TLB优化**：Huge Pages配置和使用
8. ✅ **实战案例**：数据库索引、哈希表、SIMD优化

**关键要点：**
- L1 Cache miss惩罚：100倍延迟差距
- False Sharing可导致10-100倍性能下降
- NUMA跨节点访问：2倍延迟差距
- Huge Pages可提升2-3倍性能
- Cache Line对齐：64字节边界

**性能提升总结：**
| 优化技术 | 典型提升 |
|---------|---------|
| 顺序访问 vs 随机访问 | 10-16x |
| SoA vs AoS | 2-4x |
| Cache Blocking | 2-10x |
| 避免False Sharing | 10-100x |
| 软件预取 | 1.5-3x |
| Huge Pages | 2-3x |

**工具箱：**
```bash
# Cache性能分析
perf stat -e L1-dcache-loads,L1-dcache-load-misses ./app
perf c2c record -a -- ./app  # Cache-to-cache传输分析

# NUMA分析
numactl --hardware
numastat -p <PID>
perf mem record -a -- ./app

# 内存带宽测试
./stream_benchmark
mbw 100  # Memory bandwidth benchmark
```

**下一步：**
- 课程03: 磁盘IO深度优化
- 课程04: 网络IO深度优化

恭喜你完成了内存IO深度优化课程！🎉
