---
layout: post
title:  Playing with naturals in TypeScript
description: "How Template Literal Types help manipulate number literal types"
categories: typescript
---
Introduction of Template Literal Types in TypeScript 4.1 makes the type system even more interesting. They are like template literals, but on the type level (a template literal type with a placeholder), so we can write code like this:
```typescript
type SizeUnit = 'px' | 'pt' | 'em' | 'rem'

type Size = `${number}${SizeUnit}`;
const width: Size = '10px';
```
More on that [here][template-literal-types-pr] and [here][template-literal-types-doc].

We can use this, along with other features (including recursive conditional types), to perform mathematical operations on literals representing natural numbers. I'll assume that the numbers are represented as string literal types. We can always convert from number like this: 
```typescript
type NumToStr<N extends number> = `${N}`;
```

## Manipulating digits

Digits are defined as a simple union. The most basic operations will be increasing or decreasing a digit by one.
```typescript
type Digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";

interface Incr extends Record<Digit, Digit> {
    "0": "1";
    "1": "2";
    "2": "3";
    "3": "4";
    "4": "5";
    "5": "6";
    "6": "7";
    "7": "8";
    "8": "9";
};

interface Decr extends Record<Digit, Digit> {
    "1": "0";
    "2": "1";
    "3": "2";
    "4": "3";
    "5": "4";
    "6": "5";
    "7": "6";
    "8": "7";
    "9": "8";
};

type Two = Incr["1"];
```
Carry will be represented as `"0"` or `"1"` string and result of digit addition will be represented as a pair:
```typescript
type Carry = "0" | "1";

type AddDigitsRes<C extends Carry, D extends Digit> = [C, D];
```
Along with the `AddDigits` operator, we'll define auxiliary `AddDigitsC` and `AddDigitsC2` operators which will help us handle the carried numbers.
```typescript
type AddDigits<D1 extends Digit, D2 extends Digit> = AddDigitsC2<D1, D2, "0", "0">;

type AddDigitsC<D1 extends Digit, D2 extends Digit, C extends Carry> = AddDigitsC2<D1, D2, C, "0">;

type AddDigitsC2<D1 extends Digit, D2 extends Digit, CPrev extends Carry, CNxt extends Carry> =
    D1 extends "0"
    ? (CPrev extends "0" ? AddDigitsRes<CNxt, D2> : AddDigitsC2<CPrev, D2, "0", CNxt>)
    : D2 extends "9" ? AddDigitsC2<Decr[D1], CPrev, "0", "1">
        : AddDigitsC2<Decr[D1], Incr[D2], CPrev, CNxt>;
```
`CPrev` is the carry from previous computation (say, addition of digits from previous column). I added `CNxt` for convenience to represent the carry of the current operation. The idea is pretty simple:
* as long as the first digit is not 0, decrease it and increase the second digit; save the carry to `CNxt` if needed
* if the previous carry is 1, add it
* return the result, which is the current carry `CNxt` and final value of the second digit `D2`

```typescript
type AddDigitsCTest = AddDigitsC<"4", "5", "1"> // = ["1", "0"]
```

## Chopping off digits

The general idea is to extract the last digits, add them, save the result and repeat the process on the prefixes (with last digits chopped off). Alternatively we could convert each number to a list of digits first, then perform addition on such a representation.

To do this, we need to define three auxiliary operators first:
- `Prefix` - returns prefix of a number (without last digit)
- `Last` - to extract last digit of a number
- `Empty` - to check if a number is an empty string

```typescript
type Prefix<N extends string> = N extends `${infer P}${Digit}` ? P : never;

type Ten = Prefix<"102">;
```
We simply ask TypeScript to infer the prefix by stating in the condition that it's followed by any digit. With `Last` we do this again, then use the inferred prefix to infer the actual digit:
```typescript
type Last<N extends string> = N extends `${infer P}${Digit}`
    ? N extends `${P}${infer D}`
        ? D extends Digit ? D : never
        : never
    : never;

type Six = Last<"256">;
```
The condition `D extends Digit ? D : never` might seem redundant, but this way we enforce that the operator actually returns a `Digit` (and not simply a string), which will be needed later. Typescript doesn't remember that the inferred `D` is the same `Digit` from the condition above.

The definition for `Empty` is straightforward:
```typescript
type Empty<S extends string> = S extends '' ? true : false;
```

## Basic operations

