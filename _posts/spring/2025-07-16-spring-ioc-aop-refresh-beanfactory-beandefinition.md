---
layout: post
title: "Spring IoC and AOP Explained in refresh(): How BeanFactory and BeanDefinition Work in Spring Boot"
date: 2025-07-16
categories: spring
published: true
description: "Learn how Spring IoC and AOP are initialized during the refresh() process, including BeanFactory creation, BeanDefinition registration, XML vs annotation-based configuration, and updates in Spring Boot 3.x." 
keywords: ["spring ioc explained", "spring aop refresh", "spring beanfactory creation", "spring beandefinition registration", "spring boot configuration classes", "spring boot 3 updates"]
---

# Understanding IoC and AOP in `refresh()`: How BeanFactory and BeanDefinition Power Your Application

The `refresh()` method is one of the most critical methods in the Spring framework. It serves as the **entry point for the entire application context lifecycle**, and it's also where **two core features of Spring—Inversion of Control (IoC)** and **Aspect-Oriented Programming (AOP)**—are initialized and integrated.

Both **Spring Framework** and **Spring Boot** utilize `BeanFactory` and `BeanDefinition` concepts similarly.

> ***Note (Spring Boot 3.x)***: Spring Boot has increasingly moved away from XML-based configuration in favor of annotations and Java-based configurations (`@Configuration`, `@ComponentScan`, etc.), making traditional XML bean definitions less common in modern applications.

## BeanFactory: The Core of IoC Container in Spring Framework and Spring Boot 
A `BeanFactory` is the **factory responsible for managing and instantiating beans** in Spring. During the execution of `refresh()`, the method `obtainFreshBeanFactory()` is called to retrieve or initialize the `BeanFactory`.

### In Spring Framework
```java
// From AbstractApplicationContext
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}
```
In `AbstractRefreshableApplicationContext` class, `refreshBeanFactory()` is implemented as follows:
```java
@Override
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    } catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source", ex);
    }
}
```
The default implementation of the `BeanFactory` in both Spring and Spring Boot is `DefaultListableBeanFactory`.

### In Spring Boot
In contrast, **Spring Boot's `GenericApplicationContext`** creates the `BeanFactory` **during context initialization**, not during the `refresh()` call.
```java
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException("GenericApplicationContext does not support multiple refresh attempts");
    }
    this.beanFactory.setSerializationId(getId());
}
```
Spring Boot initializes the `BeanFactory` earlier to optimize startup performance and make auto-configuration features easy.

## BeanDefinition: Metadata Driving Bean Creation in Spring IoC and Boot
A `BeanDefinition` is a **meta-description of a bean**. It includes the bean's class name, scope (singleton/prototype), properties, constructor arguments, and more. Spring parses XML configurations, Java annotations, or Java configuration classes to generate `BeanDefinition` objects.

These definitions are then used to create actual bean instances.

### XML-based Configuration Flow in Spring Framework 
Here’s how Spring processes XML-based bean definitions:
1. `obtainFreshBeanFactory()` is invoked, which internally calls `loadBeanDefinitions(beanFactory)`
2. This triggers a series of function calls leading to the parsing of the XML document.
#### Call Chain

| Class | Method |
|-------|--------|
| `AbstractXmlApplicationContext` | `loadBeanDefinitions(DefaultListableBeanFactory)` |
| `AbstractXmlApplicationContext` | `loadBeanDefinitions(XmlBeanDefinitionReader)` |
| `AbstractBeanDefinitionReader` | `loadBeanDefinitions(String... locations)` |
| `XmlBeanDefinitionReader` | `doLoadBeanDefinitions(InputSource, Resource)` |

#### Key Implementation
```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) {
    Document doc = doLoadDocument(inputSource, resource);
    int count = registerBeanDefinitions(doc, resource);
    return count;
}
```
Then, in `XmlBeanDefinitionReader`.
```java
public int registerBeanDefinitions(Document doc, Resource resource) {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```
Finally, in `DefaultBeanDefinitionDocumentReader`.
```java
protected void doRegisterBeanDefinitions(Element root) {
    this.delegate = createDelegate(getReaderContext(), root, parent);
    parseBeanDefinitions(root, this.delegate);
}
```
And at last. This is where each `<bean>` tag is transformed into a `BeanDefinition` object.
```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, "bean")) {
        processBeanDefinition(ele, delegate);
    }
}
```

### Annotation-based Configuration in Spring Boot (Auto-Configuration & Component Scan)
Spring Boot **does not rely on XML files by default**. Instead, it uses **annotation-driven configuration** with the `@Configuration` and `@ComponentScan` annotations.

The core registration logic moves to:
```java
// From AbstractApplicationContext
invokeBeanFactoryPostProcessors(beanFactory);
```
This is where Spring Boot auto-configuration classes and other `@Bean`-defined methods are scanned, parsed, and turned into `BeanDefinition` instances.

> Instead of XML files, use `@Configuration`, `@Component`, `@Service`, and `@Repository` annotations with component scanning in Spring Boot.

### Java Configuration Classes with @Configuration and @Bean in Spring Boot
Besides XML and annotation-based component scanning, **Java-based configuration using `@Configuration` classes** is another widely used approach for defining Spring beans.

This method uses **explicit Java classes and `@Bean` methods** to define beans.

#### Example
```java
@Configuration
public class AppConfig {

    @Bean
    public UserService userService() {
        return new UserServiceImpl();
    }

    @Bean
    public UserRepository userRepository() {
        return new UserRepositoryImpl();
    }
}
```
Each `@Bean` method returns a fully configured bean instance, and Spring internally registers a corresponding **`BeanDefinition`** for each method.

> Whether beans are defined via XML, annotations, or Java config, **Spring always converts them into `BeanDefinition` objects** during the context initialization phase.

### How It Works Internally: ConfigurationClassPostProcessor and BeanFactoryPostProcessors
- During the call to `refresh()`, Spring processes configuration classes through **`ConfigurationClassPostProcessor`**, a special `BeanFactoryPostProcessor`.
- It reads `@Configuration` classes, scans `@Bean` methods, and generates corresponding `BeanDefinition` entries.
- This process is triggered in the method `invokeBeanFactoryPostProcessors()` (same as for annotations).

### Summary

| Configuration Style | Description | Common In |
|---------------------|-------------|------------|
| XML-based           | Traditional, verbose, but highly explicit | Legacy projects |
| Annotation-based    | Uses `@Component`, `@Service`, etc. | Modern Spring apps |
| Java Configuration  | Uses `@Configuration` and `@Bean` methods | Spring Boot and test configurations |

## Summary of Key Differences Between Spring Framework XML and Spring Boot Annotations 

| Feature | Spring Framework (XML) | Spring Boot (Annotations) |
|--------|-------------------------|----------------------------|
| BeanFactory Creation | During `refresh()` | At context initialization |
| BeanDefinition Registration | `loadBeanDefinitions()` parses XML | `invokeBeanFactoryPostProcessors()` parses annotations |
| Default Configuration Style | XML + manual setup | Auto-config + annotations |
| Flexibility | Fully customizable | Convention over configuration |
