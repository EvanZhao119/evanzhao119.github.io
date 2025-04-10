---
layout: post
title: "Inside Spring: A Friendly Walkthrough of the Startup Process"
date: 2025-04-10
categories: spring
published: true
---

# Inside Spring: A Friendly Walkthrough of the Startup Process
> This guide breaks down how Spring Framework 6.x starts up using traditional XML configuration. We’ll walk through the `ClassPathXmlApplicationContext` flow and focus on **how beans are loaded**, **the refresh process**, and **how Spring handles circular dependencies with its three-level cache**.

## Step 0: Application Entry Point (`main()` Method)
This example demonstrates how to delegate object creation and dependency management to the Spring IoC container rather than using `new`. It exemplifies the core concepts of **Inversion of Control (IoC)** and **Dependency Injection (DI)**.
```java
public class SpringApplication {
    public static void main(String[] args) {
        // Initialize Spring context and load the XML configuration
        ApplicationContext context = new ClassPathXmlApplicationContext("application_context.xml");

        // Retrieve and use the bean
        User user = (User) context.getBean("user");
        user.setName("Evan");
    }
}
```

### `application_context.xml` Configuration File
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
         http://www.springframework.org/schema/beans
         https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- Define a bean named 'user' of type User -->
    <bean id="user" class="User" />
</beans>
```

### User Class
```java
public class User {
    private String name;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    @Override
    public String toString() {
        return "User{" + "name='" + name + '\'' + '}';
    }
}
```

## Step 1: Creating the `ApplicationContext`
Spring internally invokes the `refresh()` method to load the configuration file, parse Bean definitions, and initialize the IoC container.
```java
ApplicationContext context = new ClassPathXmlApplicationContext("application_context.xml");
```
```java

public ClassPathXmlApplicationContext(
    String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
      throws BeansException {
        super(parent);
        setConfigLocations(configLocations);
        if (refresh) {
          refresh();
        }
}
```

## Step 2: Entering `refresh()` — The Core of Context Initialization
The `refresh()` method organizes the entire lifecycle of the container. It handles BeanDefinition parsing, Bean instantiation, event broadcasting, and other critical processes.

```java
public void refresh() throws BeansException {
    synchronized (this.startupShutdownMonitor) {
        prepareRefresh(); // Initialize context state, set timestamps

        // Load and parse Bean definitions from XML
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        prepareBeanFactory(beanFactory); // Set default class loader, register default beans

        try {
            postProcessBeanFactory(beanFactory); // Extension hook for subclasses

            // Register Bean definitions parsed from XML
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register interceptors to hook into the Bean lifecycle
            registerBeanPostProcessors(beanFactory);

            initMessageSource(); // Initialize internationalization resources
            initApplicationEventMulticaster(); // Set up event broadcaster
            onRefresh(); // Extension hook
            registerListeners(); // Register event listeners

            // Instantiate all non-lazy singleton Beans (including "user")
            finishBeanFactoryInitialization(beanFactory);

            finishRefresh(); // Publish refresh-complete events
        } catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        } finally {
            resetCommonCaches(); // Clear reflection metadata
        }
    }
}
```

## Step 3: Retrieving a Bean from the `Context`
After container initialization, we use `getBean()` to retrieve the specific Bean instance from the internal BeanFactory.
```java
User user = (User) context.getBean("user");
```
Since the "user" bean was already created during container initialization, it will be retrieved directly from cache.

```java
//Invocation chain

AbstractApplicationContext#getBean()
    ↓
getBeanFactory().getBean()
    ↓
AbstractBeanFactory#doGetBean()
```

## Step 4: The Core Method `doGetBean()`
This method is the main entry point for retrieving or creating a bean. It handles singleton cache access and bean instantiation logic.
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    final String beanName = transformedBeanName(name); // Handle aliases
    Object bean;

    // Attempt to retrieve the bean from singleton cache
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // Bean found in cache — wrap and return
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // Full creation path (not triggered here)
    }

    return (T) bean;
}
```

