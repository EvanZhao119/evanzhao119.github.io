---
layout: post
title: "How a Spring Boot Web App Starts: Step-by-Step Explained"
date: 2025-04-10
categories: spring
published: true
---

# How a Spring Boot Web App Starts: Step-by-Step Explained
This article explains the full startup process of a Spring Boot Web application, from the `main()` method to the application being fully initialized. **I chose the older version(Spring Boot 1.5.7.RELEASE) because it is more understandable and fundational.** And I will provide some updates in Spring Boot 3.x.

## Step 0: App Entry Point
Everything starts from this `main()` method.
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## Step 1: Initializing SpringApplication
`SpringApplication.run()` creates a `SpringApplication` object and calls its `run()` method. Inside the constructor, `initialize()` prepares the setup.
```java
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
  return new SpringApplication(sources).run(args);
}
public SpringApplication(Object... sources) {
    initialize(sources);
}
```
### Key Actions in `initialize()`
- `setInitializers()`: Adds setup logic before context refresh.
- `setListeners()`: Registers listeners for app events.
- **(Updated in Spring Boot 3.x)**: Replaced `webEnvironment` with `WebApplicationType` enum (SERVLET, REACTIVE, NONE).
```java
private void initialize(Object[] sources) {
  if (sources != null && sources.length > 0) {
    this.sources.addAll(Arrays.asList(sources));
  }
  this.webEnvironment = deduceWebEnvironment();
  setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

## Step 2: Running the App
The code is a simplified version that retains the key parts, labeled from Step 3 to Step 9.
```java
public ConfigurableApplicationContext run(String... args) {
    listeners.starting();
    // Step 3: Parse application arguments and prepare the environment
    ApplicationArguments appArgs = new DefaultApplicationArguments(args);
    ConfigurableEnvironment env = prepareEnvironment(listeners, appArgs);
    // Step 4: Print the startup banner
    Banner banner = printBanner(env);
    // Step 5: Create the application context
    context = createApplicationContext();
    // Step 6: Prepare the context before refreshing
    prepareContext(context, env, listeners, appArgs, banner);
    // Step 7: Refresh the context (core Spring logic)
    refreshContext(context);
    // Step 8: Call custom runners after context initialization
    afterRefresh(context, appArgs);
    // Step 9: Notify listeners that startup is finished
    listeners.finished(context, null);
    return context;
}
```

## Step 3: Processing Command-Line Args and Environment
- `ApplicationArguments`: Parses command-line input.
- `ConfigurableEnvironment`: Handles environment config.

```java
ApplicationArguments appArgs = new DefaultApplicationArguments(args);
ConfigurableEnvironment env = prepareEnvironment(listeners, appArgs);
```
**(Updated)**: Spring Boot 3 uses `SpringApplicationBuilder` more for configuration.

## Step 4: Showing the Banner
Displays the startup banner.
```java
Banner banner = printBanner(env);
```
Customize it with.
```properties
banner.location=classpath:my-banner.txt
```

## Step 5: Creating the Context
Spring Boot creates a context (container), usually a web context.
```java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        contextClass = Class.forName(
            this.webEnvironment ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
}
```
**(Updated)**: New default is `AnnotationConfigServletWebServerApplicationContext`. 

## Step 6: Setting Up the Context
Before refreshing, it injects core values and beans.
```java
prepareContext(context, env, listeners, appArgs, banner);
```
**Tasks:**
- Apply initializers
- Register `ApplicationArguments` and others as beans
- Load source classes

## Step 7: Refreshing the Context
This is where Spring starts creating beans and composing everything.
```java
refreshContext(context);
((AbstractApplicationContext) context).refresh();
```
```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      prepareBeanFactory(beanFactory);

      try {
        // Allows post-processing of the bean factory in context subclasses.
        postProcessBeanFactory(beanFactory);

        // Invoke factory processors registered as beans in the context.
        // Convert annotated beans into BeanDefinition structures
        invokeBeanFactoryPostProcessors(beanFactory);

        // Register bean processors that intercept bean creation.
        // Register bean post-processors
        // AOP is implemented using the registered AnnotationAwareAspectJAutoProxyCreator
        registerBeanPostProcessors(beanFactory);

        // Initialize message source for this context.
        initMessageSource();

        // Initialize event multicaster for this context.
        initApplicationEventMulticaster();

        // Initialize other special beans in specific context subclasses.
        onRefresh();

        // Check for listener beans and register them.
        registerListeners();

        // Instantiate all remaining (non-lazy-init) singletons.
        // Create singleton beans
        finishBeanFactoryInitialization(beanFactory);

        // Last step: publish corresponding event.
        finishRefresh();
      }

      catch (BeansException ex) {
        if (logger.isWarnEnabled()) {
          logger.warn("Exception encountered during context initialization - " +
              "cancelling refresh attempt: " + ex);
        }

        // Destroy already created singletons to avoid dangling resources.
        destroyBeans();

        // Reset 'active' flag.
        cancelRefresh(ex);

        // Propagate exception to caller.
        throw ex;
      }

      finally {
        // Reset common introspection caches in Spring's core, since we
        // might not ever need metadata for singleton beans anymore...
        resetCommonCaches();
      }
    }
}
```
### What happens during `refresh()`:
1. `prepareRefresh()` prepares the context for refresh.
2. `obtainFreshBeanFactory()` creates or refreshs the internal `BeanFactory`.
3. `prepareBeanFactory(beanFactory)` configures the `BeanFactory` before it is used.
4. `postProcessBeanFactory(beanFactory)` is an extension point for subclasses.
5. `invokeBeanFactoryPostProcessors(beanFactory)` converts annotated components (@Component, etc.) into BeanDefinitions.
6. `registerBeanPostProcessors(beanFactory)` registers processors that intervene in bean creation.
7. `initMessageSource()` sets up internationalization (i18n) message handling.
8. `initApplicationEventMulticaster()` initializes the event publishing mechanism.
9. `onRefresh()` is an extension hook for subclasses (like web apps).
10. `registerListeners()` registers application event listeners.
11. `finishBeanFactoryInitialization(beanFactory)` creates all singleton beans.
12. `finishRefresh()` publish the `ContextRefreshedEvent`.

## Step 8: Running Custom Code
Spring Boot checks for `ApplicationRunner` or `CommandLineRunner` beans and runs them.
```java
protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
    callRunners(context, args);
}

