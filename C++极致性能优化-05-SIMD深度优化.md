# C++极致性能优化-05-SIMD深度优化

## 课程概述

SIMD (Single Instruction Multiple Data) 是现代CPU提供的强大数据并行处理能力，能够在单条指令中同时处理多个数据元素。掌握SIMD优化是实现极致性能的关键技术之一。

**学习目标：**
- 理解SIMD指令集演进（SSE → AVX → AVX-512）
- 掌握编译器自动向量化技术
- 精通手动向量化与Intrinsics编程
- 学习SIMD常见算法优化
- 掌握混合精度计算优化
- 理解SIMD性能陷阱与调优

**典型性能提升：**
- 自动向量化：2-4倍（SSE）、4-8倍（AVX）
- 手动SIMD优化：4-8倍（AVX）、8-16倍（AVX-512）
- 混合精度计算：10-30倍（INT8量化）

---

## 第1章 SIMD指令集概览

### 1.1 SIMD发展历程

#### 1.1.1 x86 SIMD演进路线

```
CPU SIMD演进：
1999: SSE      - 128位寄存器，4个float / 2个double
2001: SSE2     - 支持整数向量操作
2004: SSE3/SSSE3 - 水平运算、shuffle增强
2007: SSE4     - 混合运算、字符串处理
2011: AVX      - 256位寄存器，8个float / 4个double
2013: AVX2     - 256位整数向量、FMA指令
2017: AVX-512  - 512位寄存器，16个float / 8个double
2020: AVX-512 VNNI - INT8点积加速（AI推理）
```

#### 1.1.2 寄存器容量对比

```cpp
// SSE: 128位寄存器（XMM0-XMM15）
__m128  - 4 x float   (4个单精度)
__m128d - 2 x double  (2个双精度)
__m128i - 16 x int8 / 8 x int16 / 4 x int32 / 2 x int64

// AVX: 256位寄存器（YMM0-YMM15）
__m256  - 8 x float   (8个单精度)
__m256d - 4 x double  (4个双精度)
__m256i - 32 x int8 / 16 x int16 / 8 x int32 / 4 x int64

// AVX-512: 512位寄存器（ZMM0-ZMM31）
__m512  - 16 x float  (16个单精度)
__m512d - 8 x double  (8个双精度)
__m512i - 64 x int8 / 32 x int16 / 16 x int32 / 8 x int64
```

#### 1.1.3 CPU特性检测

```cpp
#include <cpuid.h>
#include <iostream>

class CPUFeatures {
public:
    bool sse;
    bool sse2;
    bool sse3;
    bool ssse3;
    bool sse4_1;
    bool sse4_2;
    bool avx;
    bool avx2;
    bool avx512f;
    bool avx512bw;
    bool avx512vl;
    bool fma;

    CPUFeatures() {
        unsigned int eax, ebx, ecx, edx;

        // CPUID Level 1
        __cpuid(1, eax, ebx, ecx, edx);
        sse     = edx & (1 << 25);
        sse2    = edx & (1 << 26);
        sse3    = ecx & (1 << 0);
        ssse3   = ecx & (1 << 9);
        sse4_1  = ecx & (1 << 19);
        sse4_2  = ecx & (1 << 20);
        avx     = ecx & (1 << 28);
        fma     = ecx & (1 << 12);

        // CPUID Level 7
        __cpuid_count(7, 0, eax, ebx, ecx, edx);
        avx2     = ebx & (1 << 5);
        avx512f  = ebx & (1 << 16);
        avx512bw = ebx & (1 << 30);
        avx512vl = ebx & (1 << 31);
    }

    void print() {
        std::cout << "CPU SIMD Features:\n";
        std::cout << "  SSE:       " << (sse ? "YES" : "NO") << "\n";
        std::cout << "  SSE2:      " << (sse2 ? "YES" : "NO") << "\n";
        std::cout << "  SSE3:      " << (sse3 ? "YES" : "NO") << "\n";
        std::cout << "  SSSE3:     " << (ssse3 ? "YES" : "NO") << "\n";
        std::cout << "  SSE4.1:    " << (sse4_1 ? "YES" : "NO") << "\n";
        std::cout << "  SSE4.2:    " << (sse4_2 ? "YES" : "NO") << "\n";
        std::cout << "  AVX:       " << (avx ? "YES" : "NO") << "\n";
        std::cout << "  AVX2:      " << (avx2 ? "YES" : "NO") << "\n";
        std::cout << "  AVX-512F:  " << (avx512f ? "YES" : "NO") << "\n";
        std::cout << "  AVX-512BW: " << (avx512bw ? "YES" : "NO") << "\n";
        std::cout << "  FMA:       " << (fma ? "YES" : "NO") << "\n";
    }
};

int main() {
    CPUFeatures features;
    features.print();
    return 0;
}
```

### 1.2 运行时分发（Runtime Dispatch）

为了在不同CPU上都能运行，需要提供多个版本的实现：

```cpp
#include <immintrin.h>

// 1. 标量版本（兜底）
float sum_scalar(const float* data, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; i++) {
        sum += data[i];
    }
    return sum;
}

// 2. SSE版本
#ifdef __SSE__
float sum_sse(const float* data, size_t n) {
    __m128 sum_vec = _mm_setzero_ps();

    size_t i;
    for (i = 0; i + 3 < n; i += 4) {
        __m128 v = _mm_loadu_ps(&data[i]);
        sum_vec = _mm_add_ps(sum_vec, v);
    }

    // 水平求和
    float sum[4];
    _mm_storeu_ps(sum, sum_vec);
    float result = sum[0] + sum[1] + sum[2] + sum[3];

    // 处理剩余
    for (; i < n; i++) {
        result += data[i];
    }

    return result;
}
#endif

// 3. AVX版本
#ifdef __AVX__
float sum_avx(const float* data, size_t n) {
    __m256 sum_vec = _mm256_setzero_ps();

    size_t i;
    for (i = 0; i + 7 < n; i += 8) {
        __m256 v = _mm256_loadu_ps(&data[i]);
        sum_vec = _mm256_add_ps(sum_vec, v);
    }

    // 水平求和
    float sum[8];
    _mm256_storeu_ps(sum, sum_vec);
    float result = sum[0] + sum[1] + sum[2] + sum[3] +
                   sum[4] + sum[5] + sum[6] + sum[7];

    for (; i < n; i++) {
        result += data[i];
    }

    return result;
}
#endif

// 4. AVX-512版本
#ifdef __AVX512F__
float sum_avx512(const float* data, size_t n) {
    __m512 sum_vec = _mm512_setzero_ps();

    size_t i;
    for (i = 0; i + 15 < n; i += 16) {
        __m512 v = _mm512_loadu_ps(&data[i]);
        sum_vec = _mm512_add_ps(sum_vec, v);
    }

    // 水平求和
    float result = _mm512_reduce_add_ps(sum_vec);

    for (; i < n; i++) {
        result += data[i];
    }

    return result;
}
#endif

// 5. 运行时分发
float sum_auto(const float* data, size_t n) {
    static CPUFeatures features;

#ifdef __AVX512F__
    if (features.avx512f) {
        return sum_avx512(data, n);
    }
#endif

#ifdef __AVX__
    if (features.avx) {
        return sum_avx(data, n);
    }
#endif

#ifdef __SSE__
    if (features.sse) {
        return sum_sse(data, n);
    }
#endif

    return sum_scalar(data, n);
}
```

**编译配置：**
```bash
# 编译包含所有SIMD版本的代码
g++ -O3 \
    -msse -msse2 -msse3 -mssse3 -msse4.1 -msse4.2 \
    -mavx -mavx2 -mfma \
    -mavx512f -mavx512bw -mavx512vl \
    -o app app.cpp

# 或使用-march=native在本机使用所有特性
g++ -O3 -march=native -o app app.cpp
```

