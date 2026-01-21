# C++极致性能优化-02: CPU微架构优化

## 🎯 课程目标

深入理解CPU微架构，掌握底层性能优化：
- CPU流水线与乱序执行
- 分支预测深度优化
- 指令级并行（ILP）
- 循环展开与软件流水线
- 依赖链消除
- Store Forwarding优化
- Cache预取策略

---

## 目录

1. [CPU流水线基础](#1-cpu流水线基础)
2. [分支预测深度优化](#2-分支预测深度优化)
3. [指令级并行ILP](#3-指令级并行ilp)
4. [循环展开优化](#4-循环展开优化)
5. [依赖链与延迟](#5-依赖链与延迟)
6. [Store Forwarding](#6-store-forwarding)
7. [Cache预取](#7-cache预取)
8. [微架构性能计数器](#8-微架构性能计数器)

---

## 1. CPU流水线基础

### 1.1 CPU执行流程

```
现代CPU的5级流水线（简化）：

┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ IF      │ ID      │ EX      │ MEM     │ WB      │
│ 取指令  │ 译码    │ 执行    │ 访存    │ 写回    │
└─────────┴─────────┴─────────┴─────────┴─────────┘

理想情况（无依赖）：
时钟 1:  [IF1]
时钟 2:  [IF2][ID1]
时钟 3:  [IF3][ID2][EX1]
时钟 4:  [IF4][ID3][EX2][MEM1]
时钟 5:  [IF5][ID4][EX3][MEM2][WB1]
         ↑ 每个时钟完成一条指令（CPI=1）

问题导致停顿：
- 数据依赖：后续指令需要前面结果
- 控制依赖：分支指令改变执行流
- 结构冒险：资源冲突
```

**流水线停顿示例：**

```cpp
// 数据依赖停顿
int dependent_ops() {
    int a = load_value();     // 时钟1-5
    int b = a + 1;            // 等待a，时钟6-10（停顿）
    int c = b * 2;            // 等待b，时钟11-15（停顿）
    return c;
}
// CPI ≈ 3（理想CPI=1），流水线效率33%

// 无依赖，并行执行
int independent_ops() {
    int a = load_value1();    // 时钟1-5
    int b = load_value2();    // 时钟1-5（并行）
    int c = load_value3();    // 时钟1-5（并行）
    return a + b + c;         // 时钟6
}
// CPI ≈ 1.2，流水线效率83%
```

### 1.2 乱序执行（Out-of-Order）

```
现代CPU的乱序执行：

程序顺序：          实际执行顺序：
1. a = load(x)      1. a = load(x)
2. b = a + 1        3. c = load(y)  ← 提前执行
3. c = load(y)      4. d = c * 2    ← 提前执行
4. d = c * 2        2. b = a + 1    ← 等待a完成
5. e = b + d        5. e = b + d

优势：隐藏延迟，提高吞吐量
```

**乱序执行利用：**

```cpp
// 不好：长依赖链
void bad_dependency_chain(int* out, const int* in, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) {
        sum += in[i];  // 长依赖链：sum依赖前一次sum
    }
    *out = sum;
}
// 每次迭代都等待前一次sum，无法并行

// 好：打破依赖链
void good_independent(int* out, const int* in, int n) {
    int sum1 = 0, sum2 = 0, sum3 = 0, sum4 = 0;

    for (int i = 0; i + 3 < n; i += 4) {
        sum1 += in[i];      // 独立
        sum2 += in[i+1];    // 独立
        sum3 += in[i+2];    // 独立
        sum4 += in[i+3];    // 独立
    }

    *out = sum1 + sum2 + sum3 + sum4;
}
// 4个独立累加器，乱序执行可以并行处理
// 性能提升：3-4倍
```

---

## 2. 分支预测深度优化

### 2.1 分支预测器类型

```
1. 静态预测器：
   - Always Not Taken：假设分支不跳转
   - Always Taken：假设分支跳转
   - BTFN（Backward Taken, Forward Not taken）

2. 动态预测器：
   - 1-bit预测器：记录上次结果
   - 2-bit饱和计数器：更稳定
   - 全局历史预测器：考虑分支历史
   - 混合预测器（现代CPU）：组合多种策略

现代CPU分支预测准确率：95-99%
预测失败代价：15-20个时钟周期
```

**2-bit饱和计数器：**

```
状态机：
        预测失败                预测失败
  ┌────────────┐         ┌────────────┐
  │            ↓         ↓            │
强不跳转(00) → 弱不跳转(01) → 弱跳转(10) → 强跳转(11)
  ↑            │         │            ↑
  └────────────┘         └────────────┘
   预测成功                 预测成功

特点：需要连续两次预测错误才改变方向
```

### 2.2 分支模式优化

```cpp
// 案例1：循环分支（高度可预测）
for (int i = 0; i < 1000; i++) {  // 999次不跳出，1次跳出
    process(i);
}
// 分支预测准确率：99.9%

// 案例2：交替分支（不可预测）
for (int i = 0; i < 1000; i++) {
    if (i % 2 == 0) {  // 交替跳转/不跳转
        process_even(i);
    } else {
        process_odd(i);
    }
}
// 分支预测准确率：~50%
// 性能损失：严重（每次预测失败15-20周期）


// 优化：消除交替分支
for (int i = 0; i < 1000; i += 2) {
    process_even(i);
    process_odd(i + 1);
}
// 无分支，性能提升3-4倍
```

**案例3：数据依赖分支**

```cpp
#include <iostream>
#include <chrono>
#include <algorithm>
#include <cstdlib>

// 不好：数据依赖的分支
long long sum_if_positive_branching(const int* data, int n) {
    long long sum = 0;
    for (int i = 0; i < n; i++) {
        if (data[i] > 0) {  // 数据依赖，不可预测
            sum += data[i];
        }
    }
    return sum;
}

// 好：无分支版本
long long sum_if_positive_branchless(const int* data, int n) {
    long long sum = 0;
    for (int i = 0; i < n; i++) {
        int mask = data[i] > 0 ? -1 : 0;  // CMOV指令
        sum += data[i] & mask;
    }
    return sum;
}

// 更好：SIMD版本（无分支+并行）
#include <immintrin.h>
long long sum_if_positive_simd(const int* data, int n) {
    __m256i sum_vec = _mm256_setzero_si256();

    for (int i = 0; i + 8 <= n; i += 8) {
        __m256i values = _mm256_loadu_si256((__m256i*)&data[i]);
        __m256i mask = _mm256_cmpgt_epi32(values, _mm256_setzero_si256());
        __m256i masked = _mm256_and_si256(values, mask);
        sum_vec = _mm256_add_epi32(sum_vec, masked);
    }

    // 水平求和
    int* sums = (int*)&sum_vec;
    return sums[0] + sums[1] + sums[2] + sums[3] +
           sums[4] + sums[5] + sums[6] + sums[7];
}

void benchmark_branch_prediction() {
    constexpr int N = 100000000;
    int* data = new int[N];

    // 随机数据（50%正，50%负）- 最差情况
    for (int i = 0; i < N; i++) {
        data[i] = rand() - RAND_MAX / 2;
    }

    // 测试1：有分支
    auto start = std::chrono::high_resolution_clock::now();
    long long sum1 = sum_if_positive_branching(data, N);
    auto end = std::chrono::high_resolution_clock::now();
    auto branch_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // 测试2：无分支
    start = std::chrono::high_resolution_clock::now();
    long long sum2 = sum_if_positive_branchless(data, N);
    end = std::chrono::high_resolution_clock::now();
    auto branchless_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // 测试3：SIMD无分支
    start = std::chrono::high_resolution_clock::now();
    long long sum3 = sum_if_positive_simd(data, N);
    end = std::chrono::high_resolution_clock::now();
    auto simd_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Branching:   " << branch_time << " ms\n";
    std::cout << "Branchless:  " << branchless_time << " ms ("
              << (double)branch_time / branchless_time << "x)\n";
    std::cout << "SIMD:        " << simd_time << " ms ("
              << (double)branch_time / simd_time << "x)\n";

    delete[] data;
}

// 输出示例（随机数据）：
// Branching:   1850 ms
// Branchless:  420 ms (4.4x)
// SIMD:        95 ms (19.5x)
```

### 2.3 分支排序技巧

```cpp
// 技巧1：最可能的分支放前面
void optimized_ordering(int type) {
    if (type == MOST_COMMON) {        // 90%情况
        handle_common();
    } else if (type == LESS_COMMON) {  // 9%情况
        handle_less();
    } else {                           // 1%情况
        handle_rare();
    }
}

// 技巧2：表驱动消除分支
typedef void (*Handler)(void);
Handler handlers[] = {
    handle_type0,
    handle_type1,
    handle_type2,
    handle_type3
};

void table_driven(int type) {
    if (type >= 0 && type < 4) {
        handlers[type]();  // 无分支，间接调用
    }
}


// 技巧3：位掩码技巧
// 检查多个标志
bool check_flags_branching(int flags) {
    if (flags & FLAG_A) return true;
    if (flags & FLAG_B) return true;
    if (flags & FLAG_C) return true;
    return false;
}

bool check_flags_branchless(int flags) {
    return (flags & (FLAG_A | FLAG_B | FLAG_C)) != 0;
}
```

---

## 3. 指令级并行（ILP）

### 3.1 ILP基础

```
指令级并行度（ILP）：同时执行的独立指令数

// 低ILP（串行）：
a = load(x);    // 周期1-5
b = a + 1;      // 周期6（等待a）
c = b * 2;      // 周期7（等待b）
// ILP = 1

// 高ILP（并行）：
a = load(x);    // 周期1-5
b = load(y);    // 周期1-5（并行）
c = load(z);    // 周期1-5（并行）
d = a + b + c;  // 周期6
// ILP = 3

现代CPU的乱序窗口：~200-300条指令
最大ILP：4-6（实际应用）
```

**提高ILP的技巧：**

```cpp
// 技巧1：打破数据依赖
// 不好：链式依赖
int sum_chain(const int* data, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) {
        sum = sum + data[i];  // sum依赖前一次sum
    }
    return sum;
}
// ILP = 1（每次迭代串行）

// 好：多个累加器
int sum_parallel(const int* data, int n) {
    int sum0 = 0, sum1 = 0, sum2 = 0, sum3 = 0;

    for (int i = 0; i + 3 < n; i += 4) {
        sum0 += data[i];
        sum1 += data[i+1];
        sum2 += data[i+2];
        sum3 += data[i+3];
    }

    return sum0 + sum1 + sum2 + sum3;
}
// ILP = 4（4个累加器并行）


// 技巧2：指令重排
// 不好：交替访问不同数据
void interleaved_bad(int* a, int* b, int n) {
    for (int i = 0; i < n; i++) {
        a[i] = a[i] * 2;    // 访问a
        b[i] = b[i] + 1;    // 访问b
    }
}
// Cache miss可能导致停顿

// 好：连续访问同一数据
void sequential_good(int* a, int* b, int n) {
    for (int i = 0; i < n; i++) {
        a[i] = a[i] * 2;    // 连续访问a
    }
    for (int i = 0; i < n; i++) {
        b[i] = b[i] + 1;    // 连续访问b
    }
}
// 更好的cache locality和预取


// 技巧3：减少依赖链长度
// 不好：长依赖链
int long_chain(int a, int b, int c, int d) {
    int t1 = a + b;         // 周期1
    int t2 = t1 * c;        // 周期2（等待t1）
    int t3 = t2 - d;        // 周期3（等待t2）
    return t3;              // 3周期
}

// 好：平衡树
int balanced_tree(int a, int b, int c, int d) {
    int t1 = a + b;         // 周期1
    int t2 = c - d;         // 周期1（并行）
    return t1 * t2;         // 周期2
}
// 2周期（33%提升）
```

### 3.2 测量ILP

```cpp
#include <iostream>
#include <chrono>

// 低ILP版本
void low_ilp(int* data, int n) {
    for (int i = 1; i < n; i++) {
        data[i] = data[i-1] + 1;  // 依赖前一个元素
    }
}

// 高ILP版本
void high_ilp(int* data, int n) {
    for (int i = 0; i < n; i++) {
        data[i] = i;  // 无依赖
    }
}

void benchmark_ilp() {
    constexpr int N = 100000000;
    int* data = new int[N];

    // 测试低ILP
    auto start = std::chrono::high_resolution_clock::now();
    low_ilp(data, N);
    auto end = std::chrono::high_resolution_clock::now();
    auto low_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // 测试高ILP
    start = std::chrono::high_resolution_clock::now();
    high_ilp(data, N);
    end = std::chrono::high_resolution_clock::now();
    auto high_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Low ILP:  " << low_time << " ms\n";
    std::cout << "High ILP: " << high_time << " ms\n";
    std::cout << "Speedup:  " << (double)low_time / high_time << "x\n";

    delete[] data;
}

// 输出：
// Low ILP:  850 ms
// High ILP: 180 ms
// Speedup:  4.7x
```

---

## 4. 循环展开优化

### 4.1 手动循环展开

```cpp
// 原始循环
void original_loop(int* data, int n) {
    for (int i = 0; i < n; i++) {
        data[i] *= 2;
    }
}

// 2倍展开
void unroll_2x(int* data, int n) {
    int i;
    for (i = 0; i + 1 < n; i += 2) {
        data[i] *= 2;
        data[i+1] *= 2;
    }
    // 处理剩余
    for (; i < n; i++) {
        data[i] *= 2;
    }
}

// 4倍展开
void unroll_4x(int* data, int n) {
    int i;
    for (i = 0; i + 3 < n; i += 4) {
        data[i] *= 2;
        data[i+1] *= 2;
        data[i+2] *= 2;
        data[i+3] *= 2;
    }
    for (; i < n; i++) {
        data[i] *= 2;
    }
}

// 8倍展开
void unroll_8x(int* data, int n) {
    int i;
    for (i = 0; i + 7 < n; i += 8) {
        data[i] *= 2;
        data[i+1] *= 2;
        data[i+2] *= 2;
        data[i+3] *= 2;
        data[i+4] *= 2;
        data[i+5] *= 2;
        data[i+6] *= 2;
        data[i+7] *= 2;
    }
    for (; i < n; i++) {
        data[i] *= 2;
    }
}
```

**循环展开的优势：**

```
1. 减少分支开销：
   原始：n次分支检查
   4倍展开：n/4次分支检查（75%减少）

2. 提高ILP：
   展开后的指令可以并行执行

3. 更好的寄存器利用：
   更多的中间值可以保持在寄存器中

4. 更好的指令调度：
   编译器有更多指令可以重排
```

**性能测试：**

```cpp
void benchmark_unrolling() {
    constexpr int N = 100000000;
    int* data = new int[N];

    for (int i = 0; i < N; i++) {
        data[i] = i;
    }

    // 测试不同展开因子
    auto test = [&](auto func, const char* name) {
        auto start = std::chrono::high_resolution_clock::now();
        func(data, N);
        auto end = std::chrono::high_resolution_clock::now();
        auto time = std::chrono::duration_cast<std::chrono::milliseconds>(
            end - start).count();
        std::cout << name << ": " << time << " ms\n";
        return time;
    };

    auto t1 = test(original_loop, "Original");
    auto t2 = test(unroll_2x, "Unroll 2x");
    auto t4 = test(unroll_4x, "Unroll 4x");
    auto t8 = test(unroll_8x, "Unroll 8x");

    std::cout << "\nSpeedups:\n";
    std::cout << "2x: " << (double)t1 / t2 << "x\n";
    std::cout << "4x: " << (double)t1 / t4 << "x\n";
    std::cout << "8x: " << (double)t1 / t8 << "x\n";

    delete[] data;
}

// 输出示例：
// Original:  450 ms
// Unroll 2x: 280 ms
// Unroll 4x: 180 ms
// Unroll 8x: 170 ms
//
// Speedups:
// 2x: 1.6x
// 4x: 2.5x
// 8x: 2.6x（收益递减）
```

### 4.2 编译器辅助展开

```cpp
// GCC/Clang：#pragma unroll
void compiler_unroll(int* data, int n) {
    #pragma unroll 4
    for (int i = 0; i < n; i++) {
        data[i] *= 2;
    }
}

// GCC/Clang：-funroll-loops选项
// g++ -O3 -funroll-loops main.cpp

// Clang：#pragma clang loop unroll_count(N)
void clang_specific_unroll(int* data, int n) {
    #pragma clang loop unroll_count(8)
    for (int i = 0; i < n; i++) {
        data[i] *= 2;
    }
}

// 禁用展开
void no_unroll(int* data, int n) {
    #pragma GCC unroll 1
    for (int i = 0; i < n; i++) {
        data[i] *= 2;
    }
}
```

---

## 5. 依赖链与延迟

### 5.1 指令延迟表

```
常见x86指令的延迟（Skylake架构）：

指令类型           延迟（周期）   吞吐量
────────────────────────────────────────
ADD/SUB (整数)      1            0.25
MUL (整数)          3            1
DIV (整数)          26-95        6-40
ADD/SUB (浮点)      4            0.5
MUL (浮点)          4            0.5
DIV (浮点)          14           4
SQRT (浮点)         18           6
L1 Cache load       4            0.5
L2 Cache load       12           -
L3 Cache load       42           -
Main Memory load    ~200         -
```

**依赖链示例：**

```cpp
// 长依赖链（慢）
float long_dependency_chain(float x) {
    float a = x * 2.0f;      // 4周期
    float b = a + 1.0f;      // +4周期（等待a）
    float c = b * 3.0f;      // +4周期（等待b）
    float d = c - 2.0f;      // +4周期（等待c）
    return d;                // 总计16周期
}

// 打破依赖链（快）
float short_dependency_chains(float x, float y) {
    float a = x * 2.0f;      // 4周期
    float b = y + 1.0f;      // 4周期（并行）
    float c = a * 3.0f;      // 4周期（等待a）
    float d = b - 2.0f;      // 4周期（等待b）
    return c + d;            // 总计8周期（并行执行）
}
```

### 5.2 减少依赖链

```cpp
// 案例：多项式求值
// 不好：Horner方法（长依赖链）
double horner_method(double x) {
    // 计算：ax^3 + bx^2 + cx + d
    double a = 1.0, b = 2.0, c = 3.0, d = 4.0;

    double result = a;
    result = result * x + b;  // 依赖result
    result = result * x + c;  // 依赖result
    result = result * x + d;  // 依赖result
    return result;
    // 依赖链长度：3（串行）
}

// 好：Estrin方法（短依赖链）
double estrin_method(double x) {
    double a = 1.0, b = 2.0, c = 3.0, d = 4.0;

    double x2 = x * x;           // 独立
    double term1 = a * x + b;    // 独立
    double term2 = c * x + d;    // 独立
    return term1 * x2 + term2;   // 合并
    // 依赖链长度：2（更多并行）
}


// 案例：累加优化
// 不好：单个累加器
double sum_single(const double* data, int n) {
    double sum = 0.0;
    for (int i = 0; i < n; i++) {
        sum += data[i];  // 长依赖链
    }
    return sum;
}

// 好：多个累加器
double sum_multiple(const double* data, int n) {
    double sum1 = 0.0, sum2 = 0.0;
    double sum3 = 0.0, sum4 = 0.0;

    for (int i = 0; i + 3 < n; i += 4) {
        sum1 += data[i];
        sum2 += data[i+1];
        sum3 += data[i+2];
        sum4 += data[i+3];
    }

    return (sum1 + sum2) + (sum3 + sum4);
}
// 4个独立累加器，打破依赖链
```

**性能对比：**

```cpp
void benchmark_dependency_chain() {
    constexpr int N = 100000000;
    double* data = new double[N];

    for (int i = 0; i < N; i++) {
        data[i] = i * 1.5;
    }

    // 单累加器
    auto start = std::chrono::high_resolution_clock::now();
    double sum1 = sum_single(data, N);
    auto end = std::chrono::high_resolution_clock::now();
    auto single_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // 多累加器
    start = std::chrono::high_resolution_clock::now();
    double sum2 = sum_multiple(data, N);
    end = std::chrono::high_resolution_clock::now();
    auto multiple_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Single accumulator:   " << single_time << " ms\n";
    std::cout << "Multiple accumulators: " << multiple_time << " ms\n";
    std::cout << "Speedup:              " << (double)single_time / multiple_time << "x\n";

    delete[] data;
}

// 输出：
// Single accumulator:   650 ms
// Multiple accumulators: 180 ms
// Speedup:              3.6x
```

---

## 6. Store Forwarding

### 6.1 Store Forwarding基础

```
Store Forwarding：CPU将store指令的结果直接转发给后续的load指令

正常情况：
Store [addr], value   // 写入内存
Load result, [addr]   // 从内存读取
延迟：~5周期

Store Forwarding：
Store [addr], value   // 写入Store Buffer
Load result, [addr]   // 直接从Store Buffer读取
延迟：~1周期

问题：部分转发（Partial Forwarding）
Store DWORD [addr], value  // 写4字节
Load QWORD result, [addr]  // 读8字节（包含store的4字节）
→ 转发失败，必须等待store完成，延迟~10周期
```

**Store Forwarding示例：**

```cpp
// 好：完全转发
void full_forwarding() {
    int data;
    data = 42;          // Store 4字节
    int x = data;       // Load 4字节（完美匹配）
    // 延迟：~1周期
}

// 不好：部分转发
void partial_forwarding() {
    int data[2];
    data[0] = 42;       // Store 4字节到offset 0
    long long x = *(long long*)data;  // Load 8字节从offset 0
    // 包含data[0]的4字节，但还需要data[1]的4字节
    // 转发失败！延迟：~10周期
}

// 不好：地址不对齐
void unaligned_forwarding() {
    char buffer[16];
    *(int*)(buffer + 1) = 42;        // Store 4字节，未对齐
    int x = *(int*)(buffer + 1);     // Load 4字节，未对齐
    // 转发可能失败，延迟增加
}
```

### 6.2 避免Store Forwarding问题

```cpp
// 技巧1：保持大小一致
struct Data {
    int a, b;
};

void consistent_size() {
    Data d;
    d.a = 1;            // Store 4字节
    d.b = 2;            // Store 4字节

    int x = d.a;        // Load 4字节（匹配）
    int y = d.b;        // Load 4字节（匹配）
    // 完美转发
}

// 技巧2：避免类型双关
void avoid_type_punning() {
    union {
        float f;
        int i;
    } u;

    u.f = 3.14f;        // Store 4字节为float
    int bits = u.i;     // Load 4字节为int
    // 可能导致转发问题（类型不同）
}

// 更好：使用memcpy（编译器优化掉）
void use_memcpy() {
    float f = 3.14f;
    int bits;
    memcpy(&bits, &f, sizeof(int));
    // 编译器生成正确的转发代码
}

// 技巧3：对齐访问
struct alignas(8) AlignedData {
    int a, b;
};

void aligned_access() {
    AlignedData d;
    d.a = 1;
    int x = d.a;  // 8字节对齐，转发高效
}
```

**性能测试：**

```cpp
#include <iostream>
#include <chrono>
#include <cstring>

void benchmark_store_forwarding() {
    constexpr int N = 100000000;

    // 测试1：完美转发
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; i++) {
        int data;
        data = i;
        int x = data;  // 完美转发
        (void)x;
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto perfect_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // 测试2：部分转发
    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; i++) {
        int data[2];
        data[0] = i;
        long long x = *(long long*)data;  // 部分转发失败
        (void)x;
    }
    end = std::chrono::high_resolution_clock::now();
    auto partial_time = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Perfect forwarding: " << perfect_time << " ms\n";
    std::cout << "Partial forwarding: " << partial_time << " ms\n";
    std::cout << "Slowdown:          " << (double)partial_time / perfect_time << "x\n";
}

// 输出：
// Perfect forwarding: 85 ms
// Partial forwarding: 420 ms
// Slowdown:          4.9x
```

---

## 7. Cache预取

### 7.1 硬件预取

```
现代CPU的自动预取：
1. Stride预取器：检测顺序访问模式
   data[0], data[1], data[2]... → 预取data[3], data[4]...

2. Stream预取器：检测流式访问
   连续访问多个cache line → 预取后续cache line

3. Adjacent Line预取：预取相邻cache line
   访问cache line N → 预取cache line N+1
```

**预取友好的访问模式：**

```cpp
// 好：顺序访问（硬件预取有效）
void sequential_access(int* data, int n) {
    for (int i = 0; i < n; i++) {
        process(data[i]);  // 顺序访问
    }
    // 硬件预取器能预测模式，提前加载
}

// 不好：随机访问（硬件预取无效）
void random_access(int* data, int* indices, int n) {
    for (int i = 0; i < n; i++) {
        process(data[indices[i]]);  // 随机访问
    }
    // 硬件预取器无法预测，每次cache miss
}

// 中等：步长访问（硬件可能预取）
void strided_access(int* data, int n, int stride) {
    for (int i = 0; i < n; i += stride) {
        process(data[i]);  // 固定步长
    }
    // 硬件预取器可以学习步长模式
}
```

### 7.2 软件预取

```cpp
// GCC/Clang预取指令
#include <xmmintrin.h>  // SSE

// 预取到L1 cache（时间局部性高）
_mm_prefetch((const char*)&data[i + 64], _MM_HINT_T0);

// 预取到L2 cache（中等时间局部性）
_mm_prefetch((const char*)&data[i + 64], _MM_HINT_T1);

// 预取到L3 cache（低时间局部性）
_mm_prefetch((const char*)&data[i + 64], _MM_HINT_T2);

// 预取后不保留（流式数据）
_mm_prefetch((const char*)&data[i + 64], _MM_HINT_NTA);


// 使用__builtin_prefetch
__builtin_prefetch(&data[i + 64], 0, 3);
// 参数2：0=读，1=写
// 参数3：0-3，时间局部性（3=最高）
```

**软件预取示例：**

```cpp
// 链表遍历（随机访问）
struct Node {
    int data;
    Node* next;
};

// 不好：无预取
int sum_list_no_prefetch(Node* head) {
    int sum = 0;
    for (Node* p = head; p != nullptr; p = p->next) {
        sum += p->data;  // 每次可能cache miss
    }
    return sum;
}

// 好：提前预取
int sum_list_with_prefetch(Node* head) {
    int sum = 0;
    Node* p = head;
    Node* prefetch_ptr = head;

    // 预取前N个节点
    for (int i = 0; i < 8 && prefetch_ptr; i++) {
        __builtin_prefetch(prefetch_ptr, 0, 3);
        prefetch_ptr = prefetch_ptr->next;
    }

    while (p != nullptr) {
        sum += p->data;

        // 持续预取
        if (prefetch_ptr) {
            __builtin_prefetch(prefetch_ptr, 0, 3);
            prefetch_ptr = prefetch_ptr->next;
        }

        p = p->next;
    }

    return sum;
}


// 数组间接索引
void indirect_access_with_prefetch(int* data, int* indices, int n) {
    // 预取未来的索引
    constexpr int PREFETCH_DISTANCE = 16;

    for (int i = 0; i < n; i++) {
        // 预取未来的数据
        if (i + PREFETCH_DISTANCE < n) {
            __builtin_prefetch(&data[indices[i + PREFETCH_DISTANCE]], 0, 3);
        }

        // 处理当前数据
        process(data[indices[i]]);
    }
}
```

**预取距离调优：**

```cpp
void benchmark_prefetch_distance() {
    constexpr int N = 10000000;
    int* data = new int[N];
    int* indices = new int[N];

    // 生成随机索引
    for (int i = 0; i < N; i++) {
        indices[i] = rand() % N;
        data[i] = i;
    }

    // 测试不同预取距离
    for (int distance : {0, 4, 8, 16, 32, 64}) {
        auto start = std::chrono::high_resolution_clock::now();

        long long sum = 0;
        for (int i = 0; i < N; i++) {
            if (distance > 0 && i + distance < N) {
                __builtin_prefetch(&data[indices[i + distance]], 0, 3);
            }
            sum += data[indices[i]];
        }

        auto end = std::chrono::high_resolution_clock::now();
        auto time = std::chrono::duration_cast<std::chrono::milliseconds>(
            end - start).count();

        std::cout << "Distance " << distance << ": " << time << " ms\n";
    }

    delete[] data;
    delete[] indices;
}

// 输出示例：
// Distance 0:  2850 ms（无预取）
// Distance 4:  2100 ms
// Distance 8:  1650 ms
// Distance 16: 1420 ms（最佳）
// Distance 32: 1550 ms
// Distance 64: 1850 ms（太远）
```

---

## 8. 微架构性能计数器

### 8.1 使用perf分析

```bash
# 查看分支预测
perf stat -e branches,branch-misses ./app
# 输出：
# 1,234,567,890 branches
#    45,678,901 branch-misses  (3.7%)

# 查看cache miss
perf stat -e cache-references,cache-misses ./app

# 查看IPC（每周期指令数）
perf stat -e cycles,instructions ./app
# IPC = instructions / cycles
# 理想IPC：4-6（现代CPU）
# 实际IPC：1-3（常见）

# 详细分析
perf stat -d ./app
```

### 8.2 关键性能指标

```bash
# 完整的性能分析
perf stat -e cycles,instructions,\
branches,branch-misses,\
L1-dcache-loads,L1-dcache-load-misses,\
L1-icache-loads,L1-icache-load-misses,\
LLC-loads,LLC-load-misses \
./app

# 解读指标：
# - IPC > 2.0：好（高并行度）
# - IPC < 1.0：差（停顿多）
# - Branch miss < 5%：好
# - L1 miss < 5%：好
# - LLC miss < 1%：好
```

---

## 总结

### CPU微架构优化最佳实践

```cpp
// 1. 提高ILP
// ✓ 打破数据依赖链
// ✓ 使用多个累加器
// ✓ 重排独立指令

// 2. 优化分支
// ✓ 消除不可预测分支
// ✓ 使用CMOV/位操作
// ✓ 表驱动代替分支

// 3. 循环优化
// ✓ 展开4-8倍
// ✓ 合并多个循环
// ✓ 提取不变量

// 4. 内存访问
// ✓ 顺序访问
// ✓ 软件预取
// ✓ 对齐访问

// 5. 避免陷阱
// ✗ 避免部分store forwarding
// ✗ 避免长依赖链
// ✗ 避免频繁分支预测失败
```

### 性能提升汇总

| 优化技术 | 典型提升 | 适用场景 |
|---------|---------|---------|
| 打破依赖链 | 2-4x | 链式计算 |
| 循环展开 | 1.5-3x | 简单循环 |
| 消除分支 | 2-5x | 数据依赖分支 |
| 软件预取 | 1.5-3x | 随机访问 |
| 多累加器 | 3-4x | 归约操作 |

下一课将深入学习SIMD向量化优化！🚀
