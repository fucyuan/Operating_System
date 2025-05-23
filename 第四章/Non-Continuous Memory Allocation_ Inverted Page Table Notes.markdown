# 非连续内存分配：反向页表

## 1. 背景：页表与逻辑地址空间的挑战

在分页机制中，\*\*页表（Page Table）\*\*是操作系统将逻辑地址映射到物理地址的核心数据结构。传统页表（正向页表）按逻辑页号索引，存储每个逻辑页面对应的物理帧号。然而，传统页表存在以下问题：

- **空间开销**：页表大小与逻辑地址空间成正比。例如，32位系统、4KB页面，逻辑地址空间4GB需要约4MB页表（2^20页面 × 4字节/项）。
- **稀疏地址空间**：进程通常只使用部分逻辑地址空间，但传统页表需为整个地址空间分配空间，造成浪费。
- **多进程开销**：多个进程的页表累积占用大量内存。

为了解决这些问题，操作系统引入了**反向页表（Inverted Page Table, IPT）**，一种以**物理内存**为索引的页表设计，显著减少空间占用，特别适合逻辑地址空间远大于物理内存的系统。

本笔记详细讲解反向页表的原理、实现方式及与正向页表的对比，通过具体例子帮助初学者理解。

## 2. 反向页表的概念

反向页表以**物理帧号**为索引，记录每个物理帧被哪个进程的哪个逻辑页面占用。与正向页表（每个进程一个，按逻辑页号索引）不同，反向页表是**系统全局的**，只有一个表，表大小与物理内存大小成正比。

### 2.1 反向页表的结构

反向页表的每条记录包含：

- **进程ID（PID）**：标识占用该帧的进程。
- **逻辑页号（Page Number）**：该进程的逻辑页面编号。
- **有效位**：表示帧是否被占用。
- **权限位**：如读、写、执行权限。
- **其他信息**（可选）：如脏位（页面是否被修改）、引用位（页面是否被访问）。

**例子**： 假设物理内存32KB，页面大小4KB（8个物理帧）。反向页表如下：

| 帧号 | 进程ID | 逻辑页号 | 有效位 | 权限 |
| --- | --- | --- | --- | --- |
| 0 | 1 | 2 | 1 | 读写 |
| 1 | 2 | 0 | 1 | 只读 |
| 2 | \- | \- | 0 | \- |
| 3 | 1 | 0 | 1 | 读写 |
| ... | ... | ... | ... | ... |

- 帧0被进程1的逻辑页面2占用。
- 帧2未被分配（空闲）。

### 2.2 反向页表的特点

- **全局性**：整个系统只有一个反向页表，表大小取决于物理内存（帧数），而非逻辑地址空间。
- **空间效率**：物理内存通常远小于逻辑地址空间，表项数少。例如，32位系统、4KB页面、512MB物理内存只需2^17（131,072）项，而正向页表需2^20项。
- **查询复杂性**：需要搜索进程ID和逻辑页号匹配的表项，查询效率低于正向页表。

## 3. 逻辑地址到物理地址的转换

反向页表的地址转换过程如下：

1. 逻辑地址：`(进程ID, 页号, 偏移量)`。
2. 搜索反向页表，查找匹配`(进程ID, 页号)`的表项，获取物理帧号。
3. 检查有效位和权限：
   - 若匹配且有效，计算物理地址。
   - 若无匹配或无效，触发**缺页中断**，操作系统加载页面。
4. 物理地址 = 物理帧号 × 页面大小 + 偏移量。

**例子**：

- 逻辑地址：`(进程ID=1, 页号=2, 偏移量=904)`。
- 页面大小：4KB（4096字节）。
- 反向页表：帧0记录`(进程ID=1, 页号=2, 有效=1)`。
- 物理帧号 = 0。
- 物理地址 = `0 × 4096 + 904 = 904`。

**挑战**：反向页表需搜索整个表，效率较低，需优化。

## 4. 反向页表的实现方式

由于反向页表查询复杂，操作系统和硬件通过以下方式优化：

- **寄存器设计**：在小型系统中，直接用寄存器存储反向页表（不常见，容量有限）。
- **关联存储器（Content-Addressable Memory, CAM）**：
  - 硬件支持并行搜索`(进程ID, 页号)`，快速匹配帧号。
  - 优点：查询速度快。
  - 缺点：硬件成本高，适合高端CPU。
- **基于哈希的计算**：
  - 使用哈希函数将`(进程ID, 页号)`映射到反向页表的一个或几个表项。
  - 哈希表存储：表项包含`(进程ID, 页号, 帧号)`，冲突通过链表或开放寻址解决。
  - 优点：查询效率高，空间占用低。
  - 缺点：可能发生哈希冲突，需额外处理。

**哈希实现例子**：

- 哈希函数：`h(PID, Page) = (PID + Page) % 表大小`。
- 逻辑地址：`(PID=1, 页号=2)`，表大小8。
- 哈希值：`(1 + 2) % 8 = 3`。
- 检查表项3：
  - 若匹配`(PID=1, 页号=2)`，返回帧号。
  - 若冲突，遍历链表查找匹配项。

