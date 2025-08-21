---
layout: post
title: "Shenandoah GC in HotSpot JVM: Truly Concurrent Non-Generational Collector"
date: 2025-05-23
categories: jvm
published: true
description: "Understand how Shenandoah GC uses Brooks Pointers, read barriers, and concurrent compaction to deliver ultra-low pause times in large-scale JVM applications."
---

# HotSpot JVM Non-Generational Garbage Collectors - Shenandoah GC

Modern garbage collectors like **G1**, **Shenandoah**, and **ZGC** are designed to address the challenges of large heaps, low-latency requirements, and scalability. Unlike traditional generational collectors, these collectors **do not strictly separate the heap into young and old generations**, but rather operate on **region-based memory layouts** for more flexibility and performance.

> ***Note***: While G1 is still technically generational, its region-based and incremental nature blurs the line. Shenandoah and ZGC, however, are **truly non-generational**.

## Overview
- Introduced as an experimental feature in **JDK 12**, production-ready by **JDK 15**.
- A **truly concurrent, low-pause-time** garbage collector.
- **Region-based**, like G1, but avoids traditional remembered sets.

## Key Innovations
- **Connection Matrix**: A global data structure that replaces remembered sets to track inter-region references.
- **Brooks Pointers**: Every object has an **extra level of indirection** (a forwarding pointer) to support concurrent object relocation.
- **Read Barriers**: Lightweight checks on object reads that enable concurrent compaction and relocation.

## Features
- **Pause times in milliseconds**, even with very large heaps (up to 4 TB).
- Nearly **all GC phases run concurrently** with application threads.
- No stop-the-world full compaction phases.

## Compared to G1

| Aspect               | Shenandoah                  | G1 GC                          |
|----------------------|-----------------------------|--------------------------------|
| Compaction           | Concurrent                  | Only during evacuation       |
| Pause Times          | Ultra-low                   | Moderate                       |
| Read Barrier         | Required                    | Not used                     |
| Suitability          | Interactive, responsive apps| General-purpose                |
| Heap Size Support    | Up to 4 TB                  | Up to ~32 GB (practically)     |

## Brooks Pointers 
A **Brooks Pointer** is an **extra field** added to every object in the heap. This field acts as a **forwarding pointer**, which either:

- Points to the object itself (in the normal state), or  
- Points to the object's new location after it has been moved during GC.

This indirection allows the collector to **move objects concurrently** without breaking references. Every time the application accesses an object, it first follows the Brooks Pointer to get the **current valid address**.

## Read Barriers

A **Read Barrier** is a lightweight piece of code that is automatically executed **on every object reference read**.

Its purpose is to:

- **Check** whether the object has been moved or is in the process of being moved by the GC.
- If needed, **update the reference** to point to the new (correct) location.
- Optionally, perform actions like logging, tracking, or correcting object state.

Read Barriers are **crucial** in concurrent and compacting GCs where objects may move at any time.
