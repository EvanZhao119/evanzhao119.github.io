---
layout: post
title: "Smart Locking in Java: From Spin to Virtual Threads"
date: 2025-04-08
categories: concurrency
published: true
---

# Smart Locking in Java: From Spin to Virtual Threads
The Java Virtual Machine (JVM) uses several smart techniques to make multithreaded programs run faster and more efficiently. These include spin locks, adaptive spinning, lock elimination, lock coarsening, biased locking, and lightweight locking.

## 1. Spin Locks & Adaptive Spinning
When a thread fails to acquire a lock, instead of immediately suspending, the JVM allows the thread to perform a busy-wait loop (i.e., spinning) for a short period. This approach avoids the high overhead of context switching when the lock is expected to be released quickly.

### Adaptive Spinning
**Adaptive spinning** dynamically adjusts the spin duration based on:
- The outcome of the previous spin attempt.
- The state of the lock owner.

If a thread recently acquired the lock quickly via spinning, the JVM may increase the spin duration for subsequent attempts.

**Best For**: Short critical sections, low contention.  
**Not Suitable For**: Long lock hold times — CPU cycles will be wasted.

> Since **JDK 9**, the JVM has gotten smarter at handling spin locks by taking into account the number of CPU cores and how threads are assigned to them. Adaptive spinning is **turned on by default**.

## 2. Lock Elimination
**Lock elimination** is based on **escape analysis**, where the JVM analyzes whether an object is accessible only within the current thread. If the object does not escape the thread’s scope, synchronization is deemed unnecessary and is **removed automatically**.

### Example
- The methods of `StringBuffer` are synchronized by default.
- However, the `sb` instance is a local variable within the method and is only accessed by the current thread, meaning there is no thread sharing.
- The JVM performs **escape analysis** and determines that `sb` does not escape the current thread. As a result, it applies **lock elimination**, removing unnecessary synchronization to improve performance.
```java
public void appendStrings(String a, String b) {
    StringBuffer sb = new StringBuffer(); // StringBuffer is synchronized
    sb.append(a);
    sb.append(b);
}
```

> Since **JDK 17**, escape analysis and scalar replacement are highly optimized, making lock elimination more effective, especially in JIT-compiled code.

## 3. Lock Coarsening
Instead of repeatedly acquiring and releasing a lock in a tight loop, the JVM may **extend the lock scope** to cover the entire sequence of operations — a technique known as **lock coarsening**.

### Before Optimization:

```java
for (int i = 0; i < 100; i++) {
    synchronized (buffer) {
        buffer.append(i);
    }
}
```

### After Coarsening:

```java
synchronized (buffer) {
    for (int i = 0; i < 100; i++) {
        buffer.append(i);
    }
}
```

> Lock coarsening is handled automatically by the **JIT compiler**, which dynamically determines the optimal locking strategy at runtime.

## 4. Biased Locking (Removed in JDK17)
**Biased locking** is a performance optimization for situations where a lock is almost always used by the same thread. In this case, the JVM "biases" the lock toward that thread by recording its ID in the object's **Mark Word**. As long as the same thread keeps using the lock, **no actual synchronization is needed** — which makes it very fast.

### What happens when another thread tries to use the lock?
- If the original thread is still holding the lock, the JVM upgrades it to a lightweight lock.
- If the lock is free, the JVM simply removes the bias and resets the lock.

### Limitations
- If the object has already computed a hash code, it can't use biased locking.
- If the thread calls `wait()` or is `interrupted`, the bias will be removed.
- Under high contention (many threads competing for the lock), biased locking doesn't help.

> ***Note:***
> - **Disabled by default in JDK 15** due to limited benefit.
> - **Completely removed in JDK 17**.
> - Modern JVMs now prefer more efficient mechanisms like lightweight locks and virtual threads.

## 5. Lightweight Locks
**Lightweight locks** are designed to reduce the overhead of heavyweight locks in situations with **mild contention**.

### Lock Acquisition
1. A thread attempts to **CAS** the Mark Word of the object to point to its own lock record.
2. If successful → lock acquired.
3. If unsuccessful, but Mark Word still points to the same thread → reentrant lock.
4. If CAS fails and another thread holds the lock → spin wait → may escalate to heavyweight lock.

### Unlocking
- CAS is used to restore the original Mark Word.
- If CAS fails, another thread may be waiting → release and wake it up.

> With enhancements in **ZGC, G1**, and **JDK 17+**, lightweight locks are still a reliable and efficient option in moderate concurrency scenarios.

## 6. Heavyweight Locks
These are **traditional OS-level mutexes** used when lock contention is high. JVM resorts to this locking mechanism when:
- Lightweight locks fail due to contention.
- Threads must be suspended and awakened.

**High overhead** due to context switching.

**Stable under high contention**, but not recommended for high-performance applications unless necessary.

## 7. Virtual Threads & Structured Concurrency
With **JDK 21**, **Virtual Threads (Project Loom)** and **Structured Concurrency** represent a major shift in concurrent programming.

### Virtual Threads
- Lightweight user-mode threads managed by the JVM.
- **Highly scalable**, thousands of threads per JVM.
- Locking becomes cheaper due to improved scheduler design.

### Structured Concurrency
- Encourages better lifecycle management of tasks.
- Enables writing **predictable and safe concurrent code**.
- Works seamlessly with traditional synchronization primitives.

> **Recommendation**: Use virtual threads combined with high-level concurrency utilities (`java.util.concurrent`) as possible.