---

## 第2章 编译器自动向量化

### 2.1 自动向量化基础

#### 2.1.1 简单循环自动向量化

```cpp
// 示例1：向量加法（完美向量化）
void vector_add(float* c, const float* a, const float* b, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}
// 编译器自动生成AVX代码：8路并行
// 性能提升：7-8倍

// 示例2：向量数乘
void vector_scale(float* out, const float* in, float scalar, int n) {
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * scalar;
    }
}
// 编译器自动向量化：8路并行

// 示例3：点积（归约操作）
float dot_product(const float* a, const float* b, int n) {
    float sum = 0.0f;
    for (int i = 0; i < n; i++) {
        sum += a[i] * b[i];  // 乘法可向量化，求和需归约
    }
    return sum;
}
// 编译器自动向量化 + 水平求和
```

#### 2.1.2 查看向量化报告

```bash
# GCC向量化报告
g++ -O3 -march=native -fopt-info-vec-optimized -fopt-info-vec-missed app.cpp

# 输出示例：
# app.cpp:10:5: optimized: loop vectorized using 32 byte vectors
# app.cpp:25:5: missed: couldn't vectorize loop (data dependencies)

# Clang向量化报告
clang++ -O3 -march=native -Rpass=loop-vectorize -Rpass-missed=loop-vectorize app.cpp

# 输出示例：
# app.cpp:10:5: remark: vectorized loop (vectorization width: 8)
# app.cpp:25:5: remark: loop not vectorized: value that could not be identified as reduction is used outside the loop
```

### 2.2 向量化友好的代码模式

#### 2.2.1 DO: 可向量化的模式

```cpp
// ✅ 1. 连续内存访问
void good_access(float* out, const float* in, int n) {
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2.0f;  // 完美：步长为1
    }
}

// ✅ 2. 固定步长访问
void strided_access(float* out, const float* in, int n) {
    for (int i = 0; i < n; i++) {
        out[i] = in[i * 2];  // 可向量化：固定步长2
    }
}

// ✅ 3. 简单归约
float simple_sum(const float* data, int n) {
    float sum = 0.0f;
    for (int i = 0; i < n; i++) {
        sum += data[i];  // 归约：编译器可优化
    }
    return sum;
}

// ✅ 4. 多个独立数组
void independent_arrays(float* a, float* b, float* c, int n) {
    for (int i = 0; i < n; i++) {
        a[i] = b[i] + c[i];  // 无依赖，完美向量化
    }
}

// ✅ 5. 已知对齐
void aligned_access(float* __restrict__ out, const float* __restrict__ in, int n) {
    // 告诉编译器out和in不重叠
    out = (float*)__builtin_assume_aligned(out, 32);  // AVX对齐
    in = (float*)__builtin_assume_aligned(in, 32);

    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2.0f;  // 对齐访问：更高效
    }
}
```

#### 2.2.2 DON'T: 阻止向量化的模式

```cpp
// ❌ 1. 数据依赖
void bad_dependency(float* data, int n) {
    for (int i = 1; i < n; i++) {
        data[i] = data[i] + data[i-1];  // 依赖前一个元素，无法向量化
    }
}

// ❌ 2. 条件分支（不可预测）
void bad_branch(float* out, const float* in, int n) {
    for (int i = 0; i < n; i++) {
        if (in[i] > 0.0f) {  // 分支：阻止向量化
            out[i] = in[i];
        } else {
            out[i] = -in[i];
        }
    }
}
// 修复：使用三元运算符或位运算消除分支
void good_branch(float* out, const float* in, int n) {
    for (int i = 0; i < n; i++) {
        out[i] = (in[i] > 0.0f) ? in[i] : -in[i];  // 可向量化
    }
}

// ❌ 3. 函数调用
void bad_call(float* out, const float* in, int n) {
    for (int i = 0; i < n; i++) {
        out[i] = some_function(in[i]);  // 函数调用：阻止向量化
    }
}
// 修复：内联函数
inline float some_function(float x) {
    return x * x + 1.0f;
}

// ❌ 4. 非连续访问（间接寻址）
void bad_indirect(float* out, const float* in, const int* indices, int n) {
    for (int i = 0; i < n; i++) {
        out[i] = in[indices[i]];  // 间接访问：难以向量化
    }
}
// AVX-512支持gather操作可部分优化

// ❌ 5. 指针别名
void bad_aliasing(float* out, float* in, int n) {
    // 编译器不确定out和in是否重叠
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2.0f;  // 可能无法向量化
    }
}
// 修复：使用__restrict__关键字
void good_aliasing(float* __restrict__ out, const float* __restrict__ in, int n) {
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2.0f;  // 可向量化
    }
}
```

### 2.3 帮助编译器向量化

#### 2.3.1 循环提示（Pragma）

```cpp
// OpenMP SIMD指令
void force_vectorize(float* out, const float* in, int n) {
    #pragma omp simd
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2.0f;
    }
}

// GCC向量化提示
void gcc_vectorize(float* out, const float* in, int n) {
    #pragma GCC ivdep  // 忽略向量依赖
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2.0f;
    }
}

// Clang向量化提示
void clang_vectorize(float* out, const float* in, int n) {
    #pragma clang loop vectorize(enable) interleave(enable)
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2.0f;
    }
}

// 指定向量化宽度
void fixed_width(float* out, const float* in, int n) {
    #pragma clang loop vectorize_width(8) interleave_count(2)
    for (int i = 0; i < n; i++) {
        out[i] = in[i] * 2.0f;
    }
}
```

#### 2.3.2 对齐提示

```cpp
// 数据对齐声明
alignas(32) float data[1024];  // AVX对齐

// 动态分配对齐内存
void* aligned_data = aligned_alloc(32, 1024 * sizeof(float));

// POSIX版本
void* aligned_data2;
posix_memalign(&aligned_data2, 32, 1024 * sizeof(float));

// 告诉编译器指针已对齐
void process_aligned(float* data, int n) {
    data = (float*)__builtin_assume_aligned(data, 32);

    for (int i = 0; i < n; i++) {
        data[i] *= 2.0f;  // 编译器知道data是32字节对齐的
    }
}
```

#### 2.3.3 循环计数提示

```cpp
// 告诉编译器循环次数
void known_count(float* data) {
    const int n = 1024;  // 常量：编译器可完全展开
    for (int i = 0; i < n; i++) {
        data[i] *= 2.0f;
    }
}

// 告诉编译器n是8的倍数
void multiple_of_8(float* data, int n) {
    // 假设n是8的倍数
    n = n & ~7;  // 清除低3位

    for (int i = 0; i < n; i++) {
        data[i] *= 2.0f;  // 无需处理剩余元素
    }
}

// 使用__builtin_assume
void assume_multiple(float* data, int n) {
    __builtin_assume(n % 8 == 0);
    __builtin_assume(n > 0);

    for (int i = 0; i < n; i++) {
        data[i] *= 2.0f;
    }
}
```

---

## 第3章 手动SIMD编程（Intrinsics）

### 3.1 SSE Intrinsics基础

#### 3.1.1 基本运算