private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<Object>();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<Object>(runners)) {
      if (runner instanceof ApplicationRunner) {
        callRunner((ApplicationRunner) runner, args);
      }
      if (runner instanceof CommandLineRunner) {
        callRunner((CommandLineRunner) runner, args);
      }
    }
}
```

## Step 9: Final Events
At the end, it sends either:
- `ApplicationReadyEvent` if startup was successful
- or `ApplicationFailedEvent` if something broke

```java
listeners.finished(context, null);
```

### Custom listener example
```java
@Component
public class SloganReadyEventListener implements ApplicationListener<ApplicationReadyEvent> {
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        //Here, you can do what you want to do.
    }
}
```
**(Updated)**: Prefer `@EventListener` annotation:
```java
@EventListener
public void onReady(ApplicationReadyEvent event) {
    // Custom logic here
}
```

## Step 10: App Is Running
So far, the app is running, with all beans ready and server listening for requests.

## Simplified Diagram

```
main() → SpringApplication.run()
      → initialize()
      → run()
         → prepareEnvironment()
         → printBanner()
         → createApplicationContext()
         → prepareContext()
         → refreshContext() → refresh()
         → afterRefresh() → callRunners()
         → listeners.finished()
```

## Key Changes from Spring Boot 1.5 to 3.x

| Feature                  | Spring Boot 1.5                             | Spring Boot 3.x (Updated)                              |
|--------------------------|---------------------------------------------|--------------------------------------------------------|
| Web Context Class        | `AnnotationConfigEmbeddedWebApplicationContext` | `AnnotationConfigServletWebServerApplicationContext`|
| Web Env Detection        | `webEnvironment` boolean                   | `WebApplicationType` enum                            |
| Event Listening          | Implements `ApplicationListener`           | Use `@EventListener`                             |
| Banner Support           | `banner.txt`, images                       | Unicode + styled banner support                   |