The logic behind addition:
* If both numbers are empty, return the accumulated result (add the carry to the front if it's positive)
* If one of the numbers is empty, add the carry to the remaining number
* Otherwise sum the current digits, accumulate the result, repeat on the prefixes

```typescript
type AddAux<N1 extends string, N2 extends string, C extends Carry, R extends string> =
    Empty<N1> extends true
    ? Empty<N2> extends true 
        ? (C extends "0" ? R : `${C}${R}`) // both numbers empty
        : AddAux<C, N2, "0", R> // first number empty
    : Empty<N2> extends true
        ? AddAux<N1, C, "0", R> // second number empty
        // add last digits and move on
        : AddDigitsC<Last<N1>, Last<N2>, C> extends AddDigitsRes<infer C1, infer DSum>
            ? AddAux<Prefix<N1>, Prefix<N2>, C1, `${DSum}${R}`>
            : never;
```
Note how we use conditional types to obtain the result:
```typescript
AddDigitsC<Last<N1>, Last<N2>, C> extends AddDigitsRes<infer C1, infer DSum>
```
It's like saying "add the digits and assign the result to new variables `C1` and `DSum`".

For subtraction we don't allow the minuend to be smaller than the subtrahend, so the logic is:
* If the minuend is empty: return the accumulated result only if there's no carry and the subtrahend is empty - otherwise return never (forbidden case)
* If the subtrahend is empty: for positive carry, subtract it from the remaining minuend - otherwise concat the remaining minuend with the accumulated result
* Otherwise subtract the digits, remember the result and continue subtraction on the prefixes

```typescript
type SubAux<N1 extends string, N2 extends string, C extends Carry, R extends string> =
    Empty<N1> extends true
    ? C extends "1"
        ? never
        : Empty<N2> extends true
            ? R
            : never
    : Empty<N2> extends true
        ? C extends "1"
            ? SubAux<N1, C, "0", R>
            : `${N1}${R}`
        : SubDigitsC<Last<N1>, Last<N2>, C> extends SubDigitsRes<infer C1, infer DDiff>
            ? SubAux<Prefix<N1>, Prefix<N2>, C1, `${DDiff}${R}`>
            : never
```
I omit implementation of the auxiliary operators on digits because it's very similar to addition. Now we want to wrap the auxiliary addition and subtraction operators in more user-friendly operators:
```typescript
type Add<N1 extends string, N2 extends string> = AddAux<N1, N2, "0", "">;
type Sub<N1 extends string, N2 extends string> =
    SubAux<N1, N2, "0", ""> extends infer U ? U extends string ? RemoveZeros<U> : never : never;
type RemoveZeros<N extends string> =
    N extends "0" ? "0" : N extends `0${infer S}` ? RemoveZeros<S> : N;
```
It turns out that there might be leading zeros after subtraction, so we remove them.

And here we go - we can add or subtract quite long numbers and the type deduction is very fast.
```typescript
type AddTest = Add<"12345678546657", "1234567890768769876764">; // 1234567903114448423421
```

## Other operations

What about other operators? Multiplication is very easy to implement using already defined operators, so I leave it as an exercise. Mod is a little bit more interesting:
```typescript
type Mod<N1 extends string, N2 extends string> = Sub<N1, N2> extends infer Diff
    ? IsNever<Diff> extends true
        ? N1
        : Diff extends "0"
            ? "0"
            : Diff extends string
                ? Mod<Diff, N2>
                : never
    : never;
```
How do we check if a given type is `never`? It turns out, we need to do it like this:
```typescript
type IsNever<T> = [T] extends [never] ? true : false
```
The question is, why can't we just say `T extends never ?`? If you have trouble figuring this out, you'll find the answer in the links below.

## Caveats

There are a few things that need to be mentioned at the end. As I noted at the beginning, we can always wrap the operators so that we work on number literals instead of string literals, like this:
```typescript
type AddN<N1 extends number, N2 extends number> = Add<IntToStr<N1>, IntToStr<N2>>;
```
However, it leads to *Numeric literals with absolute values equal to 2^53 or greater are too large to be represented accurately as integers* problem. In the case of larger numbers, this generates very weird results.

Secondly, although add/sub operations are quite reliable, operations like mul/mod are much more complex (unless we take a completely different approach) and thus they won't work on long numbers (in fact I run into *Type instantiation is excessively deep and possibly infinite* problem with a multiplicand of just 1000).

Thirdly, you might argue that digit addition could be implemented in a better way - instead of increasing/decreasing by one - using a hard-coded table for each combination. Something like this:

```typescript
type AddT<D1 extends Digit, D2 extends Digit> = {
    "0": ["0", D2];
    "1": { "0": ["0", "1"]; "1": ["0", "2"]; "2": ["0", "3"]; ... "9": ["1", "0"]; }[D2];
    "2": { "0": ["0", "2"]; "1": ["0", "3"]; "2": ["0", "4"]; ... "9": ["1", "1"]; }[D2];
    ...
    "9": { "0": ["0", "9"]; "1": ["1", "0"]; "2": ["1", "1"]; ... "9": ["1", "8"]; }[D2];
}[D1];

type AddTTest = AddT<"4", "8">; // ["1", "2"]
```
Looks nice and fast, the implementation of `AddDigitsC2` would be simplified, but surprisingly, I haven't noticed any meaningful improvement over the previous implementation.

Lastly, without the possibility to use recursive calls in conditional types, the implementations would be so much more ugly and painful.

--------

Links:
- [Template literal types and mapped type 'as' clauses PR][template-literal-types-pr]
- [Documentation of Template Literal Types][template-literal-types-doc]
- [Recursive conditional types][recursive-conditional-types-pr]
- [Checking never in conditional types][checking-never-in-conditional-types]

[template-literal-types-pr]: https://github.com/microsoft/TypeScript/pull/40336
[template-literal-types-doc]: https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html
[recursive-conditional-types-pr]: https://github.com/microsoft/TypeScript/pull/40002
[checking-never-in-conditional-types]: https://github.com/microsoft/TypeScript/issues/23182
