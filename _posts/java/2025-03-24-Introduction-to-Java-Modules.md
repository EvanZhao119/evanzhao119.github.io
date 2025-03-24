---
layout: post
title: "Introduction to Java Modules"
date: 2025-03-24
categories: java
published: true
---

# Introduction to Java Modules

Java Modules were introduced in **Java 9** as a powerful feature to better organize and encapsulate code.

They provide a modular structure for Java applications, allowing developers to explicitly declare dependencies between modules and control which classes are accessible from outside.

## 1. What are Java Modules?
A module is a logical unit of code, typically consisting of related classes, interfaces, and resources. 

Each module includes a `module-info.java` file that declares the module's name, dependencies, and which packages are exposed to other modules.

### Example Module Structure
```
myapp/
├── module-info.java
├── com/
│   └── example/
│       └── app/
│           ├── Main.java
│           └── Utils.java
```

### Sample module-info.java
```java
module com.example.app {
    requires com.example.utils;
    exports com.example.app; // Only this package is exported
}
```

## 2. Key Features of Modules

| Feature           | Description                                                        |
|------------------|--------------------------------------------------------------------|
| `requires`       | Declares dependencies on other modules                             |
| `exports`        | Specifies which packages are accessible to other modules           |
| `opens`          | Allows runtime reflective access (e.g., for JSON serialization)    |
| `provides/uses`  | Related to the Service Loader mechanism (for SPI)                  |

## 3. A Simple Use Case: Building a Multi-Module Enterprise Application

I am developing an enterprise-level application - an online ordering system. Now I decide to divide it into four modules, including `core`, `db`, `api` and `web`.
- `order.core`: Core order logic
- `order.db`: Database access module
- `order.api`: REST API module
- `order.web`: Frontend controller

```
order.core/
└── module-info.java
order.db/
└── module-info.java
order.api/
└── module-info.java
order.web/
└── module-info.java
```
This is an example of module-info.java for `order.api`.
```java
module order.api {
    requires order.core;
    requires order.db;
    requires com.fasterxml.jackson.databind; // For JSON serialization
    exports com.mycompany.order.api;
}
```

By leveraging the module system, any packages not explicitly exported are hidden by default, ensuring strong encapsulation. Dependencies between modules are clearly declared using the requires directive, and each module can be developed, tested, and deployed independently. Additionally, the JVM can validate the module graph at startup, reducing runtime errors.

In conclusion, Java Modules offer at least four key advantages: **Enhanced Encapsulation**, **Clear Dependency Management**, **Faster Startup and Improved Security**, and **Better Scalability for Large-Scale Projects**.
