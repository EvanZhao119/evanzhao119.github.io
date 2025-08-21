---
layout: post
title: "Java `static` Keyword Explained with Examples"
date: 2025-03-24
categories: java
published: true
description: "Learn the use cases of Java’s `static` keyword in variables, methods, blocks, and nested classes with code examples."
---

# Do You know Java Keyword `static`?

In Java, the `static` keyword can be applied to **variables, methods, code blocks, and nested classes**. 

## 1. `static` Variables (Class Variables)

A static variable belongs to the class rather than to any specific object. All instances of the class share the same static variable.

- Static variables are initialized when the class is loaded and only one copy is stored, shared by all instances.
- They can be accessed using `ClassName.variableName`, such as `Example.count`.

### Example:
```java
class Example {
    static int count = 0; // Static variable

    Example() {
        count++;
    }

    public static void main(String[] args) {
        Example obj1 = new Example();
        Example obj2 = new Example();
        System.out.println(Example.count); // Output: 2
    }
}
```

## 2. static Methods

A static method belongs to the class and can be called directly without creating an instance of the class.

- A static method can only access **static variables and other static methods**; it cannot directly access instance variables.
- It **cannot use** the this or super keyword.

### Example:
```java
class MathUtil {
    static int square(int x) {
        return x * x;
    }

    public static void main(String[] args) {
        System.out.println(MathUtil.square(5)); // Output: 25
    }
}
```

## 3. static Blocks (Static Initialization Blocks)

A static block executes once when the class is loaded, usually for initializing static variables.

- Executes **only once** when the class is loaded.
- Executes **before the constructor**.

### Example:
```java
class StaticBlockExample {
    static int value;

    static {
        value = 10;
        System.out.println("Static block executed");
    }

    public static void main(String[] args) {
        System.out.println("Value: " + value); // Output: 10
    }
}
```

## 4. static Nested Classes

A static nested class does not depend on an instance of the outer class and can be instantiated directly.

- A static nested class can access **static** members of the outer class but **cannot** access non-static members.

### Example:
```java
class Outer {
    static class Inner {
        void display() {
            System.out.println("Inside static nested class");
        }
    }

    public static void main(String[] args) {
        Outer.Inner obj = new Outer.Inner();
        obj.display(); // Output: "Inside static nested class"
    }
}
```

## 5. Importance of static in the main Method

The main method in Java must be static because the JVM calls it without creating an object of the class.

- Since the JVM executes main without creating an object, it **must be static** to allow direct execution.

### Example:
```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
```

## 6. Summary

The `static` keyword is essential in Java and can improve efficiency and structured program design when used correctly.

| **Usage**            | **Purpose** |
|----------------------|------------|
| `static` Variables  | Shared by all instances, created when the class is loaded |
| `static` Methods    | Can be called using class name, cannot access non-static members |
| `static` Blocks     | Executes once when the class loads, used for initialization |
| `static` Nested Classes | Can be instantiated without an outer class instance |
