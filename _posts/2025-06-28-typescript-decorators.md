---
title: "TypeScript Decorators are Awesome, Part 1"
date: 2025-06-28
tags:
  - typescript
categories:
  - TypeScript
---

TypeScript decorators are awesome. This article is about how you can use them to
build declarative APIs.

## Some Java (but don't run!)

A few months ago, I wrote an
[article](https://foojay.io/today/building-a-declarative-api-with-spring-aop-and-spel/)
on building declarative APIs with Spring AOP and SpEL that was well received and
was even featured on Baeldung's Java Weekly.

The article was about how to build an API that looks like this:

```java
@Audit(action = AuditAction.DISABLE_USER, expression = "#requests.![clientId]")
public void disableUsers(List<UserDisableRequest> requests) {
  // ...
}
```

Since this article is about TypeScript, I won't go into the details of building
this in Java. If you're curious, feel free to take a look at the article.

What I want to emphasize in this code snippet is that it's expressive, concise,
and very flexible. Even without knowing the context, it's easy to know that this
audits the `DISABLE_USER` action, and that it uses some expression (in this case
written in
[SpEL](https://docs.spring.io/spring-framework/reference/core/expressions.html),
a DSL known to Spring developers) to extract the client id fields out of the
`requests` argument. It's flexible because if you had another method to audit,
you could probably just put that annotation there, and maybe use a different
enum for the `action` value and a different expression if the arguments were
structured differently.

The first line, the one starting with `@`, is a Java
[annotation](https://docs.oracle.com/javase/tutorial/java/annotations/basics.html).
TypeScript supports a similar syntax, but it calls it a _decorator_.[^1]

## And Now Some Typescript

Decorators are
[supported natively](https://devblogs.microsoft.com/typescript/announcing-typescript-5-0/#decorators)
in TypeScript 5. Previously, a
[similar feature](https://www.typescriptlang.org/docs/handbook/decorators.html)
was accessible under the `--experimentalDecorators` flag. The new stable
decorator feature is an implementation of a
[TC39 proposal](https://github.com/tc39/proposal-decorators) to add decorators
to the ECMAScript language specification. The proposal is currently Stage 3 out
of 4, so it's reasonable to expect that decorators will be available in
ECMAScript/JavaScript in the not-so-distant future. One more reason to learn how
to use them.

To get started, let's build a decorator that measures how long a method takes to
execute. A decorator is a higher-order function that takes as parameters (1) the
original method and (2) some context information, and that returns a function to
be executed in place of the original method. For example, here is one that logs
the name of the annotated method whenever the method is invoked. Here I'm using
type `any` for simplicity.

```ts
function logged(originalMethod: any, context: any) {
  return function (this: any, ...args: any) {
    console.log(`executing method ${context.name}`);
    originalMethod.apply(this, args);
  };
}

export class SecretService {
  @logged
  shh() {
    console.log("doing something secret");
  }
}
```

When `shh()` is called, you'll see this in the logs:

```
executing method shh
doing something secret
```

What's nice is that this annotation can be packaged in a library and reused on
any method you want to log.

But this isn't so useful, so let's try something more complicated. What if you
want to measure the time it takes to execute the annotated method? This is a
common requirement when you want to measure performance of a specific code path,
for example by publishing a Prometheus counter.

In order to do this, we need to chain a promise on the annotated method:

<!-- prettier-ignore-start -->
```ts
function timed(originalMethod: any, context: any) {
  return function (this: any, ...args: any): any {
    // record the time before invoking the original method
    const startTime = performance.now();

    // note that the return value of this function is not the
    // original method, but rather a Promise that chains a
    // callback to it.
    return originalMethod.apply(this, args)
      // record the end time in a callback
      .then((result: any) => {
        const executionTime = performance.now() - startTime;
        console.log(`executed method '${context.name}' in ${executionTime} ms`);
        return result;
    });
  };
}

export class CriticalService {
  @timed
  async compute() {
    console.log("computing...");
    return new Promise((res) => setTimeout(res, 500));
  }
}
```
<!-- prettier-ignore-end -->

Calling `compute()` will log this:

```
computing...
executed method 'compute' in 501.68429199999997 ms
```

As a side note, you might notice I used
[`performance.now()`](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now)
insted of `Date.now()`. The reason is that the former is monotonic, meaning it
will never jump because of
[Network Time Protocol](https://en.wikipedia.org/wiki/Network_Time_Protocol)
(NTP) clock adjustments. This is important if you're taking deltas, because with
non-monotonic clocks you can get bad data (arbitrary deltas) during clock
adjustments.

So far, we've demonstrated that decorators can add arbitrary behaviour before or
after method execution, and that they can even return different values (in the
second example case, we returned a `Promise` that calculated a time offset).

Now let's do something a little more complex.

## Providing arguments to the decorator

Measuring time, as we did in our example, is relatively easy because it relies
only on global state, and not on the decorated method. Timing a
[nullary](https://en.wikipedia.org/wiki/Arity) method is identical to timing an
n-ary method.

But what if we want to access more context from our decorator, for example what
if the decorator publishes the execution time to Prometheus, and we want to add
a tag to the metric, to help with observability? Something like this:

```
my_method_duration_seconds_count{tier="critical"} 5
my_method_duration_seconds_sum{tier="critical"} 1.276
```

It's simple to do with decorators. First, we can add parentheses after the
identifier of a decorator: `@timed()`, and those parentheses can contain
anything that a regular function invocation would contain. That's because it
_is_ a regular function invocation. Our decorator function then becomes a
decorator _factory_: a function that returns a decorator when it is called with
the arguments that are in the parentheses! Here's what our `@timed()` decorator
looks like, now that it accepts arguments:

```ts
// a decorator factory, that accepts the arguments
// to be provided in @timed(...)
function timed(tier: "regular" | "critical") {
  // the rest is similar to what we saw before
  return function (originalMethod: any, context: any) {
    return function (this: any, ...args: any): any {
      return originalMethod.apply(this, args).then((result: any) => {
        console.log(
          `[TIER = ${tier}] executed method '${context.name}' in [TODO] ms`,
        );
        return result;
      });
    };
  };
}

export class CriticalService {
  // the argument inside the parentheses is strongly
  // typed, so the compiler will catch any typos
  @timed("critical")
  async compute() {
    return new Promise((res) => setTimeout(res, 500));
  }
}

// prints
// [TIER = critical] executed method 'compute' in [TODO] ms
```

The new decorator factory is a function, that returns a function, that returns a
function! I've omitted the timing logic for clarity, but apart from that, the
only change is that we nested the previous decorator function inside a factory
function, so that when the outer `timed` function is called with properly typed
arguments, it will return a regular decorator function.

Nested functions and [currying](https://en.wikipedia.org/wiki/Currying) tend to
look nicer when used with arrow functions (`return a => b => c => [a, b, c]`),
but in this case we need the `function` piece of syntax because the inner
`function`'s first argument is a
[`this` parameter](https://www.typescriptlang.org/docs/handbook/2/functions.html#declaring-this-in-a-function),
a special Typescript syntax that gets stripped away at compile-time and is there
just as an indication to the type system.

We just saw how we can put arbitrary expressions (anything you can pass to a
function, really) inside decorators. Now let's take it one step further. What if
we wanted to extract a tag from one of the decorated method's arguments?

## Extracting values from arguments

TypeScript has a
[`Parameters<T>`](https://www.typescriptlang.org/docs/handbook/utility-types.html#parameterstype)
type that extracts a function's arguments and creates a type from that:

```ts
function repeat(str: string, times: number) {
  return str.repeat(times);
}

// type RepeatParameters = [str: string, times: number]
type RepeatParameters = Parameters<typeof repeat>;
```

The idea is to use this type so that the decorator factory accepts a lambda that
takes in the same parameters as the decorated method. Now it's just a matter of
declaring a type parameter `ARGS` and telling the compiler that the type of the
lambda's parameters is the same as the type of the method's parameters!

```ts
function timed<ARGS extends any[]>(tagFn: (...args: ARGS) => string) {
  return function (originalMethod: (...args: ARGS) => any) {
    return function (this: any, ...args: any): any {
      console.log(`timing function with tag '${tagFn(...args)}'`);
      return originalMethod.apply(this, args);
    };
  };
}

export class CriticalService {
  // this lambda has type string[] => string
  // and if we ever change the method, the compiler will
  // know that the lambda's has to change, too!
  @timed(([tag]) => tag)
  async compute(arr: string[]) {
    return new Promise((res) => setTimeout(res, 500));
  }
}
```

Note that in this example, the `@timed` decorator is on a method that accepts an
array of strings, and the lambda simply says that the tag to use for timing
should be the first element of the array. But the same decorator would work on
any method, with any parameter types, and any lambda. Talk about reusability!

## Conclusion

In a few lines of code, we implemented a TS decorator that can be reused on any
method. We can even use the type system to type the decorator's arguments
relatively to the decorated method. This lets expose strongly typed declarative
APIs that we can use to make our code more conside, expressive, and flexible.

Decorators can also be used on things other than methods, things like fields,
classes, and accessors. For more information on decorators, take a look at Axel
Raschmayer's extensive
[article](https://2ality.com/2022/10/javascript-decorators.html) on the subject,
or at the
[TC39 proposal](https://github.com/tc39/proposal-decorators?tab=readme-ov-file#detailed-design).

## Footnotes

<!-- prettier-ignore-start -->
[^1]: Decorators are named after the [design pattern](https://refactoring.guru/design-patterns/decorator). Design pattern decorators aren't about annotations or TS decorators, but TS decorators are implemented similarly to design pattern decorators.
<!-- prettier-ignore-end -->
