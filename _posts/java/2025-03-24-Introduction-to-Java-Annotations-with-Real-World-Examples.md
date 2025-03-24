---
layout: post
title: "Introduction to Java Annotations with Real-World Examples"
date: 2025-03-24
categories: java
published: true
---

# Introduction to Java Annotations with Real-World Examples

## What are Java Annotations?
Java Annotations are a form of metadata introduced in **JDK 5** that provide **additional information about the code**. 

Annotations do not directly affect program semantics but can be **used by compilers, tools, and frameworks** to perform various operations.

## Common Use Cases
1. **Compiler Instructions**  
Example: `@Override` tells the compiler that a method overrides a superclass method.
2. **Compile-Time Processing (Code Generation, Validation)**  
Example: `@Getter` from Lombok.
3. **Runtime Processing (via Reflection)**  
Example: Spring annotations like `@Autowired`, `@RequestMapping`.

## Built-in Annotations in Java
| Annotation           | Description                                 |
|----------------------|---------------------------------------------|
| `@Override`          | Indicates a method overrides a superclass method |
| `@Deprecated`        | Marks a method/class as deprecated          |
| `@SuppressWarnings`  | Suppresses compiler warnings                |

### Example:

```java
public class MyClass {
    @Override
    public String toString() {
        return "MyClass instance";
    }

    @Deprecated
    public void oldMethod() {
        System.out.println("Old method");
    }

    @SuppressWarnings("unchecked")
    public void example() {
        List list = new ArrayList();  // Suppress warning for raw type
    }
}
```

## Creating My Own Annotations

### 1. Define `MyLog` Annotation
First the Annotation `@MyLog` is created using `@interface`, targeted at methods (`@Target(ElementType.METHOD)`) and retained at runtime (`@Retention(RetentionPolicy.RUNTIME)`). It has a `value()` field with a default string.
```java
import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyLog {
    String value() default "default log";
}
```

### 2. Use the Annotation
The `@MyLog("Calling sayHello")` annotation is applied to the `sayHello()` method of a `Service` class.
```java
public class Service {
    @MyLog("Calling sayHello")
    public void sayHello() {
        System.out.println("Hello!");
    }
}
```

### 3. Read the Annotation via Reflection
At last, reflection is used to check if `sayHello()` has the `@MyLog` annotation.
```java
import java.lang.reflect.Method;

public class Main {
    public static void main(String[] args) throws Exception {
        Method method = Service.class.getMethod("sayHello");
        if (method.isAnnotationPresent(MyLog.class)) {
            MyLog log = method.getAnnotation(MyLog.class);
            System.out.println("Log value: " + log.value());
        }

        new Service().sayHello();
    }
}
```

## Annotations in Frameworks (Spring Example)

| Annotation            | Meaning                                    |
|-----------------------|--------------------------------------------|
| `@Component`          | Declares a Spring-managed component        |
| `@Autowired`          | Injects dependencies automatically         |
| `@Controller`/`@RestController` | Marks controller classes         |
| `@RequestMapping`     | Maps HTTP requests to handler methods      |
| `@Service`            | Declares a service layer component         |

### Example:

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() {
        return "Hello Spring Boot";
    }
}
```
