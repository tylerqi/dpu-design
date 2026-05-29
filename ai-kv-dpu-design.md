# AI-KV DPU 最终硬件设计分析 (Final Design - 384TB 存储 + 64GB 本地内存版)

> [!IMPORTANT]
> 本文档为 AI-KV DPU 的**最终确定硬件设计**。明确锁定以下规格，不再迭代硬件参数：
> - 2× 400GbE 网络（合计 800Gbps）
> - 24× PCIe 5.0 通道（6× 64TB NVMe SSD，共 384TB）
> - 16MB 片上 SRAM
> - **2 通道 64GB DDR5 内存**（元数据 L2 缓存，超额冷元数据远端转储）
> - NVMe-oF KV 服务（ARM 固件实现 KV→LBA 映射）
> - **DRAM 必须 100% 排除在 IO 路径外**：数据不经过 DRAM（网络 ↔ SRAM ↔ SSD 零拷贝直通，以及 SSD ↔ SRAM ↔ SSD GC 搬运直通）

---

## 一、架构逻辑图

```mermaid
graph TD
    subgraph "AI-KV DPU (Final Design - 12nm)"
        NOC((片上互联 NoC))

        %% 网络接口
        ETH["2× 400GbE 网络接口\n(112G SerDes × 8)"]
        NET_ACC["网络/RDMA 加速引擎\n(RoCE v2, NVMe-oF)"]

        %% 控制面
        CPU["Arm CPU (4-8核, 1.0-1.2GHz)\n负责元数据调度、GC决策与异步重建\n(查表在 DRAM，不参与 IO 数据搬运)"]

        %% 硬件加速引擎
        DMA_ENGINE["零拷贝 DMA 引擎\n(P2P: 网络 ↔ SRAM ↔ SSD)\n(P2P: SSD ↔ SRAM ↔ SSD GC搬运)"]
        SEC_ACC["安全引擎\n(AES-256/SHA)"]

        %% 缓存与内存
        SRAM["片上 SRAM (16MB)\n(DMA/GC 暂存 + LBA Bitmap 缓存\n+ 热点 Bloom Filter + 队列)"]
        DDR_CTRL["2通道 DDR5 控制器"]

        %% 内部连接
        ETH <-->|"直通"| NET_ACC
        NET_ACC <--> NOC
        CPU <--> NOC
        DMA_ENGINE <--> NOC
        SEC_ACC <--> NOC
        DDR_CTRL <--> NOC
        SRAM <--> DMA_ENGINE
        SRAM <--> CPU
    end

    subgraph "外部存储"
        DDR5["2通道 DDR5-5600/6400\n(64GB, 缓存活跃 KV→LBA 映射表)"]
        SSD["6× 64TB NVMe SSD\n(PCIe 5.0 x4, NVMe 2.0\nE3.S/U.2)"]
    end

    subgraph "远端主机/内存"
        HOST["GPU 服务器集群 / 远端内存池\n(vLLM / TensorRT-LLM)\nNVMe-oF Initiator / Metadata Pool"]
    end

    %% 外部连接
    DDR_CTRL <--> DDR5
    DMA_ENGINE <-->|"PCIe 5.0 x24 RC\n(总 96GB/s)"| SSD
    HOST <-->|"RoCE v2 / NVMe-oF\n(800Gbps)"| ETH
```

---

## 二、数据路径深度分析（核心设计决策）

### 2.1 IO 数据路径（零拷贝，DRAM 100% 排除在外的直通路径）

这是本设计最关键的架构决策。IO 数据流完全绕过 DDR5 DRAM，仅通过 SRAM 作为暂存，完成网络到 SSD 的零拷贝传输。DRAM 不得存储任何 KV 数据的 Value 缓存，确保数据通路彻底排除 DRAM 干扰。

#### 写入路径 (KV Store)

