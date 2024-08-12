# Union Types for Errors

## References

- [Marat's quip (April 11)](https://jetbrains.quip.com/fOg9A3IXwD4b/Restricted-union-types)
- [Youtrack issue](https://youtrack.jetbrains.com/issue/KT-68296/Union-Types-for-Errors)

## Problem statement

> Copy from the issue

### Issue categories

1. Internal errors.
   It is a case where we would like to have a special error type that is supposed not to be able to be accessible outside of the function.
   Not only as a type (aka local classes), but also as a value (aka we do not expect to have this value in the input of the function or anywhere else). 
   This case may be viewed another special state  
   Firstly, it is demonstrated by the `last` function. 

## Approach

We had a preliminary discussion about a design where errors is a separate hierarch with the same features as an existing classes.
This approach is directed by a goal "Let's add as many features as possible to remain errors inferrable".
But it leads to several complex cases and does not have a clear use-cases in a real-world code.

Additionally, it is hard to cover even the simplest use-case with `last` function as in any such system we will encounter issue that the same error could be passed back to this function (upcasted to the errors supertype).
So this class of use-cases have to be solved using some ad-hoc approach.

The other problem of this full-flavored approach is that it will lead to a huge number of small local classes, that will be declared in every function.
If they are mapped to a separate JVM classes, it will lead to the same issue encountered in the JVM at the moment of lambda's introduction.

Thus, we moved into approach where we try to make as few features as possible that will cover all known use-cases.

## Proposal

