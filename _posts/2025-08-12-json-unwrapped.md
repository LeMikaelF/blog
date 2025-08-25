---
title: "@JsonUnwrapped, the unsung hero"
date: 2025-08-12
tags:
  - java
  - jackson
categories:
  - Java
---

Jackson is the most popular Java JSON library. It is well integrated with Spring
Boot, and it has an extensive feature set. Like with many feature-rich APIs,
there's often more than one way to do things.

In this article, I want to show you a few lesser-known tricks on how to use `@JsonUnwrapped` to very easily
manipulate JSON.

## 1 - Removing a field from JSON serialization

First, I want to show you a little trick that you can use to easily and
unobtrusively prevent a POJO field from showing up in the serialized JSON. It
can be used without modifying the POJO, which is great if you have to serialize
classes you don't own, or if you don't want to encumber the class with Jackson
annotations.

Let's pretend we have a database of persons, and we want to allow certain
clients to view all the information we have, but certain clients cannot view the
persons' date of birth. Here's the `Person` class:

```java
public record Person(
  LocalDate dateOfBirth,
  String name,
  String address
) {
}
```

We'll expose this class on a REST endpoint using Spring Web MVC, although the
following pattern isn't Spring-specific, and will apply to any application that
uses Jackson to serialize POJOs. Here's our first endpoint, where we just output
the complete `Person` without hiding the data of birth:

```java
@RestController
@RequestMapping
public class PersonController {

  @GetMapping("/person")
  Person getPerson() {
    return Persons.dummy();
  }
}
```

In lieu of a repository, I'm simply using a factory here. We can attest that
this works with a simple test:

```java
@WebMvcTest(controllers = PersonController.class)
@RequiredArgsConstructor(onConstructor_ = @Autowired)
class IgnoreJacksonFieldApplicationTests {

    private final MockMvc mockMvc;

    @Test
    void getPerson_returnsFullPerson() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.get("/person"))
                .andExpect(content().json(/* language=JSON */ """
                        {
                          "dateOfBirth" : "1945-01-01",
                          "name" : "their name",
                          "address" : "their address"
                        }
                        """, JsonCompareMode.STRICT));
    }
}
```

Now, we want to expose this POJO, but omit the date of birth in the serialized
representation. There's a few ways we can do this, using Jackson or plain Java,
but I want to show you one that I appreciate, because it's simple and non
invasive.

Without modifying the `Person` class, and without duplicating the `Person` class
and adding a mapper, here's what we can do:

```java
// in PersonController

@GetMapping("/personExternal")
NoDateOfBirth<Person> getPersonExternal() {
  return new NoDateOfBirth<>(Persons.popa());
}

record NoDateOfBirth<T>(
  @JsonIgnoreProperties("dateOfBirth")
  @JsonUnwrapped T t
) {}
```

`@JsonUnwrapped` pulls all of a field's properties one level up, so that for
example, the person's name won't be under `$.t.name`, but directly under
`$.name`. And `@JsonIgnoreProperties` is self-explanatory. `NoDateOfBirth` is a
nested class, to keep the scopes small, and it's generic, so you could hide the
DOB from any arbitrary POJO (you could for example have `NoDateOfBirth<Dog>`).

`@JsonUnwrapped` is pretty powerful, as it can be used in various scenarios. For
example, you can also use it to add fields to a POJO, or combine two POJOs into
one. But to keep this post small, I'll leave this as an exercise to the reader.

## 2 - Adding a field to JSON serialization

You can also add a field using a very similar pattern. Say you have a `Client`,
and in some cases you also want to provide the caller with their transaction
history. You could add a `ClientDTO` and a `ClientMapper` for this, but the best
code is the code you don't write, so you can avoid the extra mapper with this
simple wrapper:

```java
record WithTransactionHistory<T>(
  @JsonUnwrapped T t,
  List<Transaction> transactionHistory
) {}
```

This will pull up all of `t`'s properties to the top level, and add a
`transactionHistory` field.

## 3 - Splitting an incoming JSON into multiple POJOs

Now here's an interesting pattern if you deal with APIs that expose many fields
on the same hierarchical level. For example, I've often seen things like:

```json
{
  "groupId": "group id",
  "groupName": "group name",
  "groupTier": "group tier",
  "paymentId": "payment id",
  "paymentInfo": "payment info",
  "paymentDetails": "payment details",
  "applicationId": "application id",
  ... and so on
}
```

This payload contains information about 3 different things: group, payment, and
application. If you deserialize this into a single object, the object is now a
mixture of 3 different responsibilities. Ideally, we would want to handle it in
the application as 3 separate collections of information. We could use a DTO,
and then map this to 3 different objects.

Jackson can do this automatically using `@JsonUnwrapped`:

```java
record Group(
  String groupId,
  String groupName,
  String groupTier
) {}

record Payment(
  String paymentId,
  String paymentName,
  String paymentDetails
) {}

// if we want to carrying the full field names
// in you POJOs, we can use @JsonProperty:
record Application(
  @JsonProperty("applicationId")
  String id
) {}

// this is the POJO that will be deserialized
record MyBigDTO (
  @JsonUnwrapped Group group,
  @JsonUnwrapped Payment payment,
  @JsonUnwrapped Application application,
) {}
```

## Composing `@JsonUnwrapped` wrappers

The wrapper pattern is also composable, which can really help if, for example,
you have to expose multiple combinations of displayed/hidden fields. Say you
need to add a field `transactionHistory` to a POJO, but you also need to hide
the DOB field. You could expose something like this:

```java
WithTransactionHistory<NoDateOfBirth<Person>>
```

or even use `Object` as the controller method's return type, and let the
controller decide how to compose various wrappers.

## Conclusion

In this article, I've shown a collection of Jackson patterns using
`@JsonUnwrapped` that are unobtrusive, flexible, and expressive. I've shown a
bit of Spring here, but they'll work in any application that uses Jackson for
JSON serialization.

Happy coding!

---

The code for this article can be found
[here](https://github.com/LeMikaelF/ignore-jackson-field).