```mermaid
sequenceDiagram
    participant Host as 远端 GPU 主机
    participant ETH as 400GbE/RoCE
    participant NET as 网络引擎
    participant SRAM as SRAM (16MB)
    participant CPU as ARM CPU
    participant DRAM as DDR5 (元数据 L2 缓存)
    participant DMA as DMA 引擎
    participant SSD as 64TB NVMe SSD

    Host->>ETH: NVMe-oF KV Store(Key, Value)
    ETH->>NET: RDMA 接收 capsule
    NET->>SRAM: 将 Value 数据 DMA 至 SRAM 暂存区 (DRAM 旁路)
    NET->>CPU: 提交 KV 命令(Key) 至 CPU
    CPU->>DRAM: 查询/更新元数据 (命中本地缓存则更新；未命中则从远端调入)
    CPU->>DMA: 下发 NVMe Write(LBA, len) 至 DMA 引擎
    DMA->>SSD: 从 SRAM 读取 Value，通过 PCIe 写入 SSD
    SSD-->>DMA: Write 完成
    DMA-->>CPU: IO 完成中断
    CPU-->>NET: 生成 NVMe-oF Completion
    NET-->>Host: RDMA 回送完成通知
```

#### 读取路径 (KV Retrieve)

```mermaid
sequenceDiagram
    participant Host as 远端 GPU 主机
    participant ETH as 400GbE/RoCE
    participant NET as 网络引擎
    participant SRAM as SRAM (16MB)
    participant CPU as ARM CPU
    participant DRAM as DDR5 (元数据 L2 缓存)
    participant DMA as DMA 引擎
    participant SSD as 64TB NVMe SSD

    Host->>ETH: NVMe-oF KV Retrieve(Key)
    ETH->>NET: RDMA 接收 capsule
    NET->>CPU: 提交 KV 命令(Key) 至 CPU
    CPU->>DRAM: 查询 KV→LBA 元数据 (若 Miss，则先经 RoCE 读远端内存元数据)
    CPU->>DMA: 下发 NVMe Read(LBA, len) 至 DMA 引擎
    DMA->>SSD: 通过 PCIe 读取 SSD
    SSD-->>DMA: 数据返回
    DMA->>SRAM: 将数据暂存至 SRAM (DRAM 旁路)
    NET->>SRAM: 从 SRAM 读取数据
    NET->>ETH: RDMA 回送 Value 数据至远端主机
    ETH-->>Host: 数据到达
```

### 2.2 垃圾回收 (Garbage Collection) 数据路径 (SSD ↔ SRAM ↔ SSD)

GC 的碎片迁移同样需要绕过 DRAM，数据流仅在 SSD 和 SRAM 之间流动，保持零拷贝直通。

```mermaid
sequenceDiagram
    participant CPU as ARM CPU (GC 线程)
    participant SRAM as SRAM (9MB DMA 缓冲区)
    participant DMA as DMA 引擎
    participant SSD as 64TB NVMe SSD

    CPU->>CPU: 扫描 DRAM 统计表，定位有效槽位少于 2 的 Victim 块
    CPU->>DMA: 下发 Relocate 读指令 (Victim LBA, 长度)
    DMA->>SSD: 从旧块读取有效 Value
    SSD-->>DMA: 数据返回
    DMA->>SRAM: 将数据暂存到 SRAM
    CPU->>DMA: 下发 Relocate 写指令 (Target LBA, 长度)
    DMA->>SRAM: 从 SRAM 读取数据
    DMA->>SSD: 写入新 LBA 块对应槽位
    SSD-->>DMA: 写入完成
    DMA-->>CPU: 中断通知完成
    CPU->>CPU: 更新 DRAM 映射表，物理释放旧 LBA 块
```

### 2.3 控制面路径（元数据冷热分层，运行在 ARM + DRAM）

| 组件 | 角色 | 数据类型 |
|:---|:---|:---|
| **ARM CPU** | 运行固件，管理元数据 L2 缓存、LBA 分配、垃圾回收 (GC) 和 Bloom Filter 定期异步重建 | 不接触 IO 数据 |
| **DDR5 64GB** | 本地元数据 L2 缓存与统计表 | 1.42 亿条 32B 压缩元数据 + 3.68GB Bloom Filter + 384MB GC 统计表 |
| **SRAM 16MB** | 热点 Bloom Filter + LBA Bitmap 缓存窗口 | 包含 4MB 热点 Bloom Filter + 1MB LBA Bitmap 缓存 |

