---
layout: post
title: "Spring Boot Bean Creation Deep Dive: finishBeanFactoryInitialization, preInstantiateSingletons, and doCreateBean Explained"
date: 2025-07-18
categories: spring
published: true
description: "Step-by-step guide to Spring Boot bean creation inside finishBeanFactoryInitialization(), covering preInstantiateSingletons(), getBean(), doCreateBean(), createBeanInstance(), and Spring Boot 3.x AOT optimizations."
keywords: ["spring boot bean creation", "finishBeanFactoryInitialization explained", "spring preinstantiatesingletons", "spring doCreateBean method", "spring getbean lifecycle", "spring boot 3 aot optimizations"]
---

# How Spring Boot Actually Creates Beans — A Step-by-Step Walkthrough via `finishBeanFactoryInitialization`
After a series of initialization steps in Spring Boot, the real process of **bean creation** begins in the `finishBeanFactoryInitialization()` method. This is the core of the Spring IoC container, where control over bean instantiation shifts from the user to the Spring framework.

## Starting Point: refresh() Method in Spring Boot Bean Lifecycle
Let’s take **a common example** where a bean is annotated with `@Component`. The process starts with the `refresh()` method in the `AbstractApplicationContext` class, which then calls:
```java
// Instantiate all remaining (non-lazy-init) singletons.
finishBeanFactoryInitialization(beanFactory);
```
This method marks the beginning of actual bean creation.

## finishBeanFactoryInitialization: Preparing for Bean Creation 
In the `finishBeanFactoryInitialization()` method, we focus on:
```java
// Instantiate all remaining (non-lazy-init) singletons.
beanFactory.preInstantiateSingletons();
```
This line triggers the instantiation of all remaining singleton beans that are not marked for lazy initialization. The method `preInstantiateSingletons()` belongs to the `DefaultListableBeanFactory` class.

## preInstantiateSingletons(): Eager Singleton Bean Instantiation in Spring Boot 
Here, for each non-abstract, non-lazy singleton bean, the framework ensures they are fully instantiated.
```java
@Override
public void preInstantiateSingletons() throws BeansException {
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Pre-instantiating singletons in " + this);
    }

    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                boolean isEagerInit = (factory instanceof SmartFactoryBean &&
                    ((SmartFactoryBean<?>) factory).isEagerInit());
                if (isEagerInit) {
                    getBean(beanName);
                }
            } else {
                getBean(beanName);
            }
        }
    }
}
```

## Core Bean Creation Logic: `getBean()` and `doGetBean()`
`getBean()` calls `doGetBean()`:
```java
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}
```
The `doGetBean()` method is the central logic shared by all bean types — whether user-defined, internal processor beans, or context-related infrastructure beans.
- Checking first-level and second-level caches (to manage circular dependencies)
- Resolving dependencies recursively
- Creating the bean if not found in cache

## Bean Creation: `createBean()` and `doCreateBean()`
The `createBean()` method is defined in the `AbstractAutowireCapableBeanFactory` class and serves as the entry point for creating a new bean instance. It delegates the actual work to `doCreateBean()`.

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args)
        throws BeanCreationException {
    Object beanInstance = doCreateBean(beanName, mbd, args);
    return beanInstance;
}
```
The `doCreateBean()` method handles most of the essential steps involved in creating a bean. This includes creating the bean instance, injecting its dependencies, and applying important Spring features such as AOP proxies and lifecycle callbacks.
- **Checks for pre-existing instance** in the cache (for circular dependency resolution)
- **Creates the bean instance** using constructors
- **Records the resolved type** in the metadata
- **Handles advanced Spring features** like property injection, lifecycle methods, and aspect weaving (done after this point)

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args)
        throws BeanCreationException {

    BeanWrapper instanceWrapper = null;

    // If this is a singleton bean, remove any cached instance from the factoryBeanInstanceCache.
    // This cache is used for resolving circular dependencies.
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }

    // If we don’t already have an instance wrapper, create a new instance of the bean.
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }

    // Get the actual bean object from the wrapper.
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);

    // Determine and record the target type (i.e., the actual class of the bean).
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;

    // The following steps will:
    // - Inject dependencies (via setters, fields, constructors)
    // - Handle aware interfaces (like BeanNameAware, ApplicationContextAware)
    // - Apply post-processors (e.g., @Autowired, AOP proxies)
    // - Trigger lifecycle callbacks (@PostConstruct, InitializingBean)

    // Additional steps for property population, AOP proxying, etc. follow...
    return bean;
}
```

## Instance Creation Strategy: `createBeanInstance()` and `instantiateBean()`
In `createBeanInstance()`, Spring determines how to instantiate the bean:
```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    return instantiateBean(beanName, mbd);
}
```
This reflects the **Strategy Pattern**, where `InstantiationStrategy` (usually `SimpleInstantiationStrategy`) defines how beans are constructed.
```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    return getInstantiationStrategy().instantiate(mbd, beanName, parent);
}
```

## Final Instantiation via Reflection
The actual instantiation uses the `instantiateClass()` method in the `BeanUtils` utility class.
```java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
    ReflectionUtils.makeAccessible(ctor);
    return ctor.newInstance(args);
}
```
**This uses Java reflection to call the constructor and create the object. All metadata for instantiation (class, constructor, parameters) comes from the `BeanDefinition`.**

## **Update: Spring Boot 3.x / Spring Framework 6.x**

### 1. Jakarta EE Migration:
Spring 6 and Boot 3 have fully migrated to **Jakarta EE namespaces**.
**For example**
- `javax.annotation.PostConstruct` → `jakarta.annotation.PostConstruct`
- `javax.persistence.*` → `jakarta.persistence.*`

### 2. Native Image Support (Spring AOT):
Spring Boot 3 introduces **AOT (Ahead-of-Time) Compilation** and native image support with GraalVM. Bean definitions and dependency graphs can be processed at build time to reduce startup time.

**New behavior in `finishBeanFactoryInitialization()`**:

When AOT mode is enabled, many beans are registered via precomputed metadata, skipping reflective instantiation paths like `instantiateClass()`.

### 3. `Instantiator` Abstraction:

Spring Framework 6 introduces `BeanInstantiator` abstraction for advanced instantiation strategies that are pluggable and friendly to AOT processing.
