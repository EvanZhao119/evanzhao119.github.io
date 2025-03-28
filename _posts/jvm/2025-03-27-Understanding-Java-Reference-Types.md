---
layout: post
title: "Understanding Java Reference Types: From Strong to Phantom References and Their Practical Applications"
date: 2025-03-27
categories: jvm
published: true
---

# Understanding Java Reference Types: From Strong to Phantom References and Their Practical Applications

From JDK 1.2, Java introduced different types of references: **Strong Reference**, **Soft Reference**, **Weak Reference**, and **Phantom Reference**.

>The hierarchy of references is as follows:<br>
**Strong Reference > Soft Reference > Weak Reference > Phantom Reference**

In practice, different types of references offer a wide range of functionalities. For example:
- **Netty** framework introduced a reference counting mechanism for message management by weak references starting with version 4.x.
- The `ThreadLocal` implementation in the JDK stores its references as weak references in a map associated with the Thread object.
- Perhaps we can implement our own caching mechanism, where objects are reclaimed when memory is running low.

## 1. Strong Reference
A **strong reference** is the traditional reference assignment in Java. As long as the reference exists, the garbage collector will not reclaim the object.
```java
/**-------Strong Reference--------**/
int[] t1 = new int[10240000];
System.gc();// Full GC will not reclaim the objects
```

## 2. Soft Reference
A **soft reference** is used to refer to objects that should only be reclaimed when the JVM is low on memory. The object is eligible for garbage collection before an OutOfMemoryError occurs.
```java
/**-------Soft Reference-----------**/
SoftReference t2 = new SoftReference<>(new int[10240000]);
System.gc();// The object referenced by the soft reference remains intact
```

Next, the code is modified to continuously allocate large objects. The soft reference objects will be reclaimed only before the memory overflows (when allocation fails).
```java
/**-------Soft Reference-----------**/
SoftReference t2 = new SoftReference<>(new int[512000000]);
SoftReference t3 = new SoftReference<>(new int[512000000]);
System.gc();
```

## 3. Weak Reference
A **weak reference** allows the referenced object to live only until the next garbage collection. Regardless of whether the system has enough memory, the object will be reclaimed when the garbage collection occurs.
```java
/**-------Weak Reference-----------**/
WeakReference t3 = new WeakReference<>(new int[10240000]);
System.gc();//The weak reference object will be reclaimed after the garbage collection process.
```

## 4. Phantom Reference
A **phantom reference** does not affect the lifetime of the referenced object. It cannot be used to retrieve the object itself. The sole purpose of a phantom reference is to notify when the object is about to be garbage collected.
