# Pattern-matching

We believe that there are some strong connections between pattern-matching, compound expressions, and destructuring.
The main purpose of this document is to discover those possible connections deeper.
Please, note, even though the document presents some ideas about the possible design concrete syntax, it is not any kind of final proposal but just a way to demonstrate an alternative view or/and some point.

## Keystones

1. Use the same very syntax for all destructurings, including pattern-matching
2. Some other things to think about before making the final decision on destructurings and pattern-matching design
3. We do not suggest any "final design"; it should be considered more as an example to show certain ideas and concerns

$$
\begin{array}{lll}
\langle\text{when branch condition} \rangle  & \Coloneqq & (P\ |\ Cond)\ (\text{\color{red}if}\ \langle \text{guards} \rangle)? & \\
P &\Coloneqq &Is \\
Is &\Coloneqq & \text{{\color{red}is}}\ \langle\text{ClassName}\rangle\ Destr?\ |\ \text{{\color{red}is}}?\ Destr \\
Decl & = & \text{{\color{red}val}}\ |\ \text{{\color{red}var}} \\
Destr &\Coloneqq & (\ NamedDestr^{+}\ )\ |\ (\ PositionalDestr^{+}\ ) \\
NamedDestr &\Coloneqq & Decl?\ (\langle \text{Identifier}\rangle\ =\ )?\ \langle\text{FieldName}\rangle\ (Is\ |\ Cond)? \\
PositionalDestr &\Coloneqq & (Decl\ \langle\text{Identifier}\rangle)?\ (Cond\ |\ P)? \ |\ \_ \\
Cond &\Coloneqq & Cmp\ <Expression>\ |\ \text{\color{red}in}\ \dots \ |\ \dots \\
Cmp  &\Coloneqq & ==\ |\ !=\ |\ >=\ |\ \dots \\
\end{array}
$$

we can consider `is` to be a reversed assignment
```Kotlin
(val x, val y) = z  <===>  z is (val x, val y)
// syntax 1
fieldName is val newName@Class(val x, val y)
// syntax 2
val newName@Class(val x, val y) = fieldName
```

`dis` === `is` without *Cond* and *ClassNames*

```Kotlin
// name-based destructuring
// syntax 1
let <expr> is val k@(field1 is val x, 
                     field2 is val y@(filed1 is var m, var filed2)
// syntax 2
val k@(val x = field1, 
       val y@(var m = filed1, var filed2) = field2)
    = <expr>
```

```Kotlin
// current position-based destructuring
val (x,y,z) = <expr>
// syntax 1
let <expr> is val m@(val x, val y, var z@(val k, var p))
let <expr> is (val x, val y, var z@(val k, var p))
// syntax 2
val m@(val x, val y, var z@(val k, var p)) = <expr>
(val x, val y, var z@(val k, var p)) = <expr>
// maybe val as default:  --- then it coincides with the current syntax
val (x, y, var z@(k, var p)) = <expr>
```

## Questions

This section contains a list of questions we think important to think about before making the final decision on pattern-matching, compound expressions, and destructuring design.
We provide one of the possible answers for most of the questions;
please, do not consider those answers as final, we are more than happy to discuss them.

1. Do we need `nested` matching?
   + Marat&Misha: "Not sure"
   + Further, we *assume* that we *support nested matching*
     - Reason: it is quite common approach to support nested patterns
2. Patterns are just compact bindings or do we allow to match with specific values/conditions?
   + Further, we *assume, no*
     - I.e. patterns are patterns while specific values conditions should be placed in guards
     - Motivations: 
       * try to make "new" feature as simple as possible
       * an attempt disallow one more way to write semantically the same code
       * Actually, we think that the answer should be "yes", but do not want to "pollute" this document for now
3. Do we want to keep positional destructuring?
   + Further, we *assume, yes*
     - Motivation: we believe that name-based destructuring can be added as a new language feature keeping the existing code to compile with no changes at all; Also, it may nicely coincide with pattern-matching, so positional destructuring may be considered separately
