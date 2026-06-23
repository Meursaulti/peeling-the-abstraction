# CS61kaChow

#### 类别：计算机体系结构 / SIMD / OpenMP / 性能优化

---

### 项目概述

CS61C Project 4 主要实现并优化二维卷积运算。

项目以视频处理中的图像卷积为背景，将灰度视频帧抽象为矩阵，通过卷积核对矩阵进行模糊、锐化等处理。实现上首先完成朴素版本的二维卷积，而后使用 SIMD 指令和 OpenMP 对其进行性能优化。

课程官方要求在 Hive 机器上完成测试与 benchmark。但由于我在本地环境运行时无法使用课程提供的 Oracle：

```text
Oracle does not exist, please run on the hive machines
```

因此我额外编写了一组自定义测试，用于验证卷积结果正确性，并进行一些较为粗略的本地基准测试。

---

# 实现流程

## 朴素二维卷积

朴素实现位于：

```text
src/compute_naive.c
```

项目中的矩阵使用 row-major 形式存储，本质上是一个一维 `int32_t` 数组。

对于输入矩阵 `A` 和卷积核 `B`，输出矩阵尺寸为：

```text
output_rows = a_rows - b_rows + 1
output_cols = a_cols - b_cols + 1
```

每个输出元素通过将卷积核覆盖到输入矩阵对应窗口，并进行逐元素乘加得到。

主要实现内容包括：

- 输入矩阵合法性检查
    
- 输出矩阵尺寸计算
    
- 输出矩阵内存分配
    
- 卷积核反向遍历
    
- 四重循环完成标量卷积
    

其中需要注意的是，这里的卷积不是 cross-correlation。按照项目的卷积定义，卷积核需要在两个维度上翻转。

因此实现中使用：

```c
const int *b_ptr = b_data + kernel_size - 1;
```

并在内部循环中通过：

```c
*b_ptr--
```

从卷积核末尾向前访问，从而完成 kernel flip。

---

## SIMD 优化

优化实现位于：

```text
src/compute_optimized.c
```

在朴素版本正确的基础上，优化版本使用 AVX2 SIMD 指令对输出矩阵的列方向进行并行计算。

由于一个 256-bit AVX 向量可以容纳 8 个 `int32_t`，因此优化版本一次计算同一行中连续的 8 个输出元素。

核心思路可以表示为：

```text
8 个相邻输出位置
    ↓
共享同一个卷积核元素
    ↓
从输入矩阵中加载 8 个连续元素
    ↓
向量乘法
    ↓
向量累加
```

主要使用的 intrinsic 包括：

- `_mm256_setzero_si256`
    
- `_mm256_loadu_si256`
    
- `_mm256_set1_epi32`
    
- `_mm256_mullo_epi32`
    
- `_mm256_add_epi32`
    
- `_mm256_storeu_si256`
    

这种写法避免了在 SIMD lane 内做 horizontal reduction。

每个 lane 对应一个独立的输出元素，最终可以直接写回输出矩阵，因此整体结构较为直接。

---

## OpenMP 并行化

在 SIMD 优化之外，项目还使用 OpenMP 对输出矩阵的不同行进行并行计算。

代码中使用：

```c
#pragma omp parallel for
```

将输出矩阵的不同行分配给多个线程。

由于每个输出元素之间没有数据依赖：

```text
A matrix       只读
B matrix       只读
Output matrix  分区域写入
```

所以按输出行划分任务不会产生写冲突

并行结构可以理解为：

```text
Output Row 0  → Thread 0
Output Row 1  → Thread 1
Output Row 2  → Thread 2
...
```

---

## Tail Case 处理

SIMD 每次处理 8 个输出列。

当输出列数不能被 8 整除时，需要额外处理剩余部分。

优化版本中先计算：

```c
int simd_block = output_cols / 8;
```

前 `simd_block * 8` 列使用 SIMD 处理，剩余列回退到标量循环。

例如当输出矩阵有 11 列时：

```text
前 8 列：SIMD
后 3 列：标量 tail case
```

这样既能利用向量化提升主要计算路径的性能，也能保证任意合法矩阵尺寸下的正确性。

---

# 自定义测试

由于无法在当前环境运行课程 Oracle，我在 `test_custom` 中编写并运行了自定义测试。

测试主要覆盖：

- 小矩阵基础正确性
    
- 非对称卷积核的反向遍历
    
- SIMD 正好覆盖 8 列的情况
    
- SIMD 与 tail case 同时出现的情况
    
- 朴素版本与优化版本输出一致性
    
- 大矩阵本地性能基准测试
    

已运行测试包括：

```text
test
test-native_8
test-native_11
test-optimized_8
test-optimized_11
test-optimized_11_03_omp
test_O0
test_O0_opt
test_O3_max
```

功能性测试均正常通过。

---

# 本地测试环境

本地测试环境如下：

```text
OS:       Linux 6.6.87.2-microsoft-standard-WSL2
CPU:      AMD Ryzen 9 7945HX with Radeon Graphics
Cores:    16 cores / 32 threads
Compiler: gcc 11.4.0
SIMD:     AVX2
Parallel: OpenMP
```

Makefile 中优化相关编译参数为：

```text
-fopenmp -mavx -mavx2 -mfma
```

需要说明的是，课程官方 benchmark 会在 Hive 机器上运行，并且官方性能测试限制 OpenMP 使用 4 个线程。

而本文中的本地 benchmark 使用 32 个线程，因此结果主要用于观察优化趋势，不能完全等价于  Hive 上的最终性能评分。

---

# 基准测试

本地基准测试使用：

```text
Matrix A: 5000 x 5000
Matrix B: 5 x 5
Threads : 32
```

测试结果如下：

|版本|平均时间|吞吐量|
|---|--:|--:|
|朴素版本|1.0227 s|1.22 GOPS|
|AVX2 SIMD|0.0315 s|39.61 GOPS|
|AVX2 SIMD + OpenMP + 更高编译优化|0.0210 s|59.53 GOPS|

从结果来看，朴素版本大约为：

```text
1 GOPS
```

而优化版本可以达到：

```text
约 40 GOPS ~ 60 GOPS
```

相较朴素版本，优化后大约获得：

```text
32x ~ 49x
```

的速度提升。

由于该测试结果来自本地 WSL2 环境，且线程数、CPU 状态和系统负载都会影响结果，因此该数据只作为本地参考。
