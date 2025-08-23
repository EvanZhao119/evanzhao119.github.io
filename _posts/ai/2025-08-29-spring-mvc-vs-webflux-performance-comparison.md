---
layout: post
title: "Spring MVC vs WebFlux: Performance, Concurrency, and Benchmark Comparison"
date: 2025-08-29
categories: ai
published: true
description: "Compare Spring MVC vs Spring WebFlux with real benchmarks. Learn their performance, scalability, and best use cases for high-concurrency and AI applications."
keywords: "Spring MVC vs WebFlux, Spring WebFlux performance, Java concurrency, reactive programming, Spring benchmarks, API scalability, AI inference in Java"
tags: [spring, java, webflux, concurrency, performance, ai, reactive-programming, benchmarks]
---

# Spring MVC vs WebFlux: A Practical Look at Concurrency and Performance
In the Java backend world, **Spring MVC** has long been the go-to framework. It’s simple, reliable, and works well for most business apps. But as we move into **AI inference**, **video processing**, and **high-concurrency workloads**, a traditional blocking model starts to struggle. That’s where **Spring WebFlux** comes in. **Spring WebFlux** a **reactive, non-blocking programming model** designed to handle thousands of requests without running out of threads.

In this post, I’ll walk through the **key differences between MVC and WebFlux**, share some **benchmark results**, and highlight when you should (or shouldn’t) choose WebFlux.  

---

## 1. Thread Model: How They Handle Requests

### Spring MVC (Blocking)
- Built on **Servlet + Tomcat**
- **One request = one thread**
- If a task (e.g., AI inference) takes 300ms, that thread is blocked the whole time 
- With enough concurrent users, the thread pool fills up and requests start waiting 

Think of one **waiter** who takes your order and then personally goes to cook and bring your food. Works fine for a few tables, but falls apart in a busy restaurant.

---

### Spring WebFlux (Non-Blocking)
- Built on **Netty + Reactor**
- Threads handle incoming requests and **immediately release** them to the reactive pipeline
- When the result is ready, Reactor finds the correct connection and writes the response  

The **waiter only takes orders**, while the **kitchen and another runner** handle food delivery. The waiter is free to serve new customers right away.

---

## 2. FAQs You Might Wonder About 

**Q1: Where is the connection stored if threads get released?**  
Netty manages it. Each request has a channel bound to an event loop. Even if threads change, the channel stays alive.

**Q2: Who writes the response back?** 
When the async task (Mono/Flux) completes, Reactor picks up the right channel and pushes the result.

---

## 3. Code Comparison  

### MVC Controller
```java
@PostMapping("/check")
public ModerationResult check(@RequestPart("file") MultipartFile file) throws Exception {
    return service.classify(file);
}
```

### WebFlux Controller
```java
@PostMapping("/check")
public Mono<ModerationResult> check(@RequestPart("file") MultipartFile file) {
    return fluxService.classify(file);
}
```

Looks almost the same — but under the hood, the runtime behaves very differently.  

---

## 4. Benchmark: MVC vs WebFlux  
We stress-tested both APIs locally with **200 concurrent users** using JMeter.

(*Remember:* this was **CPU-only**, no GPU acceleration — so the absolute numbers look slow. What matters here is the relative difference.) 

| Metric | MVC API | WebFlux API |
|--------|---------|-------------|
| Requests | 1018 | **1340** |
| Errors | 0 | 0 |
| Avg. Response Time | 35,853 ms (~36s) | **26,957 ms (~27s)** |
| Min Response Time | 3,232 ms | **340 ms** |
| Max Response Time | **49,187 ms (~49s)** | 141,574 ms (~141s) |
| Median | 36,556 ms | **23,713 ms** |
| 90th Percentile | **42,631 ms** | 54,292 ms |
| 95th Percentile | **44,824 ms** | 64,867 ms |
| 99th Percentile | **46,654 ms** | 86,554 ms |
| Throughput | **4.93 req/s** | 3.31 req/s |
| APDEX (500ms / 1.5s) | 0.000 | **0.002** |

### Key Takeaways
- **Stability**: Both had zero errors 
- **Latency**:  
  - MVC = consistently high (32–47s)  
  - WebFlux = much lower on average (~27s) but with a **long tail** (some requests hit ~141s)  
- **Throughput**: MVC slightly higher (because threads were blocking), but not sustainable at scale
- **Scalability**: WebFlux shines — it avoids thread pool exhaustion, which is critical in real production workloads

---

## 5. When to Use WebFlux  

**Great for:**
- AI model inference (image, text, video)
- Long-running, high-concurrency APIs 
- API gateway layer (routing, rate-limiting, monitoring)

**Less ideal for:**
- Simple CRUD apps (MVC is easier)
- Teams new to reactive programming (steep learning curve)

---

## 6. Conclusion
Spring MVC and WebFlux are not rivals. In contrast, they are **complementary**.
- **MVC** is perfect for 80% of standard business APIs
- **WebFlux** is better for high-concurrency, resource-intensive workloads

In our benchmark, WebFlux showed **lower average latency and stronger concurrency handling**, even though it suffered from long-tail requests. In real production, with **GPU acceleration, batching, or distributed inference**, WebFlux’s advantage grows even bigger.

The future is **hybrid**: use MVC where it makes sense, and WebFlux where performance and scalability matter most. 