---
layout: post
title: "Java 21 Virtual Threads vs FixedThreadPool: Performance Comparison with 10,000 Tasks"
date: 2025-09-12
categories: concurrency
published: true
tags: ["Java", "Virtual Threads", "Concurrency", "Performance", "Benchmark"]
description: "A benchmark experiment comparing Java 21 Virtual Threads and FixedThreadPool with 10,000 blocking tasks, analyzing execution time and memory usage."
---

# Java 21 Virtual Threads vs FixedThreadPool: Performance Comparison with 10,000 Tasks
In Java 21, **Virtual Threads** have become a GA (General Availability) feature. The main goal is to **make each concurrent task as cheap as creating an object**, avoiding the complexity and memory overhead of traditional thread pools.

Traditional `FixedThreadPool` has some drawbacks compared to virtual thread pool.
1. The number of threads is fixed and cannot grow for each task.
2. With many blocking tasks, tasks must queue up, significantly increasing total execution time.
3. Each OS thread consumes a large native stack.

Virtual Threads are completely different.
1. Creation and scheduling cost is almost the same as creating an object.
2. Every task can easily get its own thread, without being limited by pool size.
3. When blocked, they are suspended without occupying a carrier OS thread.

---

## 1. Experiment Design
To illustrate the difference, we designed a simple benchmark.
- **Number of tasks**: 10,000
- **Task content**: each task sleeps for 100ms (`Thread.sleep(100ms)`) simulating I/O blocking.
- **Comparison**
	1. `Executors.newFixedThreadPool(200)`
	2. `Executors.newVirtualThreadPerTaskExecutor()`
- **Metrics observed**: total execution time and JVM memory usage increase.

### Experiment Code

```java
runWithExecutor(Executors.newFixedThreadPool(200), taskCount, "FixedThreadPool");
runWithExecutor(Executors.newVirtualThreadPerTaskExecutor(), taskCount, "VirtualThread");
```

```java
private static void printMemoryUsage(String label) {
    Runtime runtime = Runtime.getRuntime();
    long used = (runtime.totalMemory() - runtime.freeMemory()) / (1024 * 1024);
    System.out.printf("[%s] Used Memory: %d MB%n", label, used);
}
```

### Experiment Results
![Virtual Threads vs FixedThreadPool Experiment Results](/assets/images/2025_09_12_experiment_results.png "Virtual Threads vs FixedThreadPool Experiment Results")

### Comparison Table

| Executor                | Time (ms) | Memory Increase (MB) |
|--------------------------|-----------|-----------------------|
| FixedThreadPool (200)    | 5194      | 90                    |
| VirtualThread (10,000)   | 208       | 20                    |

---

## 2. Virtual Threads Conclusion

### Execution Time
- FixedThreadPool: ~5.2 seconds
- VirtualThread: ~0.2 seconds

**Virtual Threads improved throughput by more than 25x**.

![Virtual Threads vs FixedThreadPool Execution Time Comparison](/assets/images/2025_09_12_execution_time_comparison.png "Virtual Threads vs FixedThreadPool Execution Time Comparison")

### Memory Usage
- FixedThreadPool: grew from 3 MB to 93 MB (~+90 MB)
- VirtualThread: grew from 93 MB to 113 MB (~+20 MB)  

**Virtual Threads consumed significantly less memory even with 10,000 tasks**.

![Virtual Threads vs FixedThreadPool Memory Usage Increase Comparison](/assets/images/2025_09_12_memory_usage_increase_comparison.png "Virtual Threads vs FixedThreadPool Memory Usage Increase Comparison")

> This experiment demonstrates the **lightweight and high-concurrency advantage of Java 21 Virtual Threads**: enabling “one thread per task” without the heavy cost of traditional threads.

### Significant performance advantage
- `FixedThreadPool` is constrained by pool size, requiring 50 batches to finish all tasks.  
- Virtual Threads create one thread for each task, and since most of them just sleep and wake when needed, they can run almost like a single batch, making the execution time very short.

### Lower memory
- OS threads usually require MB-level stack space.
- Virtual Threads allocate their stack on the heap and grow it on demand. 10,000 threads only added tens of MB.

### Core design
- For Virtual Threads, we no longer need to artificially limit the number of threads.
- Because Virtual Threads are lightweight and scheduling is cheap, the JVM can provide each task with its own thread.

### Best use cases
- Large numbers of blocking I/O tasks (web requests, database calls, remote service invocations).
- Applications where concurrency greatly exceeds CPU cores.

### Not suitable for
- CPU-bound workloads. Virtual Threads do not make CPU faster.

---

## Related Resources
- [Project Loom: Virtual Threads](https://openjdk.org/jeps/444) 
- [Java Concurrency API Documentation](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
- [Inside Java Podcast](https://inside.java/podcast/)
- Related post: [Java ThreadPoolExecutor Explained](/concurrency/2025/07/03/java-threadpoolexecutor-explained.html)
- Related post: [Java Smart Locking with Virtual Threads](/concurrency/2025/06/28/java-smart-locking-virtual-threads.html)
- Related post: [Spring MVC vs WebFlux: A Practical Look at Concurrency and Performance](/ai/2025/08/25/spring-mvc-vs-webflux-performance-comparison.html) 
- Related post: [Building a High-Concurrency AI Moderation Service with Spring WebFlux — From Concept to Production](/ai/2025/08/27/spring-webflux-ai-nsfw-moderation-api.html)
