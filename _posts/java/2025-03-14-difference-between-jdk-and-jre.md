---
layout: post
title: "Difference Between JRE and JDK"
date: 2025-03-14
categories: java
published: true
---

# Difference Between JRE and JDK

The difference between JRE (Java Runtime Environment) and JDK (Java Development Kit) lies in their **purpose** and **components**.

## 1. JRE (Java Runtime Environment)
- **Purpose:** Used to **run** Java applications (not to compile Java applications).
- **Components:**
  - **JVM (Java Virtual Machine):** Executes Java bytecode.
  - **Core Libraries:** Provides essential libraries required for Java programs to run.
  - **Runtime Environment:** Manages application execution.
- **Who Needs It?** If you only need to run Java applications and not develop them, JRE is sufficient.

## 2. JDK (Java Development Kit)
- **Purpose:** Used to **develop and run** Java applications.
- **Components:**
  - **JRE (which includes JVM, core libraries, and runtime environment).**
  - **Development Tools:** Compiler (`javac`), debugger (`jdb`), JavaDoc (`javadoc`), and other utilities.
- **Who Needs It?** If you are a developer writing Java code, you need JDK to compile and build Java applications.

## 3. Conclusion
- If you **only need to run** Java programs → Install **JRE**.
- If you **need to develop** Java programs → Install **JDK** (which already includes JRE).

| Feature             | JRE         | JDK         |
|---------------------|------------|------------|
| Contains JVM?      | ✅ Yes      | ✅ Yes      |
| Contains JRE?      | ✅ Yes      | ✅ Yes      |
| Includes Compiler (`javac`)? | ❌ No | ✅ Yes |
| Includes Development Tools? | ❌ No | ✅ Yes |
| Purpose            | Run Java applications | Develop and run Java applications |
