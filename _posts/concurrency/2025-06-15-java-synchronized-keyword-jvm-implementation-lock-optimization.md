---
layout: post
title: "Java synchronized Keyword Explained: Mechanism, JVM Implementation, and Modern Enhancements" 
date: 2025-06-15
categories: concurrency
published: true
description: "Deep dive into the synchronized keyword in Java, covering internal JVM implementation, object header Mark Word, ObjectMonitor, lock optimizations, and best practices in modern Java concurrency."
keywords: ["Java synchronized", "synchronized keyword", "Java concurrency", "ObjectMonitor", "Mark Word", "Java lock optimization", "JVM synchronized implementation"]
---

# Understanding `synchronized` in Java: Mechanism and Modern Implementation

## 1. Overview of `synchronized`
The `synchronized` keyword in Java is a built-in mechanism to ensure synchronization across threads when accessing shared resources.
- A thread must **acquire the lock (Monitor)** before entering a synchronized block or method.
- The lock is **automatically released** when the thread exits the block—whether normally or due to an exception.
- `synchronized` is a **reentrant lock**, meaning a thread can re-enter a block it already holds the lock for.
- If a lock is held by one thread, other threads attempting to acquire it will be **blocked** until the lock is released.

## 2. Internal Implementation of `synchronized`  
Java Virtual Machine (JVM) implements synchronization using **monitors (Monitor objects)**. 

### 2.1 Method-Level Synchronization
- Declaring a method with `synchronized` applies **method-level synchronization**.
- The synchronization is **implicit**—handled by the JVM without explicit bytecode instructions.
- In the `.class` file, synchronized methods are marked with the `ACC_SYNCHRONIZED` flag.
- Before executing such a method, the JVM checks for this flag and ensures the executing thread has obtained the monitor lock:
    - For instance methods, the lock is on the **current object instance (`this`)**.
    - For static methods, the lock is on the **`Class` object**.

> From JDK 9+, performance improvements have been introduced in handling monitor acquisition for synchronized methods, especially after a biased or lightweight locking attempt fails.

### 2.2 Block-Level Synchronization
- Block-level synchronization is achieved via `synchronized(obj)` which offers **finer control**.
- The JVM uses two specific bytecode instructions:
    - `monitorenter`: Acquires the lock for `obj`.
    - `monitorexit`: Releases the lock.
- These instructions wrap the critical section and are always used in pairs to ensure proper lock release even during exceptions.

## 3. What Really Happens When You Use `synchronized`

### 3.1 Mark Word in Object Header
- Every Java object in memory contains an **Object Header** that includes:
    - **Mark Word**: Stores lock state, identity hash code, thread ID, etc.
    - **Type Pointer**: Points to class metadata.
- The Mark Word is crucial in managing lock status and transitions.

> **Biased Locking is disabled by default** starting in JDK 15 and has been **completely removed in JDK 21**. The JVM now favors lightweight or heavyweight locks, improving predictability in multi-threaded environments.

### 3.2 `ObjectMonitor` in HotSpot JVM
- In HotSpot JVM (used by most Java distributions), `ObjectMonitor` is a C++ structure managing object locking.
- Every object that requires synchronization may be associated with an `ObjectMonitor`.
- Key fields of `ObjectMonitor` include:
    - `Owner`: Thread currently holding the lock.
    - `EntryList`: Queue of threads waiting to acquire the lock.
    - `WaitSet`: Threads that have called `wait()` on the object.
    - Other flags and status bits for lock management.

> The internal structure and scheduling logic of `ObjectMonitor` have been optimized, including better support for lock elimination, fast-path locking, and improved interaction with `LockSupport`'s `park/unpark`.

## 4. Lock Optimization Techniques
Java supports multiple lock states to optimize synchronization performance.

| Lock Type     | Description                                                                 |
|---------------|-----------------------------------------------------------------------------|
| **No Lock**    | Default state when the object is not locked by any thread.                  |
| **Lightweight Lock** | Utilizes CAS (Compare-And-Swap) to acquire the lock in user space; efficient in low-contention scenarios. |
| **Biased Lock** | *Deprecated*: Previously used for non-contending threads, but removed in JDK 21. |
| **Heavyweight Lock** | Used under high contention; threads are blocked in OS kernel, involving context switches. |

## 5. Best Practices

- `synchronized` remains a **safe and reliable** concurrency mechanism in modern Java, especially for simple synchronization needs.
- For high-concurrency or performance-sensitive applications, consider using tools from `java.util.concurrent`, such as:
    - `ReentrantLock`
    - `StampedLock`
    - `ReadWriteLock`
- Always minimize the scope of synchronized code to reduce lock contention.
- Prefer **composition over inheritance** when designing synchronized utilities, and avoid nested synchronized blocks whenever possible.
