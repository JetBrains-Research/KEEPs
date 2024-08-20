# Union Types for Errors

## References

- [Marat's quip (April 11)](https://jetbrains.quip.com/fOg9A3IXwD4b/Restricted-union-types)
- [Youtrack issue](https://youtrack.jetbrains.com/issue/KT-68296/Union-Types-for-Errors)
- [Set-theoretic types](https://www.irif.fr/~gc/papers/set-theoretic-types-2022.pdf)

## Problem statement

The problem addressed in this KEEP is the support on the language level for error handling 
in a way similar to `Either` in several languages or `Result` that is already presented in Kotlin.
The difference is that we would like to have a more lightweight, flexible solution with more language-level support.

### Covered use-cases

Let's review the main use-cases for such a feature.

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
It's a typical pattern for functions that retrieve data from containers (single/singleOrNull/last/...).

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

#### Unions as tags

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

The other issue with this approach is the same as with null: input sequence may contain `NotFound` itself.
To avoid this we may declare `NotFound` inside the `last` function and does not expose it outside.
And it would be great if the type system could guarantee that this error will not be exposed outside the function.

#### ...OrThrow / ...OrNull

It's quite common to encounter functions that describe part of their effects for error cases in their names. 
This pattern could be enhanced by having only one function with a functional name like get or processRequest, 
where exceptional cases are encoded in the signature.

A typical example could be our functions `maxOf` 
([code](https://github.com/JetBrains/kotlin/blob/0938b46726b9c6938df309098316ce741815bb55/libraries/stdlib/common/src/generated/_Arrays.kt#L14700)) 
where many functions are duplicated to have `maxOfOrNull` counterpart in case of empty collections.

#### Moving unions to return types

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

### Issue categories

1. Internal errors.
   In this case we would like to have a special error state that is local to the current function/class(/package?)
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

### Background

TODO: Add information about similar features in other languages.

Zig

Rust

Either

Effects

### Goals

1. Cover both mentioned use-cases.
2. Minimize boilerplate required to operate with errors to the same level as with nulls.
3. Minimal performance overhead.
   Preferably to have an errors without data with the performance comparable to `null`.

## Proposal

### Approach

We had a preliminary discussion about a design where errors have a separate hierarch with the same features as an existing classes.
This approach is directed by a goal "Let's add as many features as possible to remain errors inferrable".
But this approach has several significant issues.

The first problem of this full-flavored approach is performance-based.
All functions that would like to have a specific internal state create a new private class.
If they are mapped to separate JVM classes, 
it leads to a significant number of small classes in JVM and class-loader calls.
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
    abstract error DbError
    error DbConnectionCancelled : DbError
    error DbTokenExpired : DbError
    // replace with:
    error DbConnectionCancelled
    error DbTokenExpired
    typealias DbError = DbTokenExpired | DbConnectionCancelled
    ```
  - Subtyping forces us to use 1to1 mapping from errors to JVM classes which have mentioned performance issues
  - Type inference for errors will never infer a supertype.
    Instead, it will just list all possible actual errors.
    So supertype is only for explicit annotations.
  - We do not want users to abuse this feature.
    And in terms of errors, the actual value of a supertype is just a list of subtypes.
    Any abstract method is a sign of abuse.
    So typealias is even more semantically sound for errors.
    In an extra cases where it is required, typealias + extension function is all you need.

### Design overview

#### Global error unions

Error type is a separate type kind, which is declared using a keyword `error`.

```kotlin
error MyError(val code: Int)
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
    contract { returns() implies (this@orThrow !is Error) }
    // ...
}

fun <T> (T | Error).orNull(): T? {
    // ...
}
```

#### Local error unions

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
(TODO: check if it is safe)
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

> Because of such limitedness of their applicability we may introduce another modifier (`local`) for local errors and use `private` modifier in a same way as for classes.
> Or just do not have such a feature and leave everything on the programmer.

> To expand the applicability of this feature, 
> we may introduce a common supertype for all errors plus all errors local to the current scope.
> F.e. `Error@last` which is a supertype of `Error` and errors local to class of `last` and `last` function itself.

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
> Because it will lead us to a more complicated system, resolution of which is subexponential AFAIK.
