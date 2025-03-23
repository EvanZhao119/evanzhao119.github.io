---
layout: post
title: "Understanding Java Access Modifiers"
date: 2025-03-23
categories: java
published: true
---

# Understanding Java Access Modifiers

## Access Modifier Table

| Modifier     | Same Class | Same Package | Subclass | Other Package |
|--------------|------------|--------------|----------|---------------|
| **private**  | ✅         | ❌           | ❌       | ❌            |
| **default**  | ✅         | ✅           | ❌       | ❌            |
| **protected**| ✅         | ✅           | ✅       | ❌            |
| **public**   | ✅         | ✅           | ✅       | ✅            |

## 1. **`private` Access Modifier**
- **Visibility**: The **`private`** modifier restricts access to the members of the class only. **Only the class where it is declared can access it.**
- **Use Case**: It’s typically used to implement **encapsulation** by hiding the details of an object.
  
### Example:
```java
class MyClass {
    private int value;  // Only accessible within MyClass

    private void displayValue() {
        System.out.println(value);
    }

    public void setValue(int value) {
        this.value = value;  // Allowed, because it's within the same class
    }
}
```

## 2. `default` (Package-Private) Access Modifier
- **Visibility**: When no access modifier is specified, it has **default** access (package-private). Members with this modifier are **only accessible within classes in the same package.**
- **Use Case**: It is used when you want to restrict access to a package but allow any class within that package to access it.

## 3. `protected` Access Modifier

- **Visibility**: The protected modifier allows access to members from: **The same package and Any subclass (even if it’s in a different package).**
- **Use Case**: `protected` is used when a member should be available to subclasses and package members, but not to other classes outside the package and inheritance hierarchy.

## 4. `public` Access Modifier

- **Visibility**: The `public` modifier allows members to be accessible from any other class in any package.
- **Use Case**: `public` is used when you want to make a class, method, or variable accessible globally. This is the most permissive modifier.


## 5. Best Practices
- **Encapsulation**: Use `private` for fields and methods to restrict access and provide public getter/setter methods if needed.
- **Package-level visibility**: Use the default access modifier for classes and methods that should only be used within the same package.
- **Inheritance**: Use `protected` for members that should be accessed by subclasses.
- **Global access**: Use `public` only when you need to expose functionality to the whole project or the external world.
