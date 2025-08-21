---
layout: post
title: "ZGC Garbage Collector in HotSpot JVM: Low-Latency Non-Generational GC"
date: 2025-05-25
categories: jvm
published: true
description: "Learn how ZGC achieves sub-10ms pause times with colored pointers and load barriers. A modern HotSpot JVM GC designed for massive heaps and scalability."
---

# HotSpot JVM Non-Generational Garbage Collectors - ZGC (Z Garbage Collector)

Modern garbage collectors like **G1**, **Shenandoah**, and **ZGC** are designed to address the challenges of large heaps, low-latency requirements, and scalability. Unlike traditional generational collectors, these collectors **do not strictly separate the heap into young and old generations**, but rather operate on **region-based memory layouts** for more flexibility and performance.

> ***Note***: While G1 is still technically generational, its region-based and incremental nature blurs the line. Shenandoah and ZGC, however, are **truly non-generational**.

## Overview
- Production-ready since **JDK 15**.
- Designed for **scalable, sub-millisecond pause times** and **massive heaps**.
- Uses **colored pointers** and **load barriers** to enable object relocation during application execution.

## Key Innovations
- **Colored Pointers**: Metadata (like GC phase bits) is embedded directly into 64-bit pointers.
- **Load Barriers**: Intercept object loads and apply necessary checks/corrections for concurrent relocation.
- **Concurrent Everything**: Marking, relocation, remapping—all run **concurrently**.

## Features
- **Pause times < 10ms** even on 1TB+ heaps.
- **No STW full GCs**, ever.
- **Region-based** with variable-size regions.
- Supports **class unloading, weak references, and finalization** all concurrently.

## Compared to Shenandoah

| Aspect             | ZGC                            | Shenandoah                     |
|--------------------|----------------------------------|--------------------------------|
| Core Technique     | Colored Pointers + Load Barriers | Brooks Pointers + Read Barriers |
| Pause Time         | < 10ms                        | Low (but typically longer)     |
| Max Heap Size      | 16 TB+                          | 4 TB                           |
| Barrier Overhead   | Lower                           | Slightly higher                |
| Ideal Use Case     | Ultra-large heap + low-latency  | Responsive UI and backends     |

## Colored Pointer

A **Colored Pointer** is a technique used in some garbage collectors—most notably **ZGC**—to **embed metadata directly into the object reference (pointer)** itself.

This metadata encodes information about the **GC state of the object**, such as,
- Whether the object is being moved
- Whether the reference needs to be remapped
- What GC phase the object is in

Modern 64-bit systems do **not use all 64 bits** of memory addresses. For example, on most systems,
- Only the **lower 48 bits** are used for actual addressing.
- The remaining **upper bits** (e.g., bits 49–63) are unused and available.

ZGC leverages these **spare bits** to store GC-related metadata—these are the "colors" in the pointer.