```cpp
#include <xmmintrin.h>  // SSE
#include <emmintrin.h>  // SSE2
#include <pmmintrin.h>  // SSE3
#include <smmintrin.h>  // SSE4.1

// 向量加法
void sse_add(float* c, const float* a, const float* b, int n) {
    int i;
    for (i = 0; i + 3 < n; i += 4) {
        __m128 va = _mm_loadu_ps(&a[i]);  // 加载4个float
        __m128 vb = _mm_loadu_ps(&b[i]);
        __m128 vc = _mm_add_ps(va, vb);    // 向量加法
        _mm_storeu_ps(&c[i], vc);          // 存储结果
    }

    // 处理剩余元素
    for (; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}

// 向量点积（SSE4.1）
float sse_dot(const float* a, const float* b, int n) {
    __m128 sum = _mm_setzero_ps();

    int i;
    for (i = 0; i + 3 < n; i += 4) {
        __m128 va = _mm_loadu_ps(&a[i]);
        __m128 vb = _mm_loadu_ps(&b[i]);
        __m128 prod = _mm_mul_ps(va, vb);
        sum = _mm_add_ps(sum, prod);
    }

    // 水平求和
    sum = _mm_hadd_ps(sum, sum);  // [a+b, c+d, a+b, c+d]
    sum = _mm_hadd_ps(sum, sum);  // [a+b+c+d, ...]

    float result = _mm_cvtss_f32(sum);

    // 处理剩余
    for (; i < n; i++) {
        result += a[i] * b[i];
    }

    return result;
}

// 向量求最大值
float sse_max(const float* data, int n) {
    __m128 max_vec = _mm_set1_ps(-INFINITY);

    int i;
    for (i = 0; i + 3 < n; i += 4) {
        __m128 v = _mm_loadu_ps(&data[i]);
        max_vec = _mm_max_ps(max_vec, v);
    }

    // 水平求最大
    float max_arr[4];
    _mm_storeu_ps(max_arr, max_vec);
    float result = std::max({max_arr[0], max_arr[1], max_arr[2], max_arr[3]});

    for (; i < n; i++) {
        result = std::max(result, data[i]);
    }

    return result;
}
```

#### 3.1.2 内存操作

```cpp
// 加载/存储
__m128 v1 = _mm_load_ps(aligned_ptr);    // 对齐加载（快）
__m128 v2 = _mm_loadu_ps(unaligned_ptr); // 非对齐加载（慢5-10%）
_mm_store_ps(aligned_ptr, v1);           // 对齐存储
_mm_storeu_ps(unaligned_ptr, v2);        // 非对齐存储

// 流式存储（绕过Cache）
_mm_stream_ps(ptr, v1);  // 直接写入内存，不污染Cache

// 预取
_mm_prefetch((const char*)ptr, _MM_HINT_T0);  // 预取到L1
_mm_prefetch((const char*)ptr, _MM_HINT_T1);  // 预取到L2
_mm_prefetch((const char*)ptr, _MM_HINT_NTA); // 非时间局部性预取

// 设置向量
__m128 zero = _mm_setzero_ps();           // [0, 0, 0, 0]
__m128 ones = _mm_set1_ps(1.0f);          // [1, 1, 1, 1]
__m128 v = _mm_set_ps(4, 3, 2, 1);        // [1, 2, 3, 4] 注意顺序！
__m128 vr = _mm_setr_ps(1, 2, 3, 4);      // [1, 2, 3, 4] 正序
```

#### 3.1.3 Shuffle与Permute

```cpp
// Shuffle示例
__m128 a = _mm_setr_ps(1, 2, 3, 4);
__m128 b = _mm_setr_ps(5, 6, 7, 8);

// _MM_SHUFFLE(z, y, x, w) 从右到左选择元素
__m128 c = _mm_shuffle_ps(a, b, _MM_SHUFFLE(0, 1, 2, 3));
// 结果：[a[3], a[2], b[1], b[0]] = [4, 3, 6, 5]

// Unpack操作（交错）
__m128 low = _mm_unpacklo_ps(a, b);   // [1, 5, 2, 6]
__m128 high = _mm_unpackhi_ps(a, b);  // [3, 7, 4, 8]

// AoS到SoA转换（Structure of Arrays）
void aos_to_soa_sse(const float* aos, float* x, float* y, float* z, float* w, int n) {
    // aos: [x0,y0,z0,w0, x1,y1,z1,w1, x2,y2,z2,w2, x3,y3,z3,w3, ...]

    for (int i = 0; i < n; i += 4) {
        __m128 v0 = _mm_loadu_ps(&aos[i*4 + 0]);   // [x0,y0,z0,w0]
        __m128 v1 = _mm_loadu_ps(&aos[i*4 + 4]);   // [x1,y1,z1,w1]
        __m128 v2 = _mm_loadu_ps(&aos[i*4 + 8]);   // [x2,y2,z2,w2]
        __m128 v3 = _mm_loadu_ps(&aos[i*4 + 12]);  // [x3,y3,z3,w3]

        // 转置4x4矩阵
        _MM_TRANSPOSE4_PS(v0, v1, v2, v3);
        // v0 = [x0,x1,x2,x3]
        // v1 = [y0,y1,y2,y3]
        // v2 = [z0,z1,z2,z3]
        // v3 = [w0,w1,w2,w3]

        _mm_storeu_ps(&x[i], v0);
        _mm_storeu_ps(&y[i], v1);
        _mm_storeu_ps(&z[i], v2);
        _mm_storeu_ps(&w[i], v3);
    }
}
```

### 3.2 AVX/AVX2 Intrinsics

#### 3.2.1 AVX基本运算

```cpp
#include <immintrin.h>  // AVX/AVX2/FMA

// 向量加法（8路并行）
void avx_add(float* c, const float* a, const float* b, int n) {
    int i;
    for (i = 0; i + 7 < n; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        __m256 vc = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(&c[i], vc);
    }

    for (; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}

// FMA（Fused Multiply-Add）：一条指令完成a*b+c
void avx_fma(float* d, const float* a, const float* b, const float* c, int n) {
    for (int i = 0; i + 7 < n; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        __m256 vc = _mm256_loadu_ps(&c[i]);
        __m256 vd = _mm256_fmadd_ps(va, vb, vc);  // a*b + c
        _mm256_storeu_ps(&d[i], vd);
    }
}

// 点积优化（使用FMA）
float avx_dot_fma(const float* a, const float* b, int n) {
    __m256 sum = _mm256_setzero_ps();

    int i;
    for (i = 0; i + 7 < n; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        sum = _mm256_fmadd_ps(va, vb, sum);  // sum += a * b
    }

    // 水平求和
    float result[8];
    _mm256_storeu_ps(result, sum);
    float total = result[0] + result[1] + result[2] + result[3] +
                  result[4] + result[5] + result[6] + result[7];

    for (; i < n; i++) {
        total += a[i] * b[i];
    }

    return total;
}
```

#### 3.2.2 AVX2整数运算

```cpp
// 整数向量加法（8个int32）
void avx2_add_i32(int32_t* c, const int32_t* a, const int32_t* b, int n) {
    for (int i = 0; i + 7 < n; i += 8) {
        __m256i va = _mm256_loadu_si256((__m256i*)&a[i]);
        __m256i vb = _mm256_loadu_si256((__m256i*)&b[i]);
        __m256i vc = _mm256_add_epi32(va, vb);
        _mm256_storeu_si256((__m256i*)&c[i], vc);
    }
}

// 字节求和（32个int8）
int avx2_sum_i8(const int8_t* data, int n) {
    __m256i sum = _mm256_setzero_si256();

    int i;
    for (i = 0; i + 31 < n; i += 32) {
        __m256i v = _mm256_loadu_si256((__m256i*)&data[i]);
        // 累加需要扩展避免溢出
        __m256i v_lo = _mm256_cvtepi8_epi16(_mm256_extracti128_si256(v, 0));
        __m256i v_hi = _mm256_cvtepi8_epi16(_mm256_extracti128_si256(v, 1));
        sum = _mm256_add_epi16(sum, v_lo);
        sum = _mm256_add_epi16(sum, v_hi);
    }

    // 水平求和（省略细节）
    // ...
}

// Gather操作（间接访问）
void avx2_gather(float* out, const float* in, const int* indices, int n) {
    for (int i = 0; i + 7 < n; i += 8) {
        __m256i vidx = _mm256_loadu_si256((__m256i*)&indices[i]);
        __m256 v = _mm256_i32gather_ps(in, vidx, 4);  // scale=4 (sizeof(float))
        _mm256_storeu_ps(&out[i], v);
    }
}
// 注意：Gather性能较差，仅在必要时使用
```

