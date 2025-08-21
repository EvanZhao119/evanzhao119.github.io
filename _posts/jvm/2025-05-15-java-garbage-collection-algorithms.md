---
layout: post
title: "Garbage Collection Algorithms in Java: What Every Developer Should Know"
date: 2025-05-15
categories: jvm
published: true
description: "A guide to GC algorithms in the JVM including Mark-Sweep, Mark-Copy, and Mark-Compact. Learn their trade-offs, performance impact, and modern collector evolution."
---

# Garbage Collection Algorithms in Java: What Every Developer Should Know

Java Virtual Machine (JVM) uses multiple garbage collection (GC) algorithms to manage memory automatically by reclaiming unused objects. 

## 1. Mark-Sweep Algorithm

It is the easiest algorithm, consisting of two phases.
1. **Mark**: Identify and mark live objects.
2. **Sweep**: Clear unmarked (dead) objects from memory.

Its main drawbacks are **Unstable performance** and **Memory fragmentation**. First, GC pause time varies depending on the number of objects. Second, it leaves behind many small memory fragments, making it difficult to allocate large objects.

## 2. Mark-Copy Algorithm
Primarily used in the **Young Generation**.

### Mechanism
Used in HotSpot’s **Serial**, **ParNew**, and **Parallel Scavenge** collectors.

- Young generation is divided into **Eden**, **Survivor From**, and **Survivor To** areas (default ratio: 8:1:1).
- New objects are allocated in Eden.
- During GC, live objects from Eden and Survivor From are copied to Survivor To.
- Eden and Survivor From are cleared.
- The roles of From and To are swapped for the next GC.

### Special Design
- **Escape Hatch (Allocation Guarantee)**: If Survivor To doesn’t have enough space, surviving objects are directly promoted to the Old Generation.

Minor GC is triggered when Eden fills up. Large objects may be directly allocated to the Old Generation. When the Old Generation is full, a **Full GC** is triggered, which is expensive and affects application performance.

### The latest
Modern collectors like **G1 GC**, **ZGC**, and **Shenandoah GC** use region-based or concurrent collection strategies, no longer relying solely on the traditional Mark-Copy model.

## 3. Mark-Compact Algorithm

Introduced to solve the fragmentation issue of the Mark-Sweep algorithm.

### Mechanism
1. **Mark**: Mark all live objects.
2. **Compact**: Move them to one end of memory.
3. Clear the space outside the compacted area.

The Mark-Compact algorithm helps fix memory fragmentation by packing all the live objects together, which also makes it easier to allocate large chunks of memory. But there’s a catch — it needs to pause the application during the process (called a Stop-The-World pause), and it's more complex and takes longer compared to the Mark-Sweep.

### The latest
The **CMS** collector, which was based on the Mark-Sweep algorithm, has now been replaced by **G1 GC** in modern JVMs. 

Unlike **CMS**, **G1 GC** uses a Mark-Compact approach for the Old Generation and performs concurrent compaction, which helps reduce long pause times and avoid memory fragmentation.
