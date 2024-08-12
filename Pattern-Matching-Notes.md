# Pattern-matching

## Keystones

1. Use the same very syntax for all destructurings, including pattern-matching
2. Some other things to think about before making the final decision on destructurings and pattern-matching design


## Questions

1. Do we need `nested` matching?
   + Marat&Misha: "Not sure"
   + Further, we *assume* that we *support nested matching*
2. Patterns are just compact bindings or do we allow to match with specific values/conditions?
   + Further, we *assume, no*
3. Do we want to keep positional destructuring?
   + Further, we *assume, yes*
4. Do we allow to mix positional and named-based destructurings on different levels?
   + Me: yes, because it seems to be that this will be a property of the type, i.e. dataclass/class
   + Further, we *assume, yes*
5. Can we use name-based destructuring on ordinary classes?
6. In name-based destructuring if one want just to destruct field but not introducing the name into the context. How?
   + Always write both names if you want to add the name into the context
     - `name = name is D(...)` introduces *vs* `name is D(...)` do not
     - I do not like `name = name`
   + Use something like `_ = name is D(...)` to avoid of introduction `name` into the context
   + Forbid binging and destructuring in one place
     - Further, we *assume this option*
     - I.e. use guards instead:
      `... name is D(...)...` ---> `... name ... if name is D(...)`
5. Do we want to have a special syntax for pattern-matching (name-based?) and destructuring to be exhaustive
   + Example: something like `(a,b)!` or `(a,b|)` or `(a,b,EOC)` meaning it is a dataclass with exactly two public fields
   + Alternative: `(a,b,..)` meaning dataclass has at least fields `a` and `b`

## Grammar

$$
\begin{array}{lll}
\langle\text{when branch condition} \rangle  & \Coloneqq & (P\ |\ \text{\color{red}in}\ \dots \ |\ \dots)\ (\text{\color{red}if}\ \langle \text{guards} \rangle)? & \\
P &\Coloneqq &Is\ |\ Dis \\
Dis &\Coloneqq & Destr \\
Is &\Coloneqq & \text{{\color{red}is}}\ \langle\text{ClassName}\rangle\ Destr? \\
Destr &\Coloneqq & (\ NamedDestr^{+}\ )\ |\ (\ PositionalDestr^{+}\ ) \\
NamedDestr &\Coloneqq & (\langle\text{Identifier}\rangle\ =\ )?\ \langle\text{FieldName}\rangle \ |\ \langle \text{FieldName} \rangle\ (Is\ |\ \text{{\color{red}dis}}\ Dis) \\
PositionalDestr &\Coloneqq & \langle\text{Identifier}\rangle \ |\ P \\
\end{array}
$$

NB: 
* $Dis$ **always succeed** while $Is$ is a usual is-check and **may fail**
* Either new name or next level destructuring
* About $\text{\color{red}dis}$ keyword:
  + We use new keyword $\text{\color{red}dis}$ to avoid situations like `name (a, b)` (meaning `name dis (a, b)`)
  + Another approach is to use `(a, b) = name` instead
    - Disadvantage: mixing left-to-right and right-to-left

## Connection to destructuring

We consider named-based destructuring to be a special case of pattern-matching which has exactly one branch and always succeed.
I.e. Pattern is always $Dis$, can be nested, but even nested patterns are $Dis$'s.

Also, in our approach ***all destructurings*** (with `val`, or in *lambdas*, or even in function arguments if one want to allow them) are in ***the exactly the same syntax***.

### Concrete syntax

We may consider $Dis$ to return a list of names to be added to the context. I.e.
* for named-based:
  + ```val a, b, c = v dis (a, b = x, c)```
  + We assume that names on left-hand side are collected by IDE automatically
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
      * if nested destructuring is allowed, the whole "structure" appears on the left
    - Danya prefers ```let v dis (a = x, var b, c)``` if one do not want to repeat names
* for positional: </br>
  ```val a, c, b = v dis (a, b, c)```
  - Cons: 
      * hard to distinguish (for "code reader") positional destructuring from named-based in the example above; Possible solution is to used another keyword like `pdis` but it is ugly

## Allowing vals and vars in patterns

We suggest to consider all bindings as `val`s by default unless some special binding is not marked as `var`.
For example, `v dis (b = x, var c, a)` or `{ (b = x, var c, a) -> ... }`.

But since simple destructuring statement it might be confusing:
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

Ordinary `is`'s remain usual Kotlin's `is`-es.

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
  P -> if (cond1) {...}
       else if (cond2) {...}
  ```

## Connection to (compound-) expressions

As stated earlier, we may consider $Dis$ to return a list of names to be added to the context.
This is exactly one case of compound expressions.

See another KEEP (not ready yet).