### 3.3 AVX-512 Intrinsics

#### 3.3.1 AVX-512基础

```cpp
#ifdef __AVX512F__
#include <immintrin.h>

// 向量加法（16路并行）
void avx512_add(float* c, const float* a, const float* b, int n) {
    for (int i = 0; i + 15 < n; i += 16) {
        __m512 va = _mm512_loadu_ps(&a[i]);
        __m512 vb = _mm512_loadu_ps(&b[i]);
        __m512 vc = _mm512_add_ps(va, vb);
        _mm512_storeu_ps(&c[i], vc);
    }
}

// 归约求和（内置）
float avx512_sum(const float* data, int n) {
    __m512 sum = _mm512_setzero_ps();

    int i;
    for (i = 0; i + 15 < n; i += 16) {
        __m512 v = _mm512_loadu_ps(&data[i]);
        sum = _mm512_add_ps(sum, v);
    }

    // 内置归约（非常高效！）
    float result = _mm512_reduce_add_ps(sum);

    for (; i < n; i++) {
        result += data[i];
    }

    return result;
}
```

#### 3.3.2 Mask操作（条件向量化）

```cpp
// AVX-512引入了mask寄存器（k0-k7）
void avx512_conditional(float* out, const float* in, int n, float threshold) {
    for (int i = 0; i + 15 < n; i += 16) {
        __m512 v = _mm512_loadu_ps(&in[i]);
        __m512 thresh = _mm512_set1_ps(threshold);

        // 比较生成mask
        __mmask16 mask = _mm512_cmp_ps_mask(v, thresh, _CMP_GT_OQ);

        // 条件执行：只处理mask为1的元素
        __m512 result = _mm512_mask_mul_ps(v, mask, v, _mm512_set1_ps(2.0f));
        // 等价于：if (in[i] > threshold) out[i] = in[i] * 2; else out[i] = in[i];

        _mm512_storeu_ps(&out[i], result);
    }
}

// ReLU激活函数（max(0, x)）
void avx512_relu(float* data, int n) {
    __m512 zero = _mm512_setzero_ps();

    for (int i = 0; i + 15 < n; i += 16) {
        __m512 v = _mm512_loadu_ps(&data[i]);
        __m512 result = _mm512_max_ps(v, zero);
        _mm512_storeu_ps(&data[i], result);
    }
}

// 条件赋值
void avx512_clip(float* data, int n, float min_val, float max_val) {
    __m512 vmin = _mm512_set1_ps(min_val);
    __m512 vmax = _mm512_set1_ps(max_val);

    for (int i = 0; i + 15 < n; i += 16) {
        __m512 v = _mm512_loadu_ps(&data[i]);
        v = _mm512_max_ps(v, vmin);  // data[i] = max(data[i], min_val)
        v = _mm512_min_ps(v, vmax);  // data[i] = min(data[i], max_val)
        _mm512_storeu_ps(&data[i], v);
    }
}
```

#### 3.3.3 AVX-512 VNNI（INT8加速）

```cpp
#ifdef __AVX512VNNI__
// INT8点积（AI推理加速）
int32_t avx512_vnni_dot_i8(const int8_t* a, const int8_t* b, int n) {
    __m512i acc = _mm512_setzero_si512();

    for (int i = 0; i + 63 < n; i += 64) {
        __m512i va = _mm512_loadu_si512((__m512i*)&a[i]);
        __m512i vb = _mm512_loadu_si512((__m512i*)&b[i]);

        // VNNI指令：一条指令完成4个乘法+累加
        acc = _mm512_dpbusd_epi32(acc, va, vb);
    }

    // 归约求和
    return _mm512_reduce_add_epi32(acc);
}
// 性能：比标量快100-200倍！
#endif
```

---

## 第4章 SIMD算法优化

### 4.1 数学函数向量化

#### 4.1.1 近似函数

```cpp
// Fast reciprocal（快速倒数）
__m256 fast_rcp_ps(__m256 x) {
    // 硬件近似（12位精度）
    __m256 rcp = _mm256_rcp_ps(x);

    // Newton-Raphson迭代提升精度
    // x_new = x_old * (2 - a * x_old)
    __m256 two = _mm256_set1_ps(2.0f);
    rcp = _mm256_mul_ps(rcp, _mm256_fnmadd_ps(x, rcp, two));

    return rcp;  // 精度提升到~24位
}

// Fast inverse square root（快速平方根倒数）
__m256 fast_rsqrt_ps(__m256 x) {
    __m256 rsqrt = _mm256_rsqrt_ps(x);

    // Newton-Raphson迭代
    __m256 half = _mm256_set1_ps(0.5f);
    __m256 three = _mm256_set1_ps(3.0f);
    __m256 x_half = _mm256_mul_ps(x, half);
    rsqrt = _mm256_mul_ps(rsqrt, _mm256_fnmadd_ps(x_half,
                          _mm256_mul_ps(rsqrt, rsqrt), three));

    return rsqrt;
}

// 向量归一化（使用fast_rsqrt）
void normalize_vectors_simd(float* vectors, int num_vectors) {
    // 每个向量3个float (x, y, z)
    for (int i = 0; i < num_vectors; i += 8) {
        // 加载8个向量的x分量
        __m256 x = _mm256_loadu_ps(&vectors[i*3 + 0]);
        __m256 y = _mm256_loadu_ps(&vectors[i*3 + 8]);
        __m256 z = _mm256_loadu_ps(&vectors[i*3 + 16]);

        // 计算长度平方：x^2 + y^2 + z^2
        __m256 len2 = _mm256_fmadd_ps(x, x,
                      _mm256_fmadd_ps(y, y, _mm256_mul_ps(z, z)));

        // 快速平方根倒数
        __m256 inv_len = fast_rsqrt_ps(len2);

        // 归一化
        x = _mm256_mul_ps(x, inv_len);
        y = _mm256_mul_ps(y, inv_len);
        z = _mm256_mul_ps(z, inv_len);

        // 存储
        _mm256_storeu_ps(&vectors[i*3 + 0], x);
        _mm256_storeu_ps(&vectors[i*3 + 8], y);
        _mm256_storeu_ps(&vectors[i*3 + 16], z);
    }
}
// 性能提升：10-15倍
```

#### 4.1.2 超越函数（exp, log, sin, cos）

```cpp
// 快速exp近似（使用泰勒级数）
__m256 fast_exp_ps(__m256 x) {
    // exp(x) ≈ 2^(x/ln2)
    __m256 log2e = _mm256_set1_ps(1.44269504f);  // 1/ln(2)
    x = _mm256_mul_ps(x, log2e);

    // 分离整数和小数部分
    __m256 fx = _mm256_floor_ps(x);
    x = _mm256_sub_ps(x, fx);

    // 泰勒展开 2^x ≈ 1 + x*ln2 + (x*ln2)^2/2 + ...
    __m256 ln2 = _mm256_set1_ps(0.693147181f);
    __m256 y = _mm256_mul_ps(x, ln2);
    __m256 z = _mm256_fmadd_ps(y, _mm256_set1_ps(0.5f), _mm256_set1_ps(1.0f));
    z = _mm256_fmadd_ps(y, z, _mm256_set1_ps(1.0f));

    // 2^整数部分（位运算）
    __m256i pow2 = _mm256_cvttps_epi32(_mm256_add_ps(fx, _mm256_set1_ps(127.0f)));
    pow2 = _mm256_slli_epi32(pow2, 23);  // 左移23位=乘以2^23

    return _mm256_mul_ps(z, _mm256_castsi256_ps(pow2));
}
```

