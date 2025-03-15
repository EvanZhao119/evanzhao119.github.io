---
layout: post
title: "Autoboxing & Unboxing in Java"
date: 2025-03-15
categories: java
published: true
---

# Autoboxing & Unboxing in Java

In Java, **autoboxing** and **unboxing** are automatic conversions between **primitive data types** and their **corresponding wrapper classes**. These features were introduced in **Java 5** to simplify coding when working with collections, generics, and object-oriented programming.

## 1. What is Autoboxing?

**Autoboxing** is the automatic conversion of a **primitive data type** into its corresponding **wrapper class** by the Java compiler.

### Example of Autoboxing
```java
public class AutoboxingExample {
    public static void main(String[] args) {
        int primitiveInt = 10;

        // Autoboxing: int → Integer
        Integer objectInt = primitiveInt;

        System.out.println("Primitive int: " + primitiveInt);
        System.out.println("Autoboxed Integer: " + objectInt);
    }
}
```

### Where Autoboxing is Useful?

- **When adding primitive values to collections** 
- **When using generic methods with primitives**

```java
import java.util.ArrayList;

public class AutoboxingExample {
    public static <T> void printValue(T value) {
        System.out.println("Value: " + value);
    }
    
    public static void main(String[] args) {
        ArrayList<Integer> numbers = new ArrayList<>();
        numbers.add(5); // Autoboxing: int → Integer
        numbers.add(10);
        System.out.println("Numbers List: " + numbers);
        
        printValue(20); // Autoboxing: int → Integer
        printValue(3.14); // Autoboxing: double → Double
    }
}
```

## 2. What is Unboxing?

**Unboxing** is the automatic conversion of a **wrapper class object** back into its corresponding **primitive type**.

### Example of Unboxing
```java
public class UnboxingExample {
    public static void main(String[] args) {
        Integer objectInt = 20;

        // Unboxing: Integer → int
        int primitiveInt = objectInt;

        System.out.println("Wrapper Integer: " + objectInt);
        System.out.println("Unboxed int: " + primitiveInt);
    }
}
```

### Where Unboxing is Useful?

- **When performing arithmetic operations on wrapper objects**
- **When passing wrapper objects to methods expecting primitives**

```java
public class UnboxingExample {
    public static void printSquare(int num) {
        System.out.println("Square: " + (num * num));
    }

    public static void main(String[] args) {
        Integer num1 = 50;
        Integer num2 = 30;

        // Unboxing happens automatically
        int sum = num1 + num2;

        System.out.println("Sum: " + sum);
        
        Integer objNum = 7;
        printSquare(objNum); // Unboxing: Integer → int
    }
}
```

## 3. Performance Considerations

While **autoboxing and unboxing** simplify code, they introduce performance overhead:

- **Memory Usage** 

    - Wrapper objects consume more memory than primitives.
    - Example: `int` takes **4 bytes**, but `Integer` takes **16 bytes** (due to object overhead).
    
- **Processing Time** 

    - Unnecessary boxing/unboxing can slow down performance in loops.
    - Example:
    ```java
     public class PerformanceTest {
         public static void main(String[] args) {
             long start = System.nanoTime();
             
             Integer sum = 0; // Autoboxing
             for (int i = 0; i < 1000000; i++) {
                 sum += i; // Unboxing & Autoboxing happening repeatedly
             }
             
             long end = System.nanoTime();
             System.out.println("Time taken: " + (end - start) + " ns");
         }
     }
     ```
    - **Fix**: Use `int` instead of `Integer` inside loops.

## 4. Summary

Autoboxing & unboxing make Java easier to work with, but understanding when and how they occur helps optimize performance. 

<table>
    <tr>
        <th>Feature</th>
        <th>Autoboxing</th>
        <th>Unboxing</th>
    </tr>
    <tr>
        <td>Definition</td>
        <td>Converts <b>primitive</b> → <b>Wrapper object</b></td>
        <td>Converts <b>Wrapper object</b> → <b>primitive</b></td>
    </tr>
    <tr>
        <td>When?</td>
        <td>Assigning a primitive value to a wrapper class</td>
        <td>Assigning a wrapper object to a primitive type</td>
    </tr>
    <tr>
        <td>Benefit</td>
        <td>Simplifies working with collections, generics</td>
        <td>Allows seamless arithmetic operations</td>
    </tr>
    <tr>
        <td>Downside</td>
        <td colspan="2">Consume more memory and implicit conversions may impact efficiency</td>
    </tr>
</table>
