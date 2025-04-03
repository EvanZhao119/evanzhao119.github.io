---
layout: post
title: "How Java Handles Method Invocation"
date: 2025-04-02
categories: jvm
published: true
---

# How Java Handles Method Invocation
In Java, method invocation is categorized into **resolution**, **static dispatch**, and **dynamic dispatch**. These categories differ based on the **timing and criteria** used to select the method.

## 1. Resolution (Static Resolution)

**Resolution** occurs at **compile time**, which means the target method for invocation is determined during the program's source code writing and compilation.

- **Static Process**: Resolution is a static process, meaning the compiler can fully determine the method call target at compile time, eliminating the need to wait until runtime.
- **Symbolic Reference to Direct Reference**: During the class loading phase, all symbolic references are resolved into direct references. For example, static methods, private methods, constructors, parent class methods, and `final` methods are determined at compile time.
- **Non-Virtual Methods**: These methods are typically referred to as "non-virtual methods" because there is no polymorphic behavior at runtime.

**Applicable Situations**:
- **Static Methods**: The invocation of static methods is related to the class itself, not the object.
- **Private Methods**: Private methods can only be accessed within the class, and the target is determined statically.
- **`final` Methods**: Since `final` methods cannot be overridden, the method call target is fixed at compile time.
- **Constructors and Parent Class Methods**: These are also resolved at compile time.

## 2. Static Dispatch
**Static dispatch** refers to the dispatch process where the method version is determined based on the static type of the reference at **compile time**.

- **Method Overloading**: Method overloading is a form of static dispatch. The compiler selects which method to invoke based on the method signature, such as the number and type of arguments.
- **Compile-Time Decision**: The dispatch happens at compile time, and the JVM does not play a role in this decision-making process.

**Example**: When multiple overloaded methods with the same name but different parameter types exist, the compiler determines the correct method based on the arguments passed.

```java
class Printer {
    void print(String s) { System.out.println("Printing String: " + s); }
    void print(int i) { System.out.println("Printing int: " + i); }
}

public class Main {
    public static void main(String[] args) {
        Printer printer = new Printer();
        printer.print("Hello");  // Static dispatch: prints the print(String s) method
        printer.print(5);        // Static dispatch: prints the print(int i) method
    }
}
```

## 3. Dynamic Dispatch
**Dynamic dispatch** occurs at **runtime** when the JVM selects the method to invoke based on the actual object type. This process is commonly associated with **polymorphism**.

- **Method Overriding**: Method overriding is the classic example of dynamic dispatch. The JVM decides which method to call based on the actual object type, not the reference type.
- **Polymorphism**: In a polymorphic situation, if a parent class reference points to a subclass object, the method invoked will be the one in the subclass.

**Example**:
```java
class Animal {
    void sound() { System.out.println("Animal makes a sound"); }
}

class Dog extends Animal {
    @Override
    void sound() { System.out.println("Dog barks"); }
}

public class Main {
    public static void main(String[] args) {
        Animal animal = new Dog(); // Dynamic dispatch
        animal.sound();  // Outputs "Dog barks"
    }
}
```

**Important Note**:
- **Fields Do Not Participate in Polymorphism**: Fields are always selected based on the reference type. If a subclass declares a field with the same name as the one in the parent class, both fields exist in the subclass, but the subclass field hides the parent class's field.

```java
class Parent {
    int field = 10;
}

class Child extends Parent {
    int field = 20;
}

public class Main {
    public static void main(String[] args) {
        Parent p = new Child();
        System.out.println(p.field);  // Outputs 10, field does not participate in dynamic dispatch
    }
}
```

## 4. Some Improvments

- **JVM Optimizations**: Modern JVMs (e.g., OpenJDK, GraalVM) offer advanced optimizations for method invocation, such as **Just-In-Time (JIT) compilation**. This allows dynamic dispatch to be further optimized during runtime, improving performance in hot code paths.

- **Lambda Expressions and Method References**: Java 8 introduced **Lambda expressions** and **method references**, which added new forms of dynamic dispatch. For example, when using a lambda expression, the JVM must choose the correct method implementation based on the actual type of the lambda, which is a form of dynamic dispatch.

```java
import java.util.function.Function;

public class Main {
    public static void main(String[] args) {
        Function<String, String> function = (s) -> s.toUpperCase(); // Dynamic dispatch
        System.out.println(function.apply("hello")); // Outputs "HELLO"
    }
}
```

## 5. Summary
The Java method invocation process is determined by **resolution, static dispatch, and dynamic dispatch**. 

Resolution happens at compile time, static dispatch depends on static types, and dynamic dispatch relies on actual object types at runtime. 

With advancements in Java technologies, particularly JVM optimizations and new features like lambda expressions, method invocation performance and flexibility have significantly improved.
