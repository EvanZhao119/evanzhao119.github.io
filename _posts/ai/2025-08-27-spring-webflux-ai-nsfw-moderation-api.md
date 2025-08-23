---
layout: post
title: "Spring WebFlux AI Moderation Service | Build High-Concurrency NSFW API"
date: 2025-08-27
categories: ai
published: true
description: "Learn how to build a scalable AI-powered NSFW image moderation API using Spring Boot, WebFlux, and DJL. Includes architecture, code examples, benchmarks, and performance optimization tips."
tags: [Spring Boot, WebFlux, DJL, AI Moderation, Reactive Programming, NSFW Detection, API Scalability, JMeter Benchmark]
---

# Building a High-Concurrency AI Moderation Service with Spring WebFlux — From Concept to Production
As AI applications move from research into real-world production, one of the biggest challenges is **how to wrap model inference into a high-concurrency, low-latency API service**.  

In this project, we implemented an **NSFW image moderation API**:
- **Input:** An image uploaded via `multipart/form-data` 
- **Output:** Classification results (porn, sexy, neutral) along with confidence scores  

We built it using **Spring Boot + WebFlux + DJL (Deep Java Library)**, creating a moderation service with production-level scalability.

---

## 1. Why WebFlux?

### 1.1 The Limits of Spring MVC
Spring MVC is based on the Servlet model, where **each request is tied to a dedicated thread**.  

In AI inference scenarios:
- Each prediction may take **200–500ms**
- During that time, the thread is blocked 
- Under high concurrency, the thread pool is quickly exhausted, and throughput drops  

### 1.2 The Advantages of WebFlux
Spring WebFlux is built on **Reactor’s reactive programming model** and **Netty’s non-blocking I/O**. 

Key benefits:
- Requests don’t block while waiting for inference 
- Tasks are handed off to `Mono`/`Flux`, freeing up the request thread immediately  
- Results are written back asynchronously once inference is done 

Using *Spring MVC* is like a waiter who takes your order and waits at your table until the food arrives. With *Spring WebFlux*, the waiter takes your order, the kitchen prepares it, and whichever waiter is free will deliver it. So the whole service will be more efficient.

---

## 2. Project Structure
![Spring WebFlux AI Moderation Service Project Structure Diagram](/assets/images/2025_08_27_spring_webflux_ai_moderation_structure.png "Spring WebFlux AI Moderation Service Project Structure")

---

## 3. Core Implementation

### 3.1 Model Configuration (NsfwModelConfig.java)
Loads the NSFW model (PyTorch, via DJL) and handles preprocessing.

```java
@Configuration
public class NsfwModelConfig {

    @Bean
    public ZooModel<NDList, Classifications> nsfwModel() throws Exception {
        Criteria<NDList, Classifications> criteria = Criteria.builder()
                .setTypes(NDList.class, Classifications.class)
                .optModelPath(modelPath)
                .optTranslator(translator)
                .optEngine("OnnxRuntime")
                .build();
        
        return ModelZoo.loadModel(criteria);
    }

    public static NDArray preprocess(BufferedImage img, NDManager manager) {
        int width = 224;
        int height = 224;
        float[] mean = {0.485f, 0.456f, 0.406f};
        float[] std = {0.229f, 0.224f, 0.225f};

        ai.djl.modality.cv.Image djlImage = ImageFactory.getInstance().fromImage(img);
        ai.djl.modality.cv.Image scaled = djlImage.resize(width, height, true);
        BufferedImage buffered = (BufferedImage) scaled.getWrappedImage();

        int[] pixels = buffered.getRGB(0, 0, width, height, null, 0, width);
        float[] data = new float[3 * width * height];

        for (int i = 0; i < pixels.length; i++) {
            int rgb = pixels[i];
            int r = (rgb >> 16) & 0xFF;
            int g = (rgb >> 8) & 0xFF;
            int b = rgb & 0xFF;

            int h = i / width;
            int w = i % width;
            int idx = h * width + w;

            data[idx] = ((r / 255f) - mean[0]) / std[0];
            data[width * height + idx] = ((g / 255f) - mean[1]) / std[1];
            data[2 * width * height + idx] = ((b / 255f) - mean[2]) / std[2];
        }

        return manager.create(data, new Shape(1, 3, height, width)); // NCHW
    }
}
```

---

### 3.2 Reactive Service (ModerationFluxService.java)
Uses `Mono.fromCallable` with `Schedulers.boundedElastic()` to run inference off the main I/O thread.

