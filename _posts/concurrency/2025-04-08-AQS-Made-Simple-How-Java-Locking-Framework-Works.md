---
layout: post
title: "AQS Made Simple: How Java’s Locking Framework Works"
date: 2025-04-08
categories: concurrency
published: true
---

# AQS Made Simple: How Java’s Locking Framework Works
**AbstractQueuedSynchronizer (AQS)** is a foundational framework in Java for building locks and synchronizers like `ReentrantLock`, `CountDownLatch`, `Semaphore`, etc. It manages a **FIFO queue** and a **state variable** to handle synchronization.

Its primary goal is to simplify the development of robust and scalable synchronization tools.

With the introduction of Virtual Threads in Java 21, traditional blocking locks (like ReentrantLock) are being re-evaluated in favor of non-blocking constructs or high-throughput alternatives like `StampedLock`.

## Core Mechanism of AQS
### State Management
AQS maintains an internal state using a `volatile int`.
```java
private volatile int state;
```
AQS provides protected methods for accessing and modifying this state.
```java
protected final int getState();
protected final void setState(int newState);
protected final boolean compareAndSetState(int expect, int update);
```
These are typically used by subclasses to control synchronization logic.

### Methods to Override
When extending AQS, developers implement the following protected methods to define custom locking behavior.
```java
protected boolean tryAcquire(int arg);
protected boolean tryRelease(int arg);
protected int tryAcquireShared(int arg);
protected boolean tryReleaseShared(int arg);
protected boolean isHeldExclusively();
```
These methods are called by higher-level functions like `acquire()` and `release()` to control how threads access shared resources.

## ReentrantLock Example
```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    // Get the current thread
    final Thread current = Thread.currentThread();

    // Get the current state value, which represents the lock status
    int c = getState();

    // If the state is 0, it means the lock is currently free
    if (c == 0) {
        // Attempt to atomically set the state to `acquires` (typically 1)
        if (compareAndSetState(0, acquires)) {
            // Set the current thread as the exclusive owner of the lock
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // If the state is not 0, the lock is already held by another thread
    // Check if the current thread is the one holding the lock (reentrant case)
    else if (current == getExclusiveOwnerThread()) {
        // If so, allow reentrancy by incrementing the state
        int nextc = c + acquires;

        // Overflow check (not likely, but needed for safety)
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");

        // Update the state with the new value
        setState(nextc);
        return true;
    }

    // Failed to acquire the lock
    return false;
}
```

## Benefits
- Generic: Supports exclusive and shared locks.
- Efficient: Relies on CAS and condition queues for performance.
- Extendable: Easy to create custom synchronizers.

Relying on blocking, AQS will be becoming less popular as Virtual Threads gradually offer a faster and more efficient way to handle concurrency without blocking.
