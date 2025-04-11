---
layout: post
title: "Understanding AOP in Spring Boot: How Proxies Are Created During Bean Initialization"
date: 2025-04-11
categories: spring
published: true
---

# Understanding AOP in Spring Boot: How Proxies Are Created During Bean Initialization
Here is a clear explanation of how Spring Boot uses AOP (Aspect-Oriented Programming) to create dynamic proxy objects during the `refresh()` method. It walks through the full process of how and when these proxies are generated during bean initialization, and highlights changes and deprecated features in newer Spring Boot versions.

## Overview: Proxy Creation During `refresh()`
The `refresh()` method in the Spring container plays a key role in setting up the application context. As part of this process, the `finishBeanFactoryInitialization()` step uses reflection to create new bean instances. After the beans are created, Spring Boot fills in their properties and performs initialization. If the bean is affected by AOP (Aspect-Oriented Programming), Spring Boot will automatically create a proxy for it using either **JDK dynamic proxies (for interface-based beans) or CGLIB (for class-based beans)**.

## JDK vs. CGLIB Dynamic Proxies
- If the class implements an interface, **JDK dynamic proxy** is used.
- If it's a regular class without interfaces, **CGLIB** is used to create a subclass-based proxy.

| Criteria                         | JDK Dynamic Proxy                     | CGLIB Proxy                             |
|----------------------------------|--------------------------------------|-----------------------------------------|
| Requirement                     | Target must implement an interface   | Target can be any class (non-final)     |
| Proxy Creation Mechanism        | `java.lang.reflect.Proxy`            | Bytecode generation via CGLIB           |
| Performance                     | Slightly faster for interfaces       | Slightly slower; more flexible          |

> **Note**: From Spring 5, CGLIB is integrated into Spring Core and does not require an external dependency.

## Enabling AOP in Spring Boot
Spring Boot enables AOP by default when you include the `spring-boot-starter-aop` dependency. This starter brings in **spring-aop** and **aspectjweaver** libraries.

### Proxy Creation in `refresh()` Flow

```java
refresh() 
  -> finishBeanFactoryInitialization() 
    -> createBean() 
      -> doCreateBean() 
        -> initializeBean() 
          -> applyBeanPostProcessorsAfterInitialization() 
            -> postProcessAfterInitialization() (by AnnotationAwareAspectJAutoProxyCreator)
```
### `registerBeanPostProcessors()` Method
This method registers `AnnotationAwareAspectJAutoProxyCreator`, a special `BeanPostProcessor` responsible for creating proxies.
```java
beanFactory.getBean(ppName, BeanPostProcessor.class);
```

### `doCreateBean()` Method
The heart of bean instantiation. Uses `createBeanInstance()` via reflection, then proceeds with:
- `populateBean()` - Field injection using processors like `AutowiredAnnotationBeanPostProcessor`
- `initializeBean()` - Includes AOP proxy creation

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
      throws BeanCreationException {
    if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    //...
    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
      populateBean(beanName, mbd, instanceWrapper);
      if (exposedObject != null) {
        exposedObject = initializeBean(beanName, exposedObject, mbd);
      }
    }
    // ...

    return exposedObject;
}
```

### `populateBean()` Method
The `populateBean()`` method handles dependency injection for newly created bean instances. It processes annotations like `@Autowired` and `@Value`, ensuring all required fields, setters, and config values are properly assigned before the bean is fully initialized.

During this step, Spring iterates through all registered BeanPostProcessors—especially `InstantiationAwareBeanPostProcessor` implementations like `AutowiredAnnotationBeanPostProcessor`.

A key internal class here is `AutowiredFieldElement`, which injects values into fields marked with `@Autowired`, using reflection:
```java
field.set(bean, value);
```
This sets a (possibly private) field on the bean using a resolved dependency from the Spring context.

#### Key Steps:
- **Scanning Metadata**: Spring finds all fields annotated with `@Autowired` or `@Value`.
- **Resolving Dependencies**: It uses `resolveDependency()` to fetch matching beans from the container.
- **Injecting via Reflection:**
```java
ReflectionUtils.makeAccessible(field);
field.set(bean, resolvedDependency);
```
- **Caching**: Injection metadata is cached to avoid re-scanning on repeated injections (e.g., for singletons).

