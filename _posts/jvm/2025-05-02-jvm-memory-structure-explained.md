---
layout: post
title: "Java Virtual Machine (JVM) Memory Structure Explained"
date: 2025-05-02
categories: jvm
published: true
description: "A detailed guide to JVM runtime memory areas: PC register, JVM stack, native stack, heap, method area, and runtime constant pool. Includes examples of OOM and StackOverflowError."
---

# Java Virtual Machine (JVM) Memory

## Introduction
Simply put, Java source code is compiled by the Java compiler into **bytecode**. 

The JVM loads this compiled bytecode and **executes it**. 

This forms the basis of Java's well-known **"write once, run anywhere"** capability, enabling platform independence by allowing the same compiled bytecode to be executed on any system equipped with a compliant Java Virtual Machine (JVM).

## JVM Runtime Memory Areas
The memory structure of the JVM is divided into ***FIVE*** main runtime memory areas.

### 1. Program Counter (PC) Register
- Each thread has its own PC register.
- It holds the address of the currently executing bytecode instruction.
- If the thread is executing a native method, the value is undefined.

### 2. Java Virtual Machine Stack
- Each thread has its own private instance.
- Stores **stack frames**, which contain:
    - Local variables
    - Operand stack
    - Dynamic linking info
    - Return addresses
- Used during method invocation and execution.

When the required call stack depth exceeds the maximum allowed by the JVM, a `StackOverflowError` will be thrown.

This typically occurs when a recursive function lacks a proper termination condition. One example as shown below.
```java
public class JVMStackOverflowError {
    public void recursiveFunc() {
        recursiveFunc(); // No termination condition
    }

    public static void main(String[] args) {
        JVMStackOverflowError example = new JVMStackOverflowError();
        example.recursiveFunc(); // Triggers StackOverflowError
    }
}
```
In the example, the `recursiveFunc()` method continuously calls itself, resulting in stack frame accumulation until the JVM can no longer allocate more stack memory for the thread. This eventually causes a `StackOverflowError`.

### 3. Native Method Stack
- Similar to the JVM stack, but used for native (non-Java) methods.
- Not all JVMs separate this; for example, HotSpot combines it with the JVM stack.

The Java Virtual Machine stack is used to store frames. The Java Virtual Machine is also free to use one or more stacks to support native methods.

### 4. Heap
- Shared across all threads.
- All class instances and arrays are allocated here.
- Main area for Garbage Collection (GC). Typically divided into Young Generation (Eden, Survivor spaces), Old Generation and (Optional) Humongous regions in G1 GC.

When new objects are continuously created and all of them remain reachable (i.e., cannot be reclaimed by the garbage collector), the heap may eventually run out of space, resulting in an `OutOfMemoryError`. 
```java
import java.util.LinkedList;
import java.util.List;

public class JVMOutOfMemoryError {
    public static void main(String[] args) {
        List<Object> list = new LinkedList<>();
        while (true) {
            list.add(new Object()); // Keeps consuming heap memory
        }
    }
}
```
In above code, a `LinkedList` keeps growing as new Object instances are continuously added to it. Since all added objects are still referenced by the list, they cannot be garbage collected, which eventually leads to a heap overflow and triggers an `OutOfMemoryError`.

### 5. Method Area
- Shared among all threads.
- Stores class metadata, including:
    - Class structures (fields, methods)
    - Runtime constant pool
    - Static variables
    - Method bytecodes

In Java 8 and later, implemented as Metaspace, which uses native memory instead of heap.

### 6Plus. Runtime Constant Pool
After introducing the five parts of JVM memory, I would like to highlight the Runtime Constant Pool separately. Although it is technically a part of the Method Area, it plays a critical role during class loading and bytecode execution.

- A per-class runtime representation of the constant pool defined in the .class file.
- Stores literals, symbolic references, method handles, etc.

## Further Topic: Direct Memory Usage in Java

Direct memory may also be frequently used during program execution. Starting from JDK 1.4, the introduction of the New Input/Output (NIO) package brought a new I/O mechanism based on Channels and Buffers. 

This allows Java programs to allocate off-heap memory directly using native libraries, and then operate on it through a `DirectByteBuffer` object, **which resides in the Java heap and serves as a reference to that off-heap memory.**

### One Example
The `ByteBuffer.allocateDirect()` method is called repeatedly in a loop to allocate 1MB of direct memory each time. The returned ByteBuffer instances (which are technically DirectByteBuffer objects) are stored in a list without being released, eventually triggering an `OutOfMemoryError: Direct buffer memory`.
```java
import java.nio.ByteBuffer;
import java.util.LinkedList;
import java.util.List;

public class DirectMemoryOOMError {
    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        List<ByteBuffer> list = new LinkedList<>();
        while (true) {
            ByteBuffer byteBuffer = ByteBuffer.allocateDirect(_1MB);
            list.add(byteBuffer);
        }
    }
}
```
