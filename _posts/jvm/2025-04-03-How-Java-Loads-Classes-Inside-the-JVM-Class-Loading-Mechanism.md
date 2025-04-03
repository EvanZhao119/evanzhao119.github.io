---
layout: post
title: "How Java Loads Classes: Inside the JVM Class Loading Mechanism"
date: 2025-04-03
categories: jvm
published: true
---

# How Java Loads Classes: Inside the JVM Class Loading Mechanism

## 1. What Is Class Loading?
Class loading refers to the process by which the Java Virtual Machine (JVM) loads class definitions (from `.class` files) into memory, verifies, transforms, resolves, and initializes them into runtime structures that the JVM can directly use.

Unlike many other languages, Java performs type loading, linking, and initialization **at runtime**, enabling dynamic loading and execution.

## 2. Class Loading Process Overview
The JVM defines **five main phases** in the class loading process:
1. **Loading**
2. **Verification**
3. **Preparation**
4. **Resolution**
5. **Initialization**

> The first three stages are part of the loading process. The latter two are part of the linking phase.

### 2.1 Loading
- The JVM uses the **fully qualified name** (e.g., `java/lang/String`) to locate the corresponding `.class` file.
- The class file is read into a **binary byte stream**.
- The byte stream is parsed into **runtime data structures**, typically stored in the **method area** (now **Metaspace**).
- A `java.lang.Class` object is created on the **heap** to serve as an access point to the class’s metadata.

> **Java 8 Update**:  
> The **PermGen space was removed** and replaced by **Metaspace**, which resides in native memory.

> **Java 15 Update**:  
> The introduction of **Hidden Classes** allows dynamically generated classes to be used internally without being exposed through a `ClassLoader`.

### 2.2 Verification
Ensures the class file is safe and conforms to the JVM specification. 
- **File format verification** (e.g., magic number, version)
- **Metadata verification** (e.g., class hierarchy validity)
- **Bytecode verification** (e.g., control flow, instruction correctness)
- **Symbolic reference verification**

> This step guarantees runtime security and prevents JVM crashes due to malformed classes.

### 2.3 Preparation
- Allocates **memory for class variables (static fields)** and sets them to default values (`0`, `null`, `false`, etc.).
- **Instance fields are not included** here; they are allocated during object instantiation.

> Java 8 and beyond: Static variables are stored on the heap along with the `Class` object rather than in the method area.

### 2.4 Resolution
- Converts **symbolic references** in the constant pool (e.g., `"java/lang/Object"`) into **direct references** (pointers or memory addresses).
- Applies to class references, field references, method references, and interface references.

> Resolution can be **eager (at load time)** or **lazy (when first used)** depending on the JVM implementation.

### 2.5 Initialization
- Executes the class’s **`<clinit>()` method**, which:
    - Assigns values to static variables
    - Executes static blocks (`static {}`)

> `<clinit>()` is **not always present**. If there are no static initializations or static blocks, it won’t be generated.

#### Thread-Safety
- JVM ensures that `<clinit>()` is **executed atomically and once** per class, even in multithreaded environments.

#### Parent Class Initialization Order
The parent class is always initialized before the child class.
```java
class Parent {
    static { System.out.println("Parent static"); }
}

class Child extends Parent {
    static { System.out.println("Child static"); }
}

public class Test {
    public static void main(String[] args) {
        new Child();
    }
}
//Output:
//Parent static
//Child static
```
