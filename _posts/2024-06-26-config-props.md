---
title: "How to merge Configuration Properties in Spring Boot"
date: 2024-06-26
tags:
  - spring
  - spring boot
categories:
  - Spring
---

_In this article, I introduce a pattern to reduce duplication between multiple `@ConfigurationProperties` instances of the same class, using
a set of shared properties that can be overridden using specific properties. This is useful for configuring multiple datasources, messaging
brokers, or any sort of configuration properties where there is a set of shared properties, and sets of overrides._

# The Problem

Suppose your application needs to talk to your database using two different users, or using the same user but two different endpoints. I've
had both these requirements come up over the last years, first to run queries using
different [workload prioritization](https://enterprise.docs.scylladb.com/stable/using-scylla/workload-prioritization.html) levels in
SyllaDB, which requires different users, and second split reads and writes to an RDBMS, sending writes to the writer, and reads to read
replicas. The problem you get when you try to implement this is that your complete configuration needs to be duplicated for each
user/endpoint variation. Let's say you have a database with a writer endpoint, a reader endoint, and an analytics user that has different
privileges:

```yaml
# writer datasource
spring:
  datasource:
    url: jdbc:postgresql://writer.mydb.io/posts
    username: application
    password: password


# reader datasource
datasources:
  reader:
    url: jdbc:postgresql://reader.mydb.io/posts
    username: application
    password: password
```

Here, the username and passwords are duplicated, meaning if I change one of them, I have to remember to update the other. But more
importantly in my opinion, it also obscures the intent. It is now harder to see that the reader has the exact same configuration as the
writer, except for its url. Code should scream its intent at you, you shouldn't need to decipher it. And configuration is code. So let's
improve this. In the next paragraphs, we're going to transform the configuration above to the code block below.

```yaml
spring:
  datasource:
    username: application
    password: password

custom:
  datasources:
    writer:
      url: jdbc:postgresql://writer.mydb.io/posts
    reader:
      url: jdbc:postgresql://reader.mydb.io/posts
```

For this, we're going to need to merge two sets of properties. There's a few ways of doing this, but I'm going to discuss what I think is
the simplest. But before this, we need to discuss how `@ConfigurationProperties` classes are initialized in Spring Boot.

# How Configuration Properties are initialized

When Spring creates a bean, and configuration properties are beans, one of the things it does to the object once it is instantiated, is to
run a series of `BeanPostProcessor`s on it. `BeanPostProcessor` is an interface consisting of these two methods:

```java
default Object postProcessBeforeInitialization(Object bean, String beanName)
    throws BeansException {
  return bean;
}

default Object postProcessAfterInitialization(Object bean, String beanName)
    throws BeansException {
  return bean;
}
```

These are two lifecycle hooks that the Spring Framework uses to perform arbitrary manipulations on beans, or even switch them out for
entirely new objects. This is how a number of container-related behaviours are implemented: injection of `@Autowired` fields,
instrumentation with `@Async`, and aspects, to name a few. Spring Boot uses a `BeanPostProcessor`,
namely `ConfigurationPropertiesBindingPostProcessor`
to inject fields into mutable configuration properties classes (classes that have setters).

The `ConfigurationPropertiesBindingPostProcessor` works by calling the relevant setter for each property it finds. If no matching property
is found, no setter is called, which is why default values that are set as field initializers, such as in the example below, don't get
overwritten to `null` when no property is specified.

```java
@Data
@ConfigurationProperties(prefix = "my-properties")
public class MyProperties {
  private String name = "default name";
}
```

Do you see where I'm going with this? If we have a fully initialized configuration properties class, we can run
the `ConfigurationPropertiesBindingPostProcessor` on it with different properties as often as we want, and the properties that were set
initially won't be affected by the next `BeanPostProcessor` runs. This is how we're going to merge our two sets of properties. And we're
going to do it as lazily as possible, meaning we're going to get the Spring Framework to do as much work as possible. But for this, we'll
need to examine one more concept: bean scopes.

# Bean Scopes

The majority of beans used by a modern Spring application have the default `singleton` scope. This means that whenever and wherever the
container injects the bean, it will always be the same instance. But it does not have to be so. Spring
supports [a few other scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html) out of the box. One of those
is the `prototype` scope. With a prototype-scoped bean, every injection will get a new object. For our objective, it means that we can
generate multiple instances of a configuration properties bean, and that each one will be independently injected by
the `ConfigurationPropertiesBindingPostProcessor`.

# Putting it all together

Our strategy will be the following:

1. generate initialized instances of our configuration properties class; and
2. for each instance, re-inject it with the property overrides.

This is actually pretty simple to do:

```java
@Configuration
class DataSourceConfig {

  @Bean(autowireCandidate = false)
  @Scope("prototype")
  DataSourceProperties commonDataSourceProperties() {
    return new DataSourceProperties();
  }

  @Bean
  @ConfigurationProperties(prefix = "custom.datasources.writer")
  DataSourceProperties writerDataSourceProperties() {
    return commonMyProperties();
  }

  @Bean
  @ConfigurationProperties(prefix = "custom.datasources.reader")
  DataSourceProperties readerDataSourceProperties() {
    return commonMyProperties();
  }
}
```

We first use Spring Boot's `DataSourceProperties` class, which is already annotated
with `@ConfigurationProperties(prefix = "spring.datasource")`. We make it a prototype bean, so that a new instance will be created every
time it is requested. We also hide it for autowiring using
the [`autowireCandidate`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Bean.html#autowireCandidate())
element, since it's not meant to be used elsewhere in the application. Then, we create two more beans that override the default prefix.
[^1] But instead of creating a new `DataSourceProperties` like we did for the first bean, we request an initialized instance, and return
that to the container, which in turn considers it to be a new bean and initializes it, only this time it's done with the overridden
properties. During this second injection, the post processor only sets the properties that it finds, so the ones that were previously set on
the object remain in place.

And that's how we get merged configuration properties!

# Testing

I recommend testing things like this, because we are entering the realm of "clever" code, or at least more unusual code. There's a few ways
to do this, but my favourite way to test things involving properties is using Spring Boot's `ApplicationContextRunner`. It was initially
designed
to [make testing autoconfiguration classes easier](https://spring.io/blog/2018/03/07/testing-auto-configurations-with-spring-boot-2-0), but
it works great for anything where you need to control the specifics of an application context, and where configuration itself is the main
target of the test. Here's how it works:

```java
class MergingTest {

  private final ApplicationContextRunner runner = new ApplicationContextRunner()
      .withUserConfiguration(DataSourceConfig.class)
      .withPropertyValues(
          "spring.datasource.url=common-url",
          "spring.datasource.username=common-username",
          "spring.datasource.password=common-password",
          "custom.writer.username=writer-username",
          "custom.reader.password=reader-password");

  @Test
  void merging() {
    runner.run(context ->
        assertThat(context)
            .getBean("writerDataSourceProperties", DataSourceProperties.class)
            .extracting(
                DataSourceProperties::getUrl,
                DataSourceProperties::getUsername,
                DataSourceProperties::getPassword)
            .containsExactly(
                "common-url",
                "writer-username",
                "common-password"));

    runner.run(context ->
        assertThat(context)
            .getBean("readerDataSourceProperties", DataSourceProperties.class)
            .extracting(
                DataSourceProperties::getUrl,
                DataSourceProperties::getUsername,
                DataSourceProperties::getPassword)
            .containsExactly(
                "common-url",
                "common-username",
                "reader-password"));
  }
}
```

This test ensures that properties are correctly overridden, and that the reader and writer beans are independent. For
instance, it will fail if someone removes the `@Scope("prototype")` annotation from the `commonDataSourceProperties` bean.

# Conclusion

We saw how to apply this pattern to data sources, but it can also be useful for anything where you want multiple configured objects that
share a common base, like HTTP connection factories and messaging brokers. This pattern is also flexible in regard to the layout of
properties. I chose to retain the default `spring.datasource` prefix for the common properties, but it would also have been possible to have
a `custom.datasources.common` prefix, which would have mirrored `custom.datasources.reader` and `custom.datasources.writer` nicely. All that
is required for this is an additional `@ConfigurationProperties(prefix = "custom.datasources.common")` on the prototype bean.

Finally, I picked a simple example, with only 3 properties and 2 sets of overrides. This pattern really shines with more complex setups,
where you might have many properties and many overrides. For example, below is a configuration for a service that had 3 datasources with
non-trivial configuration, to query a single SyllaDB database using 3 users attached to different workload prioritization levels:

```yaml
scylla:
  cluster:
    hosts: host1,host2,host3
    datacenter: datacenter1
    keyspace: objects
    port: 9042
  min-healthy-nodes: 1
  config:
    request-consistency: LOCAL_ONE
    request-timeout: 3
    tracing:
      enabled: true
  datasources:
    critical-priority:
      username: critical-priority-username
      password: critical-priority-password
    medium-priority:
      username: medium-priority-username
      password: medium-priority-password
      config:
        request-timeout: 15s
    low-priority:
      username: low-priority-username
      password: low-priority-password
      config:
        request-timeout: 60s
```

In conclusion, I hope this pattern can help make your configuration code more expressive and reduce cognitive overhead associated with
duplication. Happy coding!

---

[^1]: Spring Boot first searches for the `@ConfigurationProperties` annotation on the factory method (the `@Bean` method), then on the type, and finally on the superclasses and interfaces. This allows factory methods to override type annotations, and subclasses to override superclasses/interfaces. See `ConfigurationPropertiesBean::findAnnotation`.