## Step 5: The Three-Level Cache Mechanism in Spring (Core Concept)
```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // Level 1: Fully initialized singleton cache
    Object singletonObject = this.singletonObjects.get(beanName);

    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // Level 2: Early-exposed instances (created but not fully initialized)
            singletonObject = this.earlySingletonObjects.get(beanName);

            if (singletonObject == null && allowEarlyReference) {
                // Level 3: ObjectFactory for delayed construction
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject(); // Create early instance
                    this.earlySingletonObjects.put(beanName, singletonObject); // Promote to Level 2
                    this.singletonFactories.remove(beanName); // Remove from Level 3
                }
            }
        }
    }

    return singletonObject;
}
```
### Why Is the Three-Level Cache Needed?
To resolve **circular dependencies**, especially in setter or field-based injection.
#### Example
```java
public class A {
    public A(B b) {}
}
public class B {
    public B(A a) {}
}
```
- Constructor-based injection would fail due to infinite recursion.
- But with setter injection, Spring can create A and B separately, and **expose their early references** before full property population, using the three-level cache.

### Summary of Three-Level Cache
| Cache | Purpose | Lifespan |
|-------|---------|----------|
| `singletonObjects` | Fully initialized Beans | Persistent |
| `earlySingletonObjects` | Partially initialized Beans | Temporary |
| `singletonFactories` | Factory to create early references | Removed after use |

Spring "exposes incomplete objects" deliberately — to let dependent Beans hold a placeholder reference. Once all dependencies are injected, the object is finalized.

## Step 6: Final Bean Resolution
```java
bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
```
If the object is not a `FactoryBean`, it is returned directly. Otherwise, Spring invokes `getObject()` on the factory to get the actual bean.
```java
if (!(beanInstance instanceof FactoryBean)) {
    return beanInstance;
}
```

## Step 7: Using the Bean
At this point, we’ve successfully retrieved a Bean created and managed by the Spring container.
```java
user.setName("Evan");
```

## Summary Flow
```
main()
  → new ClassPathXmlApplicationContext()
    → refresh()
        → obtainFreshBeanFactory()
        → invokeBeanFactoryPostProcessors()
        → registerBeanPostProcessors()
        → finishBeanFactoryInitialization()
            → create Bean (uses 3-level cache)
  → getBean("user")
    → getSingleton() → return Bean from cache
```

## Key Takeaways

| Phase | Method | Purpose |
|-------|--------|---------|
| Load Configuration | `setConfigLocations()` | Set path to XML file |
| Initialize Container | `refresh()` | Master method for context lifecycle |
| Register Beans | `invokeBeanFactoryPostProcessors()` | Register `BeanDefinition` |
| Instantiate Beans | `finishBeanFactoryInitialization()` | Create all non-lazy singletons |
| Retrieve Bean | `getBean()` + `getSingleton()` | Fetch from cache or create |
| Handle Circular Dependency | `singletonObjects / earlySingletonObjects / singletonFactories` | Expose early references to break dependency loops |

## Spring 6.x and Spring Boot 3.x Updates

### 1. Java Configuration is Now the Preferred Approach
While Spring 6 still supports XML-based configuration, the mainstream practice has shifted to Java annotations and configuration classes.
```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {}

public class SpringApplication {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        User user = context.getBean(User.class);
        user.setName("Evan");
        System.out.println(user);
    }
}
```

### 2. Migration from `javax.*` to `jakarta.*` (Major Change)
Spring 6 has fully migrated to the Jakarta EE namespace, which replaces the legacy `javax.*` packages. You must update your dependencies accordingly.
```xml
<dependency>
    <groupId>jakarta.annotation</groupId>
    <artifactId>jakarta.annotation-api</artifactId>
    <version>2.1.1</version>
</dependency>
```
Updating import Statements.
```java
//Original (Before Migration):
import javax.annotation.PostConstruct;
```
```java
//Updated (After Migration to Jakarta EE)
import jakarta.annotation.PostConstruct;
```

### 3. AOT Compilation in Spring Boot 3
Spring Boot 3 introduces **Ahead-of-Time (AOT) compilation**, which improves startup time and memory usage, especially in GraalVM native image scenarios. This shifts some of the Bean registration and metadata analysis from runtime to build time for better performance.