4. Do we allow to mix positional and named-based destructurings on different levels?
   + Further, we *assume, yes*
     - Motivations: 
       * it seems to be that concrete type/kind of desctructuring is a property of the type, i.e. dataclass/class; thus, it would be strange that I can't destructure positionally a value of dataclass
5. Can we use name-based destructuring on ordinary classes?
   + Further, we *assume, yes*
6. In name-based destructuring if one want just to destruct field but not introducing the name into the context. How?
   + Approach 1: Always write both names if you want to add the name into the context
     - `name = name is D(...)` introduces *vs* `name is D(...)` do not
     - I do not like `name = name`
   + Approach 2: Use something like `_ = name is D(...)` to avoid of introduction `name` into the context, otherwise add the name
   + Approach 3: use explicit `val`/`var` declarations
     + example: C#
        ``` C#
        order switch
        {
           { Cost: var y, ... } => ...
        ```
   + Approach 3: Haskell-style and Scala-style `@`
   + Approach 5: Forbid binging and destructuring in one place
     - Further, we *assume this option*
     - I.e.
        ```Kotlin
        is A (b = a) // intros b
        is A (a) // intros a
        is A (a is B(...)) // doesn't intro a
        is A (b = a is B(...)) // Error
        is A (a, b, c is D(e, f)) // intros names a,b,e,f but not c
        is A (a, b, c) if c is D(e, f) // intros names a,b,c,e,f
        // or maybe
        is A (a, b, c) && c is D(e, f) // intros names a,b,c,e,f
        ```
     -  use guards instead:
      `... name is D(...)...` ---> `... name ... if name is D(...)`
     - `newName = name is D(...)` is confusing since right-hand side of `=` in other contexts may has boolean type
7. Do we want to have a special syntax for pattern-matching (name-based?) and destructuring to be exhaustive
   + Example: something like `(a,b)!` or `(a,b|)` or `(a,b,EOC)` meaning it is a dataclass with exactly two public fields
   + Alternative: `(a,b,..)` meaning dataclass has at least fields `a` and `b`
8. Do we want allow in name-based destructuring to match/add restrictions several times on some field?
   + example: C#  
      ```C#
      order switch
      {
        { Cost: var y, Cost: > 500.00m } => 0.05m,
        ...
      }
      ```
9. Do we want allow pattern-matching with concrete values and vars/vals using the same syntax?
   + I guess, yes
10. Conditions on fields inside patterns?
11. `and` and `or` in pattern-matchings?
12. Do we allow matter-matching on **several** values?
13. Do we want name-based and positional destructurings have the same/similar or different syntax?
    + C# uses different syntax
    + It is possible to have the same but sometimes it might be confusing since it is the property of the type

---
# Approach 1: explicit declarations

```Kotlin
class Address(val city: String, val street: String, val zip: String, ...) {...}
class User(val name: String, val age: Int, val address: Address) {...}
```
pattern-matching:
```Kotlin
val u = User (....)
when (u) { // name-based:
  // C#-style name-based
  { age: val age, age: > 18, 
    address: val address, address: { city: "Spb" } } -> ...  
  
  // no field repeats: 1
  is User(val age = age && age > 18, 
    val address = address dis ( city == "Spb" ) ) -> ...  
  
  // no field repeats: 2
  is User(val age = age, age > 18, 
    val address dis ( city == "Spb" ) ) -> ...  
  
  // no field repeats: 2
  is User(val age = age && age > 18, 
    val newAddress = address dis ( city == "Spb" ) ) -> ...  
  
  // same but no name introduction and conditions in one place
  dis ( val age = age ) if age > 18 -> ...  
  // same but Is
  is User( val age = age ) if age > 18 -> ...  

  // the same but omit `=` if names coincide
  ( val age ) if age > 18 -> ...  
}
```
destructuring:
```Kotlin
// in case of explicit declarations I insist on the following or similar destructuring design:
// here `let` is an indicator of declarations
// syntax: let <expression> <Dis_pattern>
let x dis (val age = age)
// alternatives: I do not like any of them
( val new_age = age ) = x
let ( val new_age = age ) = x
val ( val new_age = age ) = x
let (new_age) = x as (val new_age = age)
...
```
lambdas:
```Kotlin
{ ( val age ) -> ... }
{ ( val age , var newStreet = street ) -> ... }
```

