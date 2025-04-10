---
layout: post
title: "Spring Boot `refresh()` Function: `invokeBeanFactoryPostProcessors` in Action"
date: 2025-04-10
categories: spring
published: true
---

# Spring Boot `refresh()` Function: `invokeBeanFactoryPostProcessors` in Action
Spring Boot has largely replaced the traditional XML-based Spring configuration and has become the dominant framework in many modern software and Internet companies. 

Now we begin with the `refresh()` method of the `AbstractApplicationContext` class, trace the execution of the `invokeBeanFactoryPostProcessors()` method and explore how Spring Boot registers `BeanDefinition` structures. 

## Entry Point: `AbstractApplicationContext.refresh()`
This method initializes post-processing of the `BeanFactory` **after** it has been created, but **before** any beans are instantiated. One of its key roles is to register all `BeanDefinition` instances based on annotated classes like those with `@Configuration`, `@Component`, etc.
```java
invokeBeanFactoryPostProcessors(beanFactory);
```

## Delegation to `PostProcessorRegistrationDelegate`
```java
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
```
This static method first invokes all implementations of `BeanDefinitionRegistryPostProcessor`, which are used to **register `BeanDefinition` objects**, such as `ConfigurationClassPostProcessor` for processing `@Configuration` annotated classes.

### Code Fragment:
```java
List<BeanDefinitionRegistryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
    }
}
sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
registryPostProcessors.addAll(priorityOrderedPostProcessors);
invokeBeanDefinitionRegistryPostProcessors(priorityOrderedPostProcessors, registry);
```

## Step Into: `invokeBeanDefinitionRegistryPostProcessors()`
Among the post-processors, `ConfigurationClassPostProcessor` is a key player that processes classes annotated with `@Configuration`.
```java
private static void invokeBeanDefinitionRegistryPostProcessors(
    Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, 
    BeanDefinitionRegistry registry) {

    for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessBeanDefinitionRegistry(registry);
    }
}
```

## Key Logic in `postProcessBeanDefinitionRegistry()`
This method is called to register `BeanDefinition`s based on annotated configuration classes.
- It first generates a unique ID for the `registry` using `System.identityHashCode(registry)`.
- Then, it checks if this registry has already been processed by this post-processor.
    - If yes, it throws an `IllegalStateException` to avoid duplicate processing.
- If not, it marks the registry as processed by adding its ID to the `registriesPostProcessed` set.
- Finally, it calls `processConfigBeanDefinitions(registry)` to scan and register configuration classes (like those annotated with `@Configuration`, `@Component`, etc.).

This ensures that each `BeanDefinitionRegistry` is only processed once and safely initializes all configuration-related bean definitions.

```java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    int registryId = System.identityHashCode(registry);
    if (this.registriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(...);
    }

    this.registriesPostProcessed.add(registryId);
    processConfigBeanDefinitions(registry);
}
```

## Parsing @Configuration Classes
```java
ConfigurationClassParser parser = new ConfigurationClassParser(...);
parser.parse(candidates);
parser.validate();
```
- This creates a `ConfigurationClassParser` instance responsible for analyzing configuration classes.
- `parser.parse(candidates)` starts the parsing process for all `@Configuration`-annotated classes passed as `candidates`.
- After parsing, `parser.validate()` ensures the configuration is valid (e.g., checks for circular imports or invalid annotations).

### Recursive Analysis in `processConfigurationClass()` of `parse()`
- This method handles recursive parsing of a configuration class and its possible superclasses.
- It starts by converting the `ConfigurationClass` to a `SourceClass`, a wrapper that allows metadata inspection.
- Then it enters a loop, calling `doProcessConfigurationClass(...)` to process the configuration class and any additional imported or scanned configuration classes. 
- The loop continues as long as there is another class (e.g., a superclass or an imported config) to process.
```java
protected void processConfigurationClass(ConfigurationClass configClass) {
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    } while (sourceClass != null);
}
```

