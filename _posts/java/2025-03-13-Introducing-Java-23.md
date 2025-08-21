---
layout: post
title: "What’s New in Java 23? Latest Features, Updates & Improvements"
date: 2025-03-13
categories: java
description: "Explore the new features of Java 23 including pattern matching, Class-File API, ZGC generational mode, and Structured Concurrency."
---

# Introducing Java 23: What’s New in the Latest Release?
JDK 23 is the latest release of the Java SE Platform. Until March 2025, JDK 24 will be released.

Java 23 is here, bringing **exciting new features and enhancements** that improve performance, code readability, and developer productivity. Let’s take a look at some of the **key updates** in this release!

## Primitive Types in Patterns, `instanceof`, and `switch` (Preview)
Java 23 **enhances pattern matching** by allowing **primitive types** in `instanceof` checks and `switch` statements. This makes pattern-based programming more **expressive and efficient**. To eliminate manual casting and improves code readability.

### Example:
```java
void test(Object obj) {
    if (obj instanceof int i) {
        System.out.println("Integer: " + i);
    }
}
```

## Class-File API (Second Preview)
A new API for working with Java class files! This gives better introspection capabilities for frameworks and tools that need to analyze or modify bytecode. To help library and tool developers with efficient class file parsing and modification.

### Example:
```java
/**
* Read MyClass.class from disk.
* Parse the class structure using ClassFileParser.parse(classBytes).
* Extract and print the class name and method names.
**/

import java.lang.classfile.ClassFile;
import java.lang.classfile.ClassFileParser;
import java.nio.file.Files;
import java.nio.file.Path;

public class ReadClassFile {
    public static void main(String[] args) throws Exception {
        // Load a .class file (Example: "MyClass.class")
        Path classFilePath = Path.of("MyClass.class");
        byte[] classBytes = Files.readAllBytes(classFilePath);

        // Parse the class file
        ClassFile classFile = ClassFileParser.parse(classBytes);

        // Print class name
        System.out.println("Class Name: " + classFile.thisClass().name());

        // Print method names
        classFile.methods().forEach(method -> 
            System.out.println("Method: " + method.name())
        );
    }
}

```

## Markdown Documentation Comments
Javadoc now supports Markdown, making documentation cleaner and easier to read. Developers can write structured and readable documentation directly in Java.

### Example:
```java
/**
 * # My Class
 * This class does something **amazing**.
 *
 * - Supports markdown
 * - Easier to format
 */
public class MyClass { }
```

## Vector API (Eighth Incubator)
Java’s Vector API continues to evolve, providing better performance for numerical and data-processing applications. To enable hardware-optimized computations, speeding up AI, ML, and scientific applications.
### Example:
```java
import jdk.incubator.vector.*;

public class VectorAddition {
    public static void main(String[] args) {
        // Choose a specific vector species (platform-dependent size)
        VectorSpecies<Float> species = FloatVector.SPECIES_PREFERRED;

        // Define two float arrays
        float[] a = {1.0f, 2.0f, 3.0f, 4.0f};
        float[] b = {5.0f, 6.0f, 7.0f, 8.0f};
        float[] result = new float[species.length()];

        // Load the arrays into vector registers
        FloatVector va = FloatVector.fromArray(species, a, 0);
        FloatVector vb = FloatVector.fromArray(species, b, 0);

        // Perform vectorized addition
        FloatVector vc = va.add(vb);

        // Store the result back into an array
        vc.intoArray(result, 0);

        // Print results
        System.out.println("Vector Addition Result:");
        for (float v : result) {
            System.out.print(v + " ");
        }
    }
}
```

## Stream Gatherers (Second Preview)
New stream processing capabilities that allow grouping and restructuring data more flexibly. To enhance custom stream operations for data transformation.
### Example:
```java
List<String> result = Stream.of("apple", "banana", "cherry")
    .gather((word, sink) -> sink.accept(word.toUpperCase()))
    .toList();
```

## Deprecated Memory-Access Methods in sun.misc.Unsafe
Java officially deprecates certain unsafe memory-access methods, pushing developers towards safer APIs. To encourage secure memory handling and better compatibility across JVM implementations.

## ZGC: Generational Mode by Default
The Z Garbage Collector (ZGC) now uses generational mode by default, reducing pause times and improving memory efficiency. Faster and more efficient garbage collection for large-scale applications.

## Module Import Declarations (Preview)
Modules can now be imported in a cleaner way, simplifying dependency management. To improve code organization and module usability.
### Example:
```java
import module java.sql;  // Import the java.sql module

import java.sql.Connection;
import java.sql.DriverManager;

public class DatabaseExample {
    public static void main(String[] args) throws Exception {
        Connection conn = DriverManager.getConnection("jdbc:h2:mem:test");
        System.out.println("Connected: " + (conn != null));
    }
}
```

## Implicitly Declared Classes and Instance Main Methods
A major quality-of-life improvement for beginner-friendly Java coding! Java now allows implicit class definitions in **single-file programs**. Simplifing Java for scripting and educational purposes.
### Example:
```java
void main() {
    System.out.println("Hello, Java 23!");
}
```

## Structured Concurrency (Third Preview)
Java 23 continues to improve concurrency handling, making multi-threaded code more predictable. To reduce bugs and race conditions in parallel programming.

## Scoped Values (Third Preview)
Scoped values replace thread-local variables with a safer alternative, improving performance and security in concurrent applications. More efficient resource management in multi-threaded environments.
### Example:
```java
import java.lang.scoped.*;

public class ScopedValueThreadExample {
    private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

    public static void main(String[] args) {
        ScopedValue.where(REQUEST_ID, "REQ-456").run(() -> {
            new Thread(ScopedValueThreadExample::handleRequest).start();
        });
    }

    public static void handleRequest() {
        // ScopedValue is NOT inherited by the new thread
        System.out.println("Handling request with ID: " + (REQUEST_ID.isBound() ? REQUEST_ID.get() : "No Scoped Value"));
    }
}
```

## Flexible Constructor Bodies (Second Preview)
Java 23 introduces greater flexibility in constructors, allowing custom logic before initializing fields. More control over object initialization.
- You can now initialize fields and perform computations before calling super().
- You can now calculate values dynamically before calling super().
- You can apply logic before calling super().
- You can call instance methods to process data before superclass initialization.

## Final Thoughts
Java 23 enhances developer productivity with safer, faster, and more expressive features. Whether you’re working on high-performance computing, multi-threading, or modernizing Java applications, these improvements make Java more powerful than ever!

## Feel Free to Contact Me
If you have any questions, comments, or disagreements with anything, feel free to send me an email. You can find my email address on the my page.
