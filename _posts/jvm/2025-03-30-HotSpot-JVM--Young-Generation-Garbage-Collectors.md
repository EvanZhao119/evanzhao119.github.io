---
layout: post
title: "HotSpot JVM - Young Generation Garbage Collectors"
date: 2025-03-30
categories: jvm
published: true
---

# HotSpot JVM - Young Generation Garbage Collectors

The HotSpot JVM provides several garbage collectors (GC) optimized for different scenarios. This section focuses on **young generation** collectors, which are primarily responsible for handling short-lived objects. 

While these collectors are foundational and important for understanding GC mechanisms, many of them are now considered legacy and are being gradually phased out in favor of more modern collectors like G1, ZGC, and Shenandoah.

## Serial Collector
- **Single-threaded** garbage collector: uses only one thread for minor GC.
- **Mark-Copy algorithm**: divides memory into Eden and Survivor spaces and copies surviving objects.
- **Stop-The-World (STW)** pause: all application threads are paused during collection.
- It is the **default collector in client mode** of HotSpot JVM (primarily legacy use).
- **Simple and low-overhead**: best suited for single-threaded or memory-constrained environments.

> ***Note:*** In modern server-side applications, the Serial collector is mostly used for testing or very small footprint systems. It is no longer the default for most environments since **G1 has become the default collector** in newer JDKs (since **JDK 9**).

## ParNew Collector
- A **multi-threaded** version of the Serial collector.
- Also uses the **Mark-Copy algorithm**.
- Requires **Stop-The-World (STW)** pause.
- Designed to work alongside the **Concurrent Mark Sweep (CMS)** collector for the old generation.

> **Deprecated Feature:**  
> As of **JDK 14**, the CMS collector has been **removed**, and thus **ParNew is obsolete and no longer recommended**.  
> Consider using **G1** or **ZGC** instead for low-pause-time requirements.

## Parallel Scavenge Collector
- A **multi-threaded, throughput-focused** garbage collector.
- Uses the **Mark-Copy algorithm** for young generation collection.
- Also causes **Stop-The-World** pauses.
- **Optimized for high throughput** rather than low latency.
- Throughput is calculated as: `Throughput = Time spent running user application / (Time running user application + Time spent in GC)`

> ***note:*** The **Parallel GC (Parallel Scavenge + Parallel Old)** is still available and maintained.  
> However, it is being gradually replaced by **G1 GC**, which offers a better balance between throughput and pause time.
