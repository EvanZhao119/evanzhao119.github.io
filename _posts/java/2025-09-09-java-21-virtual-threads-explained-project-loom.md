---
layout: post
title: "Java 21 Feature: Virtual Threads Explained with Examples | Project Loom Guide"
date: 2025-09-09
categories: java
published: true
description: "Learn how Java 21 Virtual Threads (Project Loom) work with practical examples. Step-by-step guide covering thread creation, ExecutorService usage, performance benefits, and real-world scenarios."
tags: [Java, Java 21, Virtual Threads, Project Loom, Concurrency, Multithreading, ExecutorService, High Concurrency, I/O Performance, Reactive Alternative]
---

# Java 21 Feature: Virtual Threads Explained with Examples

## 1. Background: Why Do We Need Virtual Threads?
In traditional Java, threads are **platform threads**, which are directly mapped to the operating system (OS) threads. 

This model has several drawbacks:
- **High creation cost**: each thread consumes MB-level stack memory.
- **Limited number of threads**: an application may run out of resources with just a few thousand threads.
- **Poor performance under high concurrency**: in I/O-intensive scenarios (such as web servers or database connections), handling tens of thousands of concurrent requests becomes a bottleneck.

To solve these issues, **JDK 21 introduces Virtual Threads** as a standard feature (from **Project Loom**). The goal is to allow Java applications to create **millions of concurrent threads** with minimal overhead.

---

## 2. What Are Virtual Threads?
1. Virtual threads are **lightweight threads managed by the JVM**, not directly bound to OS threads. 
2. When a virtual thread performs a blocking I/O operation, the JVM **suspends** it and releases the underlying OS thread for other tasks. 
3. The **programming model remains the same**: virtual threads are still `java.lang.Thread`. We can keep writing code in the familiar style without learning new APIs.

---

## 3. How to Create Virtual Threads?
JDK 21 provides new APIs.
```java
// Create and start a virtual thread
Thread.startVirtualThread(() -> {
    System.out.println("Hello from: " + Thread.currentThread());
});
```
A more common approach is to use the `ExecutorService` designed for virtual threads.
```java
import java.util.concurrent.*;

public class VirtualThreadExecutorDemo {
    public static void main(String[] args) throws Exception {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 5; i++) {
                int id = i;
                executor.submit(() -> {
                    System.out.println("Running task " + id + " in " + Thread.currentThread());
                    Thread.sleep(500); // simulate blocking
                    return id;
                });
            }
        } // automatically waits for all tasks to finish
    }
}
```

--- 

## 4. Use Cases

**Best for I/O-intensive tasks**:
- Web requests and APIs  
- Database queries and transactions  
- Real-time event streaming  
- **AI inference services** (e.g., running multiple model predictions concurrently without complex async code)

**Not ideal for CPU-intensive tasks**:  
- Matrix multiplication and heavy numerical computing  
- Image and video processing  
- **Deep learning model training**

Here, the bottleneck is CPU power, not thread scheduling, so virtual threads don’t offer significant advantages.

---

## 5. Conclusion
Virtual threads in **Java 21** make concurrent programming dramatically simpler.
- We can now scale to **millions of threads** with minimal overhead.
- They allow developers to write **simple, blocking-style code** that runs efficiently under high concurrency.
- For **AI engineering**, virtual threads are particularly valuable in.
  - Serving large volumes of **inference API calls** concurrently
  - Handling **parallel model pipelines** (e.g., NLP + image recognition in the same request)
  - Building **scalable microservices** that integrate machine learning models with databases and external APIs

If building modern applications, especially **AI-powered services**, please consider using `newVirtualThreadPerTaskExecutor()` that lets you keep clean, synchronous code while still scaling to massive concurrency.

---

## Related Resources
- [Project Loom: Virtual Threads](https://openjdk.org/jeps/444)  
- [Java Concurrency API Documentation](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
- [Inside Java Podcast](https://inside.java/podcast/)    
- Related post: [Spring MVC vs WebFlux: A Practical Look at Concurrency and Performance](/ai/2025/08/25/spring-mvc-vs-webflux-performance-comparison.html)  
- Related post: [Building a High-Concurrency AI Moderation Service with Spring WebFlux — From Concept to Production](/ai/2025/08/27/spring-webflux-ai-nsfw-moderation-api.html)
