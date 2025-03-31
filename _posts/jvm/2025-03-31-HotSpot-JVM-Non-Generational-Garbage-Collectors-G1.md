---
layout: post
title: "HotSpot JVM Non-Generational Garbage Collectors - G1 GC (Garbage First Collector)"
date: 2025-03-31
categories: jvm
published: true
---

# HotSpot JVM Non-Generational Garbage Collectors - G1 GC (Garbage First Collector)

Modern garbage collectors like **G1**, **Shenandoah**, and **ZGC** are designed to address the challenges of large heaps, low-latency requirements, and scalability. Unlike traditional generational collectors, these collectors **do not strictly separate the heap into young and old generations**, but rather operate on **region-based memory layouts** for more flexibility and performance.

> ***Note***: While G1 is still technically generational, its region-based and incremental nature blurs the line. Shenandoah and ZGC, however, are **truly non-generational**.

## Key Concepts

- **Region-based memory layout**: The Java heap is divided into many equally-sized regions.
- Each **Region** is the **smallest unit of collection**. GC reclaims memory in **region multiples**, not full-heap sweeps.
- **Prioritizes regions** with the **highest garbage-to-live ratio** to maximize collection efficiency.
- **Remembered sets (RSet)** are maintained per region to track cross-region references.
- **Humongous objects** (larger than 50% of a region) are stored across multiple contiguous regions, collectively referred to as a Humongous area.

> G1 aims to balance throughput and low pause times, making it the default collector since **JDK 9**.

## GC Phases in G1

1. **Initial Mark**  
    - Marks objects directly reachable from GC Roots.
    - **Requires Stop-The-World (STW) pause**.
2. **Concurrent Mark**
    - Traverses the heap graph to identify live objects.
    - Runs **concurrently** with application threads.
3. **Final Mark (Remark)**
    - Fixes changes made during concurrent marking.
    - **STW pause required**.
4. **Cleanup & Evacuation**
    - Calculates **region-wise live data** and selects high-garbage regions for evacuation. 
    - **Moves live objects** to empty regions and reclaims old ones.
    - Requires **STW pause**, but runs with **multiple GC threads in parallel**.

## Comparison with CMS

| Aspect                     | G1 GC                        | CMS GC                      |
|----------------------------|------------------------------|-----------------------------|
| Memory Layout              | Region-based                 | Contiguous old generation   |
| Fragmentation              | No fragmentation             | Causes fragmentation        |
| Compaction                 | Compacts live data           | No compaction             |
| Pause Time                 | Lower than CMS               | Short but unpredictable     |
| Memory Overhead            | Higher (due to per-region RSet) | Lower                  |
| Concurrency Complexity     | Higher                       | Moderate                    |
| Status                    | Active (Default in JDK 9+) |  Removed (JDK 14+)         |

> G1 is more suitable for **large heaps** and modern hardware, while **CMS** was better for **smaller heaps** but is now deprecated.