---

## 三、SSD 规格与系统级分析

### 3.1 单盘规格 (基于 64TB 配置)

| 参数 | 规格 |
|:---|:---|
| **接口** | PCIe Gen5 x4, NVMe 2.0 |
| **顺序读** | 14,000 MB/s (14 GB/s) |
| **顺序写** | 9,000-9,500 MB/s (~9 GB/s) |
| **4K 随机读** | 2,800,000 IOPS |
| **4K 随机写** | 750,000-800,000 IOPS |
| **读延迟 (4K)** | < 54 μs |
| **写延迟 (4K)** | < 8 μs |
| **容量** | **64 TB** |

### 3.2 六盘聚合性能（6× 64TB SSD 通过 PCIe 5.0 x24）

| 指标 | 单盘 | 6 盘聚合 | PCIe x24 上限 | 利用率 |
|:---|:---|:---|:---|:---|
| **顺序读** | 14 GB/s | **84 GB/s** | 96 GB/s | 87.5% ✅ |
| **顺序写** | 9 GB/s | **54 GB/s** | 96 GB/s | 56.3% ✅ |
| **4K 随机读 IOPS** | 2.8M | **16.8M** | - | - |
| **4K 随机写 IOPS** | 0.8M | **4.8M** | - | - |
| **总容量** | 64 TB | **384 TB** | - | - |

---

## 四、SRAM 16MB 容量分配与设计

### 4.1 SRAM 用途分配

| 用途 | 分配 | 说明 |
|:---|:---|:---|
| **DMA 暂存缓冲区** | 9 MB | 网络↔SSD 零拷贝数据暂存，同时兼做 GC 迁移缓存 (动态共享) |
| **热点 Bloom Filter** | 4 MB | 缓存最近活跃 ~3.4M Key，快速检测 Key 存在性 |
| **LBA Bitmap 缓存** | 1 MB | 缓存活动分配窗口，管理局部块分配 |
| **NVMe-oF 队列缓存** | 2 MB | SQ/CQ 描述符缓存，降低中断延迟 |

### 4.2 本地 64GB DRAM 缓存能力估算 (384TB 存储)

| 场景 | KV 条目数 | 每条目大小 | 映射表总大小 | 存储位置 |
|:---|:---|:---|:---|:---|
| 热点 Bloom Filter (SRAM) | ~3.4M | 9.6 bits/entry | **4 MB** | 片上 SRAM |
| 压缩元数据缓存 (DRAM) | 1.42 亿 (本地缓存) | 32 B | **45.4 GB** | DDR5 64GB |
| Cuckoo Hash Table (DRAM) | 1.08 亿桶 (本地缓存) | 12 B | **13.0 GB** | DDR5 64GB |
| GC 统计表 (DRAM) | 384M 块 | 1 B/块 | **384 MB** | DDR5 64GB |
| 全量 Bloom Filter (DRAM) | 30.72 亿 (全量过滤) | 9.6 bits/entry | **3.68 GB** | DDR5 64GB |

---

## 五、ARM CPU 元数据调度与性能分析

### 5.1 映射查表延迟

| 命中位置 | 查表延迟 | 数据路径 |
|:---|:---|:---|
| **SRAM 命中** (热点 Bloom) | ~5-10 ns | 快速拦截/确定位置 |
| **本地 DRAM 命中** | ~80-120 ns | 快速读取 LBA 块地址 |
| **远端内存命中** (L2 Miss) | ~3-5 μs | 通过 RoCE 读远端元数据并换入本地 |
| **SSD 数据物理读** (零拷贝) | **~54 μs** | DRAM 不参与，SRAM 直通 |

