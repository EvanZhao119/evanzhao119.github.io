---
layout: post
title: "djl-resnet + JNI: a Practical Guide to Faster Image Preprocessing"
date: 2025-08-21
categories: ai
published: true
---

# djl-resnet + JNI: a Practical Guide to Faster Image Preprocessing
Try to move image preprocessing (**decode → resize (short side 256) → center crop 224 → CHW/float32 + normalize**) from [Java](https://evanzhao119.github.io/ai/2025/08/14/Deploying-ResNet-Models-with-Java-DJL-From-Zero-to-Hero.html) into **C/C++** via **JNI**, return a `DirectByteBuffer`, and feed it straight into DJL (Deep Java Library). 
We need to download **3rd‑party files** in advance - `stb_image.h` and `stb_image_resize2.h`.
```bash
curl -fL --retry 3 -o native/third_party/stb_image.h https://raw.githubusercontent.com/nothings/stb/master/stb_image.h
curl -fL --retry 3 -o native/third_party/stb_image_resize2.h https://raw.githubusercontent.com/nothings/stb/master/stb_image_resize2.h
```

---

## What We’ll Build
We’ll push the image preprocessing pipeline down to C/C++. Java will just call a **JNI** function and receive a `DirectByteBuffer` of `float32` in **CHW** layout. DJL will wrap it as an NDArray and run the model as usual. It keeps accuracy the same but can reduce end‑to‑end latency and CPU overhead.

---

## Project Layout
```
djl-resnet/
├─ assets/1.png 2.png 3.png
├─ src/main/java/org/estech/
│  ├─ App.java                     # Toggle between JNI and the stock DJL translator
│  ├─ NativeImageOps.java          # JNI shell (declares native methods)
│  └─ ResNetJniTranslator.java     # JNI translator (input: byte[]; output: CHW without batch dim)
├─ src/main/resources/models/resnet18/
│  ├─ traced_resnet18.pt
│  └─ synset.txt
├─ native/
│  ├─ CMakeLists.txt
│  ├─ imageops.cpp                 # Uses stb_image + stb_image_resize2
│  └─ third_party/
│     ├─ stb_image.h
│     └─ stb_image_resize2.h
└─ pom.xml
```
Final project layout:
![Project Layout](/assets/images/2025_08_21_project_layout.png)

---

## Step‑by‑Step Build

### 1) JNI shell (Java)
`src/main/java/org/estech/NativeImageOps.java`
```java
package org.estech;

import java.nio.ByteBuffer;

public final class NativeImageOps {
    static { System.loadLibrary("imageops"); } // pass -Djava.library.path at runtime
    public static native ByteBuffer preprocessToCHW(
            ByteBuffer encoded, int outW, int outH,
            float m0, float m1, float m2,
            float s0, float s1, float s2);
    public static native void freeBuffer(ByteBuffer buf);
}
```

### 2) Generate the JNI header
```bash
mvn -q -DskipTests package
# create header in native/ and keep .class in target/classes
javac -h native -cp target/classes -d target/classes src/main/java/org/estech/NativeImageOps.java
# output: native/org_estech_NativeImageOps.h
```

### 3) Native implementation (C++)
`native/imageops.cpp` (key points):
- `#define STB_IMAGE_IMPLEMENTATION` + `#include "third_party/stb_image.h"`
- `#define STB_IMAGE_RESIZE2_IMPLEMENTATION` + `#include "third_party/stb_image_resize2.h"`  
- Implement `Java_org_estech_NativeImageOps_preprocessToCHW`
    1. read encoded image pointer from a direct `ByteBuffer` 
    2. `stbi_load_from_memory` → RGB (HWC/uint8) 
    3. **`stbir_resize_uint8_linear`** to resize the **short side** to 256 (stride = width * channels) 
    4. Center-crop to 224×224
    5. HWC (u8) → CHW (float32) + normalize (ImageNet mean/std) 
    6. `NewDirectByteBuffer(chw, bytes)` to return it to Java
- Implement `Java_org_estech_NativeImageOps_freeBuffer` to free the native allocation.

```cpp
JNIEXPORT jobject JNICALL Java_org_estech_NativeImageOps_preprocessToCHW
  (JNIEnv* env, jclass,
   jobject encodedImage, jint outW, jint outH,
   jfloat mean0, jfloat mean1, jfloat mean2,
   jfloat std0,  jfloat std1,  jfloat std2)
{
    if (!encodedImage) return nullptr;
    auto* in_ptr = (unsigned char*) env->GetDirectBufferAddress(encodedImage);
    jlong in_len = env->GetDirectBufferCapacity(encodedImage);
    if (!in_ptr || in_len <= 0) return nullptr;

    int w, h, c;
    unsigned char* hwc = stbi_load_from_memory(in_ptr, (int)in_len, &w, &h, &c, 3);
    if (!hwc) return nullptr;

    // resize the short side to 256
    const int short_target = 256;
    double scale = (w < h) ? (double)short_target / w : (double)short_target / h;
    int rw = (int)std::round(w * scale);
    int rh = (int)std::round(h * scale);

    unsigned char* resized256 = (unsigned char*) std::malloc((size_t)rw * rh * 3);
    if (!resized256) { stbi_image_free(hwc); return nullptr; }

    if (!stbir_resize_uint8_linear(
            hwc, w, h, src_stride,
            resized256, rw, rh, dst_stride,
            STBIR_RGB))
    {
        std::free(resized256);
        stbi_image_free(hwc);
        return nullptr;
    }

    stbi_image_free(hwc);

    // Center-crop to 224×224
    unsigned char* cropped = (unsigned char*) std::malloc((size_t)outW * outH * 3);
    if (!cropped) { std::free(resized256); return nullptr; }
    center_crop_rgb_u8(resized256, rw, rh, cropped, outW, outH);
    std::free(resized256);

    // HWC (u8) → CHW (float32) + normalize
    size_t N = (size_t)outW * outH * 3;
    float* chw = (float*) std::malloc(N * sizeof(float));
    if (!chw) { std::free(cropped); return nullptr; }

    const float mean[3] = {mean0, mean1, mean2};
    const float stdv[3] = {std0, std1, std2};
    int hw = outW * outH;

    for (int y = 0; y < outH; ++y) {
        for (int x = 0; x < outW; ++x) {
            int i = (y * outW + x) * 3;
            // RGB -> CHW，/255 再 normalize
            float r = (cropped[i    ] / 255.0f - mean[0]) / stdv[0];
            float g = (cropped[i + 1] / 255.0f - mean[1]) / stdv[1];
            float b = (cropped[i + 2] / 255.0f - mean[2]) / stdv[2];
            int idx = y * outW + x;
            chw[idx]        = r;     // C=0
            chw[idx + hw]   = g;     // C=1
            chw[idx + 2*hw] = b;     // C=2
        }
    }
    std::free(cropped);

    return wrap_as_direct(env, chw, N * sizeof(float));
}
```

### 4) Build the dynamic library (macOS Intel)
`CMakeLists.txt` is the build recipe: where headers live, which sources to compile, that we want a **shared** library, and that we depend on JNI. CMake then creates Makefiles/Xcode projects and builds `libimageops.dylib`.

`native/CMakeLists.txt`:
```cmake
cmake_minimum_required(VERSION 3.15)
project(imageops)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

find_package(JNI REQUIRED)
include_directories(${JNI_INCLUDE_DIRS})
include_directories(${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/third_party)

add_library(imageops SHARED imageops.cpp)
```

Build:
```bash
cmake -S native -B native/build   -DJAVA_HOME="$JAVA_HOME"   -DCMAKE_OSX_ARCHITECTURES=x86_64   -DCMAKE_BUILD_TYPE=Release
cmake --build native/build --config Release
# result: native/build/libimageops.dylib
```

### 5) The JNI translator (no batch dim)
`src/main/java/org/estech/ResNetJniTranslator.java`:
```java
public class ResNetJniTranslator implements Translator<byte[], Classifications> {

    private static final int SIZE = 224;
    private static final float M0 = 0.485f, M1 = 0.456f, M2 = 0.406f;
    private static final float S0 = 0.229f, S1 = 0.224f, S2 = 0.225f;

    @Override
    public NDList processInput(TranslatorContext ctx, byte[] input) throws Exception {
        ByteBuffer encoded = ByteBuffer.allocateDirect(input.length);
        encoded.put(input).flip();

        long t0 = System.nanoTime();
        ByteBuffer chw = NativeImageOps.preprocessToCHW(
                encoded, SIZE, SIZE, M0, M1, M2, S0, S1, S2);
        long t1 = System.nanoTime();

        if (chw == null) {
            throw new IOException("JNI preprocessToCHW returned null");
        }
        System.out.printf("[JNI] preprocess: %.2f ms, bytes=%d%n",
                (t1 - t0) / 1e6, chw.remaining());

        NDManager manager = ctx.getNDManager();
        chw.rewind();
        NDArray x = manager.create(chw, new Shape(3, SIZE, SIZE), DataType.FLOAT32);

        NativeImageOps.freeBuffer(chw);

        return new NDList(x);
    }

    @Override
    public Classifications processOutput(TranslatorContext ctx, NDList list) throws Exception {
        NDArray logits = list.singletonOrThrow();

        NDArray probs;
        Shape sh = logits.getShape();
        if (sh.dimension() == 2 && sh.get(0) == 1) {
            probs = logits.softmax(1).squeeze(0);   // [1,1000] -> [1000]
        } else if (sh.dimension() == 1) {
            probs = logits.softmax(0);              // [1000]
        } else {
            int last = (int) sh.dimension() - 1;
            probs = logits.softmax(last);
            if (probs.getShape().dimension() > 1) {
                probs = probs.flatten();           
            }
        }

        List<String> synset = loadSynset(ctx, (int) probs.getShape().get(0));
        return new Classifications(synset, probs);
    }

    @Override
    public Batchifier getBatchifier() {
        return Batchifier.STACK;
    }

    private List<String> loadSynset(TranslatorContext ctx, int nClasses) throws IOException {
        try {
            List<String> labels = ctx.getModel().getArtifact("synset.txt", Utils::readLines);
            if (labels != null && !labels.isEmpty()) return labels;
        } catch (Exception ignore) { /* fallthrough */ }

        Path local = Paths.get("path/to/your/synset.txt");
        if (Files.exists(local)) {
            return Files.readAllLines(local);
        }

        try (InputStream is = ResNetJniTranslator.class.getResourceAsStream("path/to/your.txt")) {
            if (is != null) {
                return Utils.readLines(is);
            }
        }

        List<String> fallback = new ArrayList<>(nClasses);
        for (int i = 0; i < nClasses; i++) fallback.add(String.valueOf(i));
        return fallback;
    }
}
```

### 6) Switch between JNI and stock DJL in `App`
- In IntelliJ VM options:
```
-Djava.library.path=$PROJECT_DIR$/native/build -DuseJni=true
```
- Or via CLI:
```bash
-Dexec.jvmArgs="-Djava.library.path=$(pwd)/native/build"
```

`App.java`(key lines):
```java
if (useJni) {
    // JNI Translator 
    byte[] imgBytes = Files.readAllBytes(imgPath);

    Translator<byte[], Classifications> translator = new ResNetJniTranslator();

    Criteria<byte[], Classifications> criteria = Criteria.builder()
        .optEngine("PyTorch")
        .optApplication(Application.CV.IMAGE_CLASSIFICATION)
        .setTypes(byte[].class, Classifications.class)
        .optModelPath(modelDir)
        .optModelName("traced_resnet18")
        .optTranslator(translator)
        .optProgress(new ProgressBar())
        .build();

    try (ZooModel<byte[], Classifications> model = ModelZoo.loadModel(criteria);
        Predictor<byte[], Classifications> predictor = model.newPredictor()) {

        long t0 = System.nanoTime();
        Classifications result = predictor.predict(imgBytes);
        long t1 = System.nanoTime();

        System.out.printf("Predict total (JNI): %.2f ms%n", (t1 - t0) / 1e6);
        System.out.println("== Top-5 ==");
        result.topK(5).forEach(c ->
            System.out.printf("%-25s %.5f%n", c.getClassName(), c.getProbability()));
    }
} else {
    // Java DJL Translator
    Image img = ImageFactory.getInstance().fromFile(imgPath);

    Translator<Image, Classifications> translator =
        ai.djl.modality.cv.translator.ImageClassificationTranslator.builder()
            .addTransform(new Resize(256))
            .addTransform(new CenterCrop(224, 224))
            .addTransform(new ToTensor())
            .addTransform(new Normalize(
                new float[]{0.485f, 0.456f, 0.406f},
                new float[]{0.229f, 0.224f, 0.225f}))
            .optApplySoftmax(true)
            .optSynsetUrl(Paths.get("path/to/your/synset.txt")
            .toUri().toString())
            .build();

    Criteria<Image, Classifications> criteria = Criteria.builder()
            .optEngine("PyTorch")
            .optApplication(Application.CV.IMAGE_CLASSIFICATION)
            .setTypes(Image.class, Classifications.class)
            .optModelPath(modelDir)
            .optModelName("traced_resnet18")
            .optTranslator(translator)
            .optProgress(new ProgressBar())
            .build();

    try (ZooModel<Image, Classifications> model = ModelZoo.loadModel(criteria);
        Predictor<Image, Classifications> predictor = model.newPredictor()) {

        long t0 = System.nanoTime();
        Classifications result = predictor.predict(img);
        long t1 = System.nanoTime();

        System.out.printf("Predict total (DJL): %.2f ms%n", (t1 - t0) / 1e6);
        System.out.println("== Top-5 ==");
        result.topK(5).forEach(c ->
            System.out.printf("%-25s %.5f%n", c.getClassName(), c.getProbability()));
    }
}
```

---

## How the Pieces Fit Together (Java ↔ C++)
**Build time**
1. Java declares `native` methods in `NativeImageOps`.
2. `javac -h` generates `org_estech_NativeImageOps.h` with the exact JNI prototypes.
3. C++ includes that header and implements the functions.
4. CMake compiles `imageops.cpp` into `libimageops.dylib`.

**Runtime**
1. `System.loadLibrary("imageops")` maps to `libimageops.dylib` on macOS (Linux: `.so`, Windows: `.dll`).  
2. The JVM searches `-Djava.library.path` to load the library. 
3. The JVM looks up JNI symbols like `Java_org_estech_NativeImageOps_preprocessToCHW` and binds them to your `native` methods.  
4. The translator calls `NativeImageOps.preprocessToCHW(...)` → C++ preprocesses and returns a `DirectByteBuffer`.
5. Java wraps the buffer as an NDArray with shape `(3,224,224)`; `Batchifier.STACK` makes it `[N,3,224,224]` for inference.

---

## Experimental Results
### Stock DJL (Java‑side preprocessing)
```
# Image 1
Predict total (DJL): 356.75 ms
Top-1: giant panda (0.99993)

# Image 2
Predict total (DJL): 381.30 ms
Top-1: brown bear (0.99910)

# Image 3
Predict total (DJL): 417.40 ms
Top-1: tiger (0.77952)
```

### JNI preprocessing (C/C++)
```
# Image 1
[JNI] preprocess: 30.56 ms, bytes=602112
Predict total (JNI): 356.18 ms
Top-1: giant panda (0.99986)

# Image 2
[JNI] preprocess: 33.78 ms, bytes=602112
Predict total (JNI): 340.08 ms
Top-1: brown bear (0.99909)

# Image 3
[JNI] preprocess: 65.49 ms, bytes=602112
Predict total (JNI): 391.52 ms
Top-1: tiger (0.84426)
```

**Takeaways**
- **Accuracy**: Top‑1/Top‑5 match the stock DJL path (tiny probability differences are normal).  
- **Latency**: Overall **JNI ≤ DJL**. Images 2 and 3 are ~41 ms and ~26 ms faster end‑to‑end; image 1 is roughly the same.  
- **JNI preprocess time**: 30–65 ms per image (varies by image size/content; image 3 is heavier). For better efficiency, try batching multiple images in one JNI call.

---

## Appendix: Quick Q&A
- **What’s a Java “shell class”?**  
Answers: A class that only declares `native` methods. It’s the Java side of the interface; the body lives in C/C++.

- **Why does `javac -h` generate a C header?**  
Answers: The compiler reads your `native` signatures and emits the exact JNI prototypes (with the right mangled names) so C/C++ can implement them safely.
  
- **Why does `loadLibrary("imageops")` look for `libimageops.dylib`?** 
Answers: The JVM calls `System.mapLibraryName("imageops")`: macOS → `libimageops.dylib`, Linux → `libimageops.so`, Windows → `imageops.dll`.

- **What exactly does `CMakeLists.txt` do?**  
Answers: It’s the build recipe. It declares a **shared** library target, points to sources and headers, pulls in JNI include paths, and produces `libimageops.dylib`.
