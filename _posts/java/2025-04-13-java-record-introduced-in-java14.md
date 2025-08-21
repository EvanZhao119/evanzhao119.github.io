---
layout: post
title: "Java Record (Introduced in Java 14): Simplifying Data Classes"
date: 2025-04-13
categories: java
published: true
description: "Learn how Java record simplifies immutable data carriers with auto-generated methods. Covers syntax, use cases, advantages, and limitations of records."
---

# Java `record` (Introduced in Java 14)

Java **`record`** is a feature introduced in Java 14, designed to simplify class definitions—especially for Data Transfer Objects (DTOs) or simple data carriers. 

A `record` is a special type of class that provides a concise way to declare **immutable** data objects and significantly reduces boilerplate code.

## 1. Basic Definition
A `record` is an **immutable reference type** that functions similarly to a regular class but provides several simplifications. 

Java automatically generates: 
- A constructor
- Getter methods
- `equals()`, `hashCode()`, and `toString()` methods

### Example
```java
public record Person(String name, int age) {
}
```

This automatically provides:

- `public Person(String name, int age)`
- `name()` and `age()` accessor methods
- Implementations of `equals`, `hashCode`, and `toString`

## 2. How to Use Java Records
### 2.1 Creating a `record` Object
```java
public class Main {
    public static void main(String[] args) {
        Person person = new Person("John", 30);
        System.out.println(person.name());  // Output: John
        System.out.println(person.age());   // Output: 30
        System.out.println(person);         // Output: Person[name=John, age=30]
    }
}
```

### 2.2 Immutability
`record` fields are **final** and **cannot be modified** after creation.

```java
// This will cause a compilation error
person.name = "Jane";  // ❌ Error: Cannot assign a value to final variable 'name'
```

### 2.3 Auto-generated `equals`, `hashCode`, and `toString`
These methods are generated based on all fields.

```java
Person person1 = new Person("John", 30);
Person person2 = new Person("John", 30);

System.out.println(person1.equals(person2));            // true
System.out.println(person1.hashCode() == person2.hashCode()); // true
System.out.println(person1);                            // Output: Person[name=John, age=30]
```

## 3. Advantages and Features

- **Concise**: Reduces boilerplate code
- **Immutable**: Safer in concurrent/multithreaded environments
- **Auto-Generated Methods**: `equals`, `hashCode`, `toString`, and accessors

## 4. Limitations

- **No Class Inheritance**: Cannot extend other classes (implicitly extends `java.lang.Record`)
- **No Setters**: Fields are final, cannot be reassigned
- **Limited Extensibility**: You can add methods but not instance field modifiers

## 5. Using `record` with Interfaces

`record` types can implement interfaces:

```java
interface Identifiable {
    String getId();
}

public record Person(String id, String name, int age) implements Identifiable {
    @Override
    public String getId() {
        return id;
    }
}
```

## 6. Summary

| Feature                     | Description                                             |
|----------------------------|---------------------------------------------------------|
| **Immutable Fields**       | Fields are final and cannot be changed after creation   |
| **Auto-generated Methods** | Constructor, accessors, `equals`, `hashCode`, `toString` |
| **No Inheritance**         | Cannot extend classes, but can implement interfaces     |
| **Use Case**               | Great for DTOs and simple data-holding structures       |

Java `record` is ideal for:

- Data Transfer Objects (DTOs)
- Lightweight data holders
- Reducing repetitive code
- Improving code readability and maintainability