### 5.2 GC 与删除后台处理负载
- **GC 重定位调度**：GC 搬运使用 P2P DMA 自动完成，CPU 仅负责生成少量 Relocate 描述符，CPU 开销微乎其微。
- **Bloom Filter 定期重建**：CPU 在后台线程中采用双缓冲技术实现布隆过滤器异步重建。通过对 3B 个 Key 进行哈希并填充 3.68GB 位数组，耗时约 **10-15 秒**，后台核心执行，不干扰前台实时查表核心。

---

## 六、DDR5 2 通道带宽分析

### 6.1 DDR5 仅承载控制面流量 (锁定 2通道 DDR5-6400)

DRAM 彻底排除在 IO 数据路径外，不再作为数据读缓存，因此其承载的流量全部为控制面元数据与 LBA 索引查表流量：

| DDR5 流量类型 | 估算带宽需求 | 说明 |
|:---|:---|:---|
| 本地元数据哈希查表 (99.9% 命中) | ~2-4 GB/s | 每次查表读取 64B 元数据 |
| 远端元数据换入与 LRU 淘汰 | < 0.1 GB/s | 每秒极少量的 32B 元数据网络换入换出 |
| LBA Bitmap 刷回与载入 | < 0.05 GB/s | 仅在活动分配窗口越界时进行大容量 Bitmap 同步 |
| **总 DRAM 带宽需求** | **~2-4.5 GB/s** | |
| **2通道 DDR5-6400 可用带宽** | **~40-50 GB/s (顺序)** / **~15-20 GB/s (随机)** | |
| **利用率** | **10-25%** | ✅ 带宽极其充裕，元数据查询无任何排队延迟 |

---

## 七、最终规格与性能汇总

### 最终规格确认表

| 参数 | 最终规格 | 状态 |
|:---|:---|:---:|
| 网络 | 2× 400GbE (800Gbps, RoCE v2) | ✅ 锁定 |
| PCIe (SSD 侧) | 24× PCIe 5.0 (6× x4 RC) | ✅ 锁定 |
| SRAM | 16 MB (9MB DMA + 1MB Bitmap + 4MB Bloom + 2MB Queue) | ✅ 锁定 |
| DDR5 | **2 通道, 64GB, DDR5-5600/6400** | ✅ 锁定 |
| CPU | ARM 精简核, 4-8核, 1.0-1.2GHz | ✅ 锁定 |
| SSD | 6× 64TB NVMe SSD, PCIe 5.0 x4, NVMe 2.0 (总 384TB) | ✅ 锁定 |
| IO 数据路径 | **零拷贝直通** (DRAM 100% 旁路，SRAM 直通) | ✅ 锁定 |
| GC 数据路径 | **P2P DMA 零拷贝重定位** (DRAM 100% 旁路，SRAM 暂存) | ✅ 锁定 |
| 删除应对机制 | 逻辑删除标记 + 后台 CPU 双缓冲异步重建 Bloom Filter | ✅ 锁定 |
| 工艺 | 12nm | ✅ 锁定 |
| 功耗 | 75-110W (估算) | ✅ 锁定 |

### 性能指标汇总

| 指标 | 数值 | 瓶颈来源 |
|:---|:---|:---|
| **最大顺序读吞吐** | ~84 GB/s | 6× SSD 聚合上限 |
| **最大顺序写吞吐** | ~54 GB/s | NAND 物理写限制 |
| **最大 4K 随机读 IOPS** | ~16.8M | 6× SSD 聚合上限 |
| **端到端延迟 (128KB 随机读)** | **~75 μs** | SSD 物理读取与传输占 90%+ |
| **GC 写放大因子 (WAF)** | **< 1.25** | 优先收集碎片率 $\ge 75\%$ 的块 |
| **GC 带宽开销** | < 5% (约 2.7 GB/s 写 / 4.2 GB/s 读) | 自动限流以保障前台 IO 带宽 |
| **本地元数据缓存命中率** | **> 99.9%** (KV Cache 强局部性) | 本地 1.42 亿条 L2 元数据缓存覆盖 |
