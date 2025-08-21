---
layout: post
title: "Spring Boot Circular Dependency Resolution: Three-Level Caching Explained"
date: 2025-07-21
categories: spring
published: true
description: "Learn how Spring Boot resolves circular dependencies using three-level caching (singletonObjects, earlySingletonObjects, singletonFactories), with AOP proxy handling."
keywords: ["spring boot circular dependency", "spring three level cache", "spring bean lifecycle", "spring boot aop proxy", "spring dependency injection"]
---

# How Spring Boot Uses Three-Level Caching to Solve Circular Dependencies
Circular dependencies in Spring are a common issue when two or more beans depend on each other. Spring can resolve *singleton* circular dependencies but **only for property-based injection** — **not constructor injection**. Spring uses a **three-level caching mechanism** to achieve this. Spring Boot reuses Spring's default strategy.

> Starting with Spring 6.0, constructor-based circular dependencies are no longer allowed by default, and this restriction is now enforced more strictly than in previous versions.

## Three-Level Caching in Spring
Spring manages singleton beans using three caches in the `DefaultSingletonBeanRegistry`.
### 1. `singletonObjects` – First-Level Cache  
Contains fully initialized and ready-to-use singleton bean instances.
### 2. `earlySingletonObjects` – Second-Level Cache
Holds "early references" to beans currently in creation. If a bean is subject to AOP, this may hold the proxy object; otherwise, it holds the actual bean instance.
### 3. `singletonFactories` – Third-Level Cache 
Stores `ObjectFactory` instances that used to lazily generate early references to beans (or proxies) when needed. This allows other beans to inject this bean during its creation phase.

> This structure helps avoid creating multiple proxies for the same bean when using AOP, which would be a risk with just two caches.

## Example: Circular Dependency Between BeanA and BeanB
Suppose `BeanA` and `BeanB` depend on each other via field injection:
1. Spring starts creating `BeanA`, and places an `ObjectFactory` for it in `singletonFactories`.
2. While populating `BeanA`, it needs `BeanB`, which is not created yet.
3. Spring begins creating `BeanB`, and during its dependency injection phase, it finds that `BeanA` is still in the creation phase.
4. Since `singletonFactories` has an `ObjectFactory` for `BeanA`, it calls `getObject()` to retrieve an early reference and moves it into `earlySingletonObjects`.
5. `BeanB` is now fully created and placed in `singletonObjects`.
6. Control returns to complete `BeanA`, which now has `BeanB`, and then `BeanA` is finalized and placed in `singletonObjects`.

## Key Methods Involved
### 1. `doGetBean()` — Entry Point
```java
Object sharedInstance = getSingleton(beanName);
//....
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> createBean(beanName, mbd, args));
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

#### `Object sharedInstance = getSingleton(beanName);`
- Checks the three-level cache to find if the bean already exists.
- Looks into:
  - `singletonObjects` (fully created beans),
  - `earlySingletonObjects` (early-exposed beans),
  - `singletonFactories` (can generate early references).

#### `if (mbd.isSingleton())`
- Ensures the bean's scope is singleton before proceeding.
- `mbd` is the `MergedBeanDefinition`, containing the bean's configuration.

#### `sharedInstance = getSingleton(beanName, () -> createBean(beanName, mbd, args));`
- Triggers bean creation using `ObjectFactory`, if not already present.
- Stores the factory in the third-level cache to support circular dependencies.

#### `bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);`
- Resolves the actual object to return.
- If it’s a `FactoryBean`, returns its product, not the factory itself.

### 2. `getSingleton()` — Cache Retrieval Logic
The `getSingleton(beanName, allowEarlyReference)` method is used to retrieve a singleton bean from Spring’s internal caches. It first checks the fully initialized singleton cache, then looks for early exposed beans that are still in the process of creation. If `allowEarlyReference` is `true` and the bean is still being created, it may retrieve an early reference via an `ObjectFactory` from the third-level cache. This mechanism is key to resolving circular dependencies between singleton beans.
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
                singletonObject = singletonFactory.getObject();
                this.earlySingletonObjects.put(beanName, singletonObject);
                this.singletonFactories.remove(beanName);
            }
        }
    }
    return singletonObject;
}
```

