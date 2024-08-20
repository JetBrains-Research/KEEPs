## Links

[Marat's quip (April 11)](https://jetbrains.quip.com/fOg9A3IXwD4b/Restricted-union-types)

## June 4

We've discussed that nobody knows what they want from this feature.

[Scala's unions doc](https://docs.scala-lang.org/scala3/reference/new-types/union-types-spec.html)

### What they (probably) want

- Dynamic resolution.
  They would like to be able to call a function defined (not virtually) on all the types in the union.
  Examples:
  ```
  error Err1
  error Err2

  fun Err1.foo() = println("Err1")
  fun Err2.foo() = println("Err2")

  fun bar() {
    // ...
    // err: Err1 | Err2
    err.foo()
  }
  ```
  or
  ```
  error Err1 {
    fun foo() = println("Err1")
  }
  error Err2 {
    fun foo() = println("Err2")
  }

  fun bar() {
    // ...
    // err: Err1 | Err2
    err.foo()
  }
  ```
  Thoughts:
    - Here is the moment where we need a disjoint union.
    - Easy(?) to implement in compiler. Options:
        - Rewrite a call into when + call.
        - Construct a map from name to function
    - A bit tricky for a typechecker as functions may have different signatures and return values.
    - Functions should be marked with annotation or keyword
        - To make increased call price explicit at least in the definition
        - To allow typechecker to do additional checks to prevent possible issues

- Preferably no extra hierarchy for errors
- No monad due to the cost of virtuality
    - There is actually not possible to devirtualize it as if we allow people to write generic functions for all monads, they will do it, and it will be impossible to optimize (jit?)

Thoughts:
- Unions for the errors are actually as hard for the compiler as generic unions
    - We may just consider them as common unions and implement it (maybe internally) as `Either<T, E1 | E2>`
    - To handle `E1 | E2` is not hard it could be done as in scala (soft and hard unions). They have to be hard for all error types and soft for common types.
    - Possibly we may annotate the generic parameter if we allow making an implicit union on this position and define `Either<T, @ImplcitUnion E>`

Moonshots:
- `value inline` classes for users to define
    - Not sure why we need it in the scope of this feature

To do:
- Read about scala's unions
  - In DOT encoding, everything is quite straightforward
- List monad operators to add and design and think is it enough
- Read about java's error return
    - Not sure about what to read as I did not find something like this

## June 6

To discuss:
- Concern: Introduction of union error types can introduce common unions internally (Is the compiler ready for it?)

To think:
- Will error types form a separate hierarchy?
- Disjoint union vs. union? Similar to Q: Error types without multiple inheritance?
    - Q: What is the reason of disjointness?
      - T: to avoid ambiguity in error handling
      - T: to avoid issues in type inference
    - T: to decrease chance of error in matching.
    - T: by design we can form error types into a forest (prohibit multiple inheritance)
    - T: allow everything but report warnings
- Can error types be generic? Will it cause any problem when they are instantiated with another error type?
    - T: Possibly it does not because they could be encoded with `Either<Either<Either<>>>`. 
      Thus, inference is fully the same as for common types.  
    - T: Actually not because in `Either<Either<Either<>>>` order matters.
    - T: Without generics they are easy for type system.
    - P: Introduction of union error types can introduce common unions internally
      ```
      interface A<out T>
      interface B<T> : A<T>
      interface C<T> : A<T>
      
      fun <T> foo(v: Int | A<T>) {}
      
      foo(v as Int | B<Int> | C<String>) // here T instantiates to 'Int | String'
      ```
      But obviously, it may be approximated with `Serializable & Comparable<*>`
- Do we want error types to be an unions?
  - T1: no, just form a list `T | List<Error>`
  - T2: yes, `T | Error1 | Error2` Pros: we can typecheck better
  - Q: Does it give us more than just allow one to omit default branch when pattern-matching by possible errors? 
    If so, what exactly?
- Subtyping on error types?
    - T: It could be the same as for common unions
- Is variance affected by unions?
    - T: read about scala
- Runtime semantics of union? Is it the same as in `Result<>`
- At the moment we would like to make a constraint from subtyping of union types
  ```
    interface B<T>
    interface C<T> : B<T>
    interface D<T> : B<T>
    interface E<T> : C<T>, D<T>

    E<T> <: C<Int> | D<String>
  ```
  By definition (like in Scala), the least upper bound of two types is exactly the union type.
  We would like to materialize it. `C<Int> | D<String>`.
  How? What about variance? What about substitution?
  Easy solutions:
    - Disjointness? does it solve it? How to design it with and without multiple inheritance
    - No multiple inheritance => no issue?
  
## June 10

Thoughts:
- In example: 
  ```kotlin
    interface B<T>
    interface C<T> : B<T>
    interface D<T> : B<T>
    interface E<T> : C<T>, D<T>

    E<T> <: C<Int> | D<String>
  ```
  We are not able to materialize the least upper bound of `C<Int>` and `D<String>`.
  Because the actual constraint that are introduced is `T = Int | T = String`.
  Such a constraint:
  * Could not be introduced by any materialization as in any materialization as constraint from any materialization could be expressed as a single constraint of type `T (=|<:|:>) ???`, while this one could not.
  * Could not be expressed in the type checker as it introduces an exponential blow
  
  So we should just prohibit such a case.
  This is why we would like to have only a disjoint unions.

  Scala in this case resolves `T` into `Int`.
  So this code fails to compile:
  ```scala 3
    trait A[T]
    trait B[T] extends A[T]
    trait C[T] extends A[T]
    trait D[T] extends B[T] with C[T]
    
    trait Inv[-T]
    
    def foo[T](v: Inv[D[T]], vv: T) = vv
    
    def bar(v: Inv[B[Int] | C[String]], vv: String) = foo(v, vv)
  ```
  but everything is ok if you explicitly annotate the generics:
  ```scala 3
    def bar(v: Inv[B[Int] | C[String]], vv: String) = foo[String](v, vv)
  ```
- The type `A[Int] | B[String]` is actually not disjoint 
  as we are not able to distinguish different cases and generic parameters for case `B`.
  So disjointness has to be by classifiers.

- Variance + unions in Scala.
  In scala, type parameter is always a pair of upper and lower bounds.
  Variance is expressed as `Any..T` or `T..Nothing`.
  And in case of union, bounds are combined straightforwardly.

  Actually, not sure why they may relate in any way.

- In example:
  ```kotlin
  interface A<T>
  interface B<T> : A<T>
  interface C<T> : A<T>

  fun <T> foo(v: Int | A<T>) {}
      
  foo(v as Int | B<Int> | C<String>)
  
  A<T> :> B<Int> | C<String> =>
  A<T> :> B<Int> & A<T> :> C<String>
  T = Int & T = String
  ```
  We are able to typecheck a call by instantiating T with `? extends Serializable & Comparable`.
  Not sure if typechecker ready for such inference.
  So this call may be illegal.
  To easily fix this call, we have to declare `foo` as follows:
  ```kotlin
  fun foo(v: Int | A<*>) {}
  ```
  
- From quip: "How to declare special error types? Can we declare these types implicitly / locally / in place? For example, can we have `val conn: Connection | Null@loadServiceDiscoveryRepo | Null@serverByType | ...`?"
  
  Not sure about idea.
  If the error were generated deeper than in the top function (`serverByType`), we will not match it on `Null@serverByType`.
  In theory it is possible to implement using runtime tags.
  But (depending on the semantics) we will have to re-pack errors (and add conditional branching to do it) on every call.
  I guess this is enough disadvantages to not do it.

So we would like to encode an Either monad into language.
Functionality:
- Bind.
  - Bind is quite straightforward.
    We may re-use `?.` for safe call.
    As the other case remains only null if the function signature does not changed.
- Return.
  - In monad we explicitly state whether we are returning left or right.
    In out case Left (common return type) is not annotated.
    - We may annotate right with smth like `return_error`.
    - We may infer that the error is returned using types.
      In this case we have to have an explicit `error` keyword in the class declaration and care about the inheritance for errors and allow only explicit cast from error to their non-error supertypes.

Semantics:
- Inference.
  - Do we want to infer errors that function may return?
    - We are not able to use `T?` syntax for this case as it will break the smart-casts using `v === null`.
      Other cases like `?:` or `?.` are not affected as they will consider any error as null.
      So should not we ignore this case and just publish a migration patch or smth?
      Otherwise we have to introduce a new syntax for this case (aka `T | ?` or `T??`).
- Runtime.
  - In case of boxing an error as for `Result<>`.
    - When to unbox?
      Sounds that when the type is not of kind `T | Err1 | Err2`, we have to unbox it.
      Everything is the same as for `Result<>`.
    - Is union of errors remains union if it does not have a left value type?
      Then we may split it into two KEEPs.
      Unions for errors anywhere and syntax sugar for Result<>.
    - Otherwise we may also do it, but for unions we have to introduce an annotation for the generic parameter if there allowed to have a union of errors.
    - In this example:
      ```kotlin
      v: Int | Err1 | Err2
      v ?: run { 
        // here v is "Err1 | Err2" or "Nothing | Err1 | Err2" or "Error" (aka supertype of Err1 and Err2)
      } 
      ```
      The same for case:
      ```kotlin
      when (v) {
        is Int -> TODO()
        else -> {
          // v here is what?
        }
      }
      ```
  - In case of no boxing.
    - We may have a common supertype for all errors and check for error case using `isinstance` with it.
      - Then we should prohibit error type as a left value type.
- Generic parameters.
  - They have to be able to be union error types.
    - It looks quite easy to implement, just an inference for error unions.
  - May they be just a union of errors (aka `Int | T`)
    - I guess so as we are looking for `Int | T` as `Result<Int, T>`.
    - Inference does not looks harder than for the first case.
    - But case with `Int | Err1 | T` is the other case

I think that we should start with the formalization of syntactic sugar for `Result<>` without considering any unions.
It allows us to answer any questions about interpreting `Int?` as `Result<Int, Null>` and list all possible use-cases where unions may encounter. 
What about `Int? | Exception` in this case?
Is it `Result<Result<>>` or it is just prohibited (I think so, unless we've introduced unions)?
What about `T | Exception` where T instantiates into `Int?` or `Int | OtherException`?
The case is that we will not be able in runtime to distinguish different left cases?
No we are able as (I guess) kotlin handles inline classes properly.
And it will just pack the left value into the inline class.
How to handle this case for unions?
Do we have to merge errors or pack result in the same way?

## June 11

- Ask Marat how do they implemented resolution of constraint (`A<> & B<> :> C<>`). (because why mot the same for intersections?)
- Ask marat what is the intersection in Java we did not found anything.
- Choose one option: Generics or multiple inheritance. (without generics it may in theory be not disjoint)
- How to separate `return_error`

## June 12

Today we are trying to make a first step in formalization: Formalize the T? as T | Null (or Either<T, Null>).

1. We are introducing a new type `Null`.
   - `Any :> Null :> Nothing`
   - `Null` is closed for inheritance. (surprise-surprise)
2. We are introducing a new type `T | V`, error union type.
   Where `T` and `V` are arbitrary types (disjoint).
   (Maybe limit for now `V` to `Null` or `Nothing`?)
   - `T = T | Nothing`
   - `T1 | V1 :> T2 | V2` if `T1 :> T2` and `V1 :> V2`
   - Consequently, `T | V :> T`
   - `null : Nothing | Null`
3. Let's review the smart-casts in a new setting.
   Not sure why it may change anything?
   - `v === null`. `v : typeof(v) & Nothing | Null = Nothing | Null`
   - `v == null`. same as above.
   - `v ?: run { }`.
     - In the `run` we have  `v : typeof(v) & Nothing | Any`
     - If run returns in the consecutive code `v : typeof(v) & Any | Nothing`
   - `when (v) { is T -> }`
     - Inside the block `v : typeof(v) & T | Nothing`
   - `v as T`. same as above.
   - `v as? T`.
4. We are introducing function `failure : T -> Nothing | T`
5. Formalize syntax sugar
   - Monad return
     - `T?` is `T | Null`
     - Otherwise, you have to explicitly annotate the error type. (so far)
     - For inference of a return type, everything is straightforward.
       - `return null` is automatically inferred to the correct type and does not have to be boxed
       - Other errors will be boxed in `failure` function.
   - Monad bind.
     - Actually runtime operations are the same as current?
       - Because it is actually the same as `Result<>`
     - How to get a value of error?
       - Function `ofFailure: Nothing | T -> T`
     - How to smart-cast it to `Nothing | T`?
       - For `is` check due to disjointness we are able to understand if it is checked for an error or value type.

## Meeting

- Inference, not `failure` or `return_error`
  - We need soft and hard unions
- research all possibilities about type leak
- `v : Err1 | Err2` is ok
- No inference of errors in return type
- No exponent from multiple inheritance
- It is ok to have a separate elvis operator
- `T?` -> `T | Null` is useless (useful, but totally not a priority)

## June 13

- We forgot to ask about runtime representation.
- New question: do we want to have type `Nothing | T`?

- Soft and hard union rules
  - We do not infer unions in a type checker
  - Hard union is when we either propagated the expected type, and it is union (whether annotated or inferred does not matter)
  - Hard union is when we received a value with union type and trying to apply any transformation to it.

- Type leak
  - The first leak is when we have error of unions as a value
    - Ignore it until we have use-cases
  - The second leak is when we got a unions as a result of system resolutions
    - Let's investigate it
    - No multiple inheritance.
    - Actually there is no leak in case `A<T> :> B<Int> | C<String>`. 
      It is just a constraints: `T :> Int & T :> String`
      As we did not have union on any side, we did not expect a result to be a union.
      If we have something like `T :> Int | Err1 & T :> String`.
      We will have `Serializable & Comparable | Err1`.
      Actually, we are not able to not solve any system due to this constraint.
      To fail it, we have to have upper bound constraint that will be `Int | String`, 
      which is impossible as we do not have such types in the system.
    - And the behaviour is ok as it is the same as now.
      And even changing it will break backward compatibility.

- We may allow generics only in final class and it works

TODO for a week:
- Unions in different languages
- Effects as error processing
  - And other error processing


## July 23

### Local errors

> One of the motivating examples given in the talk was about lastOrNull(predicate) on a list of String? and not being able to distinguish between a null value being found versus the list not containing a value that matches the predicate. While errors could disambiguate that scenario, note that the problem isn't eliminated as we have the same issue when processing a list of errors. In that case, we wouldn't be able to disambiguate between finding that error value in the list versus the function not finding an error value that matches the predicate.

There will be no problem in such case if the error case is boxed.
(Which I highly recommend)

But there may be a case when `T = V | NotFound`.
So we have to create a unique error for our `last` function.
Which is not allowed to leave a function.
Is it possible to track it?

If it leaves a function, we will encounter the same issue.
Would we like to create an error which is not equal to any other?
Do we want to fix this case at all?
As it sounds like a very rare case which is also hard to fix.

It is already looks like attributes of types, not error type.
In theory it is possible to implement it in unsafe way:

```kotlin
error object NotFound // new category of types

inline fun <T> Sequence<T>.last(predicate: (T) -> Boolean): T {
  error object NotFoundInternal
  var last: T | NotFoundInternal = NotFoundInternal
  for (element in this) {
    if (predicate(element)) {
      last = element
    }
  }
  if (last == NotFoundInternal) throw NoSuchElementException("Sequence contains no element matching the predicate.")
  return last // smart-cast to T
}
```

But in this approach we rely on a programmer to not leak `NotFoundInternal` outside the function.

Additionally, we encounter the same issue as was with lambdas on JVM.
If all functions will declare their own internal errors, we will have a lot of classes in the heap + a lot of class-loadings.

We may implement something like tagged errors.
It is not what was proposed in the quip, as it will not lead to re-packing of values on each call.
Tag will be assigned in the function of instantiation and will be of form "tag$functionName$packageName".

Should we prohibit to leak this kind of errors out of function? (But how? Prohibit upcasting to Any?)
Or should we leave it on a programmer?
With this it will look like this:

```kotlin
inline fun <T> Sequence<T>.last(predicate: (T) -> Boolean): T {
  var last: T | `NotFound = `NotFound
  for (element in this) {
    if (predicate(element)) {
      last = element
    }
  }
  if (last == `NotFound) throw NoSuchElementException("Sequence contains no element matching the predicate.")
  return last // smart-cast to T
}
```

Such tag does not allowed to have a value.
Under the hood it is a single for all of them class:

```kotlin
error class TagError(tag: String)
```

Another option: Allow function to prohibit some errors in generics (...)

Ideas:
- Local and non-local tag
  - Local have "$functionName$packageName" in the tag
  - For locals we have to add static analysis to check that it will not leave the scope
  - Non-local is just a name
  - They are compiled into `TagError` with a tag
  - How to match and distinguish
    - We may have only local
    - We may have only non-local (but it does not solve a problem)
    - We may have both and different syntaxes (\` and # or \` and \`...@this).
      In the last option we may have local tags for class/package, but we have to track that they do not leave a scope.
    - We may have to pack-unpack them depending on the context
- Aren't this enough?
  We may add `Any?` or `Array<Any?>` into TagError and declare an associated type somewhere.
  Java interop!

### Error type inference

They said that they do not want to infer error types.
What about lambdas and expression bodies?

```kotlin
fun foo(lst: List<Int>) {
    lst.map<_, Double | MyError> {
        if (it >= 0) {
            sqrt(it.toDouble())
        } else {
            MyError
        }
    }
}
```

Looks verbose.
Other approaches:

```kotlin
fun foo(lst: List<Int>) {
    lst.map {
        if (it >= 0) {
            sqrt(it.toDouble()) as Double | Nothing
        } else {
            MyError
        }
    }
}
```

```kotlin
fun foo(lst: List<Int>) {
    lst.map {
        if (it >= 0) {
            sqrt(it.toDouble())
        } else {
            MyError as Nothing | MyError
        }
    }
}
```

```kotlin
fun foo(lst: List<Int>) {
    lst.map {
        if (it >= 0) {
            sqrt(it.toDouble())
        } else {
            failure(MyError)
            // failure is not required in other cases
        }
    }
}
```

Maybe we need a syntax for it:

```kotlin
fun foo(lst: List<Int>) {
    lst.map {
        if (it >= 0) {
            sqrt(it.toDouble())
        } else {
            MyError!
        }
    }
}
```

### Generics vs multiple inheritance

There were no questions about multiple inheritance in the issue => it is unnecessary.
So let's prefer generics.

There was a point to investigate a possible union leaks.
But I guess it was resolved to there are no leaks without multiple inheritance.
So actually generics with single inheritance rule is ok.

### Unions in different languages

Nothing interesting...

### Other error processing

Nothing interesting except effects.
Not sure how to relate it.

### Open questions

- Runtime representation of errors pros and cons
- Inference of boxing moments
- How it is allowed to interoperate error and common hierarchies
- ...

So should we start writing a keep?

### KEEP structure

We should elaborate all choices and consequences.

* Motivation (questionable)
* Structure overview
* Choices
  * Generics vs multiple inheritance
  * Runtime representation
  * Error type inference
  * Hierarchy interoperation
  * ...
* Problems and options for `last` use-case

## July 23

To think:
- Two trailing lambdas for onSuccess and onFailure

### Boxing

Pros:
- Faster checks
- Allows to have some interop between errors and common types

Cons: 
- Boxing
- Inference of boxing moments

### Interoperation with common hierarchies

Requires boxing.
May be misunderstood in case `Serializable | MySerializableError`.
May be allowed with explicit cast only (`return MySerializableError as Serializable`).

Common class extending error is very strange use-case.

### Inference of boxing moments

failure or syntax sugar => boxing.
No interoperation or only explicit interoperation => obvious boxing in return statement.chmok

## July 24

### Pre-meeting

Agenda:
- Tagged errors as a solution for `last`
  - Problem and simple solution
  - Lots of classes problem
  - Solution with tagged errors
  - Non-local tags using `@this`
  - Not thread-safe `getInstance`, just const loading when no content
  - Is this enough?
- Error type inference
  - Ask again if we would like to infer error types?
  - new syntax or failure function
- Runtime representation
  - Do we want boxing, or it is just an option?
- Do we want MyError <: Any
- Do we want T -> MyError
- Default behaviour:
  - Errors accumulated on top?

### Meeting

- Marat will think if it is enough to have only tagged errors with typealiases as interfaces.
- Marat thinks that inference in lambdas is good.

## August 2 - 5

Firstly, let's state that disjointness is not good in our case.
This is because even in the case of `last` function:

```kotlin
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

`NotFound` is not disjoint with `T`.
So we have to allow some cases of non-disjointness.

So, let's start with the formalization.
No generics so far.

### Types

- New types: `Error` (supertype for all errors), `Value` (supertype for all common types)
- New type constructor: `A | B` (error union)

### Well-formattedness

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

### Operations

- `T|_v`
  - `T <: Value` => `T|_v = T`
  - `T <: Error` => `T|_v = Nothing`
  - `T = A | B` => `T|_v = A|_v`
- `T|_e`
  - `T <: Value` => `T|_e = Nothing`
  - `T <: Error` => `T|_e = T`
  - `T = A | B` => `T|_e = A|_e | B`
- For `T <: Error`: `[T]` -- interpret `T` as a set of classifiers

### Subtyping

New axioms:
- `Any :> Error`
- `Error :> MyError` for all `MyError`
- `MyError :> Nothing`
- `Any :> Value`
- `A :> B | C <= A :> B & A :> C`
- `A | B :> C <= A|_v :> C|_v & [A|_e | B] \subset [C|_e]`

### Inference

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

## August 6

Let's add generics.

### Types

Nothing new

### Well-formattedness

Nothing new

### Operations

- For `T <: Error`: `[T]` -- interpret `T` as a mapping from classifiers to type parameters

### Subtyping

Changed only the last one:
- `A | B :> C <= A|_v :> C|_v & [A|_e | B] \subsumes [C|_e]`

### Inference

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

## August 12

Addition of local errors (`NotFound@last`) to this system is quite simple and orthogonal.
Local errors just is not a subtype of `Any` and a supertype of `Nothing`.
Thus, they could not be added to any variable and have to be stated only as a constant.

Any approach to add a subtyping for them will lead to severe complications.
F.e. we could introduce something like `Errors@last` which will be a supertype of all errors and local errors for `last` function (or class).

## August 19

Meeting:
- Generics
- Local errors
- Declaration vs no-declaration of errors

```kotlin
T | NotFound

T | NotFound<String>

T {NotFound -> V}
V <: String

v : T | NotFound<String>

fun <T> foo (v: T) {
    val found: T | NotFound<S> = NotFound
    //           ^ Error: Generic parameters incompatible in union
}
```

```kotlin
fun last(...): T {
    // Error@last
    local error NotFound
    local error Blah
    val last: T | NotFound | Blah = NotFound
    last as T
    // expands into:
    when (val last = last) {
        is NotFound -> throw IllegalStateException()
        is Blah -> throw IllegalStateException()
        else -> last
    }
    ...
}

T | E1

Any
Any@last :> NotFound

private fun <T, E : Error@C> foo(v: T | E): T | E {
}
```

```kotlin
error NotFound

const val package$NotFound = ...

fun <T> last(...): T | NotFound {
    val last: T | NotFound = NotFound
    ...
}

// vs

fun <T> last(...): T | `NotFound {
    val last: T | `NotFound = `NotFound()
    ...
}
```

```kotlin
error MyError(val code: Int, val message: Err2)

fun foo (v: Int | MyError) {
    when (v) {
        is Int -> ...
        is MyError ->  {
            // v : MyError
            v.message.code
        }
    }
}
```
