---
layout: post
title: "Java StampedLock Explained: High-Performance Read-Write Lock with Optimistic Reads"
date: 2025-06-26
categories: concurrency
published: true
description: "Understand Java StampedLock: a high-performance read-write lock supporting optimistic reads, write locks, and conversion with examples."
keywords: ["java stampedlock", "read write lock java", "optimistic read lock java", "stampedlock example", "concurrency java"]
---

# High-Performance Read-Write Lock in Java — Understanding `StampedLock`
Introduced in **Java 8**, `StampedLock` is a high-performance read-write lock designed to address the **writer starvation** issue present in `ReentrantReadWriteLock`. 

Compared to traditional locking mechanisms, `StampedLock` offers:
- **Optimistic, non-blocking reads**
- **Independence from the AQS framework**
- **Better concurrency and lower locking overhead**

> ***Note***: `StampedLock` is **not reentrant** and not suited for nested lock usage.

## 1. Comparison: `StampedLock` vs. `ReentrantReadWriteLock`

| Feature                         | StampedLock            | ReentrantReadWriteLock   |
|----------------------------------|--------------------------|----------------------------|
| AQS-based                       | No                   | Yes                     |
| Supports Optimistic Read       | Yes                  | No                      |
| Lock Downgrading               | Yes (manually)       | Yes                     |
| Reentrancy                     | No                   | Yes                     |
| Fairness Policy                | Non-fair only        | Configurable            |
| Performance (Read-heavy)       | High                 | Moderate                |
| Writer Starvation              | Solved               | Possible                |

## 2. Core Concepts of `StampedLock`

### 2.1 Optimistic Read Lock
`StampedLock` allows non-blocking reads through optimistic locking, which works under the assumption that write conflicts are rare and checks for conflicts only after the read is done.
```java
long stamp = lock.tryOptimisticRead();
try {
    // Perform read operations
    if (lock.validate(stamp)) {
        // No write occurred — safe to proceed
    } else {
        // Fallback to read lock
        stamp = lock.readLock();
    }
} finally {
    lock.unlock(stamp);
}
```
- `tryOptimisticRead()` returns a **version stamp** if no write is ongoing.
- `validate(stamp)` checks whether the version remains unchanged.

> ***Note***: In highly contended environments, excessive optimistic failures may degrade performance.

### 2.2 Lock State Management
Unlike AQS-based locks, `StampedLock` manages lock state via a `volatile long state` field.

#### Bit Partitioning (64 bits total)
| Bits       | Purpose                        |
|------------|--------------------------------|
| High 24    | Version number (incremented by write lock) |
| Bit 8      | Write lock indicator           |
| Low 7      | Read lock count                |

```java
// The number of bits used to represent the reader count (lowest 7 bits of the state)
private static final int LG_READERS = 7;

// Bit mask for the write lock (the 8th bit, value: 128)
private static final long WBIT  = 1L << LG_READERS;

// Bit mask for the read lock count (lowest 7 bits, value: 127)
private static final long RBITS = WBIT - 1L;

// Initial stamp value for the lock state (version number starts at 256)
private static final long ORIGIN = WBIT << 1;
```

### 2.3 Spin Locking
To avoid the cost of blocking threads, `StampedLock` uses **spin loops** extensively while acquiring locks.
```java
for (int spins = -1;;) {
    // Spin-wait logic for contention handling
}
```

> ***Note***: For long-duration write locks, Spin-locking will waste CPU resources.

### 2.4 Improved CLH Queue for Thread Management
`StampedLock` uses a lightweight variation of the **CLH queue** to manage contention.
- Consecutive reader threads are grouped into a `cowait` chain.
- Writer threads are queued separately and not bypassed by new readers.

```java
static final class WNode {
    volatile WNode prev, next, cowait;
    volatile Thread thread;
    volatile int status;
    final int mode; // RMODE or WMODE
}
```

> `StampedLock` **does not support condition variables**, which is **not suitable** for advanced thread coordination (e.g., `await()`/`signal()` use cases).

## 3. Limitations and Considerations
- **Non-reentrant**: Acquiring the same lock twice in the same thread leads to deadlock.
- **No condition support**: You cannot use `Condition` or equivalent APIs.
- **Manual stamp management**: Users must manually track and release locks.
- **Best suited for read-heavy scenarios**: Optimistic reads work best under low contention.
- **Compatible with Virtual Threads (Java 21+)**: Still requires careful release semantics.

## 4. Use Cases
- In-memory data caching
- Read-heavy data structures (e.g., Maps, Lists)
- High-frequency read systems under low write contention

## 5. Some Suggestions
While `StampedLock` delivers exceptional performance in specific scenarios, it introduces **higher programming complexity** and lacks features like reentrancy and conditions. 

Consider alternatives based on your needs:
- For ease of use → use `ReentrantReadWriteLock`
- For condition signaling → use `ReentrantLock + Condition`
- For non-blocking read/write → use `StampedLock`
