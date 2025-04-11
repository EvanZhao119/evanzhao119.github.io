---
layout: post
title: "Spring Cloud Config Server Explained: Behind the Scenes with Source Code"
date: 2025-04-11
categories: spring
published: true
---

# Spring Cloud Config Server Explained: Behind the Scenes with Source Code
A Spring Cloud Config Server serves as a central place to manage external configuration for applications across all environments. Here is a walkthrough of the source code behind a Spring Cloud Config Server instance and highlights the key components, initialization process, and request handling flow. 

## 1. Bootstrapping a Config Server
The following Java class launches a Config Server.
```java
@EnableEurekaClient
@EnableConfigServer
@SpringBootApplication
public class ConfigCenterApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigCenterApplication.class, args);
    }
}
```
**Note:** `@EnableEurekaClient` is used for service discovery with Eureka (which is now considered legacy). For new systems, Spring recommends using **Spring Cloud DiscoveryClient abstraction** or **Kubernetes-native service discovery**. It is not important here, because I only focus on `Config Server` implementation.

Before running the server, configuration is needed, typically in `application.yml`.

```yaml
spring:
  cloud:
    config:
      server:
        bootstrap: true
        native:
          search-locations: file:/Users/configs
```

## 2. `@EnableConfigServer` Annotation
This annotation marks the application as a Config Server. Under the hood, it only registers a `Marker` bean.
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(ConfigServerConfiguration.class)
public @interface EnableConfigServer {
}
```
```java
@Configuration(proxyBeanMethods = false)
public class ConfigServerConfiguration {
  @Bean
  public Marker enableConfigServerMarker() {
    return new Marker();
  }

  static class Marker {}
}
```

## 3. Spring’s SPI-like Mechanism: `spring.factories`
Spring Boot uses a mechanism similar to Java's Service Provider Interface (SPI), defined in `META-INF/spring.factories`. This loads auto-configuration classes during startup:
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration
=org.springframework.cloud.config.server.config.ConfigServerAutoConfiguration
```
`ConfigServerAutoConfiguration` is the main entry point.
- `@ConditionalOnBean(Marker.class)`: Ensures configuration activates only if `@EnableConfigServer` was declared.

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(ConfigServerConfiguration.Marker.class)
@EnableConfigurationProperties(ConfigServerProperties.class)
@Import({
  EnvironmentRepositoryConfiguration.class,
  CompositeConfiguration.class,
  ResourceRepositoryConfiguration.class,
  ConfigServerEncryptionConfiguration.class,
  ConfigServerMvcConfiguration.class,
  ResourceEncryptorConfiguration.class
})
public class ConfigServerAutoConfiguration {
}
```
- The `@Import(...)` annotation in `ConfigServerAutoConfiguration` brings several configuration classes into the Spring context. Each of them plays a specific role in how the Config Server operates.
    - `EnvironmentRepositoryConfiguration`: Sets up the environment repository based on the configured backend (e.g., Git, local file system, SVN, JDBC, etc.).
    - `ResourceRepositoryConfiguration`: Registers a `ResourceRepository` bean to load configuration content as Spring Resource objects, supporting raw file-based access.
    - `ConfigServerEncryptionConfiguration`: Adds encryption and decryption capabilities. It registers the `EncryptionController` for handling secure config requests (e.g., encrypting or decrypting property values).
    - `ConfigServerMvcConfiguration`: Provides web-layer configuration. It registers the `EnvironmentController` which handles HTTP requests to fetch configuration properties.
- `@EnableConfigurationProperties(ConfigServerProperties.class)`: Binds `spring.cloud.config.server.*` from the config file to Java objects.

```java
@ConfigurationProperties("spring.cloud.config.server")
public class ConfigServerProperties {
  private boolean bootstrap;
  private String prefix;
  private String defaultLabel;
  // Other properties omitted for brevity
}
```

## 4. Environment Repository Configuration
This section handles loading configuration data from various backends (Git, filesystem, JDBC, etc.)
### `JGitFactoryConfig` – Initializes the Git-based repository factory
This configuration class is only loaded when the class `TransportConfigCallback` is present (usually when JGit is on the classpath). It defines a bean for `MultipleJGitEnvironmentRepositoryFactory`, which is responsible for creating Git-based environment repositories. This factory supports additional features like custom HTTP connections or SSH-based Git access.
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(TransportConfigCallback.class)
static class JGitFactoryConfig {
  @Bean
  public MultipleJGitEnvironmentRepositoryFactory gitEnvironmentRepositoryFactory(...) {
    // Initialize the JGit repository factory
    return new MultipleJGitEnvironmentRepositoryFactory(...);
  }
}
```
### `DefaultRepositoryConfiguration` – Creates the default environment
This class defines the default `EnvironmentRepository` bean used by the Config Server if no other implementation is provided. It builds a `MultipleJGitEnvironmentRepository` from the factory, which is capable of managing multiple Git locations and branches for configuration sources.
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(value = EnvironmentRepository.class)
class DefaultRepositoryConfiguration {
  @Bean
  public MultipleJGitEnvironmentRepository defaultEnvironmentRepository(...) throws Exception {
    return gitEnvironmentRepositoryFactory.build(environmentProperties);
  }
}
```
> While `JGit` is still available, Spring Cloud Config Server now offers native support for popular source control platforms like **Azure Repos**, **GitHub (using GraphQL API v4)**, and **GitLab**. These newer integrations usually work faster and more smoothly than traditional Git-over-HTTP methods.

## 5. HTTP Request Handling with `EnvironmentController`
This controller responds to HTTP GET requests for configurations.
```java
@RestController
@RequestMapping(method = RequestMethod.GET,
    path = "${spring.cloud.config.server.prefix:}")
public class EnvironmentController {

  @RequestMapping(path = "/{name}/{profiles}/{label:.*}", produces = MediaType.APPLICATION_JSON_VALUE)
  public Environment labelled(@PathVariable String name, @PathVariable String profiles,
                              @PathVariable String label) {
    return getEnvironment(name, profiles, label, false);
  }

  public Environment getEnvironment(String name, String profiles, String label,
                                    boolean includeOrigin) {
    name = name.replace("(_)", "/");
    label = label.replace("(_)", "/");
    
    Environment environment = this.repository.findOne(name, profiles, label, includeOrigin);
    
    if (!this.acceptEmpty && (environment == null || environment.getPropertySources().isEmpty())) {
      throw new EnvironmentNotFoundException("Profile Not found");
    }
    return environment;
  }
}
```
When a configuration file is requested, the request is handled by the `EnvironmentController`, specifically through the `labelled(...)` method. Every time a client sends a request, the controller fetches the latest configuration from the repository (such as Git or file system) and returns it dynamically.

This ensures that the client always receives the most up-to-date configuration without needing to restart the server.
