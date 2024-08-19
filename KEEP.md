# Union Types for Errors

## References

- [Marat's quip (April 11)](https://jetbrains.quip.com/fOg9A3IXwD4b/Restricted-union-types)
- [Youtrack issue](https://youtrack.jetbrains.com/issue/KT-68296/Union-Types-for-Errors)
- [Set-theoretic types](https://www.irif.fr/~gc/papers/set-theoretic-types-2022.pdf)

## Problem statement

> Copy from the issue

### Issue categories

1. Internal errors.
   In this case we would like to have a special error state that is local to the current function/class(/package?)
   Example of such function is `last` function, class is `HashMap`.
   This states is expected to express some internal states and does not be exposed outside of the scope.
   Not as types but as values as well.
   So this function/class is able to rely on the fact that this error will not be in the input parameters or anywhere else.
2. External errors.
   In this case we would like to express exceptional states that could happen in the function/class.
   The example of this use-case are functions in stdlib with several overloads (`OrNull` vs `OrThrow`).
   The desired output of this feature is to unify them in a single function which could be easily handled in both manner.
   More precisely, it has to be easily transformed into exception and easily posphoned to handle using language features like conditional call and elvis operator.

### Background

Zig

Rust

Either

Effects

### Goals

1. Cover both mentioned use-cases.
2. Mainimize boilerplate required to operate with errors to the same level as with nulls.
3. Minimal performance overhead.
   Desirably to have an errors without data with the performance comparable to `null`.

## Proposal

### Approach

We had a preliminary discussion about a design where errors have a separate hierarch with the same features as an existing classes.
This approach is directed by a goal "Let's add as many features as possible to remain errors inferrable".
But this approach have several significant issues.

The first problem of this full-flavored approach is performance-based.
All functions that would like to have a specific internal state, creates a new private class.
If they are mapped to a separate JVM classes, it leads to significant amount of small classes in JVM and class-loader calls.
This performance issue is similar to one encountered in the JVM at the moment of lambda's introduction.

The other and more significant problem with full-fledged approach is not easily supported by the type inference.

When unions are discussed in terms of Kotlin's future, they are required to be disjoint.
But in the case of errors, it is not possible.
Let's consider the example of the `find` function.
We would like to replace two overloads of this function (with null and with exception) with a single version with error union.
Resulting function is expected to be the following:

```kotlin
fun <T> Iterable<T>.find(f: (T) -> Boolean): T | NotFound
```

But it is not allowed as `T` and `NotFound` are not disjoint.
So we would like to allow some kind of non-disjointness for error unions.
To achieve this, we are prohibiting subtyping between errors.
As otherwise, it will lead to the exponential type inference. 
(TODO: check this)

Thus, we moved into approach where we try to make as few features as possible that will cover all known use-cases.

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
      arg6: T | E1, // not allowed as T and E1 are not disjoint
   )
   ```

We should add the same conditional call and elvis operators for error unions as for nullable types.
For example:

```kotlin
val a: Int | MyError = foo()
val b = a ??: 0
val c = a??.toString()
```

To destruct error union, common `when` expression may be used:

```kotlin
fun foo(val content: String | ConnectionError | DbError): Int? | OtherError =
    when (content) {
        is String -> content.toInt() ??: OtherError
        is ConnectionError -> OtherError
        is DbError -> null
    }
```

#### Local error unions

Local error type is an error defined to be local.
There are several options to do this:
1. If we would like to have required error declaration:
   - ```kotlin
     fun last() {
         error NotFound
     }
     ```
     or
     ```kotlin
     class C {
         private error NotFound
     }
     ```
2. If we would like to have an optional error declaration:
   ```kotlin
    fun <T> last() {
       val last: T | `NotFound@last = `NotFound@last 
    }
   ```
   
From these types we would like to have a guarantee that they do not leak outside the scope.
Not just that their type is not exposed, but also that their value is not leaked.
If we would like to control this on the type-level:
1. Local errors are not subtypes neither of `Error` nor of `Any`.
   It may complicate the type system and writing code with such errors.
2. We may introduce something like `Error@last` which is a supertype of `Error` which includes all errors local to class of `last` and `last` itself.
    
So all of these approaches lead to significant complication of the type system.
Thus, we may just not check anything and leave it on the developer's responsibility to track the correct usage of local errors.

In this case local errors will be used in the same way. 
`NotFound` from different scopes will be different types.
Thus, the correct implementation of the `last` function will be possible, but just not guaranteed.

#### Generics in errors

We may allow not only just content in th error, but also some generic parameters.
It leads to several complications in the type inference, but does not break it.

Any union of errors may not have two errors with the same classifier and different generic parameters.
Their generic parameters will be unified respected to variance.

### Runtime representation

Despite the option to use errors (predeclared or without declaration) errors could be expressed with a single class:

```kotlin
class Error(val classifier: String, val content: Any?)
```

- Where classifier is not just a name of the error, but also a scope which it is bind to.
  And any checks for errors could be done using just string equality.
  And these strings even could be transformed into `.intern()` to reduce it to reference equality.
- Content is null for errors without data.
  How to store data that it will not lead to complication if the data is changed?

The overhead of boxing could be even removed for the errors without data. 

### Inference

#### Without generics

##### Types

- New types: `Error` (supertype for all errors), `Value` (supertype for all common types)
- New type constructor: `A | B` (error union)

##### Well-formattedness

`A | B` is well-formed if:
- `A` and `B` are well-formed
- `B <: Error`
- `B` does not contain 2 non-disjoint variables
  > It means that errors may have forms:
  > - List of explicitly written error constants (further denoted as `Errs`)
  > - `E | Errs`, where E is unbounded error variable
  > - `E1 | E2 | E3 | Errs`, where `E1` and `E2` and `E3` are disjoint error variables.
      >   F.e. DbErrors, NetworkErrors, and CacheErrors
- Constants in `B` are disjoint

##### Operations

- `T|_v`
    - `T <: Value` => `T|_v = T`
    - `T <: Error` => `T|_v = Nothing`
    - `T = A | B` => `T|_v = A|_v`
- `T|_e`
    - `T <: Value` => `T|_e = Nothing`
    - `T <: Error` => `T|_e = T`
    - `T = A | B` => `T|_e = A|_e | B`
- For `T <: Error`: `[T]` -- interpret `T` as a set of classifiers

##### Subtyping

New axioms:
- `Any :> Error`
- `Error :> MyError` for all `MyError`
- `MyError :> Nothing`
- `Any :> Value`
- `A :> B | C <= A :> B & A :> C`
- `A | B :> C <= A|_v :> C|_v & [A|_e | B] \subset [C|_e]`

##### Inference

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
  Then we may just check if the resulting type a subtype of the expected type and provide a message if not.

#### With generics

##### Types

Nothing new

##### Well-formattedness

Nothing new

##### Operations

- For `T <: Error`: `[T]` -- interpret `T` as a mapping from classifiers to type parameters

##### Subtyping

Changed only the last one:
- `A | B :> C <= A|_v :> C|_v & [A|_e | B] \subsumes [C|_e]`

##### Inference

We have to figure out how to solve constraints on error variables where they are interpreted as maps instead of sets.

In our simple algorithm, we are adding classifiers to variables one by one.
After we have added a new classifier, we initialize mapping to type parameters with the fresh variables.
Then we add constraints on type parameters for this classifier from this `\subsumes` constraint.

If it is not ambiguous, everything is finished.

When could it be ambiguous?
If we try to do it in such a constraint: `{MyErr -> Int} | A={MyErr -> B} \subsumes C={MyErr -> D}`
Example of code to produce this:

```kotlin
fun <T : Value, E : Error> foo(a: T | E | MyErr<Int>, b: T | E)

val a: Int | Nothing
val b: Int | MyErr<String>
foo(a, b)
```

In this case, we have to generate constraint depending on variance:
- If `MyErr` is invariant: `D = Int | B`. Additionally, `B = Int`. No problem there.
- If `MyErr` is covariant: `D <: Int | B`. Which is ambiguous.
- If `MyErr` is contravariant: `Int | B <: D`. Equivalent to `D :> Int & D :> B`. No problem there.

As ambiguous constraints are not possible from another side, the only problem is covariant `MyErr`.

Options to resolve this issue:
- No generics in errors (overkill as there are no problems with invariant)
- No (co)variance in errors
- Fix mapping to type parameters if we have an explicit type.
    - Strictly.

      If there is a constraint `{MyErr -> Int} | A={MyErr -> B}` (which originates to the same type) we may fix `B` to `Int` and constraint on type parameters will be unambiguous.

      For our example it will lead to error for argument `b`: `MyErr<Int> expected, but MyErr<String> found`.
      Despite the variance.
    - Softly.

      We may state that `B` is a subtype/supertype of `Int`.
        - In case of supertype we result in a constraints `Int <: B` and `D <: B`.
          And in our example `E` will be inferred as `MyErr<Serializable & Comparable<*>>`.
        - In case of subtype we result in a constraints `B <: Int` and `B <: Int`.
          And in our example `E` will be failed to infer.

The other problem is that if we treat `B` as a supertype of `Int` in case of covariant `MyErr`,
we may infer `E` to `MyErr<Serializable & Comparable<*>>`.
In this case, when we match `a` inside the function, we, unexpectedly, do not result in a `MyErr<Int>`.
So the only option is to treat `B` as a subtype of `Int`.

In case of contravariant `MyErr`, based on the same problem we have to fix `B` to be a supertype of `Int`.
So the constraint reduced from `D :> Int | B` is `D :> B` and `B :> Int`.

Summarizing, we have several options for handling generics:
- No generics
- Invariant generics fixed to explicitly written
- Generics strictly fixed to explicitly written
- Generics softly fixed to explicitly written with respect to variance

### Relation to other features

#### Smart casts

We should add support for error unions in smart casts.
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
While the semantics of it is quite clear, we just have to exclude the checked type from the union.

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
> Because it will lead us to more complicated system, resolution of which is subexponential afaik.
