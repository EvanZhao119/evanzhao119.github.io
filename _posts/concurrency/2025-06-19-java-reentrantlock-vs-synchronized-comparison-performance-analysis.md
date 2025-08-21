---
layout: post
title: "Java ReentrantLock vs synchronized: Deep Dive, Performance, and Source Code Analysis"
date: 2025-06-19
categories: concurrency
published: true
description: "Comprehensive comparison of Java's ReentrantLock and synchronized: performance, features, use cases, and source code analysis with JDK 21 updates." 
keywords: ["Java ReentrantLock vs synchronized", "Java concurrency locks", "ReentrantLock tutorial", "synchronized vs ReentrantLock performance", "Java multithreading", "AQS in Java"]
---

# Understand ReentrantLock: How It Works and How It Differs from `synchronized`
`ReentrantLock` is a powerful synchronization mechanism in Java, part of the `java.util.concurrent.locks` package. Compared with the `synchronized` keyword, `ReentrantLock` offers advanced capabilities and greater flexibility, making it suitable for complex concurrency control scenarios.

## ReentrantLock vs. synchronized

### 1. Performance Comparison
Since **JDK 6**, the performance difference between `ReentrantLock` and `synchronized` has largely diminished due to JVM-level optimizations such as **biased locking** and **lightweight locking**.

> **Update (JDK 17+ / 21)**: With the deprecation of biased locking (JEP 374), `synchronized` now performs better than `ReentrantLock` in most general-use cases. For simple mutual exclusion, prefer `synchronized` for its cleaner syntax and built-in lock management.

### 2. Feature Comparison
- **Interruptible Lock Acquisition**: `synchronized` does not support interruptible locking—once a thread starts waiting, it cannot be interrupted. In contrast, `ReentrantLock` supports interruptible lock acquisition using `lockInterruptibly()`.
- **Timeout for Acquiring Lock**: `synchronized` provides no mechanism to time out while trying to acquire a lock. `ReentrantLock` allows setting a timeout with `tryLock(long timeout, TimeUnit unit)`.
- **Fairness Control**: `synchronized` always uses a non-fair locking strategy—threads may acquire the lock out of order. `ReentrantLock` offers optional fairness, ensuring first-come-first-served behavior if configured.
- **Multiple Condition Variables**: `synchronized` supports only a single monitor condition, accessed via `wait()`, `notify()`, and `notifyAll()`. `ReentrantLock` supports multiple `Condition` objects through `newCondition()`, allowing more fine-grained control.
- **Reentrancy**: Both `synchronized` and `ReentrantLock` are reentrant, allowing the same thread to acquire the same lock multiple times.
- **Automatic Lock Release**: `synchronized` automatically releases the lock when exiting a `synchronized` block, even in case of exceptions. With `ReentrantLock`, the developer must manually release the lock using `unlock()`, typically in a `finally` block.

| Feature                         | `synchronized`          | `ReentrantLock`                 |
|----------------------------------|--------------------------|---------------------------------|
| Interruptible lock acquisition  | No                    | Yes                           |
| Timeout for acquiring lock      | No                    | Yes                           |
| Fairness control                | No                    | Yes (optional)               |
| Multiple condition variables    | No (only `wait/notify`) | Yes (`newCondition()`)       |
| Reentrancy                      | Yes                   | Yes                           |
| Automatic lock release          | Yes (via block exit)  | No (must call `unlock()`)    |

> `ReentrantLock` remains essential when fine-grained lock control, fairness, or advanced waiting mechanisms are needed.

## How to Use ReentrantLock

```java
Lock lock = new ReentrantLock(); // Non-fair lock by default
lock.lock();
try {
    // Critical section
} finally {
    lock.unlock();
}
```

### Fair Lock Example

```java
Lock fairLock = new ReentrantLock(true); // Explicitly use a fair lock
```

## ReentrantLock Source Code Analysis (Based on JDK 21)
### Class Structure Overview
```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {}
    static final class NonfairSync extends Sync {}
    static final class FairSync extends Sync {}
}
```
Internally, `ReentrantLock` uses the powerful AQS (AbstractQueuedSynchronizer) framework to manage thread queuing and state.


### Lock Acquisition (`lock()`)
#### Nonfair Lock
```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
- First, it attempts to acquire the lock using a **CAS** operation.
- If it fails, the thread enters the AQS queue and waits for the lock.

#### Fair Lock
```java
final void lock() {
    acquire(1); // Always enqueues the thread to maintain fairness
}
```

> Fair locks check `hasQueuedPredecessors()` to determine if the thread should wait in line, ensuring first-come-first-served behavior.

#### tryAcquire() Logic
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
- If the lock is free, it tries to acquire it with CAS.
- If the current thread already owns the lock, it increases the state (reentrant).
- Otherwise, the thread enters a **spin-then-block** loop in AQS.

### Lock Release (`unlock()`)
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
#### tryRelease
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
- Only the owning thread can release the lock.
- If the state drops to 0, the lock is released.

#### unparkSuccessor
```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }

    if (s != null)
        LockSupport.unpark(s.thread);
}
```
- This wakes up the next thread in the queue to try acquiring the lock.

## Summary & Best Practices
### When to Use What?
- **Use `synchronized` when**
    - You need simple and readable lock logic.
    - You want the JVM to manage lock lifecycle.
    - Performance is not a concern in fine-tuned scenarios.
- **Use `ReentrantLock` when**
    - You need timed or interruptible lock attempts.
    - You require fairness or multiple conditions.
    - You need non-block-structured locking.

> `synchronized` continues to receive JVM-level optimizations and is recommended for general locking needs. `ReentrantLock` shines in advanced scenarios.
