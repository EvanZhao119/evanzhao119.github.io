---
layout: post
title: "Reactive Spring: Reactive Web Explained"
date: 2025-09-22
categories: spring
published: true
description: "A beginner-friendly guide to Spring WebFlux, covering Mono vs Flux, backpressure, WebClient, and the difference from Spring MVC. Includes code examples and key takeaways."
keywords: ["Reactive Spring", "Spring WebFlux tutorial", "Mono vs Flux", "Spring WebClient", "Spring MVC vs WebFlux", "Reactive programming in Java"]
---

# Reactive Spring: Reactive Web Explained
Spring MVC works on top of the Servlet API and follows a “one thread per request” model. This means that every incoming request needs its own thread. When the number of requests grows very largely, the server may end up managing too many threads, which leads to extra memory and CPU cost.

Spring WebFlux takes a different approach. It is built on the Reactive Streams standard and uses non-blocking I/O with an event-loop style of processing. Instead of creating a new thread for every request, a small number of threads can handle many requests efficiently. This makes it much better for applications that spend a lot of time waiting on I/O operations, such as calling external services or databases. WebFlux is powered by Project Reactor, which introduces the core types Flux and Mono for handling reactive data flows.

*The key idea:* WebFlux is not meant to replace Spring MVC. Rather, it provides an alternative choice when you need applications that are scalable and non-blocking.

---

## 1. WebFlux Programming Models
### (1) Annotation-based
Similar to Spring MVC (`@RestController`, `@RequestMapping`), but the return type is **Mono/Flux** instead of plain objects.

**Annotation style feels familiar for MVC developers.**

```java
@RestController
public class GreetingController {
    @GetMapping("/hello")
    public Mono<String> hello() {
        return Mono.just("Hello, Reactive World!");
    }
}
```

### (2) Functional Routing
A more functional, DSL-style approach using `RouterFunction` and `HandlerFunction`.

**Functional style is lightweight and flexible, similar to frameworks like Express.js or Ktor.**
```java
@Configuration
public class RouterConfig {
    @Bean
    public RouterFunction<ServerResponse> route(GreetingHandler handler) {
        return RouterFunctions
                .route(RequestPredicates.GET("/hello"), handler::hello);
    }
}

@Component
public class GreetingHandler {
    public Mono<ServerResponse> hello(ServerRequest request) {
        return ServerResponse.ok().bodyValue("Hello, Reactive World!");
    }
}
```

---

## 2. Return Types: Mono vs Flux
- **Mono<T>**: at most 1 element (think `Optional` or `Future`).
- **Flux<T>**: 0…N elements (think `Stream`). 

```java
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable String id) {
    return userRepository.findById(id);
}

@GetMapping("/users")
public Flux<User> getAllUsers() {
    return userRepository.findAll();
}
```

---

## 3. Backpressure
Reactive Streams include **backpressure**: the subscriber tells the publisher *“I can only handle 5 items right now.”*  

```java
Flux.range(1, 1000)
    .log()
    .limitRate(100); // request 100 items at a time
```

This prevents overload and keeps the system stable.

---

## 4. WebClient (Replacing RestTemplate)
- `RestTemplate` is **blocking**. 
- **WebClient** is **non-blocking** and integrates with Mono/Flux.

```java
WebClient client = WebClient.create("http://localhost:8080");

Mono<User> user = client.get()
        .uri("/users/{id}", "123")
        .retrieve()
        .bodyToMono(User.class);
```
WebClient is the modern choice for microservices and concurrent API calls.

---

## 5. WebFlux vs Spring MVC

| Feature | Spring MVC | Spring WebFlux |
|---------|------------|----------------|
| Foundation | Servlet API (blocking I/O) | Reactive Streams + Netty/Undertow/Tomcat (non-blocking I/O) |
| Concurrency | Thread per request | Small event-loop thread pool |
| Return Types | Plain objects | Mono / Flux |
| HTTP Client | RestTemplate | WebClient |
| Best Use Case | CPU-bound apps, moderate traffic | I/O-heavy, high-concurrency apps |

---

## 6. Example: Streaming Data
This streams data to the client using **Server-Sent Events (SSE)**.  
The front-end can use `EventSource` to receive updates in real time.
```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream() {
    return Flux.interval(Duration.ofSeconds(1))
               .map(seq -> "Message " + seq);
}
```

---

## 7. Key Takeaways
1. WebFlux = **reactive programming + non-blocking I/O**.  
2. Core types: `Mono` and `Flux`.  
3. Two styles: Annotation-based (MVC-like) and Functional routing (DSL-like).  
4. Use **WebClient**, not RestTemplate, for async calls.  
5. Backpressure ensures system stability.  
6. Best for **high-concurrency, I/O-bound applications**.  

---

## Related Resources
- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/reference/web/webflux.html)  
- [Project Reactor Reference Guide](https://projectreactor.io/docs/core/release/reference/)  
- [Reactive Streams Specification](https://www.reactive-streams.org/)  
- [Spring WebClient Guide](https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html)  
- Related post: [Spring MVC vs WebFlux: A Practical Look at Concurrency and Performance](/ai/2025/08/25/spring-mvc-vs-webflux-performance-comparison.html)  
- Related post: [Building a High-Concurrency AI Moderation Service with Spring WebFlux — From Concept to Production](/ai/2025/08/27/spring-webflux-ai-nsfw-moderation-api.html)  
- Related post: [Monitoring Spring WebFlux with Prometheus and Grafana](/ai/2025/08/31/spring-webflux-monitoring-prometheus-grafana.html)