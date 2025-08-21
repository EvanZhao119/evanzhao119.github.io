---
layout: post
title: "Introduction to Nested Classes in Java: Static, Inner, Local, Anonymous"
date: 2025-04-10
categories: java
published: true
description: "A complete guide to Java nested classes, including static, inner, local, and anonymous classes. Learn best practices and real-world usage examples."
---

# Introduction to Nested Classes in Java

In Java, **nested classes** are classes defined within the body of another class. This feature is primarily used to logically group classes that are only relevant to their enclosing class, thereby improving encapsulation and code clarity. **Nested classes are commonly used when one class is tightly bound to another and is not intended to be used independently.**

There are **four main types of nested classes** in Java: **Static Nested Classes, Non-static Nested Classes (Inner Classes), Local Classes and Anonymous Classes.**

## 1. Static Nested Classes
These are defined with the `static` keyword and do not have access to instance variables or methods of the enclosing class. They are used when the nested class does not need a reference to the outer class. Static nested classes behave much like regular top-level classes but are scoped within their outer class.

```java
//Define a Static Nested Class
class Outer {
    static class StaticNested {
        void display() {
            System.out.println("Inside static nested class");
        }
    }
}
//Create an Instance of a Static Nested Class
Outer.StaticNested nested = new Outer.StaticNested();
nested.display();
```

## 2. Non-static Nested Classes (Inner Classes)
These classes have access to all members (including private) of their enclosing class. They are associated with an instance of the outer class and can be used when the nested class logically requires access to the outer class’s members.

```java
//Define a Non-static (Inner) Class
class Outer {
    class Inner {
        void show() {
            System.out.println("Inside inner class");
        }
    }
}
//Instantiate an Inner Class
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
inner.show();
```

## 3. Local Classes
These are inner classes declared within a method or a block of code. They are only visible within the scope they are defined in and are often used for simple, short-lived tasks, such as implementing a listener or performing calculations.

```java
public class LocalClassExample {
    public void printMessage() {
        class LocalPrinter {
            void print() {
                System.out.println("Hello from the Local Class!");
            }
        }

        LocalPrinter printer = new LocalPrinter();
        printer.print();
    }

    public static void main(String[] args) {
        LocalClassExample example = new LocalClassExample();
        example.printMessage();
    }
}
```

## 4. Anonymous Classes
These are local classes without a name, usually used to instantiate objects with certain behaviors, typically for interfaces or abstract classes. They are often passed as arguments to methods or used in GUI event handling.

```java
public class AnonymousClassExample {
    public static void main(String[] args) {
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("Running from Anonymous Class!");
            }
        };

        Thread thread = new Thread(task);
        thread.start();
    }
}
```

## 5. Best Practices

- Use **static nested classes** when the nested class does not need to access instance members of the outer class.
- Use **inner classes** when the nested class requires access to the outer class’s members.
- Use **local or anonymous classes** for one-time use cases or for quick, localized implementations like event listeners or simple threads.
- **Avoid overusing nested classes, as excessive nesting can make code harder to understand and maintain.**

## 6. Define My Own Map Structure Inspired by `java.util.HashMap`
In this example, I define a simplified custom map structure that mimics the basic behavior of Java's `HashMap`. I use **a static nested class** called `Entry` to represent key-value pairs, just like `HashMap` uses its internal `Node<K,V>` class.

The `Entry<K, V>` class is used to hold key-value pairs and is defined as a static nested class because it does not need access to the outer `MySimpleMap` instance.

```java
import java.util.ArrayList;
import java.util.List;

public class MySimpleMap<K, V> {

    // Static nested class representing a key-value pair (Entry)
    private static class Entry<K, V> {
        K key;
        V value;

        Entry(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    // List to store all entries
    private List<Entry<K, V>> entries;

    public MySimpleMap() {
        entries = new ArrayList<>();
    }

    // Method to add or update a key-value pair
    public void put(K key, V value) {
        for (Entry<K, V> entry : entries) {
            if (entry.key.equals(key)) {
                entry.value = value; // Update value if key already exists
                return;
            }
        }
        entries.add(new Entry<>(key, value)); // Otherwise, add new entry
    }

    // Method to retrieve the value by key
    public V get(K key) {
        for (Entry<K, V> entry : entries) {
            if (entry.key.equals(key)) {
                return entry.value;
            }
        }
        return null; // Return null if key is not found
    }

    // Check if a key exists in the map
    public boolean containsKey(K key) {
        for (Entry<K, V> entry : entries) {
            if (entry.key.equals(key)) {
                return true;
            }
        }
        return false;
    }
}
```