### 4.2 字符串SIMD优化

#### 4.2.1 字符串比较

```cpp
// SIMD字符串相等比较
bool simd_strcmp(const char* s1, const char* s2, size_t len) {
    size_t i;

    for (i = 0; i + 31 < len; i += 32) {
        __m256i v1 = _mm256_loadu_si256((__m256i*)&s1[i]);
        __m256i v2 = _mm256_loadu_si256((__m256i*)&s2[i]);
        __m256i cmp = _mm256_cmpeq_epi8(v1, v2);

        int mask = _mm256_movemask_epi8(cmp);
        if (mask != -1) {  // 不是全部相等
            return false;
        }
    }

    // 处理剩余
    for (; i < len; i++) {
        if (s1[i] != s2[i]) return false;
    }

    return true;
}
// 性能提升：10-20倍
```

#### 4.2.2 字符串搜索

```cpp
// 查找字符在字符串中的位置
int simd_strchr(const char* str, char c, size_t len) {
    __m256i target = _mm256_set1_epi8(c);

    for (size_t i = 0; i + 31 < len; i += 32) {
        __m256i v = _mm256_loadu_si256((__m256i*)&str[i]);
        __m256i cmp = _mm256_cmpeq_epi8(v, target);

        int mask = _mm256_movemask_epi8(cmp);
        if (mask != 0) {
            // 找到第一个匹配
            return i + __builtin_ctz(mask);
        }
    }

    return -1;  // 未找到
}
```

### 4.3 排序与搜索

#### 4.3.1 Bitonic排序网络

```cpp
// 8个元素的bitonic排序（AVX）
__m256 bitonic_sort_8(__m256 v) {
    // Stage 1
    __m256 tmp = _mm256_shuffle_ps(v, v, _MM_SHUFFLE(2,3,0,1));
    __m256 min1 = _mm256_min_ps(v, tmp);
    __m256 max1 = _mm256_max_ps(v, tmp);
    v = _mm256_blend_ps(min1, max1, 0b10101010);

    // Stage 2
    tmp = _mm256_shuffle_ps(v, v, _MM_SHUFFLE(1,0,3,2));
    __m256 min2 = _mm256_min_ps(v, tmp);
    __m256 max2 = _mm256_max_ps(v, tmp);
    v = _mm256_blend_ps(min2, max2, 0b11001100);

    // Stage 3
    tmp = _mm256_permute2f128_ps(v, v, 1);
    __m256 min3 = _mm256_min_ps(v, tmp);
    __m256 max3 = _mm256_max_ps(v, tmp);
    v = _mm256_blend_ps(min3, max3, 0b11110000);

    return v;
}
```

#### 4.3.2 SIMD二分查找

```cpp
// 并行二分查找（在已排序数组中查找16个值）
void simd_binary_search_avx512(const float* sorted_array, int n,
                                const float* queries, int* results, int num_queries) {
    for (int i = 0; i + 15 < num_queries; i += 16) {
        __m512 q = _mm512_loadu_ps(&queries[i]);
        __m512i result_idx = _mm512_setzero_si512();

        int left = 0, right = n - 1;
        while (left < right) {
            int mid = (left + right) / 2;
            __m512 mid_val = _mm512_set1_ps(sorted_array[mid]);

            __mmask16 mask = _mm512_cmp_ps_mask(q, mid_val, _CMP_GT_OQ);
            // 根据mask调整left或right（简化示例）
            // ...
        }

        _mm512_storeu_si512((__m512i*)&results[i], result_idx);
    }
}
```

### 4.4 图像处理

#### 4.4.1 RGB到灰度转换

```cpp
// Grayscale = 0.299*R + 0.587*G + 0.114*B
void rgb_to_gray_avx(const uint8_t* rgb, uint8_t* gray, int num_pixels) {
    __m256 coef_r = _mm256_set1_ps(0.299f);
    __m256 coef_g = _mm256_set1_ps(0.587f);
    __m256 coef_b = _mm256_set1_ps(0.114f);

    for (int i = 0; i < num_pixels; i += 8) {
        // 加载8个像素（24字节）
        __m128i rgb_u8 = _mm_loadu_si128((__m128i*)&rgb[i*3]);

        // 解交错RGB (需要复杂的shuffle操作)
        // 简化版：逐个处理
        uint8_t r[8], g[8], b[8];
        for (int j = 0; j < 8; j++) {
            r[j] = rgb[(i+j)*3 + 0];
            g[j] = rgb[(i+j)*3 + 1];
            b[j] = rgb[(i+j)*3 + 2];
        }

        // 转换为float并计算
        __m256i r_i = _mm256_cvtepu8_epi32(_mm_loadl_epi64((__m128i*)r));
        __m256i g_i = _mm256_cvtepu8_epi32(_mm_loadl_epi64((__m128i*)g));
        __m256i b_i = _mm256_cvtepu8_epi32(_mm_loadl_epi64((__m128i*)b));

        __m256 r_f = _mm256_cvtepi32_ps(r_i);
        __m256 g_f = _mm256_cvtepi32_ps(g_i);
        __m256 b_f = _mm256_cvtepi32_ps(b_i);

        __m256 result = _mm256_mul_ps(r_f, coef_r);
        result = _mm256_fmadd_ps(g_f, coef_g, result);
        result = _mm256_fmadd_ps(b_f, coef_b, result);

        // 转回uint8
        __m256i result_i = _mm256_cvtps_epi32(result);
        __m128i result_u8 = _mm256_cvtepi32_epi8(result_i);
        _mm_storel_epi64((__m128i*)&gray[i], result_u8);
    }
}
// 性能提升：5-8倍
```

#### 4.4.2 卷积滤波

```cpp
// 3x3高斯模糊（简化版）
void gaussian_blur_3x3_simd(const float* in, float* out, int width, int height) {
    // 高斯核
    alignas(32) float kernel[9] = {
        1/16.0f, 2/16.0f, 1/16.0f,
        2/16.0f, 4/16.0f, 2/16.0f,
        1/16.0f, 2/16.0f, 1/16.0f
    };

    for (int y = 1; y < height - 1; y++) {
        for (int x = 0; x + 7 < width - 1; x += 8) {
            __m256 sum = _mm256_setzero_ps();

            for (int ky = -1; ky <= 1; ky++) {
                for (int kx = -1; kx <= 1; kx++) {
                    __m256 pixel = _mm256_loadu_ps(&in[(y+ky)*width + x+kx]);
                    __m256 k = _mm256_set1_ps(kernel[(ky+1)*3 + (kx+1)]);
                    sum = _mm256_fmadd_ps(pixel, k, sum);
                }
            }

            _mm256_storeu_ps(&out[y*width + x], sum);
        }
    }
}
```

---

## 第5章 混合精度与量化

### 5.1 FP16半精度计算

#### 5.1.1 FP16基础

