---
layout: post
title: "Python Inference Service with FastAPI and gRPC – REST vs RPC Deployment Guide"
date: 2025-08-16
categories: ai
published: true
description: "Step-by-step tutorial on building a Python inference service using FastAPI (HTTP) and gRPC, comparing REST vs RPC performance."
keywords: ["Python inference service", "FastAPI tutorial", "gRPC tutorial", "REST vs gRPC", "AI model deployment", "machine learning API"]
---

# Implementing a Python Inference Service with HTTP (FastAPI) and RPC (gRPC) Interfaces

## Introduction
When deploying AI models in production, we often face a choice:

**Should we expose the service via HTTP (REST/JSON) or RPC (gRPC)?**

This tutorial walks through building a **Python inference service** with both **FastAPI (HTTP)** and **gRPC**, then compares their performance and usability.

In modern AI applications, it’s common to deploy machine learning models as services so that different systems and programming languages can use them easily. For example, a Java backend system might need to use a PyTorch-trained model. Deploying the model in a Python service and exposing an API allows the Java system to send input data and receive predictions.

In this project, we built a **Python inference service** that supports **two interface types**:
1. **HTTP Interface** using FastAPI (supports JSON and multipart/form-data requests)
2. **gRPC Interface** using Protocol Buffers (efficient binary RPC calls)

Finally it provides a hands-on guide for machine learning API deployment using Python, showing how to serve AI models with both REST (FastAPI) and gRPC for production-grade inference services.

---

## Project Setup: Loading ResNet18 and API Interfaces
- **Load the ResNet18 model from local weights** (avoiding online download and SSL issues). 
- Accept an image from the client, perform preprocessing, run inference, and return the top-K classification results.
- Implement both **HTTP (FastAPI)** and **gRPC** interfaces.

---

## Technology Stack for Python Inference Service (FastAPI + gRPC)

| Technology | Purpose | Why Chosen |
|------------|---------|------------|
| **PyTorch / torchvision** | Model loading & inference | Industry-standard deep learning framework with torchvision for pretrained models and transforms. |
| **FastAPI** | HTTP service framework | Asynchronous, high-performance web framework with automatic OpenAPI documentation. |
| **gRPC** | High-performance RPC | Compact binary protocol, HTTP/2, language-neutral, ideal for low-latency microservices. |
| **Pillow (PIL)** | Image processing | Easy to load and manipulate images before feeding them to the model. |
| **python-multipart** | HTTP file uploads | Required for multipart/form-data in FastAPI. |
| **NumPy (<2)** | Tensor operations compatibility | Ensures compatibility with PyTorch 2.x, which is compiled against NumPy 1.x. |

Project Structure Below:
![Project Structure for Python Inference Service with FastAPI and gRPC](/assets/images/2025_08_16_project_structure.png)

---

## Building the HTTP Inference API with FastAPI

### API Design

- **POST `/predict/file`**: Accepts an uploaded image file via `multipart/form-data`.
- **POST `/predict/b64`**: Accepts a base64-encoded image string (useful for mobile or cross-language scenarios).

### Core Inference Code (`fastapi_app/infer.py`)
```python
import torch
from torchvision import models
from PIL import Image
from typing import List, Tuple

_MODEL = None
_TRANSFORM = None

def _load_model():
    global _MODEL, _TRANSFORM
    if _MODEL is None:
        weights_path = "/path/to/resnet18-f37072fd.pth"
        _MODEL = models.resnet18()
        state_dict = torch.load(weights_path, map_location=torch.device("cpu"))
        _MODEL.load_state_dict(state_dict, strict=False)
        _MODEL.eval()

        default_weights = models.ResNet18_Weights.DEFAULT
        _TRANSFORM = default_weights.transforms()
    return _MODEL, _TRANSFORM

def preprocess(img: Image.Image) -> torch.Tensor:
    _, transform = _load_model()
    return transform(img).unsqueeze(0)

@torch.inference_mode()
def predict(img: Image.Image, topk: int = 5) -> List[Tuple[str, float]]:
    model, _ = _load_model()
    x = preprocess(img)
    logits = model(x)
    probs = torch.softmax(logits, dim=1)[0]
    topk_probs, topk_idxs = torch.topk(probs, k=topk)
    classes = models.ResNet18_Weights.DEFAULT.meta["categories"]
    return [(classes[i], float(topk_probs[j])) for j, i in enumerate(topk_idxs)]
```

