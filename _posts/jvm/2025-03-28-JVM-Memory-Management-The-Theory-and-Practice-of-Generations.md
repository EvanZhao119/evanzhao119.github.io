---
layout: post
title: "JVM Memory Management: The Theory and Practice of Generations"
date: 2025-03-28
categories: jvm
published: true
---

# JVM Memory Management: The Theory and Practice of Generations

## Generational Hypotheses
There are three key hypotheses behind the generational design of the Java Virtual Machine (JVM).

### 1. Weak Generational Hypothesis
> Most objects die young.

This hypothesis suggests that a large number of Java objects have a very short lifespan and become unreachable shortly after allocation.

### 2. Strong Generational Hypothesis
> The longer an object has survived, the more likely it is to continue surviving.

This implies that objects which have survived multiple garbage collection (GC) cycles are likely to remain in memory for a long time.

### 3. Cross-Generational Reference Hypothesis
> References from older generations to younger generations are relatively rare.

This assumption is crucial for optimizing GC performance. Since references from the old generation to the young generation are infrequent, **the garbage collector does not need to scan the entire old generation when collecting the young generation**.

**Implementation in JVM:**  
To support this hypothesis, the HotSpot JVM uses a structure called the **card table**, maintained by a technique known as **write barrier**. This mechanism records cross-generational references efficiently.

## JVM Memory Generation in Practice

### Heap Space Generations

The Java heap is divided into **Young Generation** and **Old Generation**. Based on these, the method area used to be referred to as the **Permanent Generation (PermGen)** in earlier Java versions.

Since **Java 8**, **PermGen has been removed** and replaced with **Metaspace**, which resides in native memory rather than the heap.

### Appel-style Garbage Collection

The **Young Generation** is further divided into:
- **Eden Space**
- **Two Survivor Spaces (From and To)**

In collectors like **Serial**, **ParNew**, and **Parallel Scavenge**, the default space ratio between Eden and a single Survivor space is **8:1**.

Most new objects are allocated in Eden, and during GC, surviving objects are copied between Survivor spaces before being promoted to the Old Generation.

***Java 8+ Note***: Always keep in mind that PermGen is obsolete. Metaspace is now the standard for class metadata storage, and its size is dynamically managed in native memory.