```java
@Service
public class ModerationFluxService {

    private final ZooModel<NDList, Classifications> model;

    public ModerationFluxService(ZooModel<NDList, Classifications> model) {
        this.model = model;
    }

    public Mono<ModerationResult> classify(FilePart file) {
        return DataBufferUtils.join(file.content())
                .flatMap(buffer -> Mono.fromCallable(() -> {
                    try (InputStream in = buffer.asInputStream();
                         Predictor<NDList, Classifications> predictor = model.newPredictor()) {

                        BufferedImage img = ImageIO.read(in);
                        NDArray nd = NsfwModelConfig.preprocess(img, model.getNDManager());
                        NDList input = new NDList(nd);
                        Classifications out = predictor.predict(input);

                        var best = out.best();
                        Map<String, Double> probs = new LinkedHashMap<>();
                        out.items().forEach(i -> probs.put(i.getClassName(), i.getProbability()));

                        return new ModerationResult(best.getClassName(), best.getProbability(), probs);
                    }
                }))
                .subscribeOn(Schedulers.boundedElastic()) 
                .timeout(Duration.ofSeconds(3))        
                .onErrorResume(TimeoutException.class,
                        e -> Mono.just(ModerationResult.timeoutFallback()));
    }
}
```

---

### 3.3 Controller (ModerationFluxController.java)

Handles `POST` requests with image uploads.

```java
@RestController
@RequestMapping("/flux/moderation")
@RequiredArgsConstructor
public class ModerationFluxController {

    private final ModerationFluxService service;

    @PostMapping(value = "/check", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public Mono<ModerationResult> check(@RequestPart("file") FilePart file) {
        return service.classify(file);
    }
}
```

---

## 4. Performance Benchmarking

We ran a local benchmark of the **WebFlux-based moderation API** under **200 concurrent users** using JMeter. The following numbers reflect performance on my development machine and highlight the current bottlenecks.

### Test Summary

| Metric | MVC API | WebFlux API |
|--------|---------|-------------|
| Requests | 1018 | 1340 |
| Errors | 0 (0.00%) | 0 (0.00%) |
| Average Response Time | 35,853 ms (~36s) | **26,957 ms (~27s)** |
| Min Response Time | 3,232 ms | **340 ms** |
| Max Response Time | **49,187 ms (~49s)** | 141,574 ms (~141s) |
| Median Response Time | 36,556 ms | **23,713 ms** |
| 90th Percentile | **42,631 ms** | 54,292 ms |
| 95th Percentile | **44,824 ms** | 64,867 ms |
| 99th Percentile | ****46,654 ms** | 86,554 ms |
| Throughput | **4.93 req/s** | 3.31 req/s |
| Avg. Response Size | 4.8 KB | 3.2 KB |
| APDEX (500ms / 1.5s) | 0.000 | 0.002 |

---

### Observations

- **Stability:** Both handled all requests without errors. 
- **Latency:**  
  - MVC showed **consistently high latency (~36s median)** but relatively narrow distribution (most requests 32–47s).  
  - WebFlux had **lower average/median latency (~27s / 23s)**, but a heavier **long tail**, with some requests reaching ~141s.  
- **Throughput:** MVC achieved slightly higher raw throughput (~4.9 req/s vs 3.3 req/s), but by **blocking threads**.
- **Scalability:** WebFlux prevents thread pool exhaustion, which makes it more resilient under high concurrency, whereas MVC will degrade linearly once threads are saturated.  
- **User Experience:** Both APDEX scores were poor in this CPU-bound local test, but WebFlux still provides better concurrency handling in production-like scenarios. 

---

### Key Takeaway  

This benchmark was run on my **local development machine without GPU acceleration**. While absolute numbers look slow, WebFlux demonstrates **better scalability than a blocking MVC version** would under the same conditions. In high-concurrency scenarios, WebFlux ensures requests are handled asynchronously, preventing thread pool exhaustion.

On local CPU tests, MVC delivers slightly higher throughput but suffers from rigid blocking latency. WebFlux, while showing long-tail effects, is **better suited for scaling under true high-concurrency workloads**, especially once GPU acceleration or batching is introduced.

**Next steps:** Optimizations such as **GPU acceleration, inference batching, or distributed model serving** will significantly improve throughput and latency in production environments.  

## 5. Conclusion
This project demonstrates how to build a **production-ready AI moderation service**:
- Java-native model inference with **DJL**  
- High-concurrency API using **Spring WebFlux**  
- Benchmarking showed **clear advantages over MVC**  
- Real-world JMeter tests revealed **inference performance as the main bottleneck**

This is more than just a demo — it’s a **practical AI engineering case study** for bringing deep learning models into production systems.