### 3. `createBean()` — Preparing for Circular Dependency Resolution
If the singleton bean is not found in the cache during the initial call to `getSingleton(beanName)`, Spring proceeds to create the bean using the `ObjectFactory` provided in the second call.
```java
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> createBean(beanName, mbd, args));
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```
This triggers the `createBean()` method, which eventually delegates to the `doCreateBean()` method in the `AbstractAutowireCapableBeanFactory` class. During this process, if circular dependency resolution is allowed and the bean is currently in creation, Spring **registers an early exposure factory** to the third-level cache via `addSingletonFactory()`.
```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    this.singletonFactories.put(beanName, singletonFactory);
    this.earlySingletonObjects.remove(beanName);
}
```
This call ensures that other beans encountering this one during their injection phase can access an early reference to it (or its proxy) even before it has been fully initialized. 

### 4. `getEarlyBeanReference()` — AOP-Aware Reference
```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                bean = ((SmartInstantiationAwareBeanPostProcessor) bp)
                         .getEarlyBeanReference(bean, beanName);
            }
        }
    }
    return bean;
}
```
This method is used to return an early reference to a bean, often a proxy, during its creation. It is invoked by the `ObjectFactory` registered in the third-level cache (`singletonFactories`) via `addSingletonFactory()`.

When another bean requires this one during dependency injection, Spring uses this method to provide an early reference, enabling **circular dependency resolution**.

If any registered `BeanPostProcessor` implements `SmartInstantiationAwareBeanPostProcessor`, it can decide whether a proxy is needed and wrap the original bean accordingly. This ensures only **one proxy is created**, even if the bean is injected before full initialization, avoiding issues with duplicated proxies and maintaining the singleton guarantee.

This mechanism is especially important in AOP scenarios, where proxies must be injected even before full bean initialization.

## Why Three Caches?
### Why Not Just Two Caches?
At first glance, two levels seem enough:
- singletonObjects – the final instance
- earlySingletonObjects – the "just-in-time" object exposed for injection

However, in circular dependency scenarios involving AOP, two caches are insufficient.
#### The Problem with Two Caches
Imagine two beans with circular dependencies:
```java
@Service
class A {
    @Autowired B b;
}

@Service
class B {
    @Autowired A a;
}
```
Assume that Spring must create a proxy for A.
Now, during creation:
1. Spring starts creating `A`, instantiates the raw object.
2. Before completing `A`, it needs to inject `B`, so it starts creating `B`.
3. `B` depends on `A`, but `A` is not yet fully initialized.

Without the third cache:
- Spring can only expose the raw object `A` via `earlySingletonObjects`.
- `B` gets this raw object injected.
- Later, during `A`'s post-processing phase, Spring detects AOP and creates a proxy (`proxyA`), placing it into `singletonObjects`.

**Result:**
- One place (`B`) has raw `A`, another has `proxyA`
- Singleton scope is broken

### The Solution: A Third-Level Cache
To address this, Spring introduces `singletonFactories`, the third-level cache. This stores a lazily executed `ObjectFactory`.
- During creation of `A`, Spring executes:
```java
addSingletonFactory("A", () -> getEarlyBeanReference("A", A));
```
- The factory is placed in `singletonFactories` **before dependency injection begins**.
- When `B` tries to inject `A`, Spring:
    - Misses in first-level and second-level caches,
    - Falls back to `singletonFactories`,
    - Executes the `ObjectFactory`, which internally:
        - Checks if `A` needs proxying
        - Wraps the raw object if needed (`wrapIfNecessary()`)
        - Produces `proxyA`
        - Puts `proxyA` into second-level cache `earlySingletonObjects`
- Now `B` receives `proxyA` during injection.
- When `A` finishes initialization, Spring finds the proxy already exists and uses it — no duplicate proxy creation.

>  **[Common Misconception]** Some believe that Spring uses three caches only for legacy reasons. In fact, the third cache (`singletonFactories`) is essential — it allows Spring to delay proxy creation and handle circular dependencies correctly and efficiently.

> There is no such thing as a “perfect” design — every solution involves trade-offs. While it is technically possible to reduce the number of caches, doing so would make the system more complex. You would need additional logic to determine whether a bean is fully created, whether it is a proxy, and whether it is safe to inject early. This would make the entire framework harder to understand and maintain.
