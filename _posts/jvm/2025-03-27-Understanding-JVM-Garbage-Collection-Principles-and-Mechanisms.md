---
layout: post
title: "Understanding JVM Garbage Collection: Principles and Mechanisms"
date: 2025-03-27
categories: jvm
published: true
---

# Understanding JVM Garbage Collection: Principles and Mechanisms

According to the memory structure of the Java Virtual Machine (JVM), the **Program Counter**, **Java Virtual Machine Stack**, and **Native Method Stack** are **thread-private**. They are created when a thread is created and destroyed when the thread ends. Therefore, they do not require garbage collection, as their lifecycle is tied directly to the thread.

In contrast, the **Hea**p and the **Method Area** (also referred to as the **Metaspace** in modern JVMs) are shared among all threads. All object instances and arrays are allocated in the heap space, which is the **primary area collected by garbage collection (GC)**. 

## How Does the JVM Determine Whether an Object Can Be Reclaimed?

### 1. Reference Counting

This method keeps track of how many references each object has. When an object’s reference count drops to zero, it is considered unreachable and can be reclaimed.

However, this method has a significant drawback — it **cannot handle circular references**. 
For instance, if object A references object B and object B references A, both will have non-zero reference counts even if they are no longer accessible from anywhere else in the program.

> **Note:** Modern JVMs do **not** use reference counting as the primary garbage collection strategy due to this limitation.

### 2. Reachability Analysis (Graph-Based Approach)

The JVM uses **reachability analysis** to determine whether an object is still in use. It starts from a set of well-known objects called **GC Roots** and traverses the object graph along reference paths. Any object **not reachable** from the GC Roots is considered 'garbage' and is eligible for collection.

## What Are GC Roots?

**GC Roots** are special references that act as starting points for the garbage collector to explore object reachability.

### GC Roots include:
- **Local variables** in the virtual machine stack of active threads.
- **JNI references** in the native method stack.
- **Static variables** in the Method Area (Metaspace).
- **Constants** stored in the runtime constant pool (e.g., string literals).
- Objects referenced by **locks or monitors**, i.e., objects held by `synchronized` blocks.
- Internal references maintained by the JVM, such as:
    - **Class objects**, **class loaders**, **exception objects**,
    - **JMXBeans**, **JVMTI callbacks**, and
    - **JIT compiler code cache**.

Additionally, some garbage collectors (like G1 or ZGC) may designate **temporary GC Roots** during specific phases of garbage collection (e.g., Minor GC or incremental GC).

> **Note:** Starting with **Java 8**, the Method Area has been replaced by **Metaspace**, which resides in **native memory** rather than the JVM heap.

## Summary
- Thread-private memory areas (PC, VM Stack, Native Method Stack) do not require GC.
- The Heap and Metaspace are shared and require GC management.
- **Reachability analysis** is the modern approach used in the JVM to determine if an object is garbage.
- GC Roots serve as entry points to identify all live objects.

## A Final Note: Garbage Collection in the Method Area

Compared to the heap, **garbage collection in the method area (also known as Metaspace in modern JVMs)** is less efficient. This is primarily because the method area contains class metadata, which seldom changes and is rarely discarded once loaded.

With the growing use of **hot-swapping, reflection, and dynamic proxies**, the JVM is increasingly expected to support garbage collection in the method area to help free up memory occupied by obsolete class metadata.

> **Note:** According to the JVM specification, **implementations are not required to support garbage collection or compaction in the method area**. As shown in the excerpt below:
>
> > *"Although the method area is logically part of the heap, simple implementations may choose not to either garbage collect or compact it."*
>
> This means it is left to the discretion of the JVM implementation whether to manage memory reclamation in the method area.

In real-world scenarios, modern JVMs like **HotSpot** provide **conditional support for class unloading through garbage collection**, particularly in situations involving **class redefinition**, **runtime-generated classes**, or frameworks such as **Spring** and **OSGi** that dynamically manipulate the classloader.