```cpp
#ifdef __F16C__
#include <immintrin.h>

// FP32转FP16
void fp32_to_fp16(const float* fp32, uint16_t* fp16, int n) {
    for (int i = 0; i + 7 < n; i += 8) {
        __m256 v = _mm256_loadu_ps(&fp32[i]);
        __m128i h = _mm256_cvtps_ph(v, _MM_FROUND_TO_NEAREST_INT);
        _mm_storeu_si128((__m128i*)&fp16[i], h);
    }
}

// FP16转FP32
void fp16_to_fp32(const uint16_t* fp16, float* fp32, int n) {
    for (int i = 0; i + 7 < n; i += 8) {
        __m128i h = _mm_loadu_si128((__m128i*)&fp16[i]);
        __m256 v = _mm256_cvtph_ps(h);
        _mm256_storeu_ps(&fp32[i], v);
    }
}

// FP16点积（转换到FP32计算）
float fp16_dot_product(const uint16_t* a, const uint16_t* b, int n) {
    __m256 sum = _mm256_setzero_ps();

    for (int i = 0; i + 7 < n; i += 8) {
        __m128i ha = _mm_loadu_si128((__m128i*)&a[i]);
        __m128i hb = _mm_loadu_si128((__m128i*)&b[i]);

        __m256 va = _mm256_cvtph_ps(ha);
        __m256 vb = _mm256_cvtph_ps(hb);

        sum = _mm256_fmadd_ps(va, vb, sum);
    }

    return _mm256_reduce_add_ps(sum);
}
// 内存节省：50%，性能略有下降（转换开销）
#endif
```

### 5.2 INT8量化

#### 5.2.1 对称量化

```cpp
// 对称量化：量化值 = round(原始值 / scale)
struct QuantizedTensor {
    int8_t* data;
    float scale;
    int size;
};

// FP32 -> INT8量化
QuantizedTensor quantize_symmetric(const float* fp32, int n) {
    // 找到绝对值最大值
    float max_val = 0.0f;
    for (int i = 0; i < n; i++) {
        max_val = std::max(max_val, std::abs(fp32[i]));
    }

    float scale = max_val / 127.0f;
    int8_t* quantized = new int8_t[n];

    // AVX2量化
    __m256 scale_vec = _mm256_set1_ps(1.0f / scale);
    for (int i = 0; i + 7 < n; i += 8) {
        __m256 v = _mm256_loadu_ps(&fp32[i]);
        v = _mm256_mul_ps(v, scale_vec);

        // 四舍五入并转换为int32
        __m256i v_i32 = _mm256_cvtps_epi32(v);

        // int32 -> int8（饱和）
        __m128i v_i16 = _mm256_cvtepi32_epi16(v_i32);
        __m128i v_i8 = _mm_cvtepi16_epi8(v_i16);

        _mm_storel_epi64((__m128i*)&quantized[i], v_i8);
    }

    return {quantized, scale, n};
}

// INT8 -> FP32反量化
void dequantize_symmetric(const QuantizedTensor& q, float* fp32) {
    __m256 scale_vec = _mm256_set1_ps(q.scale);

    for (int i = 0; i + 7 < q.size; i += 8) {
        // 加载int8
        __m128i v_i8 = _mm_loadl_epi64((__m128i*)&q.data[i]);

        // int8 -> int32
        __m256i v_i32 = _mm256_cvtepi8_epi32(v_i8);

        // int32 -> float
        __m256 v_f = _mm256_cvtepi32_ps(v_i32);

        // 乘以scale
        v_f = _mm256_mul_ps(v_f, scale_vec);

        _mm256_storeu_ps(&fp32[i], v_f);
    }
}
```

#### 5.2.2 INT8矩阵乘法（VNNI）

```cpp
#ifdef __AVX512VNNI__
// INT8矩阵乘法：C = A * B
// A: M x K (int8), B: K x N (int8), C: M x N (int32)
void matmul_int8_vnni(const int8_t* A, const int8_t* B, int32_t* C,
                      int M, int N, int K) {
    for (int m = 0; m < M; m++) {
        for (int n = 0; n + 15 < N; n += 16) {
            __m512i acc = _mm512_setzero_si512();

            for (int k = 0; k + 3 < K; k += 4) {
                // 加载A的4个元素（广播）
                int32_t a_val;
                memcpy(&a_val, &A[m*K + k], 4);
                __m512i a_vec = _mm512_set1_epi32(a_val);

                // 加载B的4x16块
                __m512i b_vec = _mm512_loadu_si512(&B[k*N + n]);

                // VNNI指令：4个乘法+累加
                acc = _mm512_dpbusd_epi32(acc, a_vec, b_vec);
            }

            _mm512_storeu_si512(&C[m*N + n], acc);
        }
    }
}
// 性能：比FP32快10-20倍，内存减少75%
#endif
```

### 5.3 动态量化

```cpp
// 动态量化：运行时计算scale
class DynamicQuantizer {
public:
    static void quantize_dynamic(const float* input, int8_t* output,
                                  float& scale, int n) {
        // 1. 找最大值（SIMD）
        __m256 max_vec = _mm256_setzero_ps();
        for (int i = 0; i + 7 < n; i += 8) {
            __m256 v = _mm256_loadu_ps(&input[i]);
            v = _mm256_andnot_ps(_mm256_set1_ps(-0.0f), v);  // abs
            max_vec = _mm256_max_ps(max_vec, v);
        }
        float max_val = _mm256_reduce_max_ps(max_vec);

        // 2. 计算scale
        scale = max_val / 127.0f;

        // 3. 量化
        __m256 scale_inv = _mm256_set1_ps(1.0f / scale);
        for (int i = 0; i + 7 < n; i += 8) {
            __m256 v = _mm256_loadu_ps(&input[i]);
            v = _mm256_mul_ps(v, scale_inv);
            __m256i v_i32 = _mm256_cvtps_epi32(v);
            // 转换为int8...
        }
    }
};
```

---

## 第6章 性能陷阱与调优

### 6.1 常见性能陷阱

#### 6.1.1 未对齐访问

```cpp
// ❌ 不好：未对齐访问
float data[100];  // 可能未对齐
__m256 v = _mm256_load_ps(data);  // 如果未对齐会崩溃！

// ✅ 好：使用loadu或保证对齐
__m256 v = _mm256_loadu_ps(data);  // 安全但慢

// ✅ 更好：保证对齐
alignas(32) float data[100];
__m256 v = _mm256_load_ps(data);  // 快且安全
```

**性能影响：**
- 对齐加载：1 cycle
- 未对齐加载：2-3 cycles
- 跨越Cache Line边界：10+ cycles

#### 6.1.2 过度使用Shuffle

```cpp
// ❌ 不好：频繁shuffle
for (int i = 0; i < n; i += 8) {
    __m256 v = _mm256_loadu_ps(&data[i]);
    v = _mm256_shuffle_ps(v, v, _MM_SHUFFLE(2,3,0,1));  // 1 cycle
    v = _mm256_permute_ps(v, 0x1b);                     // 1 cycle
    v = _mm256_permute2f128_ps(v, v, 1);                // 3 cycles
    // ...总共5+ cycles仅用于数据重排
}

// ✅ 好：优化数据布局避免shuffle
// 使用SoA而不是AoS
```

#### 6.1.3 Gather/Scatter滥用

```cpp
// ❌ 不好：使用gather（性能差）
__m256i indices = _mm256_loadu_si256((__m256i*)idx);
__m256 v = _mm256_i32gather_ps(data, indices, 4);
// Gather延迟：~10-20 cycles

// ✅ 好：重组数据避免gather
// 预处理：将需要的数据打包到连续内存
```

#### 6.1.4 过早优化小数据

```cpp
// ❌ 不好：对小数据用SIMD
void add_small(float* c, const float* a, const float* b) {
    __m256 va = _mm256_loadu_ps(a);  // 假设只有8个元素
    __m256 vb = _mm256_loadu_ps(b);
    __m256 vc = _mm256_add_ps(va, vb);
    _mm256_storeu_ps(c, vc);
}
// 对于8个元素，SIMD开销可能大于收益

// ✅ 好：有阈值判断
void add_auto(float* c, const float* a, const float* b, int n) {
    if (n < 32) {
        // 标量版本
        for (int i = 0; i < n; i++) c[i] = a[i] + b[i];
    } else {
        // SIMD版本
        // ...
    }
}
```

