---
layout: post
title: "Understanding the Two Variations of the `while` Loop in Java"
date: 2025-03-19
categories: java
published: true
---

# Understanding the Two Variations of the `while` Loop in Java

In Java, there are two types of `while` loops:

1. **`while` loop (pre-test loop)**
2. **`do-while` loop (post-test loop)**

The key difference between them is **when the condition is checked**. In a `while` loop, the condition is tested before executing the loop body, while in a `do-while` loop, the loop body is executed at least once before the condition is checked.

## 1. `while` Loop (Pre-Test Loop)

The `while` loop first checks the condition before executing the block of code. If the condition is `false` at the start, the loop never runs.The loop executes **only if** the condition is `true` at the beginning.

### Syntax:
```java
while (condition) {
    // Code block to be executed
}
```

## 2. `do-while` Loop (Post-Test Loop)

The `do-while` loop executes the body first and **then** checks the condition. This guarantees at least **one execution**, even if the condition is initially `false`.The loop body **always executes at least once** before checking the condition.

### Syntax:
```java
do {
    // Code block to be executed at least once
} while (condition);
```

## 3. `while` vs `do-while` in Real Scenarios

### Reading a File Using `while`

If the file is empty, the loop won’t run at all.

```java
import java.io.*;

public class FileReadingWhile {
    public static void main(String[] args) {
        try {
            BufferedReader reader = new BufferedReader(new FileReader("test.txt"));
            String line = reader.readLine(); // Read the first line

            while (line != null) { // Check before reading
                System.out.println(line);
                line = reader.readLine();
            }

            reader.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### Menu System Using `do-while`

The menu **must be displayed at least once** before exiting.

```java
import java.util.Scanner;

public class MenuExample {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        int choice;

        do {
            System.out.println("Menu:");
            System.out.println("1. Start Game");
            System.out.println("2. View Score");
            System.out.println("3. Exit");
            System.out.print("Enter your choice: ");
            choice = scanner.nextInt();

            switch (choice) {
                case 1 -> System.out.println("Starting Game...");
                case 2 -> System.out.println("Your score: 100");
                case 3 -> System.out.println("Exiting...");
                default -> System.out.println("Invalid choice. Try again.");
            }

        } while (choice != 3); // Ensure the menu appears at least once

        scanner.close();
    }
}
```

## 4. Conclusion

- **Use `while`** when a loop should **not run** if the condition is false from the beginning.
- **Use `do-while`** when the loop **must execute at least once**, such as in user interaction scenarios.
- Understanding when to use each loop can help you write more **efficient and bug-free** code.

