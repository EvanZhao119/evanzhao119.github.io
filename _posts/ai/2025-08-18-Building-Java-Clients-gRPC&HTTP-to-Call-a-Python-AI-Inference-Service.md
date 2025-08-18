---
layout: post
title: "Building Java Clients (gRPC & HTTP) to Call a Python AI Inference Service"
date: 2025-08-18
categories: ai
published: true
---

# Building Java Clients (gRPC & HTTP) to Call a Python AI Inference Service
In this post, I will built **two Java clients** —one using **gRPC** and one using **HTTP**— to call a **[Python AI inference service](https://evanzhao119.github.io/ai/2025/08/16/Implementing-a-Python-Inference-Service-with-HTTP-FastAPI-and-RPC-gRPC-Interfaces.html)** running a ResNet18 model. 

The Python server provides **two endpoints**:
- A **gRPC service** for high-performance, structured communication
- An **HTTP API** for compatibility with tools like `curl` or browser requests

Finally, the article will show a working Java project that can talk to the Python server using **either RPC or HTTP**.

---

## 1. Project Overview

- Write a **Java client** to send image data to the server and get **Top-K classification predictions**.
- Support both **gRPC calls** and **HTTP requests**.

Project Structure Below
![Result Examples](/assets/images/2025_08_18_project_structure.png)

---

## 2. Java Project Setup - Maven Dependencies
Key libraries:
- `grpc-netty-shaded`, `grpc-protobuf`, `grpc-stub`
- `protobuf-java`
- `okhttp` (HTTP client)
- `jackson-databind` (JSON parsing)

Example snippet from `pom.xml`:

```xml
<dependencies>
    <!-- gRPC -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.63.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.63.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.63.0</version>
    </dependency>

    <!-- Protobuf runtime -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>3.25.3</version>
    </dependency>

    <!-- HTTP client & JSON -->
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
        <version>4.12.0</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.1</version>
    </dependency>
</dependencies>
```

---

## 3. Implementing the gRPC Client
**`Client.java`** reads an image file, sends it to the Python gRPC server, and prints predictions.
```java
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("127.0.0.1", 50051)
    .usePlaintext()
    .build();

ClassifierGrpc.ClassifierBlockingStub stub = ClassifierGrpc.newBlockingStub(channel);

ImageRequest request = ImageRequest.newBuilder()
    .setImage(ByteString.copyFrom(Files.readAllBytes(Paths.get(imagePath))))
    .setTopk(topk)
    .build();

PredictionResponse resp = stub.predict(request);
resp.getTopkList().forEach(p ->
    System.out.printf("%-30s %.4f%n", p.getLabel(), p.getProb())
);
```

---

## 4. Implementing the HTTP Client
Our `FastAPI` endpoint expects:
- **POST** to `/predict/file?topk=5`
- `multipart/form-data` with `file=@image`

**`HttpTester.java`**:
```java
public void predictMultipart(String imagePath, int topk) throws IOException {
    File img = new File(imagePath);
    RequestBody fileBody = RequestBody.create(img, MediaType.parse("image/*"));

    MultipartBody requestBody = new MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("file", img.getName(), fileBody)
            .build();

    String urlWithParam = String.format("%s?topk=%d", endpoint, topk);

    Request request = new Request.Builder()
            .url(urlWithParam)
            .post(requestBody)
            .addHeader("accept", "application/json")
            .build();

    try (Response resp = client.newCall(request).execute()) {
        String json = resp.body().string();
        System.out.println(json);
    }
}
```

---

## 5. Results
This project implements HTTP and gRPC clients against the same Python inference server. **gRPC** delivered faster responses with smaller payloads, while **HTTP** was simpler to test and integrate during development. With gRPC and HTTP clients in Java, we can talk to the same AI inference server in whichever way makes the most sense for our scenario.