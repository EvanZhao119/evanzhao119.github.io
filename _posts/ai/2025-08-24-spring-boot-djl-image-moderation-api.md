---
layout: post
title: "Spring Boot + DJL: High-Performance Image Moderation API with AI & Java Concurrency"
date: 2025-08-24
categories: ai
published: true
description: "Learn how to build a scalable Image Moderation API using Spring Boot and DJL (Deep Java Library). Covers model loading, memory management, async concurrency, and production deployment."
keywords: ["Spring Boot DJL tutorial", "Image Moderation API Java", "AI content moderation Spring Boot", "Deep Java Library DJL", "Java concurrency optimization", "Spring Boot AI API"]
---

# Spring Boot + DJL: Build a High-Performance Image Moderation API
In this post, we walk through how to design and implement a **high-performance Image Moderation API** using **Spring Boot** and **Deep Java Library (DJL)**. 

The focus is on **DJL memory management**, **Java concurrency optimization with Spring Boot**, and building a scalable **AI-powered Image Moderation API** ready for production workloads.

## Why Use Spring Boot and DJL for Building an AI Image Moderation API?
- **Spring Boot** provides a lightweight, production-ready Java framework.
- **DJL (Deep Java Library)** allows you to run AI/ML models natively in Java.
- Perfect for **AI-powered content moderation**, e.g., detecting NSFW or unsafe images.

## Architecture of Spring Boot + DJL Image Moderation API
The system consists of:
1. **Spring Boot REST API** – handles HTTP requests.
2. **DJL Model Service** – loads and reuses pre-trained models.
3. **Async Thread Pool** – improves concurrency handling.
4. **Memory Management Layer** – ensures efficient predictor lifecycle.

### Core Modules

| Module                     | Description                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| `NsfwModelConfig`          | Loads the PyTorch model once and registers it as a global singleton bean    |
| `ModerationService`        | Handles image preprocessing and prediction, supports sync and async modes   |
| `AsyncConfig`              | Configures thread pool and enables Spring’s `@Async` annotation              |
| `ModerationController`     | Defines RESTful endpoints for image moderation                              |
| `GlobalExceptionHandler`   | Catches and formats all exceptions with consistent error responses           |
| Swagger Integration        | Generates interactive API documentation for easy testing and collaboration  |

---

## Memory Management in DJL for AI Image
1. **Singleton Model Loading** → load once, reuse across predictors. 
2. **Predictor Lifecycle** → managed via `try-with-resources` to prevent memory leaks. 
3. **Batch Processing** → reduce redundant overhead.

```java
@Bean
public ZooModel<NDList, Classifications> nsfwModel() {
    return model;
}
```

---

## Java Concurrency Optimization with Spring Boot Async

Using Spring’s async capabilities, we move model inference to a background thread, freeing up the request-handling thread immediately.

### Async Thread Pool Configuration

```java
@EnableAsync
@Configuration
public class AsyncConfig {
    @Bean("aiTaskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-Task-");
        executor.initialize();
        return executor;
    }
}
```

### Async Method

```java
@Async("aiTaskExecutor")
public CompletableFuture<ModerationResult> classifyAsync(MultipartFile file) {
    return CompletableFuture.completedFuture(classify(file));
}
```

### Benefits:
- The main thread is released immediately.
- The model inference happens in a background thread from the pool.
- Ideal for batch processing or high-frequency image upload scenarios.

---

## Performance Benchmark: Sync vs Async API in Spring Boot
I tested `/check` and `/check-async` with 30 concurrent requests using `ApacheBench`.

- The async version significantly outperforms the sync version in throughput and latency.
- Non-200 responses are expected due to missing files in the test payload — validation works as intended.

| Metric                 | `/check` (Sync)     | `/check-async` (Async) |
|------------------------|---------------------|--------------------------|
| Total Time             | 0.040 sec           | 0.028 sec                |
| Requests per second    | 757.58 req/sec      | 1064.96 req/sec          |
| Avg Response Time      | 6.6 ms              | 4.7 ms                   |
| Max Response Time      | 11 ms               | 5 ms                     |
| Non-200 Responses      | 31                  | 31                       |

These results confirm that using Spring Boot with DJL can deliver high-performance AI content moderation in Java, especially when combined with proper concurrency and memory optimization techniques.

---

## Best Practices for Building AI-Powered Image Moderation APIs

1. **Efficient model reuse** was achieved by loading the DJL model once during application startup and registering it as a Spring singleton bean. This avoids the overhead of reloading the model on every request.
2. I ensured **safe memory management** by using `try-with-resources` when creating `Predictor` instances, which automatically closes and releases resources after each inference task.
3. To handle **high concurrency**, we enabled asynchronous processing with Spring's `@Async` annotation and a custom thread pool. This allows the system to process multiple image classification tasks in parallel without blocking the main thread.
4. The API was designed with a **developer-friendly REST structure**, and integrated with `Swagger` for automatically generated, interactive API documentation.
5. Finally, a **centralized global exception handler** was implemented to catch and return consistent, clean error messages across the entire application, improving maintainability and user experience.

---

So far, we have demonstrated how **Java + DJL + Spring Boot** can serve as a powerful combo for deploying scalable AI services. To take this project further, we can continue to upgrade the model (e.g., CLIP) for improved accuracy and integrate OpenCV for better image preprocessing. Adding message queue support like Kafka will enable async task handling at scale. Finally, containerizing the service with Docker and deploying it on Kubernetes will enhance scalability and make the system cloud-ready.

---

## Related Resources
- [DJL Official Documentation](https://docs.djl.ai/)
- [Spring Boot Async Guide](https://spring.io/guides/gs/async-method/)
- Related post: [Deploying ResNet Models with Java DJL – From Zero to Hero](/ai/2025/08/14/deploy-resnet-java-djl-tutorial.html)
- Related post: [Java gRPC & HTTP Clients for Python Inference](/ai/2025/08/18/java-grpc-http-clients-python-inference.html)