# Union Types for Errors

## Problem statement

### Overview

In this KEEP, we propose a new way of error-handling supported on the language level in Kotlin.
It is expected to cover cases similar to those covered by `Either` in different languages.

It is possible to express full-power `Either` in Kotlin using sealed classes.
Moreover, there is already a `Result` class in the standard library 
that have a similar purpose but limited to `Throwable` as errors.
The advantage of the proposed feature over these approaches is 
that it has better language support, flexibility and performance.

### Covered use-cases

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

##### Unions as tags

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

This code struggles with the same issues as were with `null`: 
we do not have guarantees that `NotFound` will not be in the input sequence.
To avoid this we may declare `NotFound` private to `last` function and does not expose it outside:

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

It is still possible that `NotFound` will leak due to programmer's mistake, 
but for now it depends only on the developer of the function, not on the function's users.
But it may be useful to provide a type level guarantee that `NotFound` will not leak outside the function. 

#### ...OrThrow / ...OrNull

It's quite common to encounter functions that describe part of their effects for error cases in their names.
This pattern could be enhanced by having only one function with a functional name like `get` or `processRequest`,
where exceptional cases are encoded in the signature.

A typical example could be our functions `maxOf` 
([code](https://github.com/JetBrains/kotlin/blob/0938b46726b9c6938df309098316ce741815bb55/libraries/stdlib/common/src/generated/_Arrays.kt#L14700)) 
where many functions are duplicated to have `maxOfOrNull` counterpart in case of empty collections.

##### Moving unions to return types

With the error union types, it could be possible to define one method that can encode all the necessary error cases:

```kotlin
// getOrThrow -> get
fun <T> get(): T | Error

// maxOrNull -> max
fun IntArray.max(): Int | NoSuchElement

// awaitSingleOrNull -> awaitSingle
fun <T> awaitSingle(): T | NoSuchElement
```

However, to make the pattern shine in Kotlin, 
it is essential to provide deconstruction operators for such unions at call-sites.

This is where our presentation of unions as 'main' or 'extras' also works well
We can consider operators `??.` (to compound errors on the left side) 
and `??:` (elvis analogue, to evaluate RHS if there is an error)

<!---
### Issue categories

1. Internal errors.
   In this case, we would like to have a special error state that is local to the current function/class(/package?)
   Example of such a function is `last`.
   These states are expected to express some internal states and do not be exposed outside the scope.
   Not as types but as values as well.
   So this function/class is able to rely on the fact that this error will not be in the input parameters or anywhere else.
2. External errors.
   In this case, we would like to express exceptional states that could happen in the function/class.
   The example of this use-case are functions in stdlib with several overloads (`OrNull` vs `OrThrow`).
   The desired output of this feature is to unify them in a single function which could be easily handled in both manners.
   More precisely, it has to be easily transformed into exception 
   and easily postponed to handle using language features like conditional call and elvis operator.
-->

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
- `Either` with several errors is sensitive to the structure and order:
  `Either<A, Either<B, C>>` is not the same as `Either<Either<A, B>, C>` or `Either<B, Either<A, C>>`.
  Which may lead to complications for code refactoring or reordering.
- Error type in `Either` is a common type in a Kotlin type system.
  Consequently, if you would like to have several possible errors, you have to create a sealed hierarchy for them.
  And you will not be able to return a subset of this hierarchy by handling only a part of errors.
- Even with the introduction of arbitrary unions, due to expressiveness of common types, unions on them will be limited.

#### Error union type in Zig

The most similar feature to the proposed one is [error union types in Zig](https://ziglang.org/documentation/master/#Error-Union-Type).

The broad overview of the feature is the following:
- Error types are just tags without data, represented as integers at runtime.
- Arbitrary unions are allowed.
- Allowed inference of error in return type. (But it is limited)
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
While they are great to track some io interactions and more complex exceptions, 
they may be too complex to handle simple cases like function `last` or that user's age is not in the range.

### Goals

1. Cover mentioned use-cases.
2. Minimize boilerplate required to operate with errors to the same level as with nulls.
3. Minimize performance overhead.
   Preferably to have an errors without data with the performance comparable to `null`.

## Proposal

### Approach

We had a preliminary discussion about a design where errors have a separate hierarch with the same features as an existing classes.
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

### Design overview

Error type is a separate type kind, which is declared using a soft keyword `error`.

```kotlin
error class MyError(val code: Int)
```

> There is an approach where an error type could be declared in-place using different syntax.
> For example:
> ```kotlin
> fun foo(): Int | `MyError
> // or
> fun foo(): Int | `MyError(Int)
> ```
>
> But this approach looks worse than the proposed one.
> - Firstly, it does not align with the existing Kotlin style
> - Secondly, it does not allow making declaration-site optimizations
> - Finally, it allows clashing errors from different libraries. Which is not expected in 99% of cases.

Error type may not have any supertypes (use delegation instead)
other than new common supertype for all of them, `Error`.
We are also introducing new type `Value` which is a supertype for all non-error types.
The resulting subtype hierarchy between those types is the following:
1. `Any :> Value` and `Any :> Error`
2. `Value :> Int`, `Value :> String`, etc.
3. `Error :> MyError`, `Error :> ConnectionError`, etc.
4. `MyError :> Nothing`, `ConnectionError :> Nothing`, etc.

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

We should add the same conditional call and elvis operators for error unions as for nullable types.
For example:

```kotlin
val a: Int | MyError = foo()
val b = a ??: 0
val c = a??.toString()
```

To destruct error union, common `when` expression may be utilized:

```kotlin
fun foo(val content: String | ConnectionError | DbError): Int? | OtherError =
    when (content) {
        is String -> content.toInt() ??: OtherError
        is ConnectionError -> OtherError
        is DbError -> null
    }
```

To get a value from the error, you have to
- (smart-)cast it to specific error
- Then it is possible to use classic field reference or destructuring syntax as for common data classes:

```kotlin
error NetworkError(val code: Int, val message: String)

fun foo() {
    val v: Json | NetworkError = request()
    if (v is NetworkError) {
        val (code, message) = v
        // or
        val code = v.code
    }
}
```

> Due to another kind of errors it is possible to allow call `.code` 
> if we have union of errors where each error have `code` field.
> But it requires additional investigation.

To allow easily transform `errors` into exceptions or null, we may introduce a new functions:
    
```kotlin
fun <T> (T | Error).orThrow() {
    contract { returns() implies (this !is Error) }
    // ...
}

fun <T> (T | Error).orNull(): T? {
    // ...
}
```

### Relation with `null`

The problem of `null` is that it is currently used both as a special value and as an error.
Are these cases really different?
As we are introducing a special construction for errors, what is the future of `null`?

Firstly, let's answer the question: Are we really interested in this division?
At first sight, it may affect how user handles errors and special values.
- Errors may be:
  - Processed with different logic.
    For example, display different messages for user or try to silently refresh the token in case of network error.
    Required language support: `when` expression.
  - Processed with simple same logic.
    For example, pass to logger, replace with default value, return them or throw an exception.
    Required language support: elvis operator + `orThrow`, `asException`, `check`, `require` functions + `!!` operator.
  - Processed with complex same logic.
    For example, restore state or retry the operation.
    Required language support: `is Error` condition for `if` expression.
  - Accumulated by different calls to be bunch-processed later.
    To not process them into C-style error codes.
    Required language support: safe call operator.
  - TODO...
- Special values may be:
  - Processed with different logic.
    Required language support: `when` expression.
  - Replaced with default value (are they actually a value in this case?).
    Required language support: elvis operator.

As a result, we are interested in this division because special values do not require any special language support.
While errors require that all mentioned operators applied only to errors, not to special values.
For example, if we consider `null` as a special value, we have to have an operator that filters out only errors.

Secondly, let's iterate over all nine possibilities of error design and relation to `null`:

|                              | Errors are only errors | Errors explicitly divided into errors and states | Errors implicitly divided into errors and states |
|------------------------------|------------------------|--------------------------------------------------|--------------------------------------------------|
| `null` is only error         | 3                      | 1.1.1                                            | 1.2                                              |
| `null` is only special value | 2.1                    | 1.1.2                                            | 1.2                                              |
| `null` is both               | 2.2                    | 1.1.3                                            | 1.2                                              |

Numbers are references to the elaboration of the combinations.

Elaboration of single choices:
- "`null` is only error".
  In this case, we consider:
  - Users should use `null` in cases function may fail, and they do not care about the reason.
  - Users should not use `null` as a built-in `Optional`.
  
  Consequences:
  -  `null` have to be deprecated in favor of error `Null`.
     Because if it is only an error, it is better to merge the two concepts.
  -  It sounds impossible as it means that we have to push all the users to rewrite their code with `Optional`
- "`null` is only special value".
  In this case, we consider:
  - Users should use `null` only as a built-in `Optional`.
  - Users should not use `null` as an error in favor of explicit and more informative errors. 
    Or just `T | Failure` if they do not care about the reason.
  
  Consequences:
  - We have to have separate operators for `null` and errors.
    As they have to exist for `null` due to backward compatibility and for errors due to the feature requirements.
  - We have to push users to rewrite their code where `null` is used as an error.
    Which is ok as it is actually why we are introducing this feature,
    and it is easy to do as they will have a backward compatible operators.
- "`null` is both".
  In this case we consider users should use `null` in any case they want.
- Errors are only errors.
  In this case, we consider that errors are used only as errors.
  And any case and issue where they are used as special values should be considered as a bad design and ignored.
- Errors explicitly divided into errors and states.
  In this case, we consider that errors are used both as errors and as special values, 
  and it is implemented on the language level.
  For example:
  ```kotlin
  error NetworkError(val code: Int, val message: String)
  state NoValue
  
  fun request(): Int | NoValue | NetworkError
  ```
  
  Consequences:
  - We have to filter out only errors in the operators.
  - Unpredictable behavior for the constructs that looks similar.
  - Looks alien to the OO-language design.
- Errors implicitly divided into errors and states.
  In this case, we consider that errors are used both as errors and as special values, 
  but it is not implemented on the language level.
  For example:
  ```kotlin
  error NetworkError(val code: Int, val message: String)
  error NoValue
  
  fun request(): Int | NoValue | NetworkError
  ```
  
  Consequences:
  - We should allow user to choose what to filter out in this specific case.
  - We have to cover some cases that may be considered as a bad design for OO-language.

Let's review the combinations on a specific example.
We have a function `request` that may return `Optional<Int>` or `NetworkErrors` 
(`NetworkErrors` should be considered as a typealias on several errors).
Which return type is expected?

1. ```kotlin
   fun request(): Int | NoValue | NetworkErrors
   ```
   In this case, errors are used both as special values and as errors.
   If we see this as an expected design, we state that errors may be used anywhere as a special value.
   And there may be more than one special value (f.e. `Int | Uninitialized | NoValue`).
   Whether this "error" is an error or special value may be determined on the declaration-site or use-site.
   
   1. Declaration-site.
      In this case we have to introduce a new modifier `state` for errors that are used as special values.
      ```kotlin
      error class NetworkError(val code: Int, val message: String)
      state class NoValue
      ```
      We also have to have an operator that filters out only errors.
      1. If `null` is considered as an error, we may re-use the same operators as for nullable types.
      2. If `null` is considered as a special value, we have to introduce a new operators.
      3. If `null` is considered as both, we are:
         - not able to transform `null` into `Null`.
         - have to introduce a new operators.
         - have to allow merging operators for errors and null concisely.
           For cases where null is used as an error.

         So this case looks strictly worse than any other.
   2. Use-site.
      In this case, we have to allow user to choose what is considered as an error in this context.
      To allow this, we have to add a new generic parameter for operators.
      ```kotlin
      val v = request() ?:<NetworkErrors> return
      val v = request()?.<NetworkErrors>process()
      val v = request()!!<NetworkErrors>
      ```
      > It may be called as "Sad Elvis operator".
   
      In this case null could be unconditionally transformed into `Null` and user will decide what to filter out.

   Consequences:
   1. We should deprecate `null` in favor of error `Null`.
     It will force people to use more sound names for errors in their cases and leave `Null` only for legacy and Java/JVM interop.

   Advantages:
   1. No `null`.
   2. Less syntactic categories, no different (but similar) handling of `null` and errors in compiler and programmer's mind.
   
   Disadvantages:
   1. It looks alien to the OO language design.
   2. It may be harder to design `Error` typing to be comfortable for both errors and special values. TODO: reflect on this.
2. ```kotlin
   fun request(): Int? | NetworkErrors
   ```
   This case is divided into two:
   1. `null` is used only as a special value.
      Errors are used only as errors.
      
      Consequences:
      1. We do not consider `null` as error in our design.
         (Only in terms of backward compatibility)
      2. We disallow any future view on `null` as an error `Null`.
      3. Theoretically, it is possible to change the semantics of operators for `null` into operators for errors and do not introduce new operators.

      Advantages:
      1. No significant changes in the language.
      2. Overhead-less optional for JVM in composition with errors
   
      Disadvantages:
      1. `null` exists.
      2. `null` is too similar to errors but different.
         People may continue to use it as an error and report something as a bad design.
      3. Different operators for `null` and errors.
         More complicated syntax.
      4. After release of project Valhalla, there will be another overhead-less `Optional` in the language.
         (But it will have an overhead if it is mixed with errors)
   2. `null` is allowed to be used as both.
      It is mostly the same as the previous case.
      Differences:
      - We have to add an operator that filters both nulls and errors.
        Otherwise, we state that if you used `null` as an error, you do not have any operators.
        It is bad and easier to just consider option where null is only a special value.
      - It is not possible to change the semantics of operators for `null` into operators for errors.
3. ```kotlin
   fun request(): Optional<Int> | NetworkErrors
   ```
   In this case, we consider users should design hierarchy for values.
   Errors are used only as errors and named properly.
   `null` is used as a specific error `Null`. 

   Consequences:
   1. We should deprecate `null` in favor of error `Null`.
   2. Operators could be easily re-used from nullable types.
   
   Advantages:
   1. No `null`
   2. Less syntactic categories, no different (but similar) handling of `null` and errors in compiler and programmer's mind.
   
   Disadvantages:
   1. Requires more letters to better design the value hierarchy.
   2. No overhead-less optional for JVM.
   3. Users may continue to use null for optional and do not use errors at all.
   
TODO: more advantages and disadvantages
TODO: formatting (numbering only for suboptions)
TODO: discuss

> IMO we should go for the option 2.1
> Thus, `null` has to be considered only as a special value, and errors are only as errors.
> Use-cases where `null` is used as error or errors as special values we should consider as bad design 
> and do not cover them.

### New operators

To make the feature more usable, we have to introduce the same operators as for nullable types.
The simple idea is to replicate them with the same semantics:
- `??.` -- conditional call
- `??:` -- elvis operator
- `?!!` -- force operator

Or we may introduce a new symbol for errors to make operators shorter: `#.`, `#:`, `#!!`/`##`/`#!`.

As we consider null as a special value, we may ignore cases where user would like to filter both nulls and errors.
This is because it is intended to be a different operations, 
thus happen together in rare cases where two consecutive operators represent two different operations.
For example:

```kotlin
val v = foo() ??: return
              ?: return
```

This code is either a bad design because `foo` should use one more error that is hidden under `null`.
Or it is a good code that just does the same operations in case of two totally different cases 
(error happens in `foo` or `foo` returned `Optional.None` aka `null`).

Safe call on both nulls and errors is not possible,
but if you use `T?` as optional value, then your function should get `T?` as a receiver.
Thus, the cases where it is required may be considered as a bad design.

#### More new operators

If we want to continue considering null as both a special value and an error.
Then we should cover cases considered as a bad design in the previous section.
Thus, we have to introduce operators that filter out both nulls and errors.
They may look like:
- `???.`, `???:`, `??!!`
- `#?.`, `#?:`, `##!!`

<!--

#### No new operators

If we consider option where no new operators required, the only problem is functions like this:

```kotlin
fun <T : Any> containsNulls(l: List<T?>): Boolean {
   l.forEach { it ?: return true }
   return false
}
```

Yes, this function is not written well, but it may exist and user may expect that it will work as written.

-->

### Difference between class and object

TODO: discuss do we want to separate `error object` and `error class` or not.

> I propose not to introduce `error object` and have only `error class`, 
> which is optimized to unique if they do not have data.

We are able to transform errors without data into unique constants even without `object` keyword.
Shouldn't we leave it on the optimization level?
It will allow users to write errors easier as they do not have to think about the choice between `object` and `class`.

### Error unions as types for compilation phases

TODO: to filter impossible ancestors or states

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

### Inference

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
  > - `E1 | E2 | E3 | Errs`, where `E1` and `E2` and `E3` are disjoint error variables.
      >   F.e. DbErrors, NetworkErrors, and CacheErrors
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
> t ??: return
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

If we would like to handle these cases in some way we may introduce a new modifier `state` for errors
that are expected to use as special values.

## References

- [Marat's quip (April 11)](https://jetbrains.quip.com/fOg9A3IXwD4b/Restricted-union-types)
- [Youtrack issue](https://youtrack.jetbrains.com/issue/KT-68296/Union-Types-for-Errors)

## Future possibilities

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
