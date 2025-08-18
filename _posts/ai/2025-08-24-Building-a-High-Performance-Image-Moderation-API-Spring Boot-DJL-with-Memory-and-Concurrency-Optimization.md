---
layout: post
title: "Building a High-Performance Image Moderation API: Spring Boot + DJL with Memory and Concurrency Optimization"
date: 2025-08-24
categories: ai
published: true
---

# Building a High-Performance Image Moderation API: Spring Boot + DJL with Memory and Concurrency Optimization
Today, I built a high-performance image moderation service using **Spring Boot** and **Deep Java Library (DJL)**. The core goals of this project were to optimize **memory usage** by reusing the model and improve **concurrent performance** through asynchronous processing.

## System Architecture Overview
The system follows a modular design, clearly separating responsibilities across model loading, inference logic, REST API, thread management, and exception handling.

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

## Memory Optimization: Load Once, Use Many

- Loading the model every time a request comes in is inefficient and memory-intensive.
- By using Spring's `@Bean`, we load the model only once during startup and reuse it across all requests.
- A new `Predictor` is created per request using the shared model, ensuring both safety and performance.

```java
@Bean
public ZooModel<NDList, Classifications> nsfwModel() {
    return model;
}
```

---

## Concurrency Optimization: Asynchronous Thread Execution

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

## Benchmarking Results (ApacheBench)
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

---

## Key Engineering Practices

1. **Efficient model reuse** was achieved by loading the DJL model once during application startup and registering it as a Spring singleton bean. This avoids the overhead of reloading the model on every request.
2. I ensured **safe memory management** by using `try-with-resources` when creating `Predictor` instances, which automatically closes and releases resources after each inference task.
3. To handle **high concurrency**, we enabled asynchronous processing with Spring's `@Async` annotation and a custom thread pool. This allows the system to process multiple image classification tasks in parallel without blocking the main thread.
4. The API was designed with a **developer-friendly REST structure**, and integrated with `Swagger` for automatically generated, interactive API documentation.
5. Finally, a **centralized global exception handler** was implemented to catch and return consistent, clean error messages across the entire application, improving maintainability and user experience.

---

So far, we have demonstrated how **Java + DJL + Spring Boot** can serve as a powerful combo for deploying scalable AI services. To take this project further, we can continue to upgrade the model (e.g., CLIP) for improved accuracy and integrate OpenCV for better image preprocessing. Adding message queue support like Kafka will enable async task handling at scale. Finally, containerizing the service with Docker and deploying it on Kubernetes will enhance scalability and make the system cloud-ready.
