---
title: "Consuming nested payloads? Work smarter, not harder."
date: 2024-06-26
tags:
  - spring
  - spring boot
  - jackson
  - JsonPath
  - REST APIs
categories:
  - Spring
math: true
---

_In this article, I start by examining the problems of consuming deeply nested JSON APIs using Jackson. I then introduce a solution using
JsonPath, and refine it into a declarative API by tapping into the underlying Spring messaging framework. The final result is a powerful,
yet simple and expressive API._

If you work with JSON APIs, you've probably encountered cases where the payload you needed to deserialize was deeply nested. Nesting is a
natural part or JSON, and is often a sign of healthy future-proofing. However, our applications often do not need to worry about
JSON-specific nesting. Isolating the JSON structure from the application layer is a good practice, since it simplifies the code that deals
with business logic and decouples it from any evolution of the API.

In this article, I will start by discussing the typical Jackson/Spring Boot pattern for deserializing JSON. I will then introduce JsonPath
as a way of simplifying JSON retrieval. Finally, I will build on this solution to propose a declarative API that leverages the JsonPath DSL
and Spring's extension points to solve the pain points that the first pattern brings. This will also give us an opportunity to explore
reflection and data binding in Spring.

As an example of a complex nested API, I'll
use [Cloudtrail log files](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitor-with-cloudtrail.html), which contain a wealth of
information. For this article, we'll pretend that we're
consuming this API, but we're only interested in knowing who (the username)
terminated which EC2 instances (the instance ids), and at what time (the event's timestamp). Here is an abbreviated typical Cloudtrail log:

[//]: # (@formatter:off)
```json
{
  "Records": [
    {
      "eventVersion": "1.03",
      "userIdentity": {
        "type": "Root",
        "principalId": "123456789012",
        "arn": "arn:aws:iam::123456789012:root",
        "accountId": "123456789012",
        "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
        "userName": "user" // we need this
      },
      "eventTime": "2016-05-20T08:27:45Z", // we need this
      //snip
      "responseElements": {
        "instancesSet": {
          "items": [
            {
              "instanceId": "i-1a2b3c4d" // we need this
              //snip
            }
          ]
        }
      }
      //snip
    }
  ]
}
```
[//]: # (@formatter:on)

# Level 1: DTO+mapper

This is the vanilla pattern for consuming APIs with Spring (Boot) and Jackson. You create classes, which can even be private to the class
consuming the message, and structure them in a way that Jackson will correctly map the payload to the class. You then convert it to a more
domain-oriented class using a mapper:

[//]: # (@formatter:off)
```java
// the DTO
record CloudTrailLogDTO(@JsonProperty("Records") List<CloudtrailRecord> records) { }

record CloudtrailRecord(OffsetDateTime eventTime, RequestParameters requestParameters) { }

record RequestParameters(UserIdentity userIdentity, InstanceSet instanceSet) { }

record UserIdentity(String arn) { }

record InstanceSet(List<Item> items) { }

record Item(String instanceId) { }

// the domain object
@Builder
private record CloudTraiLog(String userArn,
                            OffsetDateTime eventTime,
                            List<String> instanceIds) { }

// the mapper
class CloudtrailLogMapper {
  private CloudTraiLog map(CloudTrailLogDTO cloudtrailLogDTO) {
    CloudtrailRecord firstRecord = cloudtrailLogDTO.records().get(0);

    return CloudTraiLog.builder()
      .userArn(firstRecord.userIdentity().arn())
      .eventTime(firstRecord.eventTime())
      .instanceIds(firstRecord.requestParameters()
        .instancesSet()
        .items()
        .stream()
        .map(Item::instanceId)
        .toList())
      .build();
  }
}
```
[//]: # (@formatter:on)

This is how I first learned to deserialize JSON in Spring. It's a standard practice and has a few benefits that make it so common. First,
it's easy. You just write classes, and Jackson will map them to a nested structure. And second, it enforces a clear separation between your
presentation layer and your domain.

But as you can see, it's not the most readable code. This pattern gets quite verbose as your payloads grow in complexity. In this case, 7
classes are required just to convert the payload to a domain type so that the application can deal with it. We also need to iterate on all
the `items` to retrieve the `instanceId` out of each one.

It also doesn't scale well. If you start consuming other similar payloads, for example to monitor ECS cluster creations instead of EC2
instance terminations, you'll either have to reuse some of those objects and create an entanglement of DTOs that belong to multiple APIs, or
you'll have to duplicate the common classes. Having 7 classes is still manageable, but if a requirement suddenly comes up to consume 3
different CloudTrail event types, that's 21 classes!

Is all this ceremony really necessary? I think not.

## Let's talk dynamic

Let's take a step back and pretend that we're handling this payload in a dynamically typed language such as Javascript for a moment. All the
verbosity and ceremony around deserializing JSON quickly disappears when we think dynamic:

[//]: # (@formatter:off)
```js
const record = payload.Records[0];

const dto = {
  userArn: record.userIdentity.arn,
  eventTime: record.eventTime,
  instancesIds: record.requestParameters.instancesSet.items.map(item => item.instanceId)
};
```
[//]: # (@formatter:on)

This is much simpler, and as long as it's properly tested, I would be as comfortable with this as with the strongly typed Java version
above. The bad news is, it's not Java. But the good news is, we can bring some of this simplicity to our Java context by delegating the
type-awkward parts to a [domain-specific language](https://en.wikipedia.org/wiki/Domain-specific_language), or DSL.

# Deserializing with JsonPath

[JsonPath](https://datatracker.ietf.org/doc/rfc9535/) is a DSL focused on retrieving values from JSON. It's inspired
by [XPath](https://www.w3.org/TR/1999/REC-xpath-19991116/), a similar language aimed at addressing parts of an XML document. In Java, the
most popular implementation is [Jayway JsonPath](https://github.com/json-path/JsonPath). It ships with Spring Boot Test, but it's not on the
production classpath of a Spring Boot application by default, so we have
to [import it](https://mvnrepository.com/artifact/com.jayway.jsonpath/json-path).

To deserialize a JSON using JsonPath, we have to start from a `String`, a `byte[]` or an `InputStream` instead of the `Map<String, Object>`
or the other objects exposed by Jackson. One option is to just request one of these types in your controller and pass it to a factory method
in the DTO. Here's what this looks like:

```java

@PostMapping(value = "cloudtrail/logs/jsonpath")
void consumeLog(InputStream json) {
  service.consumeLog(new CloudTrailLogDTO(json));
}

@Getter
private static class CloudTrailLogDTO {

  private static final JsonPath USER_ARN_JSON_PATH =
    JsonPath.compile("$.Records[0].userIdentity.arn");
  private static final JsonPath EVENT_TIME_JSON_PATH =
    JsonPath.compile("$.Records[0].eventTime");
  private static final JsonPath INSTANCE_IDS_JSON_PATH =
    JsonPath.compile("$.Records[0].requestParameters.instancesSet.items[*].instanceId");

  private final String userArn;
  private final OffsetDateTime eventTime;
  private final List<String> instanceIds;

  private CloudTrailLogDTO(InputStream json) {
    DocumentContext documentContext = JsonPath.parse(json);
    this.userArn = documentContext.read(USER_ARN_JSON_PATH);
    this.eventTime = OffsetDateTime.parse(documentContext.read(EVENT_TIME_JSON_PATH));
    this.instanceIds = documentContext.read(INSTANCE_IDS_JSON_PATH);
  }
}
```

We specify the JsonPath expressions representing the parts of the JSON we want to store in the DTO's fields, and store those expressions
in `private static final` fields, so that they are compiled only once and can be reused for each deserialization. This pattern is a bit more
expressive, because the path expressions stand out and tell you right away which parts of the JSON go into which fields. It also decouples
the structure of the API from the structure of the DTO, so you don't necessarily need a separate domain type, or even a mapper. You might
also have noticed that we don't need to loop through the `items` anymore, since the `[*]` JsonPath operator takes care of it for us.[^1]
Despite all this, there is still a layer of indirection that slightly obscures the path-to-field relation by adding some less than desirable
boilerplate, and we can improve this.

This leads me to our final solution, which prominently displays the JsonPath expressions used for each field and also saves you from the
toil associated with manually compiling JsonPath expressions and assigning the values to each field.

# Declarative JsonPath mapping

Our final solution has our DTO looking like this:

```java
private record CloudTraiLogDTO(
  @JsonPath("$.Records[0].userIdentity.arn")
  String userArn,
  @JsonPath("$.Records[0].eventTime")
  OffsetDateTime eventTime,
  @JsonPath("$.Records[0].requestParameters.instancesSet.items[*].instanceId")
  List<String> instanceIds) {
}
```

This makes it immediately obvious what parts of the payload map to which fields. The DTO is also unencumbered by boilerplate. This means you
have less code to read when figuring out what it does. Maybe you wonder where all the boilerplate went. This is an example of using a
framework to stow away boilerplate or routine code and leave your application cleaner.

In my experience, moving complexity off of the critical path, meaning the code you read and work on when maintaining an application, always
results in an overall higher amount of complexity, but since some of it is stowed away, it results in a lower effective complexity, like if
the effective (cognitive) complexity of our code is the sum of its parts, weighted by how exposed each part is. A little like:

$$ C_{\text{effective}} = \sum_{i=1}^{n} w_i \cdot C_i $$

where:

<ul>
  <li>\( C_{\text{effective}} \) is the effective cognitive complexity of your application,</li>
  <li>\( C_i \) is the cognitive complexity of the \(i\)-th part of the code,</li>
  <li>\( w_i \) is the weight representing how exposed the \(i\)-th part is, and</li>
  <li>\( n \) is the total number of parts.</li>
</ul>

This explains why a small Spring Boot application can have a low cognitive complexity, meaning it's easy to understand and maintain, even
though it contains a significant amount of very complex code (think of all the dependencies you rely on). The exposed bit is just what you
own and maintain, yet you can still benefit from a lot of code you'll never even think about. I'm sure this has been discussed before, so if
you know of anything in the literature that covers this subject, please DM me.

End digression.

What I was saying is that to fulfill this elegant and simple API, we have to work a bit harder, but the code we write will stay out of our
way, so overall we're still making our application simpler to maintain.

Since from the framework's point of view, the object containing the `@JsonPath` annotation could be anything, we first need to inspect the
DTO's class to learn about its fields and their annotations, then parse and evaluate the JsonPath expressions, then convert the properties
as required (for example, from a `String` to an `OffsetDateTime`), and finally we need to construct a POJO containing the resolved
properties.

Spring makes this easier by supplying two useful classes: `ReflectionUtils`, and `DataBinder`. The first is a collection of
reflection-related methods, one of which, `doWithFields`, lets you run a callback on all fields of a class hierarchy. We'll use this to
inspect the fields and their annotations. The second is a class that lets
you [bind properties](https://docs.spring.io/spring-framework/reference/core/validation/beans-beans.html) to arbitrary classes, similar to
how properties are bound to `@ConfigurationProperties` classes in Spring Boot.

First, let's define a method to deserialize an `InputStream` into an object containing `@JsonPath` annotations. For this, we create a class
that we'll call `JsonPathDeserializer`, which will receive a `ConversionService` in its constructor. This class is Spring's conversion Swiss
army knife, and it acts like a parameterizable converter. We also initialize a field that will serve as a cache for the Json Path
expressions that we'll parse, so that we don't duplicate work. Then we inspect each field using `ReflectionUtils.doWithFields()`, and for
each JsonPath expression we find, we compile and cache it, we use it extract the value from the JSON, we convert it the field's type using
the `ConversionService`, and we store it in a map with the field's name as a key. Finally, we use `DataBinder` to construct the DTO using
the map we built. `DataBinder` can also inject properties using setters, but supporting immutable objects only should be enough.

Here's the complete deserializer:

```java
class JsonPathDeserializer {
  private final ConversionService conversionService;
  private final ConcurrentHashMap<String, com.jayway.jsonpath.JsonPath>
    documentContextCache = new ConcurrentHashMap<>();

  JsonPathDeserializer(ConversionService conversionService) {
    this.conversionService = conversionService;
  }

  <T> T deserialize(@Nullable String input, Class<T> clazz) {
    DocumentContext documentContext = com.jayway.jsonpath.JsonPath.parse(input);

    Map<String, Object> properties = new HashMap<>();

    ReflectionUtils.doWithFields(clazz, field -> {
      JsonPath expression = AnnotationUtils.findAnnotation(field, JsonPath.class);

      if (expression != null) {
        var jsonPath = documentContextCache.computeIfAbsent(
          expression.value(), ignored ->
            com.jayway.jsonpath.JsonPath.compile(expression.value()));
        Object value = documentContext.read(jsonPath);
        Object converted = conversionService.convert(value, field.getType());

        properties.put(field.getName(), converted);
      }
    });

    DataBinder dataBinder = new DataBinder(null);
    dataBinder.setTargetType(ResolvableType.forType(clazz));
    dataBinder.construct(new DataBinder.ValueResolver() {
      @Override
      public Object resolveValue(String name, Class<?> type) {
        return properties.get(name);
      }

      @Override
      public Set<String> getNames() {
        return properties.keySet();
      }
    });

    //noinspection unchecked
    return (T) dataBinder.getTarget();
  }
}
```

This can be used on its own, or in combination with a framework. The way you hook this into the framework depends on what kind of listener
you're working with, but the common logic stays the same. For this article, we're going to integrate it with Spring Web MVC. This pattern
can easily be applied to other type of listeners, like `@KafkaListener` or`@SqsListener`. For Spring Web MVC, we need a `WebMvcConfigurer`
to register a `HandlerMethodArgumentResolver`, an interface used by Spring to convert a request into an injectable argument.[^2]

The `HandlerMethodArgumentResolver` is also responsible for saying which arguments it applies to. I chose to support all arguments where at
least one field is annotated with our `@JsonPath` annotation. For this, we use the same `ReflectionUtils.doWithFields()` method. We could 
cache the
result, since a given type will either always match or always not match, but the result of `supportsParameter()` is already cached, and our
resolver shouldn't be called often, since we add it to the end of the framework's resolvers (in `resolvers.add()`, `add()`
appends to the list).

```java

@Component
@RequiredArgsConstructor
class JsonPathWebMvcConfigurer implements WebMvcConfigurer {

  private final ObjectMapper objectMapper;
  private final ObjectProvider<ConversionService> conversionService;

  @Override
  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    resolvers.add(new JsonPathHandlerMethodArgumentResolver(objectMapper,
      conversionService));
  }
}

class JsonPathHandlerMethodArgumentResolver implements
  HandlerMethodArgumentResolver {

  private final JsonPathDeserializer deserializer;

  JsonPathHandlerMethodArgumentResolver(ObjectMapper objectMapper,
                                        ObjectProvider<ConversionService> conversionService) {
    deserializer = new JsonPathDeserializer(objectMapper, conversionService);
  }

  @Override
  public boolean supportsParameter(MethodParameter parameter) {
    AtomicBoolean found = new AtomicBoolean();
    ReflectionUtils.doWithFields(parameter.getParameterType(),
      field -> {
        if (!found.get() && AnnotationUtils.findAnnotation(field, JsonPath.class) != null) {
          found.set(true);
        }
      });
    return found.get();
  }

  @SneakyThrows
  @Override
  public Object resolveArgument(MethodParameter parameter,
                                ModelAndViewContainer mavContainer,
                                NativeWebRequest webRequest,
                                WebDataBinderFactory binderFactory) {
    return deserializer.deserialize(requireNonNull(
        webRequest.getNativeRequest(HttpServletRequest.class)).getInputStream(),
      parameter.getParameterType());
  }
}

// by the way, this is our JsonPath annotation
@Retention(RetentionPolicy.RUNTIME)
@interface JsonPath {
  String value() default "";
}
```

And with these two classes, Spring Web MVC will inject all controller arguments that have fields annotated with our `@JsonPath`
annotation. Just don't annotate the argument with `@RequestBody`, because another argument resolver (`RequestBodyArgumentResolver`) will
return `true` from `supportsParameter` and our JsonPath resolver won't run.

# Conclusion

Consuming nested payloads can require lots of boilerplate and be tedious, especially if you're only interested in small segments of the
payload. In this article, I showed a way to simplify the DTOs by leveraging JsonPath to extract just the relevant sections. We used the
CloudTrail log API as an example, but many real-world APIs present consumers with this challenge, such as
Ticketmaster's [Discovery API](https://developer.ticketmaster.com/products-and-docs/apis/discovery-api/v2/), or the
[Amadeus API](https://developers.amadeus.com/self-service/category/flights/api-doc/flight-offers-price/api-reference) for flight pricing. We
also saw that hooking into the framework can be a good way to simplify our applications by hiding complexity far from the critical path of
our application code.

_The code from this article and a test suite can be found [here](https://github.com/LeMikaelF/jsonpath-deserialization-blog)._

-----

[//]: # (@formatter:off)
[^1]: JsonPath also supports filters, projections, array slices, recursive addressing, and more. Read more about it [here](https://support.smartbear.com/alertsite/docs/monitors/api/endpoint/jsonpath.html).
[^2]: Spring Kafka and Spring Cloud use a slightly different `HandlerMethodArgumentResolver` interface, this one from the Spring Messaging project.

[//]: # (@formatter:on)