### FastAPI App Entrypoint (`fastapi_app/app.py`)
```python
from fastapi import FastAPI, UploadFile, File
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from infer import predict
from PIL import Image
import base64, io

app = FastAPI(title="ResNet18 Inference Service")

class B64ImageRequest(BaseModel):
    image_base64: str
    topk: int = 5

@app.post("/predict/file")
async def predict_file(file: UploadFile = File(...), topk: int = 5):
    img = Image.open(file.file).convert("RGB")
    res = predict(img, topk)
    return JSONResponse({"topk": [{"label": l, "prob": p} for l, p in res]})

@app.post("/predict/b64")
async def predict_b64(req: B64ImageRequest):
    img = Image.open(io.BytesIO(base64.b64decode(req.image_base64))).convert("RGB")
    res = predict(img, req.topk)
    return JSONResponse({"topk": [{"label": l, "prob": p} for l, p in res]})
```

### Testing with `curl`
It is easy to test using `curl` command.

```bash
curl -X POST "http://127.0.0.1:8000/predict/file?topk=5"   
	-F "file=@/path/to/image.jpg"
```

---

## Implementing the gRPC Inference API with Protocol Buffers

### Protocol Buffers Definition (`grpc_app/protos/classify.proto`)

```proto
syntax = "proto3";

package classify;

service Classifier {
  rpc Predict (ImageRequest) returns (PredictionResponse) {}
}

message ImageRequest {
  bytes image = 1;
  int32 topk = 2;
}

message Prediction {
  string label = 1;
  float prob = 2;
}

message PredictionResponse {
  repeated Prediction topk = 1;
}
```

### Inference Logic (`grpc_app/infer.py`)

*Same model loading logic as FastAPI, using local weights*

### gRPC Server (`grpc_app/server.py`)
```python
import grpc
from concurrent import futures
import classify_pb2, classify_pb2_grpc
from infer import predict_bytes

class ClassifierServicer(classify_pb2_grpc.ClassifierServicer):
    def Predict(self, request, context):
        preds = predict_bytes(request.image, topk=request.topk or 5)
        return classify_pb2.PredictionResponse(
            topk=[classify_pb2.Prediction(label=l, prob=p) for l, p in preds]
        )

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=4))
    classify_pb2_grpc.add_ClassifierServicer_to_server(ClassifierServicer(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```

### Testing gRPC Client
For testing, we implement a gRPC client.
```python
import grpc, classify_pb2, classify_pb2_grpc

with open("image.jpg", "rb") as f:
    data = f.read()

channel = grpc.insecure_channel("127.0.0.1:50051")
stub = classify_pb2_grpc.ClassifierStub(channel)
resp = stub.Predict(classify_pb2.ImageRequest(image=data, topk=5))
print(resp)
```

---

## HTTP vs gRPC Performance Comparison and Deployment Takeaways

| Feature | HTTP (FastAPI) | gRPC |
|---------|----------------|------|
| Protocol | JSON / multipart | Protobuf (binary) |
| Performance | Slower (text encoding, larger payloads) | Faster (compact binary, HTTP/2) |
| Debugging | Easier (curl, Postman) | Harder (need generated stubs) |
| Cross-language | Excellent | Excellent |
| Complexity | Low | Medium (requires .proto, codegen) |

- **HTTP** is ideal for quick integration, easy debugging, and scenarios where clients can easily send JSON or form-data.
- **gRPC** is better for low-latency, high-throughput systems where performance matters.

Using local weights avoids external dependencies, improves startup time, and makes deployments more stable.

In production, the choice depends on the trade-off between **ease of integration** and **performance requirements**. 

---

## Related Resources

- Related post: [Deploying ResNet Models with Java DJL – From Zero to Hero](/ai/2025/08/14/deploy-resnet-java-djl-tutorial.html)
  Learn how to integrate and deploy deep learning models in **Java** using DJL.

- Related post: [PyTorch Housing Price Prediction Demo (Multi-feature)](/ai/2025/08/07/pytorch-housing-price-prediction-multi-feature.html)
  Hands-on guide to applying linear regression in PyTorch with multiple features.

- [FastAPI Official Documentation](https://fastapi.tiangolo.com/)  
  Comprehensive docs for building high-performance Python APIs.

- [gRPC Documentation](https://grpc.io/docs/)
  Learn more about gRPC core concepts, Protocol Buffers, and API design best practices.