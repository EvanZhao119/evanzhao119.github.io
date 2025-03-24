---
layout: post
title: "Comparison: Java Lambda Expressions vs. Anonymous Interface Implementations"
date: 2025-03-24
categories: java
published: true
---

# Comparison: Java Lambda Expressions vs. Anonymous Interface Implementations

## 1. Comparison Summary

| Feature | Lambda Expression | Anonymous Interface Implementation |
|--------|-------------------|------------------------------------|
| Syntax | More concise | Verbose |
| Readability | Clearer, especially for functional interfaces | Less readable with nested logic |
| Scope of `this` | Refers to the outer class instance | Refers to the anonymous inner class instance |
| Applicable To | Only functional interfaces (one abstract method) | Any interface or class |
| Compilation | Compiles to a private static method | Compiles to a separate .class file |
| Performance | More efficient (compile-time optimization) | Slightly slower (runtime instantiation) |

## 2. Example: `Runnable` Interface

### 1. Using Anonymous Inner Class
```java
public class RunnableExample {
    public static void main(String[] args) {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println("Running from Anonymous Class");
            }
        };

        Thread thread = new Thread(runnable);
        thread.start();
    }
}
```

### 2. Using Lambda Expression

```java
public class RunnableExample {
    public static void main(String[] args) {
        Runnable runnable = () -> System.out.println("Running from Lambda");

        Thread thread = new Thread(runnable);
        thread.start();
    }
}
```

---

### 3. Most Concise Form

```java
public class RunnableExample {
    public static void main(String[] args) {
        new Thread(() -> System.out.println("Running directly from Lambda")).start();
    }
}
```

## 3. Important Differences

### 1. The Meaning of `this`
```java
public class Main {
    String name = "OuterClass";

    public void test() {
        Runnable anon = new Runnable() {
            String name = "InnerClass";

            @Override
            public void run() {
                System.out.println(this.name); // Refers to InnerClass
            }
        };

        Runnable lambda = () -> {
            System.out.println(this.name); // Refers to OuterClass
        };

        anon.run();
        lambda.run();
    }

    public static void main(String[] args) {
        new Main().test();
    }
}
```

### 2. Functional Interface Limitation

Lambda expressions only work with interfaces that contain **exactly one abstract method**.

```java
interface NotFunctional {
    void method1();
    void method2();
}

// This will cause an error:
// NotFunctional nf = () -> System.out.println("Invalid");
```
