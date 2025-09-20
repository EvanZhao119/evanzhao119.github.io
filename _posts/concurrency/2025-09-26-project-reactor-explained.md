---
layout: post
title: "Project Reactor Guide: Explained with Examples"
date: 2025-09-26
categories: concurrency
published: true
description: "A beginner-friendly guide to Project Reactor. Learn Mono, Flux, operators, backpressure, and schedulers with practical examples for reactive programming in Java."
tags: ["Project Reactor", "Reactive Streams", "Java Reactive Programming", "Mono vs Flux", "Spring WebFlux", "Reactor Operators", "Backpressure"]
---

# Project Reactor: Explained with Examples
I will give you a clear, beginner-friendly explanation of **Project Reactor**, based on the official documentation. We'll cover the basics, core types, common operators, and advanced features — all with practical examples.

---

## 1. What is Project Reactor?
- **Reactor** is a **JVM reactive programming library** based on the [Reactive Streams Specification](https://www.reactive-streams.org/).
- Key features:
    - **Asynchronous** and non-blocking.
    - **Backpressure support**: downstream tells upstream how much data it can handle.
    - **Functional API**: similar to Java Streams, but works with **asynchronous data flow**.
    - The foundation of **Spring WebFlux**.

---

## 2. Core Types
Reactor provides two main `Publisher` types.

### Mono
Represents a stream with **0 or 1 element**.
Common use cases: HTTP responses, database lookups.
```java
Mono<String> mono = Mono.just("Hello Reactor");
mono.subscribe(System.out::println);
```

### Flux
Represents a stream with **0 to N elements**.
Common use cases: event streams, message queues.
```java
Flux<Integer> flux = Flux.range(1, 5);
flux.subscribe(System.out::println);
```

### Subscription Model
Data only flows after you call `subscribe()`.
```java
Flux.range(1, 3)
    .map(i -> i * 2)
    .subscribe(System.out::println); // Output: 2, 4, 6
```

---

## 3. Operators
Reactor comes with many operators, like SQL + Java Stream combined.

### Transforming
- `map`: one-to-one transformation
- `flatMap`: flatten async streams

```java
Flux.just("a", "b")
    .flatMap(s -> Mono.just(s.toUpperCase()))
    .subscribe(System.out::println);
```

### Filtering
- `filter`: keep matching elements
- `take`: take first N elements
- `skip`: skip first N elements

### Combining
- `merge`: merge multiple streams (concurrent)
- `concat`: concatenate streams in order
- `zip`: pair elements

```java
Flux.zip(Flux.just(1, 2), Flux.just("A", "B"))
    .subscribe(System.out::println); 
// Output: (1,A) (2,B)
```

### Error Handling
- `onErrorReturn`: fallback value
- `onErrorResume`: fallback stream
- `retry`: retry operation

---

## 4. Advanced Features
### Backpressure
- A core Reactive Streams concept: **downstream controls demand**.  
- `Flux` and `Mono` handle this automatically, preventing overload.

### Schedulers
By default, Reactor runs in the current thread.  
Use `publishOn` or `subscribeOn` to switch thread pools.  

```java
Flux.range(1, 10)
    .publishOn(Schedulers.boundedElastic())
    .map(i -> i * 2)
    .subscribe(System.out::println);
```

Common schedulers:
- `immediate()`: current thread
- `single()`: single-threaded
- `boundedElastic()`: elastic thread pool (good for blocking tasks)
- `parallel()`: parallel thread pool (CPU-intensive tasks)

### Cold vs Hot Publishers
- **Cold publisher**: each subscriber gets a fresh stream (e.g., `Flux.range`).
- **Hot publisher**: data is emitted regardless of subscribers (e.g., `ConnectableFlux`).

---

## 5. Summary
- **Mono**: async single value.  
- **Flux**: async stream of many values.  
- **Operators**: SQL/Stream-like tools for data flow.  
- **Backpressure & Schedulers**: handle performance and resource management. 
- Reactor is the foundation of **Spring WebFlux** and modern Java reactive programming. 

---

## Related Resources
- [Project Reactor Reference Guide](https://projectreactor.io/docs/core/release/reference/)  
- [Reactive Streams Specification](https://www.reactive-streams.org/)  
- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/reference/web/webflux.html)  
- [Spring WebClient Guide](https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html)  
- Related post: [Reactive Spring: Reactive Web Explained](/spring/2025/09/22/reactive-spring-reactive-web-explained.html)  
- Related post: [Spring MVC vs WebFlux: A Practical Look at Concurrency and Performance](/ai/2025/08/25/spring-mvc-vs-webflux-performance-comparison.html)  
- Related post: [Building a High-Concurrency AI Moderation Service with Spring WebFlux — From Concept to Production](/ai/2025/08/27/spring-webflux-ai-nsfw-moderation-api.html)  
- Related post: [Monitoring Spring WebFlux with Prometheus and Grafana](/ai/2025/08/31/spring-webflux-monitoring-prometheus-grafana.html) 