### 6.2 SIMD调优技巧

#### 6.2.1 循环展开

```cpp
// 基础版本
void sum_basic(float* out, const float* in, int n) {
    for (int i = 0; i + 7 < n; i += 8) {
        __m256 v = _mm256_loadu_ps(&in[i]);
        // process v
    }
}

// 展开2倍（隐藏延迟）
void sum_unroll2(float* out, const float* in, int n) {
    for (int i = 0; i + 15 < n; i += 16) {
        __m256 v0 = _mm256_loadu_ps(&in[i]);
        __m256 v1 = _mm256_loadu_ps(&in[i+8]);
        // process v0, v1 in parallel
    }
}

// 展开4倍（最佳ILP）
void sum_unroll4(float* out, const float* in, int n) {
    for (int i = 0; i + 31 < n; i += 32) {
        __m256 v0 = _mm256_loadu_ps(&in[i]);
        __m256 v1 = _mm256_loadu_ps(&in[i+8]);
        __m256 v2 = _mm256_loadu_ps(&in[i+16]);
        __m256 v3 = _mm256_loadu_ps(&in[i+24]);
        // 四个独立流水线
    }
}
// 性能提升：1.5-2倍
```

#### 6.2.2 多累加器

```cpp
// ❌ 单累加器（依赖链长）
float dot_single(const float* a, const float* b, int n) {
    __m256 sum = _mm256_setzero_ps();
    for (int i = 0; i + 7 < n; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        sum = _mm256_fmadd_ps(va, vb, sum);  // 依赖前一次的sum
    }
    return _mm256_reduce_add_ps(sum);
}
// FMA延迟：4-5 cycles，吞吐量：0.5 cycles
// 依赖链限制了ILP

// ✅ 4个累加器（打破依赖）
float dot_multi(const float* a, const float* b, int n) {
    __m256 sum0 = _mm256_setzero_ps();
    __m256 sum1 = _mm256_setzero_ps();
    __m256 sum2 = _mm256_setzero_ps();
    __m256 sum3 = _mm256_setzero_ps();

    for (int i = 0; i + 31 < n; i += 32) {
        __m256 va0 = _mm256_loadu_ps(&a[i]);
        __m256 vb0 = _mm256_loadu_ps(&b[i]);
        sum0 = _mm256_fmadd_ps(va0, vb0, sum0);

        __m256 va1 = _mm256_loadu_ps(&a[i+8]);
        __m256 vb1 = _mm256_loadu_ps(&b[i+8]);
        sum1 = _mm256_fmadd_ps(va1, vb1, sum1);

        __m256 va2 = _mm256_loadu_ps(&a[i+16]);
        __m256 vb2 = _mm256_loadu_ps(&b[i+16]);
        sum2 = _mm256_fmadd_ps(va2, vb2, sum2);

        __m256 va3 = _mm256_loadu_ps(&a[i+24]);
        __m256 vb3 = _mm256_loadu_ps(&b[i+24]);
        sum3 = _mm256_fmadd_ps(va3, vb3, sum3);
    }

    sum0 = _mm256_add_ps(sum0, sum1);
    sum2 = _mm256_add_ps(sum2, sum3);
    sum0 = _mm256_add_ps(sum0, sum2);

    return _mm256_reduce_add_ps(sum0);
}
// 性能提升：3-4倍
```

#### 6.2.3 软件流水

```cpp
// 软件流水：预取下一轮数据
void pipelined_process(float* out, const float* in, int n) {
    if (n < 16) return;

    // 预加载第一批
    __m256 v0 = _mm256_loadu_ps(&in[0]);
    __m256 v1 = _mm256_loadu_ps(&in[8]);

    for (int i = 0; i + 31 < n; i += 16) {
        // 预取下一轮
        _mm_prefetch((const char*)&in[i+32], _MM_HINT_T0);

        // 处理当前批
        __m256 result0 = _mm256_mul_ps(v0, v0);
        __m256 result1 = _mm256_mul_ps(v1, v1);

        // 加载下一批（与计算重叠）
        __m256 v0_next = _mm256_loadu_ps(&in[i+16]);
        __m256 v1_next = _mm256_loadu_ps(&in[i+24]);

        // 存储结果
        _mm256_storeu_ps(&out[i], result0);
        _mm256_storeu_ps(&out[i+8], result1);

        // 移动到下一批
        v0 = v0_next;
        v1 = v1_next;
    }
}
```

### 6.3 性能测量

#### 6.3.1 微基准测试

```cpp
#include <benchmark/benchmark.h>

static void BM_Scalar_Sum(benchmark::State& state) {
    const int n = state.range(0);
    std::vector<float> data(n, 1.0f);

    for (auto _ : state) {
        float sum = 0;
        for (int i = 0; i < n; i++) {
            sum += data[i];
        }
        benchmark::DoNotOptimize(sum);
    }

    state.SetItemsProcessed(state.iterations() * n);
    state.SetBytesProcessed(state.iterations() * n * sizeof(float));
}
BENCHMARK(BM_Scalar_Sum)->Range(1024, 1<<20);

static void BM_AVX_Sum(benchmark::State& state) {
    const int n = state.range(0);
    alignas(32) std::vector<float> data(n, 1.0f);

    for (auto _ : state) {
        __m256 sum_vec = _mm256_setzero_ps();
        for (int i = 0; i + 7 < n; i += 8) {
            __m256 v = _mm256_load_ps(&data[i]);
            sum_vec = _mm256_add_ps(sum_vec, v);
        }
        float sum = _mm256_reduce_add_ps(sum_vec);
        benchmark::DoNotOptimize(sum);
    }

    state.SetItemsProcessed(state.iterations() * n);
    state.SetBytesProcessed(state.iterations() * n * sizeof(float));
}
BENCHMARK(BM_AVX_Sum)->Range(1024, 1<<20);

BENCHMARK_MAIN();
```

**运行结果示例：**
```
---------------------------------------------------------------
Benchmark                    Time             CPU   Iterations
---------------------------------------------------------------
BM_Scalar_Sum/1024        1250 ns         1250 ns       560000   Throughput: 3.2 GB/s
BM_AVX_Sum/1024            180 ns          180 ns      3890000   Throughput: 22.7 GB/s
```

#### 6.3.2 perf性能分析

```bash
# 编译
g++ -O3 -march=native -g simd_app.cpp -o simd_app

# perf stat
perf stat -e cycles,instructions,branches,branch-misses,\
cache-references,cache-misses,\
fp_arith_inst_retired.128b_packed_single,\
fp_arith_inst_retired.256b_packed_single \
./simd_app

# 输出示例：
#  10,000,000,000  cycles
#  40,000,000,000  instructions      # IPC = 4.0（很好！）
#     100,000,000  branches
#       1,000,000  branch-misses     # 1% miss rate（很好）
#   2,000,000,000  fp_arith_inst_retired.256b_packed_single  # AVX使用率
```

---

## 第7章 实战案例

### 7.1 案例1：神经网络推理加速

#### 7.1.1 全连接层（FP32）

```cpp
// 矩阵向量乘法：y = Wx + b
// W: M x N, x: N x 1, y: M x 1
void fc_layer_avx(const float* W, const float* x, const float* b,
                  float* y, int M, int N) {
    for (int m = 0; m < M; m++) {
        __m256 sum = _mm256_setzero_ps();

        int n;
        for (n = 0; n + 7 < N; n += 8) {
            __m256 w = _mm256_loadu_ps(&W[m*N + n]);
            __m256 x_vec = _mm256_loadu_ps(&x[n]);
            sum = _mm256_fmadd_ps(w, x_vec, sum);
        }

        float result = _mm256_reduce_add_ps(sum);

        // 处理剩余
        for (; n < N; n++) {
            result += W[m*N + n] * x[n];
        }

        y[m] = result + b[m];
    }
}
// 性能提升：6-8倍
```

