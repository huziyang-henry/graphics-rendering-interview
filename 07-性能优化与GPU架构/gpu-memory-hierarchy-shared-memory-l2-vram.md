---
id: Q07.07
title: "GPU的内存层次结构是怎样的？Shared Memory、L1 Cache、L2 Cache、VRAM的访问延迟和带宽分别是多少量级？"
chapter: 7
chapter_name: "性能优化与GPU架构"
difficulty: advanced
knowledge_points:
  - id: "07.05"
    name: "GPU内存层次与带宽优化"
tags: ["memory-hierarchy", "shared-memory", "l1-cache", "l2-cache", "vram"]
---

# Q07.07 GPU的内存层次结构是怎样的？Shared Memory、L1 Cache、L2 Cache、VRAM的访问延迟和带宽分别是多少量级？

**难度：** 🔵 高级
**所属章节：** [性能优化与GPU架构](./index.md)

---

## 🎯 结论

- GPU的内存层次从快到慢依次为：Register（寄存器）→ Shared Memory（共享内存）→ L1 Cache → L2 Cache → VRAM（显存）。访问延迟从约1个时钟周期到数百个时钟周期不等，带宽从数十TB/s到数百GB/s。理解这一层次结构对于编写高效的Compute Shader和优化渲染性能至关重要。

## 📐 原理解析

### GPU内存层次详解

- Register（寄存器）：每个线程私有，容量极小（每个线程约255个32位寄存器），延迟约1-4个时钟周期，带宽极高（数TB/s）。寄存器是GPU中速度最快的存储，但数量有限，超出限制会导致Register Spilling（溢出到Local Memory，即L1 Cache）
- Shared Memory：每个SM/Block共享，容量约32-128KB（视GPU架构而定），延迟约5-30个时钟周期，带宽约1-10TB/s。由程序员显式管理（__shared__ in CUDA），适合Block内线程的数据共享和复用
- L1 Cache：每个SM私有，容量约32-128KB（现代架构中L1与Shared Memory共享物理存储），延迟约30-100个时钟周期，带宽约1-5TB/s。用于缓存Global Memory的访问和Register Spilling
- L2 Cache：全芯片所有SM共享，容量约2-8MB，延迟约200-400个时钟周期，带宽约1-3TB/s。用于缓存所有SM对Global Memory的访问，提供跨Block的数据复用
- VRAM（显存）：容量数GB到数十GB（GDDR6/HBM3），延迟约400-800个时钟周期，带宽约数百GB/s到数TB/s（HBM3可达3TB/s）。是容量最大但速度最慢的存储层级

### 缓存层次的设计目标

GPU内存层次的核心目标是利用数据局部性（Locality）缩小高速计算单元与大容量慢速显存之间的速度差距。时间局部性（Temporal Locality）指同一数据在短时间内被多次访问——L1/L2 Cache通过缓存最近访问的数据来利用这一特性。空间局部性（Spatial Locality）指相邻地址的数据会被连续访问——Cache Line（通常32-128字节）一次加载多个相邻数据来利用这一特性。


## 🛠 工程实践

### 利用Shared Memory加速邻域计算

在Compute Shader中，如果每个线程需要访问相邻线程处理的数据（如卷积滤波、矩阵乘法、图像处理），可以将邻域数据加载到Shared Memory中。例如在图像模糊中，每个线程负责一个像素但需要读取周围 $N \times N$ 像素的数据。通过将Tile（数据块）加载到Shared Memory，每个数据只需从Global Memory读取一次，后续访问从Shared Memory读取，带宽节省可达 $N$ 倍。

### 纹理缓存（Texture Cache）对2D空间局部性的优化

GPU的Texture Cache（纹理缓存）是针对2D空间局部性优化的只读缓存。当Fragment Shader采样纹理时，相邻像素通常采样纹理的相邻区域，Texture Cache会预取相邻的Texel。对于2D数据访问模式（如屏幕空间后处理），纹理采样比直接读取Buffer更高效，因为Texture Cache的命中率更高。

### 矩阵乘法的Tiling优化

经典GEMM优化中，将矩阵分块（Tile）加载到Shared Memory，每个Block处理一个Tile对。通过Shared Memory的数据复用，将Global Memory的访问量从 $O(N^3)$ 降低到 $O(N^3/T)$ ，其中 $T$ 是Tile大小。这是GPU上矩阵乘法性能优化的基础技术。


## ⚠️ 踩坑经验

### Shared Memory的Bank Conflict

Shared Memory被分为多个Bank（通常32个），每个Bank每个时钟周期只能服务一次访问。如果同一Warp内的多个线程同时访问同一Bank的不同地址（Bank Conflict），访问会被串行化，延迟增加。最严重的情况是所有线程访问同一Bank（ $N$ -way Bank Conflict，延迟增加 $N$ 倍）。解决方案包括填充（Padding）——在数据布局中插入空元素使访问分散到不同Bank。

### L2 Cache的容量限制

L2 Cache通常只有几MB，对于大数据集（如大型纹理、G-Buffer）远远不够。当工作集超过L2容量时，Cache Miss率急剧上升，大量访问直接落到VRAM，带宽压力增大。在延迟渲染中，G-Buffer的多个Render Target（Albedo + Normal + Depth + ...）可能占用数十MB，远超L2容量，导致严重的Cache Thrashing。

### Register Spilling的性能影响

当Shader使用的寄存器数量超过硬件限制时（如CUDA中每个线程最多255个寄存器），编译器会将溢出的寄存器存储到Local Memory（实际是L1 Cache/VRAM）。Register Spilling会导致访问延迟从1-4周期增加到数百周期，性能可能下降数倍。减少Register使用的方法包括：减少局部变量数量、使用更少的精度（float vs double）、手动优化Shader代码结构。


## 🔮 延伸思考

### 不同GPU架构的缓存层次差异

NVIDIA从Ampere架构开始将L1 Cache和Shared Memory统一为同一物理存储，程序员可以配置两者的比例。AMD的RDNA3架构也有类似的LDS（Local Data Share）与L1共享设计。Apple GPU（M1/M2）采用统一内存架构（Unified Memory），CPU和GPU共享同一物理内存，缓存层次与传统离散GPU有显著不同。理解目标架构的缓存特征对性能优化至关重要。

### HBM（高带宽内存）对带宽瓶颈的缓解

HBM（High Bandwidth Memory）通过将多个DRAM芯片堆叠并与GPU封装在一起，配合极宽的总线（如HBM3的1024-bit总线），提供远超GDDR6的带宽（HBM3可达3TB/s vs GDDR6X的1TB/s）。NVIDIA的数据中心GPU（A100/H100）和AMD的Instinct系列都采用HBM。HBM显著缓解了带宽瓶颈，但成本极高，目前主要用于高端计算卡。

---

[← 返回 性能优化与GPU架构 题目列表](./index.md) | [返回总纲](../00-总纲.md)
