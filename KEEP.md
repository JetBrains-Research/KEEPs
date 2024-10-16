# Union Types for Errors

* **Type**: Design proposal
* **Authors**: Roman Venediktov, Daniil Berezun
* **Contributors**: Marat Akhin, Mikhail Zarechenskiy
* **Status**: In research
* **Related YouTrack issue**: [KT-68296](https://youtrack.jetbrains.com/issue/KT-68296/Union-Types-for-Errors)

## Overview

This KEEP introduces *union error types*,
which allows for better handling of expected failure states for values and function results.

## Problem statement

### Motivation

#### In-place tags

A typical showcase of using in-place tags is `Sequence.last` function that is written as follows:

```kotlin
inline fun <T> Sequence<T>.last(predicate: (T) -> Boolean): T {
    var last: T? = null
    var found = false
    for (element in this) {
        if (predicate(element)) {
            last = element
            found = true
        }
    }
    if (!found) throw NoSuchElementException("Sequence contains no element matching the predicate.")
    @Suppress("UNCHECKED_CAST")
    return last as T
}
```

The variable found is used specifically for cases when a predicate looks like `{ it == null }`. 
It's a typical pattern for functions that retrieve data from containers (`single`/`singleOrNull`/`last`/...).

One way to rewrite the code without using the variable `found` is to use some kind of private tag:

```kotlin
object NotFound

inline fun <T> Sequence<T>.last(predicate: (T) -> Boolean): T {
    var last: Any? = NotFound // We have to use the most common type Any?
    for (element in this) {
        if (predicate(element)) {
            last = element
        }
    }
    if (last == NotFound) throw NoSuchElementException("Sequence contains no element matching the predicate.")
    @Suppress("UNCHECKED_CAST")
    return last as T // unchecked cast
}
```

It might be a bit shorter, but we still have to trick our type system by using the most common type `Any?` 
and performing an unchecked cast at the end.

Combining the ideas of tags and unions, the mentioned code could be rewritten as follows:

```kotlin
error object NotFound // new category of types

inline fun <T> Sequence<T>.last(predicate: (T) -> Boolean): T {
    var last: T | NotFound = NotFound
    for (element in this) {
        if (predicate(element)) {
            last = element
        }
    }
    if (last == NotFound) throw NoSuchElementException("Sequence contains no element matching the predicate.")
    return last // smart-cast to T
}
```

Our functions work for all `T` and it doesn't have unchecked casts

To make this function fully correct, we have to prevent `NotFound` from leaking outside the function:

```kotlin
inline fun <T> Sequence<T>.last(predicate: (T) -> Boolean): T {
    error object NotFound
    
    var last: T | NotFound = NotFound
    for (element in this) {
        if (predicate(element)) {
            last = element
        }
    }
    if (last == NotFound) throw NoSuchElementException("Sequence contains no element matching the predicate.")
    return last // smart-cast to T
}
```

#### ...OrThrow / ...OrNull

It's quite common to encounter functions that describe part of their effects for error cases in their names.
This pattern could be enhanced by having only one function with a functional name like `get` or `processRequest`,
where exceptional cases are encoded in the signature.

A typical example could be our functions `maxOf` 
([code](https://github.com/JetBrains/kotlin/blob/0938b46726b9c6938df309098316ce741815bb55/libraries/stdlib/common/src/generated/_Arrays.kt#L14700)) 
where many functions are duplicated to have `maxOfOrNull` counterpart in case of empty collections.

With the error union types, it is possible to define one method that can encode all the necessary error cases:

```kotlin
// getOrThrow -> get
fun <T> get(): T | Error

// maxOrNull -> max
fun IntArray.max(): Int | NoSuchElement

// awaitSingleOrNull -> awaitSingle
fun <T> awaitSingle(): T | NoSuchElement
```

And programmer is able at the call-site 
transform them into sound exceptions using `!!` operator or process them in any way.

### Background

#### Either

`Either` is a common way to handle errors in functional programming languages.
It is parametrized with two types: one for the successful result and one for the error.
It has two constructors: `Left` and `Right` which are used to store the result and the error respectively.
In Kotlin, it is possible to implement `Either` using sealed classes:

```kotlin
sealed class Either<out L, out R>
data class Left<out L>(val error: L) : Either<L, Nothing>()
data class Right<out R>(val value: R) : Either<Nothing, R>()
```

To eliminate `Either` you could use common `when` expression:

```kotlin
val result: Either<Int, String> = ...
when (result) {
    is Left -> println("Error: ${result.error}")
    is Right -> println("Value: ${result.value}")
}
```

It is possible to implement `Either` on the language level with the same operators as proposed for error unions.
But this approach has several disadvantages:
- Composition of `Either` is sensitive to the structure and order:
  `Either<A, Either<B, C>>` is not the same as `Either<Either<A, B>, C>` or `Either<B, Either<A, C>>`.
  Which may lead to complications for subtyping and code refactoring or reordering.
- Error type in `Either` is a common type in a Kotlin type system.
  Consequently, if you would like to have several possible errors, you have to create a sealed hierarchy for them.
  And you will not be able to return a subset of this hierarchy by handling only a part of errors.
- Even with the introduction of arbitrary unions, due to expressiveness of common types, unions on them will be limited.

#### Error union type in Zig

The most similar feature to the proposed one is [error union types in Zig](https://ziglang.org/documentation/master/#Error-Union-Type).

The broad overview of the feature is the following:
- Error types are just tags without data, represented as integers at runtime.
- Equal names implies equality of errors.
- Arbitrary error unions are allowed.
- Allowed inference of error in return type.
- In debug mode, tracing of error bubbling is enabled.

#### Effects as error handling

Effects is another technique to handle errors.
It is used in languages like: TODO:...

The idea is that functions may trigger some effects, which are handled by the respective handler at the callstack.
The difference with exceptions is that effects are:
- Type-safe, as all possible effects have to be handled.
- Allows continuing execution after the effect is handled.

Effects is a very promising technique that is actively researched and used in some languages.
They may even be implemented in Kotlin in future using coroutines and context parameters.
The good example of their adoption for OO language is [Scala's capabilities](https://docs.scala-lang.org/scala3/reference/experimental/cc.html).

But we state that their use-cases are different from the proposed feature.
While they are great to track some IO interactions and unrecoverable exceptions, 
they may be too complex to handle simple cases like function `last` or that user's age is not in the range.

### Goals

1. Cover mentioned use-cases.
2. Minimize boilerplate required to operate with errors to the same level as with nulls.
3. Minimize performance overhead.
   Preferably to have an errors without data with the performance comparable to `null`.

## Proposal

### Approach

TODO: rewrite & move

We had a preliminary discussion about a design where errors have a separate hierarchy with the same features as existing classes.
This approach is directed by a goal "Let's add as many features as possible to remain errors inferrable".
But this approach has several significant issues.

The first problem of this full-flavored approach is performance-based.
All functions that would like to have a specific internal state create a new private class.
Mapping them to separate JVM classes leads to a significant number of small classes in JVM and class-loader calls.
This performance issue is similar to one encountered in the JVM at the moment of lambda's introduction.

The other and more significant problem with full-fledged approach is
that it is not easily supported by the type inference.
When unions are discussed in terms of Kotlin's future, they are required to be disjoint.
But in the case of errors, it is not possible.
Let's consider the example of the `find` function.
We would like to replace two overloads of this function (with null and with exception) with a single version with error union.
Resulting function is expected to be the following:

```kotlin
fun <T> Iterable<T>.find(f: (T) -> Boolean): T | NotFound
```

But it is not disjoint as `T` may contain `NotFound` itself.
So we would like to allow some kind of non-disjointness for error unions.
To achieve this, we have to use a slightly different approach compared to one used for the whole language.

Thus, we moved into approach where we try to make as few features as possible that will cover all known use-cases.

With this approach, we do not allow for error types:
- Generics.

  Generics provide the main issue with unions in Kotlin.
  While for disjoint unions it is possible to do something,
  in a non-disjoint approach we easily encounter the following issue:
  ```kotlin
  fun <T> foo(v: T) {
    val a: T | MyError<String> = v
    // ...
    if (a is MyError) {
      a // Expected: MyError<String>, Actual: MyError<*>
    }
  }
  ```
  In this code we may expect that if `a` is `MyError`, then it is `MyError<String>`.
  The issue is that we do not know anything about generic parameters of `MyError` in `T`.
  Possible solutions to this issue are:
  - Consider this behavior as ok.
    But it is really confusing if generic parameters become use-less in unexpected case.
  - Prohibit union of type variable and error with generic parameters.
    But we allow such a union in case of no generic parameters.
    Quite a confusing behavior.
  - Do not allow generics for errors.
    This approach is the chosen one. 
    Advantages:
    - It will not generate anything confusing.
    - There are no clear use-cases where generics are significant for errors.
    - There is an opinion that they will generate more issues in type inference which we could not establish so far.
- Subtyping.

  Actually, it is possible to add subtyping for errors.
  But there are several cons against this option:
  - There are no clear use-cases where subtyping is significant and could not be replaced with typealias.
    More precisely:
    ```kotlin
    abstract error class DbError
    error class DbConnectionCancelled : DbError
    error class DbTokenExpired : DbError
    // replace with:
    error class DbConnectionCancelled
    error class DbTokenExpired
    typealias DbError = DbTokenExpired | DbConnectionCancelled
    ```
  - Subtyping forces us to use 1to1 mapping from errors to JVM classes which have mentioned performance issues.
  - Type inference for errors will never infer a supertype.
    Instead, it will just list all possible actual errors.
    So supertype is only for explicit annotations.
  - We do not want users to abuse this feature.
    And in terms of errors, the actual value of a supertype is just a list of subtypes.
    Any abstract method is a sign of abuse.
    So typealias is even more semantically sound for errors.
    In an extra cases where it is required, typealias + extension function is all you need.

### Type syntax

Error type is a separate type kind, which is declared using a soft keyword `error`.

```kotlin
error class MyError(val code: Int)
```

Error type may not have any supertypes
other than new common supertype for all of them, `Error`.
We are also introducing new type `Value` which is a supertype for all non-error types.
The resulting subtype hierarchy between those types is the following:
1. `Any :> Value` and `Any :> Error`
2. `Value :> Int`, `Value :> String`, etc.
3. `Error :> MyError`, `Error :> ConnectionError`, etc.
4. `MyError :> Nothing`, `ConnectionError :> Nothing`, etc.

> TODO: Discuss: 
> 
> IMO we should name `Any` as a supertype for all values and `Ref` (name is really problematic) as a supertype for everything.
> Cons:
> - in cases `fun <T> foo(t: T): T & Any` programmer means that `T & Any` is a value without any other states (like `null` before)
> - The same in cases like `class C(val v: Any)` it does not accept any error before (only null existed), 
>   why should it start accepting null or any other error now?
> - So it is a requirement of backward compatibility
> 
> We may not introduce a supertype for Any, but introduce a syntax `Any??` for `Any | Error`.
> 
> TODO: existing `Any?` usages which expected to be able to assiged to any value. 

These types may be used in a union with common type:

```kotlin
fun foo(val content: String | ConnectionError | DbError): Int? | OtherError
```

Limitations on those types:
1. It may contain only one non-error type.
   And it has to be written in the leftmost position.
2. Error type may contain only disjoint type variables.
   Even if they are rigid.
   For example:
   ```kotlin
   fun <T, V : Value, E1 : Error, E2 : Error, EDB: DbErrors, EP: ParseErrors> foo(
      arg1: T | MyError, // allowed
      arg2: V | MyError, // allowed
      arg3: V | EDB | EP, // allowed
      arg4: V | E1 | MyError, // allowed
      arg5: String | E1 | E2, // not allowed as E1 and E2 are not disjoint
      arg6: T | E1, // not allowed as error component of T and E1 are not disjoint
   )
   ```
   
> Actually, any limitations other than first one are consequences of the inference algorithm.
> TODO: review the algorithm, formalize requirement better, check for forward compatibility

### Relation with `null`

`null` is another special value which is typed orthogonal to class hierarchy.
There already exists a special handling like smart casts, operators and not-null generic types.
If we would like to have these two features separately, 
we have to introduce whole new machinery not only in the compiler but also in the syntax.
It leads to different safe call operators for `null` and for errors.
What if we would like to make a safe call that filters out only errors, only nulls or both?
To cover all of these cases and all possible future interactions,
we have to cover quadratic number of operators and features.

While actually, `null` is mostly used as an error and even if it is used as optional value,
ideas of optional and either are quite similar.
So we decide to merge these two features into one:
- `null` is become a special value of type `error object Null`.
- `T?` is become a syntax sugar for `T | Null`.
- `?.`, `!!`, `?:` becomes an operators for errors

This change is backward compatible, as behavior is not changed for current code.
There is simple translation from nullable types to error unions with only one error, `Null`.

### Operating with errors

The most straightforward way to destruct error union is to use `when` expression:

```kotlin
fun foo(val content: String | ConnectionError | DbError) =
    when (content) {
        is String -> response(content)
        is ConnectionError -> {
            // Here content is smart-casted to ConnectionError
            // So you can access its properties
            logger.error("Connection error: ${content.code}")
        }
        is DbError -> logger.error("DB error: ${content.message}")
    }
```

> TODO: there is some issue with refering to `DbError` with name content.
> Maybe it is reason why other languages introduce two return values for error and value.
> IMO to store them in a single value much better, but maybe we need a syntax for creating a new name or inplace destruction could be enough.
> ```kotlin
> fun foo(val content: String | ConnectionError | DbError) =
>     when (content) {
>         is String -> response(content)
>         is err: ConnectionError -> logger.error("Connection error: ${err.code}")
>         is DbError(message) -> logger.error("DB error: ${message}")
>     }
> ```
> This issue is close to pattern matching

To not check for error after each call as in C, there are several operators: 

- Safe call operator. `?.`
  
  Now it expands into the following:
  ```kotlin
  val v2 = v1?.foo()
  // expands into
  val v2 = if (v1 is Error) v1 else v1.foo()
  ```
  
  So the errors in `v2` is a union of errors in `v1` and errors in `foo()`.
- Bang-bang operator. `!!`
  
  Now it expands into the following:
  ```kotlin
  val v2 = v1!!
  // expands into
  val v2 = 
    if (v1 is Error) {
      throw ErrorException(v1)
    } else {
      v1
    }
  ```
- Elvis operator. `?:`
  
  It would be consistent to expand it into the following:
  ```kotlin
  val v2 = v1 ?: v3
  // expands into
  val v2 = if (v1 is Error) v3 else v1
  ```
  
  But the issue there is that errors will be aggressively swallowed.
  For instance, if a library introduces a new error, 
  all the previous code will silently continue to work, while it may not be expected.
  While good design here should imply a compilation error, which forces programmer to handle new error.
  
  So, it is decided to make it equivalent to function `elvis`, defined as follows:
  ```kotlin
  fun <T : Value, E : Error> elvis(v1: T | Null, v3: T | E): T | E = if (v1 is Null) v3 else v1
  ```
  So, elvis operator will be applicable only for values where there are only `Null` error.
  
  > TODO: in Zig it is swallowing: https://ziglang.org/documentation/master/#catch
  > 
  > TODO: discuss if we should filter out null even if there are other errors.
  > 
  > TODO: discuss if it encourages usage of `Null` instead of meaningful errors.
  > 
  > TODO: Maybe we should filter out any single error in elvis operator? (what about typealias?)

#### New operator

TODO: discuss section

It is fine to apply all existing operators to errors, but with errors we may often encounter the following pattern:

```kotlin
val v = when (val tmp = foo()) {
    is NetworkError -> TODO("Some code")
    is DbError | CacheError -> TODO("Some other code")
    is Error -> tmp // or `throw tmp`
    else -> tmp // Value
}
```

It may be quite useful to introduce a syntax sugar over this pattern.
It is quite complex to find a good syntax for it.
Some options:

- No changes in language, just auxiliary function(s).
  Suboptions:
  - ```kotlin
    val v = foo()
        .on<NetworkError> { TODO("Some code") }
        .on<DbError | CacheError> { TODO("Some other code") }
    ```
- No new syntax, just bind error in a new variable in elvis operator.
  Suboptions: 
  - Implicitly introduce a new variable with name `it` (or `err`).
    ```kotlin
    val v = foo() ?: when (it) {
        is NetworkError -> 0
        is DbError -> 1
        else -> throw it // or err
    }
    ```
    - Backward compatibility issue: there may be existing variables with name `it` or `err`.
    - Issue: no possibility to use another name.
  - Explicitly introduce a new variable.
    Possible syntaxes:
    - ```kotlin
      val v = foo() ?: when (val e) {
          is NetworkError -> 0
          is DbError -> 1
          else -> throw e
      }
      ```
    - ```kotlin
      val v = foo() ?: e -> when (e) {
          is NetworkError -> 0
          is DbError -> 1
          else -> throw e
      }
      ```
- New elvis-like operator that binds variable.
  ```kotlin
  val v = foo() ?| when (err) {
      is NetworkError -> 0
      is DbError -> 1
      else -> throw err // or it
  }
  ```
  ```kotlin
  val v = foo() ?| e -> when (e) {
      is NetworkError -> 0
      is DbError -> 1
      else -> throw e
  }
  ```
- Totally new operator to remove `when` subexpression.
    Suboptions:
    - ```kotlin
      val v = foo() ? { e ->
          is NetworkError -> 0
          is DbError -> 1
          else -> throw it
      }
      ```
    - ```kotlin
      val v = foo() ?| { e ->
          is NetworkError -> 0
          is DbError -> 1
          else -> throw it
      }
      ```
    - ```kotlin
      val v = foo() ?when (val e) {
          is NetworkError -> 0
          is DbError -> 1
          else -> throw it
      }
      ```
  - ```kotlin
        val v = foo() ?! when {
            is NetworkError -> 0
            is DbError -> 1
            else -> throw it
        }
        ```
    - ```kotlin
        val v = foo() !! {
            is NetworkError -> 0
            is DbError -> 1
        }
        ```
    - ```kotlin
        val v = foo() !! when {
            is NetworkError -> 0
            is DbError -> 1
            else -> throw it
        }
        ```
    
> We think that word `catch` should not be used as it is already used for exceptions.
> And we should not create mental relation between them.

### Runtime representation

All errors could be expressed (internally) with a single class:

```kotlin
class Error(val classifier: String, val content: Any?)
// or
class Error<T>(val classifier: String, val content: T)
```

- Classifier is not just a name of the error, but also a scope where it is declared.
  Any checks for errors could be performed using just string equality.
  And these strings even could be transformed into `.intern()` to reduce it to reference equality.
- Content is null for errors without data.
- If there is some data, there are several options to store them.
  - If there is one field, we may store it directly.
  - If there are several fields, we may declare backed data class or store them in the array

  These options have to be discussed in the context of binary compatibility and performance.

With this approach and declaration of the error could be transformed into:
- For errors without data: just a val with the only instance of this error.
  This allows operating with this error as with `null`, because we are able to use reference equality.
  This case is actually transformed into "Another special value for reference with the performance comparable to null."
- For errors with data: constructing function or nothing if we would like to construct them in-place.

### Typing

#### Types

- New types: `Error` (supertype for all errors), `Value` (supertype for all common types)
- New type constructor: `A | B` (error union)

#### Well-formattedness

`A | B` is well-formed if:
- `A` and `B` are well-formed
- `B <: Error`
- `B` does not contain 2 non-disjoint variables
  > It means that errors may have forms:
  > - List of explicitly written error constants (further denoted as `Errs`)
  > - `E | Errs`, where E is an unbounded error variable
  > - `E1 | E2 | E3 | Errs`, where `E1` and `E2` and `E3` are error variables with disjoint upper bounds.
      F.e. DbErrors, NetworkErrors, and CacheErrors
- Constants in `B` are disjoint

#### Operations

- `T|_v`
    - `T <: Value` => `T|_v = T`
    - `T <: Error` => `T|_v = Nothing`
    - `T = A | B` => `T|_v = A|_v`
- `T|_e`
    - `T <: Value` => `T|_e = Nothing`
    - `T <: Error` => `T|_e = T`
    - `T = A | B` => `T|_e = A|_e | B`
- For `T <: Error`: `[T]` -- interpret `T` as a set of classifiers

#### Subtyping

New axioms:
- `Any :> Error`
- `Error :> MyError` for all `MyError`
- `MyError :> Nothing`
- `Any :> Value`
- `A :> B | C <= A :> B & A :> C`
- `A | B :> C <= A|_v :> C|_v & [A|_e | B] \subset [C|_e]`

#### Inference

First idea is that we replace any flexible variable that is not subtype of `Value` or subtype of `Error` with two variables: `T|_v` and `T|_e`.

Then we have to figure out how to solve constraints on error variables where they are interpreted as sets.
Actually, we have a list of constraints on sets with kinds:
- `A \subset B`, where `A` and `B` are predefined sets or variables
- `A \union B \subset C`, where
    - `A` and `C` are predefined sets or variables
    - `B` is a predefined set

Simple algorithm:
- Start with empty sets for all variables
- Fix all constraints that are not satisfied by adding (minimal amount of) elements into sets

Have complexity around `O(T * log(T) * C)`, where
- `T` is a number of elements in all sets
- `C` is a number of constraints

Such approaches usually lead to complex error messages.
Possible solutions:
- As we incorporate constraints one by one, we know the first one that is not satisfied => we may provide a message for it
- If we drop all upper-bounding constraints,
  we will infer the smallest possible sets satisfying all lower-bounding constraints.
  Then we may just check if the resulting type is a subtype of the expected type and provide a message if not.

#### Backward compatibility with nullability

TODO: review mapping, consider corner cases

### Relation to other features

#### Smart casts

We have to add support for error unions in smart casts.
For example:

```kotlin
val v: Int | Error1 | Error2 = foo()
if (v is Error1) {
    v // Error1
} else {
    v // Int | Error2
}
``` 

It is straightforward to make a smart cast in a first branch.
Actually, it works almost the same as now.

While the second branch is more complicated as it does not have any analogues in the compiler for now.
While the semantics of it is quite clear: 
We have to exclude the checked type from the union 
(more precisely, exclude all union items that are subtypes of the checked type).

> For this case:
> ```kotlin
> val v: Int | E = foo()
> if (v is Error1) {
>   v // Error1
> } else {
>   v // Int | E
> }
> ```
> We do not want to infer something like `E \ Error1`.
> Because it will lead us to a more complicated system, resolution of which is subexponential, non-deterministic, 
> and it produces worse error messages.
> (see [Set-theoretic types](https://www.irif.fr/~gc/papers/set-theoretic-types-2022.pdf))
> 
> On the contrary, if we have something like:
> ```kotlin
> val t: T
> t!!
> ```
> We should smart-cast `t` to `T & Value`

## Extra considerations

### Mixing with value types

#### Kotlin's value types

TODO

#### Valhalla's value types

TODO

### Abuse cases

Error types may be (ab)used as special values instead of errors.

For example, it may be used like this:

```kotlin
sealed interface Tree {
    object Leaf : Tree
    data class Node(val left: Tree, val right: Tree) : Tree
}

// replaced with:

class Node(...) 
error Leaf
typealias Tree = Node | Leaf

// or even 

error Leaf
error Node(...)
typealias Tree = Unit | Node | Leaf   
```

All these cases are considered as a misuse of the feature.
It looks impossible and impractical to prevent them so we have to rely on the developer's discipline.

## References

- [Marat's quip (April 11)](https://jetbrains.quip.com/fOg9A3IXwD4b/Restricted-union-types)

## Future possibilities

### Origin of errors

TODO: it is possible to bubble errors if they were called with some specific modifier?

### Local error unions

Local error type is an error defined in a specific scope.
They express some exceptional state of the function/class that exists only in internal logic.
They are expected not to leave it neither on the type level nor on the value level.
They are declared in the same way as global errors but with visibility modifiers.

```kotlin
fun last() {
   error NotFound
}

class C {
   private error NotFound
}
```

It is straightforward to control their scope on the type level as it is the same as for the common local classes.

To track their scope on the value level, we declare them as not subtypes neither of `Error` nor of `Any`.
But we leave them as a supertype of `Nothing`.
Because of this, for every value that may contain this error, it has to be directly expressed in the type.
Thus, if a type is not exposed out of the declared scope, the value is not exposed either.

The issue with this approach is that we are not able to pass value with this error in any function.
Even if we would like to have a private function expected to accept such value as an argument,
we have to specify it explicitly.
And if we would like for this function to optionally accept this error, we have to write such a boilerplate:
```kotlin
private error MyError

private fun <T, LOCE : MyError> foo(v : T | LOCE): T | LOCE
```
As a result, any value with such errors has significantly limited usability.

The main use-case of local errors is `last` function.
It is possible to implement it using a common errors.
If we just limit the scope of the error on the type-level (common private error)
and developer will track that this error is not exposed outside the function.
But with local errors, it is possible to guarantee the correctness of non-exposure on the type level.

> Because of such limitedness of their applicability we may introduce another modifier (`local`) for local errors
> and use `private` modifier in a same way as for classes.

> To expand the applicability of this feature,
> we may introduce a common supertype for all errors plus all errors local to the current scope.
> F.e. `Error@last` which is a supertype of `Error` and errors local to class of `last` and `last` function itself.

> TODO: discuss if this feature really needed.
>
> IMO it is too complicated, not so useful and does not align with the other language.

### Richer type system for errors

TODO: Exclusions

TODO: more complex non-disjoint cases for variables (E1 : Error, E2 : DBError)

TODO?: `Err(v)` representing errors of variable(argument) `v`

TODO: `Int??` as a shorthand for `Int | Error` or `Int | $E`