## 5. 反向页表与TLB的结合

\*\*TLB（Translation Lookaside Buffer）\*\*是CPU内部的高速缓存，存储常用页面映射，加速反向页表查询：

- TLB缓存`(进程ID, 页号) → 帧号`的映射。
- **命中**：直接从TLB获取帧号，跳过反向页表搜索。
- **缺失**：搜索反向页表（或哈希表），更新TLB。

**例子**：

- 逻辑地址：`(PID=1, 页号=2, 偏移量=904)`。
- TLB内容：

| 进程ID | 页号 | 帧号 | 有效位 |
| --- | --- | --- | --- |
| 1 | 2 | 0 | 1 |

- TLB命中：帧号0，物理地址 = `0 × 4096 + 904 = 904`。
- TLB缺失：搜索反向页表，更新TLB。

## 6. 正向页表与反向页表的对比

| 特性 | 正向页表 | 反向页表 |
| --- | --- | --- |
| **索引方式** | 按逻辑页号索引 | 按物理帧号索引 |
| **表数量** | 每个进程一个 | 系统全局一个 |
| **空间开销** | 与逻辑地址空间成正比（大） | 与物理内存成正比（小） |
| **查询效率** | 直接索引，O(1) | 需搜索或哈希，O(n)或O(1) |
| **适用场景** | 逻辑地址空间稀疏，进程多 | 物理内存小，逻辑地址空间大 |
| **硬件支持** | MMU+TLB | 需关联存储器或哈希硬件支持 |

## 7. 硬件与软件的配合

反向页表依赖硬件和软件协同工作：

- **硬件**：
  - **MMU**：执行地址转换，检查权限。
  - **TLB**：缓存常用映射，加速查询。
  - **关联存储器**：支持并行搜索（高端CPU）。
- **软件**：
  - 操作系统维护反向页表，处理缺页中断。
  - 实现哈希算法或冲突解决策略。
  - 页面替换算法（如LRU）决定换出页面。

**例子**（高端CPU）：

- CPU使用关联存储器快速匹配`(PID, 页号)`。
- 操作系统更新反向页表，处理TLB缺失和页面换入换出。

## 8. 具体例子：反向页表应用

假设系统物理内存32KB（8帧，4KB/帧），两个进程（PID=1, PID=2），逻辑地址空间16KB（4页面/进程）。反向页表如下：

| 帧号 | 进程ID | 逻辑页号 | 有效位 | 权限 |
| --- | --- | --- | --- | --- |
| 0 | 1 | 2 | 1 | 读写 |
| 1 | 2 | 0 | 1 | 只读 |
| 2 | \- | \- | 0 | \- |
| 3 | 1 | 0 | 1 | 读写 |
| ... | ... | ... | ... | ... |

**场景1**：访问逻辑地址`(PID=1, 页号=2, 偏移量=904)`：

- TLB命中：`(PID=1, 页号=2) → 帧0`。
- 物理地址 = `0 × 4096 + 904 = 904`。
- TLB缺失：搜索反向页表，找到帧0，更新TLB。

**场景2**：访问逻辑地址`(PID=2, 页号=1, 偏移量=100)`：

- TLB缺失，反向页表无匹配，触发缺页中断。
- 操作系统分配帧2，更新反向页表：
  - 帧2：`(PID=2, 页号=1, 有效=1)`。
- 物理地址 = `2 × 4096 + 100 = 8292`。

**场景3**：哈希实现：

- 哈希函数：`h(PID, Page) = (PID + Page) % 8`。
- 地址`(PID=1, 页号=2)`：哈希值 = `(1 + 2) % 8 = 3`。
- 检查表项3，未匹配，遍历链表找到`(PID=1, 页号=2)`，返回帧0。

## 9. 反向页表与分页、分段的关系

- **分页机制**：
  - 反向页表是分页的优化，适合物理内存受限的系统。
  - 提供无外部碎片、支持虚拟内存的优点。
- **分段机制**：
  - 基于程序逻辑结构，段大小可变，可能产生外部碎片。
  - 反向页表不直接支持分段，但可结合分段分页混合机制。

## 10. 总结

反向页表是一种以物理帧为索引的页表设计，显著减少空间开销，适合逻辑地址空间远大于物理内存的系统。核心特点包括：

- **全局表**：系统只有一个反向页表，表项数与物理帧数成正比。
- **查询优化**：通过哈希、关联存储器或TLB加速地址转换。
- **硬件软件协同**：高端CPU的硬件支持（如关联存储器）与操作系统配合提升效率。
- **空间效率**：相比正向页表，节省内存，适合稀疏地址空间。

对于初学者，理解反向页表的索引方式（物理帧而非逻辑页面）及通过哈希或TLB优化的查询过程是关键。例子（如地址转换和哈希搜索）有助于直观掌握其工作原理。后续课程将探讨虚拟内存和页面替换策略的实现。