<!-- ```Kotlin
class User(val name: String, val age: Int) { ... }
val u = User (....)
when (u) { // name-based:
  is User{ age: val age, age: > 18 } -> ...  // C#-style name-based
  is User( val age = age, age > 18 ) -> ...  // ?
  is User( val age = age ) if age > 18 -> ...  // no name introduction and conditions in one place
  is User( val age ) if age > 18 -> ...  // the same but omit `=` if names coinside
}
``` -->

---
# Approach 2: implicit declarations


-----
-----
-----
-----
-----
-----
-----
-----
-----
-----
## Grammar

Here we present the grammar for patterns in when-branch conditions.
Each pattern $P$ is either
* **fallible** --- $Is$ pattern, i.e. pattern matching which may fail, or 
* **infallible** --- $Dis$ pattern, i.e. pattern matching which always succeeds.

Those patterns are very similar but can be distinguished by the keyword being used: `is` for fallible $Is$ patterns and `dis` for infallible $Dis$ patterns.
We believe that infallible $Dis$ patterns can be successfully used for name-based destructuring in the very the same syntax whenever the destructuring happens, while fallible $Is$ patterns can be used in when-branch conditions (discussed later: or maybe in any other condition).
Since it is expected that pattern-matching on some specific pattern may fail we allow to mix fallible $Is$ patterns and `dis` for infallible $Dis$ patterns on different levels of patterns inside one when-branch condition.

$$
\begin{array}{lll}
\langle\text{when branch condition} \rangle  & \Coloneqq & (P\ |\ \text{\color{red}in}\ \dots \ |\ \dots)\ (\text{\color{red}if}\ \langle \text{guards} \rangle)? & \\
P &\Coloneqq &Is\ |\ Dis \\
Dis &\Coloneqq & Destr \\
Is &\Coloneqq & \text{{\color{red}is}}\ \langle\text{ClassName}\rangle\ Destr? \\
Destr &\Coloneqq & (\ NamedDestr^{+}\ )\ |\ (\ PositionalDestr^{+}\ ) \\
NamedDestr &\Coloneqq & (\langle\text{Identifier}\rangle\ =\ )?\ \langle\text{FieldName}\rangle \ |\ \langle \text{FieldName} \rangle\ (Is\ |\ \text{{\color{red}dis}}\ Dis) \\
PositionalDestr &\Coloneqq & \langle\text{Identifier}\rangle \ |\ \_\ |\ P\ |\ Cond\\
Cond &\Coloneqq & Cmp\ <Expression>\ |\ ... \\
Cmp  &\Coloneqq & ==\ |\ !=\ |\ >=\ |\ ... \\
\end{array}
$$

NB: 
* $Dis$ itself **always succeed** while $Is$ is a usual is-check and **may fail**
* Either new name or next level destructuring
  ```Java
  is A(newName = name is D(...))            // Error
  is A(newName = name) if newName is D(...) // Ok
  is A(newName = name) if name is D(...)    // Error
  is A(name) if name is D(...)              // Ok
  is A(name is D(...)) // Ok, but doesn't add `name` to scope
  ```
* About $\text{\color{red}dis}$ keyword:
  + We use new keyword $\text{\color{red}dis}$ to avoid situations like `name (a, b)` (meaning `name dis (a, b)`)
  + Another approach is to use `(a, b) = name` instead
    - Disadvantage: mixing left-to-right and right-to-left

## Connection to destructuring

We consider **destructuring** to be a special case of pattern-matching which has exactly one branch and always succeed.
I.e. Pattern is always infallible $Dis$ pattern, can be nested, but even nested patterns are infallible $Dis$ patterns.
Note, since those infallible $Dis$ patterns are nested, each level can be both positional or named-based.
NB: this approach doesn't disallow old-fashioned positional destructuring.

Also, in our approach ***all destructurings*** (with `val`, or in *lambdas*, or even in function arguments if one want to allow them) are in ***the exactly the same syntax***.

### Concrete syntax

We may consider $Dis$ to return a list of names to be added to the context. I.e.
* for named-based:
  + ```val a, b, c = v dis (a, b = x, c)```
  + We assume that **names on left-hand side are collected by IDE automatically**
  + Note the order doesn't matter on both sides:</br>
    the same as ```val c, b, a = v dis (a, b = x, c)```</br>
    the same as ```val c, b, a = v dis (b = x, c, a)```
  + Marat prefers: ```val (a, b=x, c) = v```
    - Pros: 
      * looks familiar
      * doesn't repeat names
    - Cons: 
      * mixing right-to-left and left-to-right 
      * hard to write since fields are not in the scope when writing left-hand side
      * if nested destructuring is allowed, the whole "structure" appears on the left making it hard to find the place variable is actually defined by "naked eye"
    - Danya prefers ```let v dis (a = x, var b, c)``` if one do not want to repeat names
* for positional: </br>
  ```val a, c, b = v dis (a, b, c)``` or in let-syntax ```let v dis (a, b, c)```
  - Cons: 
      * hard to distinguish (for "code reader") positional destructuring from named-based in the example above; Possible solution is to used another keyword like `pdis` but it is ugly
  - Note, this doesn't forbids old-fashioned way `val (a,b,c) = v`
    * Can be used for a long "transaction period" if one decides one day to deprecate old-fashioned style

Also note, that, for example, in lambdas 
``` Kotlin
{ (a, newName = name, b) -> ... } // Ok, name-based
{ (a, b, c) -> ... } // Ok, whether name-based or positional is used depend on particular type of the value being destructured
{ (a, newName = name, field_name dis (b, _, c)) -> ... } // Ok, top-level name-based destructuring and positional destructuring for `field_name`
{ (a, newName = name, field_name ) -> 
  let field_name dis (b, _, c) 
  ...
} // Ok, use if you want `field_name` in context
```

## Allowing vals and vars in patterns

We suggest to consider all bindings as `val`s by default unless some special binding is not marked as `var`.
For example, `v dis (b = x, var c, a)` and `{ (b = x, var c, a) -> ... }`.

But with simple destructuring statement it might be confusing:
</br>
```val a, var c, b = v dis (a, var b, c)```
</br>
it might be good to use something like `let` instead
</br>
```let a, var c, b = v dis (a, var b, c)```
</br>
or
</br>
```let a, c, b = v dis (a, var b, c)``` // Crap

And once again: Danya prefers ```let v dis (a = x, var b, c)``` if one do not want to repeat names.

Syntax `(val x, var y) = Point(x,y)` we find violating usual way declarations are done in Kotlin and has a very limited use-case. 

## Tuples

1. RETURN TUPLES BACK!!!!
2. In far future positional destructuring only for tuples, and collections, and maybe some other very specific cases but *not for dataclasses*

## Connection to is in expressions (and Java)

By default, ordinary `is`'s remain usual Kotlin's `is`-es.

I.e. Java-style `if (x is D(y,z)) { ... }` is not allowed.
Currently and with our syntax it is a sugar over
```
when {
  x is D(x,y) -> ...
}
```
which, by-the-way, is forbidden by the presented grammar.

Maybe discussed later after making decision on final design, if any, of compound expressions.

## Other

* `is D(x == true)` --go_to_hell--> `is D(x) if x == true`
* No multiple guards?
  ```
  P | cond1 -> ...
    | cond2 -> ...
  ```
  --go_to_hell-->
  ```
  P -> when {
         cond1 -> ...
         cond2 -> ...
       }
  ```

---
---
---
---

## Connection to (compound-) expressions (!!TODO!! Unfinished)

There is a [KEEP for compounds expressions for Kotlin](TODO) where we describe some common approaches for compound expressions in different programming languages and propose to add C-style compound expressions in if-conditions and when-expression-scrutinee.
E.g.
```Java
if (val x = 5, // intro x, 
               // x can be used further but not in else-branch
    val y = f(x,...); // intro y, use x
    cond(x,y)) {      // condition
  ... y ... 
} else { ... } // no x nor y here
```

Another approach to add compound-expressions is to allow writing usual `val`/`var` declarations inside an expression.
E.g. `cond1 && val x = ... && cond2(x)`,
meaning that `x` will be only initialised iff `cond1 == true` and is added in scope ones control-flow reached the declaration.

---
### Cons #1
This introduces some ambiguity with current ad-hoc design of `when`-expressions:
```
when (val x = ...) { ... }
```
Here `val x = ...` "returns" the value of `x`,
 while in case of 
```
cond1 && val x = ... && cond2(x)
```
`val x = ...` either always "returns" `true` or maybe in some particular cases (e.g. `null`) `false`.
In other words, in different contexts the same declaration may have different semantics.

### Cons #2

This feature may easily make code "unreadable". E.g.
```
if (x == 5 && 
  val y = someVeryLongFunctionNameWithDozenVeryLongArguments(
    veryLongLongLongLongLongLongArgument#1,
    veryLongLongLongLongLongLongArgument#2,
    someFunctionCallWithSeveralArguments,
    ....
  ) &&
  someConditionOnXAndY(x,y)) { ... }
```
And we do not see any good way of disallowing thing like that.
In a lot of cases it looks like a use-case for scopes (like `.run` blocks) rather than compounds.

### Cons #3

It is essentially the same as **Cons #2** but we would like to explicitly state it:
if one allow to write an arbitrary declaration inside an expression and/or condition it is too easy to abuse the feature.

For example, having a condition with a declaration inside one may use `if` expression in the declaration and another declaration inside and so on.



---

**QUESTION**: Do we want to allow declarations inside expressions?
* We do not have any strong opinion here, even though it easy to imagine that it would be sometimes nice to write some code like that;
But unfortunately "nice" or "Danya would like it" doesn't mean "there is some really strong use-cases for it".
* Anyway, hereafter let's *assume, yes*, we want to allow them or something similar



Nevertheless, it contains nothing about the connection with destructuring and patterns.
Further in this text, we focus on these connections.

As it was stated earlier, we may consider infallible $Dis$ patterns to return a list of names to be added to the context.
This is exactly one case of compound expressions.
E.g.

!!TODO!!

Moreover, we may consider fallible $Is$ patterns to return a list of names to be added to the context also.

!!TODO!!


<!-- ```
.run {
  let p dis (x,y)
  when (p) {
    (x,y) if x == 1 && y > 2 -> ...
    (x,y) if x == 2 && y > 2 -> ...
    ...
  }
}

{ (x, newName = name dis (a, b), z) -> ...  } // go to hell
{ (x, newName = name , z) -> 
  let newName dis (a, b)
  ...  
} // OK
``` -->

---

## Other languages

### Scala

* links
  * [1](https://stackoverflow.com/questions/18841031/lots-of-nested-match-case-in-pattern-matching)
  * [2](https://blog.rockthejvm.com/8-pm-tricks/)
  * [3](https://docs.scala-lang.org/tour/pattern-matching.html)
  * [4](https://www.baeldung.com/scala/pattern-matching)
* allows @
* conditions
* guards
* named-destr
* nesting
* literals
  
### C# allows everything =)

* [pattern-matching](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/pattern-matching)
* [destruct](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/deconstruct)

```C#
//	public class Order(int Items, decimal Cost){
//		public int Items;
//		public decimal Cost;
//	}; // also OK except positional (  > 10,  > 1000.00m) => 0.10m,
	public record Order(int Items, decimal Cost);

	public static decimal CalculateDiscount(Order order) =>
    order switch
    {
        (  > 10,  > 1000.00m) => 0.10m,
        { Items: > 10, Cost: > 1000.00m } => 0.10m,
        { Items: > 10, Cost: > 1000.00m } or {Items: > 1000 } => 0.15m,
        { Items: > 5, Cost: > 500.00m } and { Items: > 5, Cost: var y, Cost: > 500.00m } => 0.05m,
        { Cost: > 250.00m, Cost: var cost, Cost: var y, Items: > 1, Cost: < 300.00m } => cost+y,
        null => throw new ArgumentNullException(nameof(order), "Can't calculate discount on null order"),
        var someObject => 0m,
    };
```