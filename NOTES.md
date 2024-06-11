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
- Concern: Introduction of union error types can introduce common unions internally (Is the compiler read for it?)

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
- At the moment we would like to make a constraint from subtypings of union types
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
  ```
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