## Detailed Parsing with `doProcessConfigurationClass()`
This method processes `@ComponentScan` annotations inside a `@Configuration` class.
- It first retrieves all `@ComponentScan` annotations using `AnnotationConfigUtils.attributesForRepeatable(...)`.
- For each `@ComponentScan`, it uses `componentScanParser.parse(...)` to scan the specified packages and find candidate components (like classes annotated with `@Component`, `@Service`, etc.).
- Each found component is wrapped as a `BeanDefinitionHolder`.
- Then, it checks if any of these scanned classes are also `@Configuration` classes using `ConfigurationClassUtils.checkConfigurationClassCandidate(...)`.
    - If yes, it recursively parses them using the `parse()` method.

This enables Spring to discover and process nested or indirectly referenced configuration classes automatically.

```java
protected final SourceClass doProcessConfigurationClass(...) {
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(...);
    for (AnnotationAttributes componentScan : componentScans) {
        Set<BeanDefinitionHolder> scannedBeanDefinitions =
            this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());

        for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
            if (ConfigurationClassUtils.checkConfigurationClassCandidate(...)) {
                parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
            }
        }
    }
}
```

## Deep Dive: `ComponentScanAnnotationParser.parse()`
- This code creates a `ClassPathBeanDefinitionScanner`, which is responsible for scanning specified packages to find candidate components (e.g., classes annotated with `@Component`, `@Service`, `@Repository`, `@Controller`, etc.).
- The `basePackages` value is usually extracted from the `@ComponentScan` annotation.
- `scanner.doScan(...)` performs the actual scanning of these packages and returns a set of `BeanDefinitionHolder` objects representing the discovered beans.
- These beans will later be registered with the `BeanFactory`.
```java
ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(...);
return scanner.doScan(StringUtils.toStringArray(basePackages));
```

## Scanning Components with `ClassPathBeanDefinitionScanner`
```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    for (String basePackage : basePackages) {
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);

        for (BeanDefinition candidate : candidates) {
            // Assign scope, name, and proxy metadata
            // Register into BeanFactory
            registerBeanDefinition(definitionHolder, this.registry);
        }
    }
    return beanDefinitions;
}
```
- This method performs **component scanning** within the specified `basePackages`.
- For each base package:
    1. It calls `findCandidateComponents(basePackage)` to locate all eligible classes annotated with `@Component`, `@Service`, `@Repository`, etc.
    2. It then iterates over the discovered `BeanDefinition` objects.
- For each candidate:
    - It assigns scope metadata (like singleton or prototype).
    - Generates a bean name.
    - Applies any necessary proxy mode (such as for scoped beans).
    - Finally, it **registers the bean** using `registerBeanDefinition(...)`, which adds it to the Spring `BeanFactory`.

All valid component classes in the specified packages are converted to `BeanDefinition` objects and registered, so Spring can manage and inject them as needed.

## **Spring Boot 3.x Enhancements**
1. **Kotlin and Functional Bean Registration** (Optional):
   
In addition to `@Configuration`, Spring Boot 3.x also supports Kotlin DSLs and programmatic registration via `GenericApplicationContext.registerBean(...)`.

2. **Record Support**:
   
Java `record` classes can now be used directly as `@ConfigurationProperties` or `@Component` types, improving immutability and readability.

3. **Declarative Initialization with `ApplicationContextInitializer`**:
   
Although `ConfigurationClassPostProcessor` is still used, more initialization is now often handled using lambda-based registration or externalized configuration loading strategies for performance and simplicity.

4. **Module Scanning and Native Configuration**:

Spring Boot 3.x adds support for GraalVM and native images. To make applications start faster and use less memory, Spring now allows you to give explicit hints about which classes and methods should be kept at compile time. This is done using annotations like `@NativeHint`, `@ImportRuntimeHints`, and through the AOT (Ahead-Of-Time) compilation process. These tools help Spring avoid relying on runtime reflection, which native images don't support well.
