# Pattern matching and friends

In these notes, we would like to express our vision on pattern matching in Kotlin ans related features like compound expressions, guards, etc. 

## Related reading

- https://youtrack.jetbrains.com/issue/KT-186/Support-pattern-matching-with-complex-patterns
- https://youtrack.jetbrains.com/issue/OSIP-398/Non-stable-release-for-Object-name-based-destructuring
- https://youtrack.jetbrains.com/issue/KT-43871/Collection-literals
- https://youtrack.jetbrains.com/issue/KT-13626/Guard-conditions-in-when-with-subject
- https://github.com/nekitivlev/CompoundExpressions/blob/main/CompoundExpressionsProposalFull.md
- https://github.com/BlaBlaHuman/kotlin-comprehensions/blob/main/list-comprehension.md

## 31 July

- No chances for structural equality in OO languages. 
- Features:
  - let/where syntax
  - lazy syntax and behaviour
    - Bahaviour with smart-casts (where is it typed?)
    - Main use-case:
      ```kotlin
      lazy val condition1 = ...
      lazy val condition2 = ...
      lazy val condition3 = ...
      lazy val condition4 = ...
      if (condition1 && condition2 && (condition3 || condition4)) {
        ...
      }
      ```
    - Why not `inline val`: lazy val may be typed in the context of the first usage 
  - Guards were already introduced, so out of scope
  - Pattern matching semantics and predictable & readable syntax
    - `is Pair(val a, Pair(listOf(1,2), 2))` expands into:
      ```kotlin
      scr is Pair &&
      (scr as Pair).component2().equals(Pair(listOf(1,2), 2))
      ```
    - `is Pair(val a, is Pair(listOf(1,2), 2))` expands into:
      ```kotlin
      scr is Pair &&
      (scr as Pair).component2() is Pair &&
      ((scr as Pair).component2() as Pair).component1().equals(listOf(1,2)) &&
      ((scr as Pair).component2() as Pair).component2().equals(2)
      ```

## 1 August

Notes on sources from romanv:

- https://youtrack.jetbrains.com/issue/KT-186/Support-pattern-matching-with-complex-patterns
  - Use-less holywar about name-based vs positional-based destructuring
    - We should cover both
  - We have to have a concise syntax if the name of variable matches the name of the field
  - Nice idea for name based:
    ```kotlin
    when (scr) {
      is Clazz(val name = .fieldName is ...) -> ...
    }
    ```
    I like the idea of "." before field name because it makes it obvious which of names is field.
    In this example, it is useless, but if we do not have a declaration, it is better:
    ```kotlin
    when (scr) {
      is Clazz(.fieldName is ...) -> ...
    }
    ```
    Additionally, it helps for IDE to initialize the completion.
    But it is not Kotliny.
  - JEP: https://openjdk.org/jeps/405
    - Nothing interesting. Strange design for smart-casts.
  - C#: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns
    - "and", "or", "not" for patterns?
    - Not allowed to mix binding and constrained pattern
      - In java too
      - In scala it is done using "@" 
- https://youtrack.jetbrains.com/issue/OSIP-398/Non-stable-release-for-Object-name-based-destructuring
  - In progress, but no KEEP insofar.
  - Lots of issues with syntax
    - Backward compatibility
    - Kotliny style
- https://youtrack.jetbrains.com/issue/KT-43871/Collection-literals
  - Nothing interesting. We just have to keep in mind syntax conflicts. 
- https://youtrack.jetbrains.com/issue/KT-13626/Guard-conditions-in-when-with-subject
  - Simple `if` with the following expression.
  - They state that they covers most of use-cases of pattern matching...
- https://github.com/nekitivlev/CompoundExpressions/blob/main/CompoundExpressionsProposalFull.md
  - No OO languages have something like let-in syntax.
  - Idea of `scope {}` syntax may be wrong as it does not significantly differ from `run {}` syntax.
    And it eats two lines...
- https://github.com/BlaBlaHuman/kotlin-comprehensions/blob/main/list-comprehension.md
  - Not useful for pattern-matching. 

### Patters syntax:

```
`pat = is ClassName `destructing?
`destructing = \(`namedDestructing | `positionalDestructing\)
`namedDestructing = `binding? .fieldName `condition
`positionalDestructing = `binding? `condition
`binding = (val | var) name =
`condition = `pat | `inCondition | == `equalityCondition | ...
```

- We have to use either "==" before `equalityCondition` or "." before `fieldName` to distinguish them. otherwise it will be ambiguity.
- Positional destructing with bindings looks weird...
- How to align with named destructing?

### Meeting notes:

- We have to either prohibit to simultaneously match and declare variables or introduce a new syntax for it.
  Because we would like to not introduce a new variable in the scope.
  - `is Clazz(fieldname is ...)` is an introduction of a new name same as fieldname and pattern on it
  - `is Clazz(.fieldname is ...)` is just a pattern on a field 
  - or `is Clazz(_=fieldname is ...)`

## 7 August

Summarizing on the pattern-matching:
- For data-classes.
  There are two modes seme as for destructuring: Positional and Named.
  - Positional.
      
    Choices:
    - (var|val) for the binding or val by default.
    - Possibility of nested patterns.
    - Possibility of binding with nested destruction.
        
    Thoughts: 
    - (var|val) is not required as writing a name is already an introduction.
      Not requiring them will fully align the syntax with the destructuring.
    - Nested patterns are looks weird but not much and extremely useful.
    - Is it ok to have a binding with nested destruction?
      `is Clazz(fieldName is Clazz2(fieldName2))`
      The issue is that it potentially conflicts with the nested destruction, 
      where the nested elements should be on the left (while there on the right).
      
    So actually the current idea is `bindingName? condition? | _`
  - Named.
      
    Choices:
    - (var|val) for the binding or val by default.
    - Possibility of nested patterns.
    - Differentiation between binding + pattern and just pattern.
        
    Thoughts:
    - (var|val) is not required as writing a name is already an introduction.
      Not requiring them will fully align the syntax with the destructuring.
    - For named matching we have to differentiate between binding + pattern and pattern.
      The idea is to use `.` or `_=` before the field name.
    - In theory we may use `.` before name of the field to use an arbitrary val in the pattern.
      It allows for pattern matching of the arbitrary class.
- For common classes.
  It may be allowed to just use any val using `.` before the field name.
  Logically, it is just a simplified syntax for short conditions

## 8 August

What do we want from this document:
