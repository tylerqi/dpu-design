# DAOS Scalability Architecture Design

This repository contains the architectural design for supporting a large-scale **DAOS (Distributed Asynchronous Object Storage)** cluster with **1,000+ nodes** (supporting tens of thousands of client compute nodes).

## Key Design & Reference Documents

- **[Detailed Architecture Design Document (EN)](file:///mnt/dev/daos-cluster/Architecture_Design.md)**: A deep-dive architecture design that details scaling bottlenecks, mitigations, infrastructure specifications, and comparative analysis.
- **[Detailed Architecture Design Document (CN/中文)](file:///mnt/dev/daos-cluster/Architecture_Design_CN.md)**: 深入的架构设计文档，涵盖扩展瓶颈、缓解措施、基础设施规范以及对比分析。
- **[DAOS Scalability for Exascale](https://daos.io/blog/daos-scalability-architecture-for-exascale/)**
- **[Optimized Container Scout Implementation](https://daos.io/blog/optimized-cont-scout-implementation-for-daos/)**

## Summary of Design Considerations

To scale a DAOS cluster efficiently, the architecture incorporates the following core design paradigms:

1. **Adaptive Scaling Modes (Ceph vs. Lustre)**:
   - *Small-Scale Mode (< 128 nodes - "Ceph" Way)*: Deploys a symmetric, hyper-converged architecture where all nodes run a single monolithic pool and namespace container.
   - *Large-Scale Mode (128 to 1,000+ nodes - "Lustre" Way)*: Partitions the cluster into smaller, independent sub-cluster pools. Filesystem directories (subtrees) are mapped to different pools/containers using the **DAOS Unified Namespace (UNS)** to isolate control traffic.
2. **Mitigations for Scaling Bottlenecks**:
   - *Raft Consensus & Transactions*: Raft consensus and lockless epoch-based MVCC transactions are isolated within each sub-cluster container, preventing global serialization bottlenecks.
   - *SWIM Gossip Protocol*: Tuned gossip parameters and localized sub-cluster domains reduce failure detection network traffic from $O(N^2)$ to $O((N/M)^2)$.
   - *Data Resilience & Fault Domains*: Configures a hierarchical `Rack -> Node -> Engine` failure domain hierarchy. Nodes are vertically sliced across racks to enable wide-stripe Erasure Coding (e.g., `OC_EC_16P2`) at the rack-tolerance level.
   - *RDMA / IB Scaling*: Transitioning to UCX transport using Dynamically Connected Transport (DCT / `DC_X`) prevents Queue Pair (QP) cache thrashing on NIC HCAs.
   - *Container Scout*: Distributed metadata auditing and asynchronous epoch-based garbage collection ensure optimal space reclaim without affecting the active I/O path.
