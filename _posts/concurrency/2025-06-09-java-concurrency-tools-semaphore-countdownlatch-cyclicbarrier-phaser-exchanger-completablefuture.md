---
layout: post
title: "Java Concurrency Tools Explained: Semaphore, CountDownLatch, CyclicBarrier, Phaser, Exchanger, and CompletableFuture"
date: 2025-06-09
categories: concurrency
published: true
description: "Learn how to use Java concurrency utilities including Semaphore, CountDownLatch, CyclicBarrier, Phaser, Exchanger, and CompletableFuture with examples, code, and best practices."
keywords: ["Java concurrency tools", "Semaphore", "CountDownLatch", "CyclicBarrier", "Phaser", "Exchanger", "CompletableFuture", "Java multithreading", "asynchronous programming"]
---

# Java Concurrency Tools: What I Use and Why
> Featuring `Semaphore`, `CyclicBarrier`, `CountDownLatch`, and modern concurrency tools like `Phaser`, `Exchanger`, and `CompletableFuture`.


## 1. Semaphore
A `Semaphore` is a concurrency utility that controls access to a resource by managing a set number of permits. Threads must acquire a permit before proceeding and release it afterward. It is useful for limiting concurrent access.
 
```java
import java.util.concurrent.Semaphore;

public class SemaphoreStudy {
    private static final Semaphore sp = new Semaphore(1);
    private static int cnt = 0;

    public static void main(String[] args){
        Runnable r1 = () -> {
            try {
                for (int i = 0; i < 100; ++i) {
                    sp.acquire();
                    Thread.sleep(1);
                    System.out.println("R1 " + System.currentTimeMillis() + " " + cnt++);
                    sp.release();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        };

        Runnable r2 = () -> {
            try {
                for (int i = 0; i < 100; ++i) {
                    sp.acquire();
                    Thread.sleep(1);
                    System.out.println("R2 " + System.currentTimeMillis() + " " + cnt++);
                    sp.release();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        };

        new Thread(r1).start();
        new Thread(r2).start();
    }
}
```
Two threads update a shared variable `cnt` concurrently. Only one thread can access the critical section at a time, due to the semaphore with one permit.

**Sample Output (truncated)**
```
R1 1712205844441 0  
R2 1712205844442 1  
R1 1712205844443 2  
...
```

## 2. CyclicBarrier
`CyclicBarrier` is used to synchronize a fixed number of threads at a common barrier point. All threads must call `await()` before any of them proceeds. After release, the barrier is reset for reuse.
```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierStudy {
    private static CyclicBarrier cb = new CyclicBarrier(2, () -> {
        System.out.println("CyclicBarrierStudy finish.");
    });

    public static void main(String[] args){
        Runnable r1 = () -> {
            try {
                System.out.println("R1 running.");
                cb.await();
                System.out.println("R1 finish.");
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        };

        Runnable r2 = () -> {
            try {
                System.out.println("R2 running.");
                cb.await();
                System.out.println("R2 finish.");
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        };

        new Thread(r1).start();
        new Thread(r2).start();
    }
}
```
Two threads reach a common barrier. Once both call `await()`, a predefined action is executed, and then they both proceed.

**Sample Output**
```
R1 running.  
R2 running.  
CyclicBarrierStudy finish.  
R2 finish.  
R1 finish.  
```

## 3. CountDownLatch
`CountDownLatch` allows one or more threads to wait until a set of events has occurred. A counter is reduced by `countDown()` and threads await its value reaching zero using `await()`.
```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchStudy {
    private static final CountDownLatch cdl = new CountDownLatch(1);

    public static void main(String[] args){
        Runnable r1 = () -> {
            try {
                System.out.println("R1 waiting.");
                cdl.await();
                System.out.println("R1 finish.");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };

        Runnable r2 = () -> {
            System.out.println("R2 running.");
            cdl.countDown();
            System.out.println("R2 finish.");
        };

        new Thread(r1).start();
        new Thread(r2).start();
    }
}
```
Thread `r1` waits until thread `r2` calls `countDown()`. Once the latch reaches zero, `r1` continues execution.
**Sample Output**
```
R1 waiting.  
R2 running.  
R2 finish.  
R1 finish.  
```

## 4. Phaser
`Phaser` is a flexible and reusable synchronization barrier supporting dynamic thread registration and multi-phase coordination. It is often a better choice than `CyclicBarrier` or `CountDownLatch` in many use cases.
```java
import java.util.concurrent.Phaser;

public class PhaserStudy {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(3); // Register 3 parties

        Runnable task = () -> {
            System.out.println(Thread.currentThread().getName() + " phase 1");
            phaser.arriveAndAwaitAdvance();

            System.out.println(Thread.currentThread().getName() + " phase 2");
            phaser.arriveAndAwaitAdvance();
        };

        for (int i = 0; i < 3; i++) {
            new Thread(task).start();
        }
    }
}
```
Three threads synchronize at each of two phases. The `Phaser` handles advancing to the next phase automatically after all threads arrive.

**Sample Output**
```
Thread-0 phase 1  
Thread-1 phase 1  
Thread-2 phase 1  
Thread-2 phase 2  
Thread-1 phase 2  
Thread-0 phase 2  
```

## 5. Exchanger
`Exchanger` is used for exchanging data between two threads at a synchronization point.

```java
import java.util.concurrent.Exchanger;

public class ExchangerStudy {
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();

        new Thread(() -> {
            try {
                String msg = exchanger.exchange("From A");
                System.out.println("A got: " + msg);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        new Thread(() -> {
            try {
                String msg = exchanger.exchange("From B");
                System.out.println("B got: " + msg);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```
Two threads exchange messages using a synchronization point. Each receives the message sent by the other.

**Sample Output**
```
B got: From A  
A got: From B  
```

## 6. CompletableFuture
Introduced in Java 8, `CompletableFuture` is a powerful framework for asynchronous programming. It supports chaining, combining, and composing tasks.

```java
import java.util.concurrent.*;

public class CompletableFutureStudy {
    public static void main(String[] args) {
        CompletableFuture.supplyAsync(() -> {
            return "Step 1";
        }).thenApply(result -> {
            return result + " -> Step 2";
        }).thenAccept(System.out::println);
    }
}
```
An asynchronous task runs in the background and is chained with another computation. Final result is printed at the end.

**Sample Output**
```
Step 1 -> Step 2  
```

## Summary: When to Use Each Java Concurrency Tools

### `Semaphore`
Use it when you need to **limit access to a shared resource**, like controlling the number of threads accessing a database or connection pool.  
It's ideal for implementing **rate-limiting** or **resource permits**.

### `CountDownLatch`
Best for **one-time event coordination**, such as making one thread wait until a set of other threads finish their tasks  
_(e.g., wait for multiple services to initialize before proceeding)._

### `CyclicBarrier`
Choose this when you want a **fixed group of threads to wait for each other and then proceed together**, often used in **multi-threaded computation phases** _(e.g., matrix processing)_.

### `Phaser`
A more flexible alternative to `CyclicBarrier` and `CountDownLatch`, great for **multi-phase tasks** with **dynamic thread participation**.  
Use it when **thread count may change** or when **multiple rounds of coordination** are needed.

### `Exchanger`
Best suited for **two-thread collaboration**, where each thread needs to **exchange data at a synchronization point**.  
Useful in **pipeline stages** or **producer-consumer pairs**.

### `CompletableFuture`
Ideal for **asynchronous programming** and **task chaining**.  
Use it when you need to **run tasks in parallel**, **handle dependencies**, and **process results in a non-blocking way**  
_(e.g., combining multiple API calls)_.
