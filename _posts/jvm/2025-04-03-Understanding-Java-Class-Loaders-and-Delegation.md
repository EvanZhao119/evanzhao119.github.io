---
layout: post
title: "Understanding Java Class Loaders and Delegation"
date: 2025-04-03
categories: jvm
published: true
---

# Understanding Java Class Loaders and Delegation

## 1. Class Uniqueness and Class Loaders
In the Java Virtual Machine (JVM), **the uniqueness of a class** is determined by **its fully qualified name plus the class loader that loaded it**. 
```java
<class loader instance> + <package.class name>
```
Two classes with the same name loaded by different class loaders are treated as **completely distinct types** by the JVM.

## 2. Parent Delegation Model
Java's class loading mechanism follows the **Parent Delegation Model**, where **a class loader first delegates the class-loading request to its parent** before attempting to load the class itself.

### 2.1 Types of Class Loaders
| Class Loader | Implemented In | Responsibility |
|--------------|----------------|----------------|
| **Bootstrap ClassLoader** | C++ | Loads core Java classes from `JAVA_HOME/lib` and classes specified by `-Xbootclasspath`. Appears as `null` in output. |
| **Extension ClassLoader** | Java | Loads classes from `JAVA_HOME/lib/ext` or paths specified by `java.ext.dirs`. |
| **Application ClassLoader** | Java | Loads classes from the user classpath (`classpath`). This is the default loader for most applications. |

As of Java 9, the **Extension ClassLoader has been removed** and replaced by the **Platform ClassLoader**, reflecting the introduction of the Java Module System.

## 3. Class Loader Hierarchy (Java 9+)
From Java 9 onward, the Platform ClassLoader replaces the Extension ClassLoader and is responsible for loading platform modules (excluding core modules).

```text
                +--------------------------+
                |   Bootstrap ClassLoader  |
                +--------------------------+
                            |
                +--------------------------+
                |Platform ClassLoader (new)|
                +--------------------------+
                            |
                +--------------------------+
                |  Application ClassLoader |
                +--------------------------+

```

## 4. Scenarios that Break the Parent Delegation Model
Although delegation is the default model, certain real-world use cases break this pattern deliberately.

### 4.1 JDBC Mechanism
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.18</version>
</dependency>
```
```java
public class DriverStudy {
    public static void main(String[] args) throws SQLException {
        System.out.println(Driver.class.getClassLoader());
        System.out.println(DriverManager.class.getClassLoader());
        System.out.println(com.mysql.cj.jdbc.Driver.class.getClassLoader());

        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306", "root", "123456");
    }
}
```
**Sample output (Java 8):**
```java
null  // Driver and DriverManager are loaded by Bootstrap ClassLoader
null
sun.misc.Launcher$AppClassLoader@xxx  // MySQL driver loaded by Application ClassLoader
```

### 4.2 Internal Analysis of JDBC Loading
- `DriverManager` is loaded by the Bootstrap ClassLoader.
- Inside `getConnection()`, the loader:
    - Retrieves **the context class loader** via `Thread.currentThread().getContextClassLoader()`.
    - Uses this loader to locate `com.mysql.cj.jdbc.Driver`.
    - Verifies the class using `Class.forName()` and loads it if allowed.

#### Breaking Point
Although DriverManager is loaded by the Bootstrap ClassLoader, it uses the thread context class loader—typically the Application ClassLoader—to discover and load third-party JDBC drivers dynamically. 

This breaks the parent delegation model and follows the Service Provider Interface (SPI) pattern, which relies on the META-INF/services directory and the ServiceLoader mechanism.

## 5. Java 9+ Module System and Class Loading
With the introduction of Java Modules (Jigsaw) in Java 9:
- Class loading is determined by **module ownership**, not just classpath.
- Platform and application class loaders must check if a class belongs to a system module before delegating.
- If a class belongs to a specific module, the corresponding module class loader loads it, bypassing parent delegation.

>The Java 9+ module system formally relaxes strict parent delegation, favoring modular encapsulation.

## 6. Other Scenarios that Break Delegation

Frameworks such as **JNDI** and **OSGi** often **intentionally break the parent delegation model** by using **custom class loader hierarchies**. These hierarchies are designed to support **modularization**, **dynamic loading**, and **hot-swapping** of components.

Such frameworks typically rely on:
- **Child-first loading** (also known as reverse delegation)
- Allowing **child class loaders to load classes before delegating to parent loaders**

This approach enables better **isolation**, **versioning**, and **runtime flexibility**, which are essential in modular systems or plugin-based architectures.