### `initializeBean()` Method
The `initializeBean()` method is the final step in the Spring bean lifecycle before the bean is considered ready for use. It performs three key operations:
1. **Invokes Aware interfaces** (e.g., `BeanNameAware`, `ApplicationContextAware`)
2. **Calls initialization methods** (like `@PostConstruct` or `afterPropertiesSet()`)
3. **Applies `BeanPostProcessors` after initialization, where AOP proxy creation happens**
### AOP Integration Happens Here
The crucial step for AOP is:
```java
wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
```
This line tells Spring to apply all registered `BeanPostProcessors` to the bean after it has been initialized. **Among them is the `AnnotationAwareAspectJAutoProxyCreator`, which is responsible for creating AOP proxies.**

Internally, this process looks like:
```java
AnnotationAwareAspectJAutoProxyCreator 
   ⟶ AbstractAutoProxyCreator.postProcessAfterInitialization()
```
If the bean matches AOP criteria (e.g., it has methods annotated with `@Around`, `@Before`, `@After`, etc.), this method will:

- **Determine if a proxy is needed** (`wrapIfNecessary()`)
- **Build a proxy object** using JDK dynamic proxy or CGLIB
- **Return the proxy instead of the original bean**

So, after `initializeBean()`, the object stored in the Spring container is no longer the raw bean, but a proxy that wraps it and intercepts method calls to apply cross-cutting logic (like logging, transactions, etc.).

> In short: `initializeBean()` completes the setup of the bean and applies AOP if necessary, ensuring any method with advice will be executed through a proxy.

## AOP Proxy Creation Logic Summary
### `wrapIfNecessary()`
Determines if a proxy is needed for the bean. If yes:
- Constructs `PointcutAdvisor` instances (e.g., `InstantiationModelAwarePointcutAdvisorImpl`)
- Calls `createProxy()`

### `createProxy()`
Creates the proxy via `ProxyFactory`. It uses `JdkDynamicAopProxy` or `ObjenesisCglibAopProxy`:
```java
return proxyFactory.getProxy(getProxyClassLoader());
```

### `DefaultAopProxyFactory.createAopProxy()`
Determines proxy mechanism:
```java
if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
    return new JdkDynamicAopProxy(config);
} else {
    return new ObjenesisCglibAopProxy(config);
}
```
> **Deprecated Note**: Spring Boot 1.5.x relied heavily on external CGLIB. As of Spring 5, internal support for CGLIB (via ASM) is used and the old `cglib-nodep` is no longer necessary.

### `CglibAopProxy.getProxy()`
Generates the subclass via ASM bytecode generation:
```java
enhancer.setSuperclass(proxySuperClass);
enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
return enhancer.create();
```

## Debugging Order Summary (Updated for Spring 5+)
This table serves as a quick reference to help you locate the key methods involved in Spring Boot's AOP proxy creation process. It's especially useful when debugging the bean initialization flow, allowing you to trace the sequence of method calls across core Spring classes — from `refresh()` all the way to the proxy object being created.

| Class                                 | Method                                      |
|--------------------------------------|---------------------------------------------|
| AbstractApplicationContext           | `refresh()`                                 |
| DefaultListableBeanFactory           | `preInstantiateSingletons()`                |
| AbstractBeanFactory                  | `doGetBean()`                               |
| AbstractAutowireCapableBeanFactory   | `createBean()`                              |
| AbstractAutowireCapableBeanFactory   | `doCreateBean()`                            |
| AbstractAutowireCapableBeanFactory   | `initializeBean()`                          |
| AbstractAutowireCapableBeanFactory   | `applyBeanPostProcessorsAfterInitialization()` |
| AbstractAutoProxyCreator             | `postProcessAfterInitialization()`          |
| AbstractAutoProxyCreator             | `wrapIfNecessary()`                         |
| AbstractAutoProxyCreator             | `createProxy()`                             |
| ProxyFactory                         | `getProxy()`                                |
| CglibAopProxy / JdkDynamicAopProxy   | `getProxy()`                                |

## A Summary At Last
Spring Boot uses a sophisticated and extensible AOP mechanism. During bean initialization in the `refresh()` method.
- Beans are created and populated
- `BeanPostProcessors` like `AnnotationAwareAspectJAutoProxyCreator` determine if a bean requires proxying
- AOP proxies are created using JDK dynamic proxies or CGLIB, depending on interface presence

> **Updated for Spring Boot 2.x+**
>
> - **CGLIB Integration**: Now part of Spring Core, no need for separate dependencies
> - **Proxy Caching & Efficiency**: Optimizations have improved bean startup time and proxy reuse
> - **AOT Compilation (Spring Boot 3)**: Native image support changes some dynamics (e.g., use of proxies with limitations)