#### 7.1.2 卷积层（INT8量化）

```cpp
#ifdef __AVX512VNNI__
// 1D卷积（量化版本）
void conv1d_int8_vnni(const int8_t* input, const int8_t* kernel,
                      int32_t* output,
                      int input_len, int kernel_size, int stride) {
    for (int i = 0; i + kernel_size <= input_len; i += stride) {
        __m512i acc = _mm512_setzero_si512();

        for (int k = 0; k + 63 < kernel_size; k += 64) {
            __m512i in_vec = _mm512_loadu_si512(&input[i + k]);
            __m512i ker_vec = _mm512_loadu_si512(&kernel[k]);

            // VNNI: 4个int8乘法 + int32累加
            acc = _mm512_dpbusd_epi32(acc, in_vec, ker_vec);
        }

        output[i/stride] = _mm512_reduce_add_epi32(acc);
    }
}
// 性能提升：20-50倍（相比FP32）
#endif
```

### 7.2 案例2：音频DSP

#### 7.2.1 FIR滤波器

```cpp
// FIR滤波：y[n] = Σ h[k] * x[n-k]
void fir_filter_avx(const float* input, const float* coeffs,
                    float* output, int signal_len, int num_taps) {
    for (int n = num_taps - 1; n < signal_len; n++) {
        __m256 sum = _mm256_setzero_ps();

        int k;
        for (k = 0; k + 7 < num_taps; k += 8) {
            __m256 h = _mm256_loadu_ps(&coeffs[k]);
            __m256 x = _mm256_loadu_ps(&input[n - k - 7]);  // 反向
            x = _mm256_permute_ps(x, _MM_SHUFFLE(0,1,2,3)); // 翻转
            sum = _mm256_fmadd_ps(h, x, sum);
        }

        output[n] = _mm256_reduce_add_ps(sum);
    }
}
```

#### 7.2.2 FFT（Fast Fourier Transform）

```cpp
// Cooley-Tukey FFT（简化版，N=8）
void fft_8_avx(const float* real_in, const float* imag_in,
               float* real_out, float* imag_out) {
    // 旋转因子
    alignas(32) float cos_table[4] = {1.0f, 0.707f, 0.0f, -0.707f};
    alignas(32) float sin_table[4] = {0.0f, -0.707f, -1.0f, -0.707f};

    __m256 r_in = _mm256_loadu_ps(real_in);
    __m256 i_in = _mm256_loadu_ps(imag_in);

    // Butterfly操作（简化）
    // Stage 1
    __m256 r_tmp = _mm256_shuffle_ps(r_in, r_in, _MM_SHUFFLE(2,3,0,1));
    __m256 i_tmp = _mm256_shuffle_ps(i_in, i_in, _MM_SHUFFLE(2,3,0,1));

    __m256 r_out_vec = _mm256_add_ps(r_in, r_tmp);
    __m256 i_out_vec = _mm256_add_ps(i_in, i_tmp);

    _mm256_storeu_ps(real_out, r_out_vec);
    _mm256_storeu_ps(imag_out, i_out_vec);
}
```

### 7.3 案例3：科学计算

#### 7.3.1 矩阵乘法（GEMM）

```cpp
// C = A * B (简化版，M=N=K，都是8的倍数)
void gemm_avx(const float* A, const float* B, float* C, int N) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j += 8) {
            __m256 c_vec = _mm256_setzero_ps();

            for (int k = 0; k < N; k++) {
                __m256 a_broadcast = _mm256_broadcast_ss(&A[i*N + k]);
                __m256 b_vec = _mm256_loadu_ps(&B[k*N + j]);
                c_vec = _mm256_fmadd_ps(a_broadcast, b_vec, c_vec);
            }

            _mm256_storeu_ps(&C[i*N + j], c_vec);
        }
    }
}
// 实际优化还需分块（tiling）、寄存器重用等
```

#### 7.3.2 蒙特卡洛模拟

```cpp
// 并行生成随机数+计算
#include <random>

double monte_carlo_pi_avx(int num_samples) {
    std::mt19937 gen(42);
    std::uniform_real_distribution<float> dis(0.0f, 1.0f);

    __m256 count_vec = _mm256_setzero_ps();
    __m256 one = _mm256_set1_ps(1.0f);

    for (int i = 0; i < num_samples; i += 8) {
        // 生成8对随机数
        alignas(32) float x[8], y[8];
        for (int j = 0; j < 8; j++) {
            x[j] = dis(gen);
            y[j] = dis(gen);
        }

        __m256 x_vec = _mm256_load_ps(x);
        __m256 y_vec = _mm256_load_ps(y);

        // 计算 x^2 + y^2
        __m256 dist2 = _mm256_fmadd_ps(x_vec, x_vec, _mm256_mul_ps(y_vec, y_vec));

        // 比较 < 1.0
        __m256 mask = _mm256_cmp_ps(dist2, one, _CMP_LT_OQ);

        // 累加（mask转换为1.0或0.0）
        count_vec = _mm256_add_ps(count_vec, _mm256_and_ps(mask, one));
    }

    double count = _mm256_reduce_add_ps(count_vec);
    return 4.0 * count / num_samples;
}
```

---

## 第8章 总结与最佳实践

### 8.1 SIMD优化检查清单

#### 编译前
- [ ] 检测CPU特性，确定目标指令集
- [ ] 准备多个版本（scalar, SSE, AVX, AVX-512）
- [ ] 设计运行时分发机制

#### 数据布局
- [ ] 确保关键数据32/64字节对齐
- [ ] 使用SoA而不是AoS（如果适用）
- [ ] 预处理数据避免shuffle/gather

#### 循环优化
- [ ] 循环展开2-4倍
- [ ] 使用多累加器打破依赖链
- [ ] 添加软件预取
- [ ] 处理循环剩余元素

#### 指令选择
- [ ] 优先使用FMA而不是MUL+ADD
- [ ] 避免频繁类型转换
- [ ] 最小化shuffle/permute操作
- [ ] 使用mask操作而不是分支

#### 验证
- [ ] 正确性测试（对比标量版本）
- [ ] 性能基准测试
- [ ] perf分析IPC、Cache miss
- [ ] 检查编译器生成的汇编

### 8.2 性能提升预期

| 场景 | SSE (4x) | AVX (8x) | AVX-512 (16x) | 备注 |
|------|----------|----------|---------------|------|
| 向量加法 | 3.5x | 7x | 14x | 内存带宽限制 |
| 点积 | 3x | 6x | 12x | FMA优化 |
| 矩阵乘法 | 2-3x | 4-6x | 8-12x | 需分块优化 |
| 字符串比较 | 8-12x | 16-24x | 32-48x | 无计算，纯内存 |
| INT8推理 | - | - | 20-50x | VNNI专用 |

### 8.3 推荐资源

**文档：**
- Intel Intrinsics Guide: https://software.intel.com/intrinsics
- Agner Fog优化手册: https://agner.org/optimize/

**工具：**
- Compiler Explorer (godbolt.org)
- Intel VTune Profiler
- perf + FlameGraph

**库：**
- Intel MKL (Math Kernel Library)
- Eigen (C++线性代数库)
- xsimd (C++跨平台SIMD抽象)

---

**恭喜你完成SIMD深度优化课程！** 🎉

通过掌握SIMD技术，你已经拥有了在现代CPU上实现极致性能的关键能力。记住：**测量、优化、再测量**，持续迭代才能达到最佳性能！
