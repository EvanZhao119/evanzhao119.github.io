---
layout: post
title: "HotSpot JVM - Old Generation Garbage Collectors"
date: 2025-03-30
categories: jvm
published: true
---

# HotSpot JVM - Old Generation Garbage Collectors

In the HotSpot JVM, the **old generation** is where long-lived objects reside. Garbage collection in this space is generally more expensive and less frequent than in the young generation.

> Many old generation collectors listed below are now deprecated or legacy. While they are useful for foundational learning, **modern applications should prefer collectors like G1, ZGC, or Shenandoah**, which manage both young and old generations efficiently.

## Serial Old Collector
- The old generation version of the **Serial collector**.
- **Single-threaded** garbage collection.
- Uses the **Mark-Compact algorithm** (Mark-Sweep-Compact).
- Causes a **Stop-The-World (STW)** pause for the entire collection process.
- Suitable for single-core or memory-constrained environments.

> Still available but only recommended for testing, educational purposes, or very small applications.

## Parallel Old Collector
- The old generation counterpart of the **Parallel Scavenge** collector.
- A **multi-threaded collector**.
- Also uses the **Mark-Compact algorithm**.
- Causes **Stop-The-World (STW)** pauses during collection.
- Optimized for **throughput**, making it ideal for batch processing or CPU-constrained environments.

> Best used in combination with **Parallel Scavenge** for **throughput-first** applications.

## CMS (Concurrent Mark Sweep) Collector
- A low-pause collector designed to **minimize GC pause times**.
- Uses the **Mark-Sweep algorithm** and includes four phases:
    1. **Initial Mark (STW)**: Marks objects directly reachable from GC Roots. Requires all application threads to pause.
    2. **Concurrent Mark**: Traverses the object graph starting from GC Roots. Runs **concurrently** with application threads.
    3. **Remark (STW)**: Fixes any changes made by the application during the concurrent phase. Also a pause phase.
    4. **Concurrent Sweep**: Reclaims memory of unreachable objects, allowing **concurrent execution** with application threads.
- Advantages:
    - **Low pause times**: Only the initial and remark phases require pausing application threads.
    - **Concurrent marking and sweeping** reduces overall GC latency.

- Disadvantages
    - **CPU-intensive**: Concurrent threads compete with application threads. 
    - **Floating garbage**: Objects allocated during the concurrent mark may not be collected in the current cycle.
    - **Concurrent Mode Failure**: If memory is exhausted before GC completes, a fallback **Serial Old GC** is triggered, causing a full STW pause.
    - **Fragmentation**: Since memory is not compacted, CMS can lead to **memory fragmentation** over time.

> **Deprecated Notice:**  
> **CMS was officially removed in JDK 14**.  
> Modern applications should use **G1 GC**, **ZGC**, or **Shenandoah** as alternatives for low-pause requirements.

Understanding Serial, Parallel, and CMS collectors helps build a strong foundation, but when working on real-world Java systems today, it’s recommended to prioritize **G1, ZGC, or Shenandoah** for better performance and scalability.
