---
title: "Paper Notes: HAKES — High Availability Key-value Engine for Search"
date: 2026-01-19
author: "Suman Roy"
description: ""
tags: ["Distributed Systems", "VectorSearch", "Architecture"]
---


HAKES is a vector search system designed to scale to billions of vectors while maintaining sub-millisecond tail latencies. Unlike monolithic vector databases, HAKES adopts a **disaggregated architecture** that separates storage, indexing, and search layers to achieve high availability and seamless horizontal scaling.

### TL;DR
HAKES addresses the limitations of traditional vector databases—specifically resource contention (“heat”) and the high cost of graph-based indices. By employing a **two-stage Filter-and-Refine architecture** based on **IVF + PQ**, it offloads persistent data to Cloud Storage and uses a unified cluster management approach to handle planet-scale workloads.

### Three Key Innovations
1.  **Filter-and-Refine Architecture:** Instead of a single-step search, HAKES uses a highly compressed “Filter” stage to prune 99% of the search space before a “Refine” stage performs exact distance calculations on a tiny subset of candidates.
2.  **Disaggregated Storage:** By using Cloud Storage (S3/GCS) as the persistent backing store, HAKES decouples the dataset size from the cluster’s RAM capacity. Workers are stateless and load data on demand.
3.  **Optimized Quantization (OPQ):** HAKES utilizes a rotation-based quantization strategy that minimizes information loss, allowing high-dimensional vectors to be compressed into a few bytes while maintaining high search accuracy.

---

### The Architecture: Disaggregated Responsibilities

HAKES breaks away from monolithic design by splitting responsibilities across specialized components:

* **Cloud Storage (The Backing Store):** Serves as the central repository for all raw vectors and index partitions. This allows the system to scale storage independently of compute.
* **FilterWorkers:** Memory-optimized nodes that store compressed **PQ codes**. They handle the bulk of the query volume by narrowing down millions of vectors to a set of `N` high-probability candidates.
* **RefineWorkers:** Compute-optimized nodes that store **full-precision vectors**. They receive candidates from the FilterWorkers and perform final, accurate distance scoring.
* **Metadata Service:** Coordinates the mapping between index partitions in Cloud Storage and the active Worker nodes.



---

### The Search Path: Filter and Refine

To achieve “Planet-Scale” efficiency, HAKES avoids expensive global searches. It follows a two-step mathematical workflow:

#### Step 1: The Filter Stage (Coarse Search)
The query `x` first hits the **FilterWorkers**. These nodes utilize an Inverted File (IVF) structure:
* **Partition Pruning:** The worker identifies the `nprobe` closest centroids to the query.
* **Approximate Computation:** Inside those partitions, it performs **Asymmetric Distance Computation (ADC)** using Product Quantization (PQ) codes.
* **Efficiency:** This relies on **lookup tables** rather than floating-point math, significantly reducing CPU “heat.”

#### Step 2: The Refine Stage (Exact Scoring)
The filter stage outputs a candidate set `R'`. These are passed to the **RefineWorkers**:
* The worker fetches the original, high-dimensional vectors for `R'` from its local cache or Cloud Storage.
* It calculates the exact distance (e.g., Euclidean or Cosine).
* This step ensures high recall while performing expensive math only on the most relevant candidates.



#### Visualizing the Data Flow
```text
[ Query ]
    |
    v
+------------------+      +-----------------------+
|  FilterWorkers   |      |     Cloud Storage     |
| (PQ Code Lookups)|<---->| (Index/Vector Shards) |
+---------|--------+      +-----------|-----------+
          |                           |
    [ Candidates ]                    |
          |                           |
+---------v--------+                  |
|  RefineWorkers   |<-----------------+
| (Exact Distance) |      (Fetch Raw Vectors)
+---------|--------+
          |
    [ Top-K Result ]
```

### Indexing via Disaggregated Workers

HAKES treats indexing as an asynchronous, parallelizable task rather than a global cluster lock event. This is critical for maintaining high availability during massive data updates.

* **Global Parameter Training:** A coordinator pulls a representative sample from Cloud Storage to learn the **Global IVF Centroids** and **Global PQ Codebooks**.
* **Parallel Quantization:** Once parameters are fixed, stateless **Indexing Workers** quantize the dataset in parallel. Each worker reads a chunk from Cloud Storage, converts vectors to PQ codes, and writes the resulting **Index Segments** back to the persistent store.
* **Global Consistency:** Because the codebook is global, any node in the cluster can interpret any partition, making data migration and rebalancing trivial.



---

### Managing “Heat” and Scalability

In a planet-scale system, specific partitions often become “hot” due to query popularity (e.g., a trending topic in a search engine). HAKES manages this through a coordination layer that treats partitions as first-class citizens:

* **Granular Sharding:** By using a partition-based approach (IVF), “heat” is localized. A cluster manager can monitor the QPS of individual partitions.
* **Replication & Migration:** If a partition becomes a bottleneck, the orchestrator can replicate that specific segment from Cloud Storage to another node without re-indexing or affecting the rest of the cluster. This allows for rapid horizontal scaling of "hot" data while keeping "cold" data cost-efficiently stored.

---

### Appendix: How PQ Distance Works

To understand the speed of the Filter Stage, consider how HAKES calculates distance without ever accessing the original high-dimensional vectors. The system uses **Product Quantization (PQ)** to transform expensive floating-point math into simple memory lookups.

By splitting a vector into **m** subspaces, the squared distance is approximated through the following logic:

**dist(q, v)² ≈ Σ [lookup_tableⱼ[codeⱼ]]** *(where j ranges from 1 to m)*

**Why this enables "Planet-Scale" speed:**

* **No Floating-Point Multiplication:** The CPU avoids the most "heat-intensive" mathematical operations by bypassing raw vector multiplication.
* **Integer Efficiency:** The search process is reduced to **m** integer lookups and **m** additions per vector.
* **Cache Locality:** Because the lookup tables are extremely small, they stay within the processor’s **L1/L2 cache**. This prevents the CPU from waiting on the relatively slow main memory (RAM), allowing a single core to evaluate millions of candidates per second.

In short, HAKES achieves sub-millisecond latency not just through disaggregation, but by reducing the fundamental physics of search into the most efficient operations a modern CPU can perform.

---

*The content presented here are my notes based on the HAKES research paper. It serves as a practical example of how to handle high-volume vector search by separating logical coordination from physical hardware constraints.*

