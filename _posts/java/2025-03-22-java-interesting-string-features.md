---
layout: post
title: "Interesting String Features in Java: Literals, Compact Strings & Text Blocks"
date: 2025-03-22
categories: java
published: true
description: "Learn about unique Java String features including String Pool, Compact Strings, and Text Blocks with practical code examples."
---

# Some Interesting `String` Features in Java

## 1. String Literals: Constants and Singletons

In Java, if the same string literal (e.g., `"Hello World"`) is used in multiple variable declarations, the Java Virtual Machine (JVM) may create only **one instance** in memory. Thus, string literals act as **constants** or **singletons**.

### constants or singleton

In this case, both str1 and str2 point to the **same** String object stored in the **String Pool** of the JVM.

```java
String str1 = "Hello World";
String str2 = "Hello World";
```

JVM maintains a String Constant Pool, where all string literals are stored. Even if classes from different projects are compiled separately, they may still share the same constant String objects at runtime.

### Separate String Objects

If you want two String variables to point to separate objects, use the new operator.

Although both strings contain the same text, the JVM creates two different objects in memory.

```java
String str1 = new String("Hello World");
String str2 = new String("Hello World");
```

## 2. Compact Strings

Introduced in Java 9, Compact Strings optimize **memory usage** for String objects.

- When a String is created, the JVM **detects** if it contains only **ISO-8859-1/Latin-1** characters.
- If so, it stores the String internally as a **byte array** (byte[]) instead of a char[], reducing memory consumption (each character takes only one byte).
- This check occurs at the time of string creation and does not affect the **immutability** of the String.

**Before Java 9 (UTF-16 representation)**

- **`"Hello"` → Stored as a `char[]` with 10 bytes (2 bytes per character).**
- `"你好"` → Stored as a `char[]` with 4 bytes (since non-Latin-1 characters require UTF-16).
```java
public class StringMemoryTest {
    public static void main(String[] args) {
        String str1 = "Hello"; // 5 characters → 10 bytes (UTF-16, 2 bytes per char)
        String str2 = "你好";   // 2 characters → 4 bytes (UTF-16)
        
        System.out.println(str1);
        System.out.println(str2);
    }
}
```

**After Java 9 (Compact Strings enabled)**

- **`"Hello"` → Stored in a `byte[]` (5 bytes total) instead of a `char[]` (10 bytes).**
- `"你好"` → Since it contains non-Latin-1 characters, it falls back to UTF-16 using `char[]` (4 bytes).
```java
public class CompactStringTest {
    public static void main(String[] args) {
        String latinString = "Hello"; // Uses byte[] internally (5 bytes)
        String unicodeString = "你好"; // Uses char[] internally (4 bytes)
        
        System.out.println(latinString);
        System.out.println(unicodeString);
    }
}
```

## 3. Java Text Blocks

**Introduced in Java 13 (as a preview) and officially released in Java 15**, Text Blocks provide a more readable and convenient way to define multi-line strings in Java. This feature eliminates the need for **escape sequences (\n, \", etc.)** and improves code maintainability.

A text block is a multi-line string literal enclosed within **triple double-quotes (""")**. It automatically preserves new lines and indentation, making it an ideal choice for defining formatted text such as:
- HTML
- JSON
- SQL queries
- Multi-line logs

### Features and Advantages
- **Automatic Line Breaks**

Text blocks preserve new lines as they appear in the source code.
```java
String message = """
    Hello,
    This is a multi-line message.
    Have a great day!
    """;
System.out.println(message);
```
```java
Hello,
This is a multi-line message.
Have a great day!
```
- **No Need to Escape Double Quotes (`"`)**

In regular Java strings, double quotes inside the string must be escaped (`\"`).
With text blocks, escaping is unnecessary.
```java
String json = """
    {
        "name": "Alice",
        "age": 25
    }
    """;
System.out.println(json);
```
```java
{
    "name": "Alice",
    "age": 25
}
```
- **Supports Indentation Handling**

By default, Java removes common leading whitespace based on the leftmost non-space character.

```java
String sqlQuery = """
        SELECT * FROM users
        WHERE active = true
        ORDER BY created_at DESC;
    """;

```

The leading whitespace in each line is automatically aligned.

- **Handling Escaping and Special Characters**

Although escaping quotes (`"`) is not required, certain characters still need escaping such as Triple double-quotes (`"""`) and Backslashes (`\`).

```java
String text = """
    This is a "quote" inside a text block.
    To use triple quotes, escape one: \"""
    This is how you add \"\\\" inside text blocks.
    """;
System.out.println(text);
```

```java
This is a "quote" inside a text block.
To use triple quotes, escape one: """
This is how you add "\"" inside text blocks.
```

- **String Formatting with .formatted()**

Text blocks support String formatting using .formatted().

```java
String template = """
    Name: %s
    Age: %d
    Country: %s
    """.formatted("Alice", 25, "Canada");

System.out.println(template);
```

```java
Name: Alice
Age: 25
Country: Canada
```

### Use Cases of Java Text Blocks
- JSON or XML Formatting
- Writing SQL Queries
- Multi-line Logging and Messages
- HTML Content Representation

```java
String sql = """
    SELECT id, name, email
    FROM users
    WHERE status = 'active'
    ORDER BY created_at DESC;
    """;
```

```java
String html = """
    <html>
        <head>
            <title>Java Text Blocks</title>
        </head>
        <body>
            <h1>Welcome to Java 15!</h1>
        </body>
    </html>
    """;
```

## Summary

| Feature           | Description                                                            | Available Since |
|------------------|------------------------------------------------------------------------|----------------|
| **String Literals** | Identical string literals are stored in the **String Pool**, preventing redundant object creation. | All Versions |
| **String Singleton** | String literals with the same value share the **same instance**, unless explicitly created with `new String()`. | All Versions |
| **Compact Strings** | If a `String` contains only **Latin-1** characters, it is stored using `byte[]` to reduce memory usage. | Java 9+ |
| **Text Blocks** | Multi-line strings with **automatic formatting** and **built-in support for quotes**. | Java 13+ |
