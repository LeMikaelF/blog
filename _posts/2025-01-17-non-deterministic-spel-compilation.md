---
title: "Patching SpEL for Better Compilation Performance"
date: 2025-01-18
tags:
  - spring
  - aop
  - spel
categories:
  - Spring
hidden: true
---

# DRAFT DRAFT DRAFT

The Spring Expression Language (SpEL) is an expressive language that is typically used for computing simple expressions. To provide better
performance in latency-sensitives cases, SpEL
can [compile expressions](https://docs.spring.io/spring-framework/reference/core/expressions/evaluation.html#expressions-spel-compilation)
by generating bytecode, instead of evaluating the expressions in interpreted mode. This can result in a drastic performance boost, with
reported speedups of [25x](https://docs.spring.io/spring-framework/reference/core/expressions/evaluation.html#expressions-spel-compilation),
[36x](https://stackoverflow.com/a/62729338/7096763) and even [58x](https://stackoverflow.com/a/44165853/7096763) in one case.[^1] This makes
SpEL a candidate for hot paths, such as logging and high-volume event processing. There are however a few caveats to this, one of which
makes it impossible to know whether an expression containing boolean operators is compilable, unfortunately making the SpEL compiler
unreliable for critical scenarios, such as high-volume data ingestion or logging filters.

In this article, we will discuss this limitation, and see how we can address it by patching the SpEL runtime. In the process, we will learn
about the internals of SpEL. Finally, we will briefly discuss an application of this in a Logback logging filter.

## The SpEL compiler

By default, SpEL compilation is disabled, and expressions will run in interpreted mode. We can enable compilation either for a specific
`SpelExpression`, by setting `CompilerMode.IMMEDIATE` or `CompilerMode.MIXED` on its configuration, or globally, by setting
the `spring.expression.compiler.mode` property to `immediate`
or `mixed` ([documentation](https://docs.spring.io/spring-framework/reference/core/expressions/evaluation.html#expressions-compiler-configuration)).
Under `immediate`, an expression will run in interpreted mode the first time around, and in compiled mode from then on, given that the
expression is compilable
(more on that later). Under `mixed`, a given expression will run in interpreted mode, and after a number of iterations (currently
[100](https://github.com/spring-projects/spring-framework/blob/main/spring-expression/src/main/java/org/springframework/expression/spel/standard/SpelExpression.java#L54)),
it will be compiled and start running in compiled mode.

Only a subset of SpEL is compilable. For instance, expressions using assignment, bean references, or collection projection
are [not compilable](https://docs.spring.io/spring-framework/reference/core/expressions/evaluation.html#expressions-compiler-limitations).
But the most important limitation, according to me, is due to the way the compiler gets its typing information:

> Due to the lack of typing around expressions, the compiler uses information gathered during the interpreted evaluations of an
> expression when performing compilation. For example, it does not know the type of a property reference purely from the expression, but
> during the first interpreted evaluation, it finds out what it is.

In other words, a SpEL Abstract Syntax Tree ([AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)) node is only compilable if it has
been previously executed in interpreted mode, and if all nodes have been evaluated. This is a problem for the `and`
and `or` boolean operators, since they don't always evaluate both of their operands. Take the following expressions:

```
true and true   // left and right side are evaluated
true and false  // left and right side are evaluated
false and false // only left side
false and true  // only left side
```

If we write the expression `false and true` using boolean literals, SpEL will still be able to compile it, because it knows from the types
of boolean literals from the get go. But if we add an indirection to the mix, and the operands are method calls or property accesses, SpEL
can no longer know the type of the operands before evaluating them, and therefore it cannot compile the expression, even if the left and
right operands are themselves compilable expressions. The following test reproduces this issue:

```java

@Test
void unCompilableExpression() {
  // declare and parse an expression
  Expression expression = new SpelExpressionParser().parseExpression("left and right");

  // set the values of "left" and "right" on the evaluation context
  SimpleEvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding()
    .withRootObject(new ExpressionRoot(false, true))
    .build();

  // SpelCompiler.compile() tries to compile the expression,
  // returning true if it succeeded, and false otherwise.
  // If true, further evaluations of the `SpelExpression` will
  // run in compiled mode.
  // TODO check if they will revert to interpreted on exceptions if compiler is off or mixed
  assertThat(SpelCompiler.compile(expression))
    .isFalse();

  // evaluate the expression
  assertThat(expression.getValue(context))
    .isEqualTo(false);

  // check that the expression is compilable;
  // this time, it's not because the second operand
  // has never been evaluated
  assertThat(SpelCompiler.compile(expression))
    .isFalse();
}

public record ExpressionRoot(boolean left, boolean right) {
}
```

If we run this test again with a `new ExpressionRoot(true, true)` evaluation context, we'll see that expression _does_ compile after we
evaluate the expression. The difference depends only on short-circuiting behaviour of the `and` operator.

This is problematic, because it means that **as soon as you have any boolean operators in a SpEL expression, its compilability is
non-deterministic**. Consider this expression:

```
T(java.util.concurrent.ThreadLocalRandom).current().nextBoolean() && right
```

Is it compilable? Depends on what `nextBoolean()` evaluates to. But what if we need the expression to compile, because we can't afford to
pay the performance cost of interpreted mode? Well, we might just have to get creative...

## Making SpEL compilation deterministic

A possible solution to this is to ensure that there is no short-circuiting boolean operators in the expressions. Unfortunately, SpEL has no
boolean non-short-circuiting operators (Java's `&` and `|`).[^2] We could always use something like this:

```java
// equivalent to a non-short-circuiting AND
"false && true left ? right : (right && false)"
```

But wow, that's ugly. Although, what if we could hide this from end users? What if we could transform the regular `and` and `or` into
non-short-circuiting operators, run them this way the first time around to let the interpreter gather the necessary type information, and
then revert back to short-circuiting? This is what the next section is about.

## Hacking SpEL

Before we get our hands dirty, a quick word about the SpEL interpreter. It's a tree-walk interpreter, meaning it traverses
an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) (AST) and evaluates the nodes as it goes.[^3] SpEL nodes
implement the `SpelNode` interface and know how to both interpret and compile themselves. Here is an overview of the `SpelNode` interface:

```java
public interface SpelNode {
  // ...
  @Nullable
  Object getValue(ExpressionState expressionState) throws EvaluationException;

  int getChildCount();

  SpelNode getChild(int index);

  default void generateCode(MethodVisitor methodVisitor, CodeFlow codeFlow) {
    // throw exception if not implemented
  }
  // ...
}
```

This interface is directly implemented by a single abstract class, `SpelNodeImpl`, that in turn is extended by a grand total of 21
classes: `OpAnd` and `OpOr` for the `and` and `or` operators, but also `ConstructorReference`, `Assign`, `Elvis` (the
[Elvis operator](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/operator-elvis.html#page-title)),
`MethodReference`, and `BeanReference`. The AST root is therefore a `SpelNode` and we can traverse the tree by accessing each node's
children using their `getChild(int)` method. And we can get this AST by calling `getAst()` on a parsed SpEL expression.

Knowing this, here is how we can ensure that SpEL expressions are not uncompilable because of boolean short-circuiting. We'll intercept
the `SpelExpression` coming from a `SpelExpressionParser`, inspect its AST, and modify it by swapping out the `OpAnd` and `OpOr` nodes for
custom nodes that will run the first evaluation without short-circuiting.

We'll do this in three parts. First, we'll write a `SpelExpressionParser` that we can use to intercept the AST. Second, we'll write custom
AST nodes to implement our logic. And third, we'll modify the AST to use our custom nodes.

## Step 1: Writing a custom parser (kind of)

When I think "intercepting a return value", the design pattern that springs to mind
is [Decorator](https://en.wikipedia.org/wiki/Decorator_pattern). Just implement an interface or extend a class, and accept a class to
delegate invocations to. So we'll kind of write a parser, in the sense that we'll be extending `SpelExpressionParser`, but we will delegate
all the heavy lifting to the regular `SpelExpressionParser`:

```java

@RequiredArgsConstructor
public class CustomSpelExpressionParser extends SpelExpressionParser {

  private final SpelParserConfiguration configuration = new SpelParserConfiguration();
  private final SpelExpressionParser delegate = new SpelExpressionParser(configuration);

  @Override
  public Expression parseExpression(String expressionString,
                                    @Nullable ParserContext context) throws ParseException {

    SpelExpression expression =
      (SpelExpression) delegate.parseExpression(expressionString, context);

    // yay!
    SpelNode ast = expression.getAST();

    return expression;
  }
}
```

Now that we just wrote a complete parser in 5 minutes 🤓, let's see how we can modify the AST nodes concerning us, `OpAnd` and `OpOr`.

## Step 2: Custom AST nodes

What we want is to "patch" `and` and `or` operators so that they won't short-circuit the first time around, so that the interpreter can
build up the ¬necessary the type information for the compiler. As we saw earlier, all SpEL AST nodes have one method,
`getValueInternal`, that they implement in order to provide their evaluation behaviour. For
`OpAnd`, that's just evaluating the two expressions at the left and at the right of the operator, while obeying the short-circuiting
contract:

```java
public class OpAnd extends Operator {
  // ...
  public TypedValue getValueInternal(ExpressionState state) throws EvaluationException {
    if (!getBooleanValue(state, getLeftOperand())) {
      return BooleanTypedValue.FALSE;
    }
    return BooleanTypedValue.forValue(getBooleanValue(state, getRightOperand()));
  }
  // ...
}
```

`OpOr` does pretty much the same thing. From now on, we'll only worry about `OpAnd`, because `OpOr` is just its mirror image. Already, we
can see that the boolean ASTs are nothing special. Actually, they are a bit special, since they do know how generate bytecode equivalent to
their `getValueInternal` method to compile themselves, but we don't have to worry about that.[^4] The good thing is, since nodes are
compilable only starting with their second evaluation, as long as we force the expressions to run in compiled mode right from the second
evaluation by setting `CompilerMode.IMMEDIATE` on the expression, we don't have any work to do!

```java
class NonShortCircuitingFirstEvaluationAnd extends OpAnd {

  public NonShortCircuitingFirstEvaluationAnd(int startPos, int endPos, SpelNodeImpl... operands) {
    super(startPos, endPos, operands);
  }

  @Override
  public TypedValue getValueInternal(ExpressionState state) throws EvaluationException {
    Boolean left = (Boolean) getLeftOperand().getValue(state);
    Boolean right = (Boolean) getRightOperand().getValue(state);

    return BooleanTypedValue.forValue(left && right);
  }
}
```

I'm omitting some null-checking and error reporting from the snippet above to keep this article simple, but that's really all there is to
it.

Now, let's see how we can modify the AST to replace `OpAnd` and `OpOr` with our custom nodes. The AST an immutable data structure and is a
tree, so we'll use a standard [post-order traversal](https://en.wikipedia.org/wiki/Tree_traversal). The AST nodes are immutable, so to swap
out a node, we actually have to copy everything up to from that node. It helps to visualize this problem. In the image below, if we want to
replace node D, we also have to "tell" node B that its left child is now D′, and repeat the process for node A. And since nodes are
immutable, we need to replace them with a copy that contains the updated children.

![image showing the process of replacing a node in a tree](/assets/img/replacing a node in an immutable tree.jpg)

One limitation of doing this with SpEL nodes is that some node types, like `Assign`, which represents the
SpEL [assignment operator](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/operators.html#expressions-assignment),
don't expose the necessary methods to copy the nodes. So to keep this article simple, and also because it is sufficient for many cases, we
will only swap out `OpAnd` and `OpOr` nodes that are either top-level or direct children other `OpAnd` or `OpOr` nodes. This means that we
will be able to handle this expression:

```
#exception instanceof T(IllegalArgumentException) 
and 
#exception.message matches '[ERROR].*' 
or 
#myVar > 5 
```

but not this one:

```
#logger.log(true && false)
```

because the `OpAnd` is a child of the node representing the `#logger.log` operation. This limitation is not significant, since boolean
expressions in SpEL are usually used directly.

## Step 3: modifying the AST

And with this, we can write the rest of our parser:

```java

@Override
public Expression parseExpression(String expressionString, @Nullable ParserContext context) throws ParseException {
  SpelExpression expression = (SpelExpression) delegate.parseExpression(expressionString, context);

  SpelNode ast = expression.getAST();

  return new SpelExpression(expressionString, transformAst((SpelNodeImpl) ast), configuration);
}

// ...

public SpelNodeImpl transformAst(SpelNodeImpl root) {
  return switch (root) {
    case OpAnd opAnd -> new NonShortCircuitingFirstEvaluationAnd(
      opAnd.getStartPosition(),
      opAnd.getEndPosition(),
      // recursive case
      transformChildren(getChildren(opAnd)));
    case OpOr opOr -> new NonShortCircuitingFirstEvaluationAnd(
      opOr.getStartPosition(),
      opOr.getEndPosition(),
      // other recursive case
      transformChildren(getChildren(opOr)));
    // base case
    default -> root;
  };
}

private SpelNodeImpl[] transformChildren(SpelNodeImpl... children) {
  return Arrays.stream(children)
    .map(this::transformAst)
    .toArray(SpelNodeImpl[]::new);
}

// nothing to see here, just populating an array
private SpelNodeImpl[] getChildren(SpelNodeImpl root) {
  SpelNodeImpl[] children = new SpelNodeImpl[root.getChildCount()];
  for (int i = 0; i < root.getChildCount(); i++) {
    children[i] = (SpelNodeImpl) root.getChild(i);
  }
  return children;
}
```

Now if we use this parser and run it through the test at the beginning of this article, we'll see that the expression now compiles just
fine!

## Applying this to a Logback Filter

I first stumbled upon the problem of boolean operator compilability when writing a
Logback [TurboFilter](https://logback.qos.ch/manual/filters.html#TurboFilter). We wanted to be able to downgrade an `ERROR` log to `INFO`
whenever it matched an arbitrary expression, so that our error logs wouldn't get polluted by errors we know could be ignored. I'll be
dedicating a separate article to this filter, because it came with interesting challenges, but here's part of a `logback.xml` configuration
demonstrating its usage:

```xml

<turboFilter class="com.example.filters.ExpressionBasedLevelConverterFilter">
  <ExpressionFilterCriteria>
    <Expression>
      level == ERROR
      and
      throwable instanceof T(java.lang.IllegalArgumentException)
      and
      throwable.message == 'something specific'
      and
      logger == 'com.example.application.MyService'
    </Expression>
  </ExpressionFilterCriteria>
</turboFilter>
```

Upon initialization, the filter will:

1. parse its expression
2. evaluate it once using dummy objects to populate the expression variables (`level`, `throwable`, `logger`, and a few others)
3. try to compile it

If the expression is uncompilable, the filter will throw an exception to prevent running the interpreted expression on a critical code path
(TurboFilters are called on every log request–TRACE, DEBUG, INFO, WARN, and ERROR). After this point, the filter is ready to run the
compiled expression without adding undue latency to the application. And since the first evaluation of the expression uses dummy arguments,
it doesn't matter that the operators don't short-circuit!

## Conclusion

We started by discussing how SpEL's compiler can lead to notable performance improvements. It does have some limitations,
one of which causes the compilability of expressions containing boolean expressions to be non-deterministic. We remediated this by patching
the AST nodes corresponding to the `and` and the `or` operators. Finally, we introduced a concrete application of this by introducing a
Logback filter based on SpEL expressions.

This pattern of modifying the AST of a SpEL expression can be reused for other purposes. For example, a common use case for SpEL is to run
certain authorization checks whenever a method is called, via the `@PreAuthorize` annotation. In certain situations, these checks will
include more than one blocking call, for
example calling two different systems via HTTP to check if the user is authorized in any of them. In Java code, it would be easy to run
these checks concurrently, using `CompletableFuture`. In SpEL, we could extend the language to provide `&&&` and `|||`
operators that would evaluate both of their operands concurrently, and return as soon as they have enough information. For example, if one
operand of an `|||` returns `true`, the operator could immediately return `true` and possibly interrupt/cancel the other operand.

SpEL is a useful language with untapped potential. Its limitations are easily overcome when we consider how accessible its internals are.


[//]: # (TODO mention null-safe index operator https://github.com/spring-projects/spring-framework/pull/29847)

[//]: # (TODO mention groovy and dynamically typed JVM languages, maybe SpEL could take a page from their book. The parser dates back from 2013, maybe that's before invokeDynamic)

# Footnotes

[//]: # (@formatter:off)
[^1]: An early benchmark from 2013 even achieved a [1500x speedup](https://github.com/spring-projects/spring-framework/issues/15571#issuecomment-453405791) by compiling an expression (!). I do not know if these results still stand, though.  
[^2]: I found interesting that the Java Language Specification calls the plain `&&` operator "conditional-and operator", and the bitwise `&` "boolean logical operator" ([source](https://docs.oracle.com/javase/specs/jls/se23/html/jls-15.html#jls-15.22.2)).
[^3]: If you want to know more about ASTs and interpreters but don't feel like getting a master's degree in compilers, Ruslan Spivak has a series of articles called "Let's Build a Simple Interpreter". [Part 7](https://ruslanspivak.com/lsbasi-part7/) covers ASTs and tree-walk interpreters. If you still find yourself wanting more, I can't recommend enough Robert Nystrom's [_Crafting Interpreters_](https://craftinginterpreters.com/). It's 600 pages long, but it's a pleasant read and is packed with insights on computer science and software in general.
[^4]: If you are curious about how the AST nodes generate bytecode, take a look at [`OpAnd.generateCode()`](https://github.com/spring-projects/spring-framework/blob/main/spring-expression/src/main/java/org/springframework/expression/spel/ast/OpAnd.java#L86).
[//]: # (@formatter:on)





