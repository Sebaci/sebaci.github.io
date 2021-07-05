---
layout: post
title:  Programming in TypeScript types with tuples
description: "Exploring advanced features to play with tuple and function types"
categories: typescript
---
The TypeScript type system has become increasingly powerful, allowing us to create very complex types today. In fact it's already shown to be [Turing complete][ts-turing-complete]. Many advanced utilities, including manipulation of object, list and function types can be found in the [ts-toolbelt][ts-toolbelt] library. Here I make a quick recap of some of the more advanced TypeScript features and then give a little introduction on how to use them to manipulate tuples like array values, but on the type level.

## Conditional types

Conditional type is a type-level counterpart of conditional expressions with ternary operators. It lets us select a type depending on whether given subtyping relation is satisfied:

```typescript
SomeType extends OtherType ? TrueType : FalseType;
```

A simple exapmle using generics: operator that returns element type if input type is array, or type itself otherwise
```typescript
type Flatten<T> = T extends Array<infer E> ? E : T;
```

The `infer` keywoard lets us introduce a new type variable `E` in the condition and leave the task of infering concrete type to TypeScript.

Another exapmle, an operator already defined in TypeScript library:

```typescript
type ReturnType<F> = F extends (...args: never[]) => infer R ? R : never;
```

You can think about type operators as type-level counterparts of functions, taking one (or more) type as a parameter and returning a different one.

## Indexed access types, mapped types and keyof operator

With these constructs we can build new type upon existing object type using index signature syntax.

```typescript
type DefinedKeysAux<O> = {
    [K in keyof O]: O[K] extends undefined ? never : K
};

type DefinedKeys<O> = DefinedKeysAux<O>[keyof O];
```

The first example is mapped type where we iterate through all keys of `O` and replace the value type with the key itself unless it's assignable to undefined. The `keyof` returns the union of keys and the indexed access type `O[K]` is used to obtain the value type.

The `DefinedKeys` operator then uses index syntax on newly created type and as a result we get a union of keys, for which `O` had defined values:

```typescript
type ExampleObj = {
    foo: string;
    date: Date;
    nested: {
        bar: null
    },
    qux: undefined
}

// "foo" | "date" | "nested"
type DefinedKeysExample = DefinedKeys<ExampleObj>;
```

Example usage below. The operator is called `QuasiJSON` because it's not totally accurate, e.g. it doesn't differentiate between `null` and `undefined`:

```typescript
type QuasiJSON<O> = {
    [K in DefinedKeys<O>]: O[K] extends Date
        ? string
        : O[K] extends object
            ? QuasiJSON<O[K]>
            : O[K]
};

const json: QuasiJSON<ExampleObj> = {
    foo: 'str',
    date: '01-01-2021',
    nested: { }
};
```

## Variadic tuple types

Tuples type represents arrays of fixed length, not necessarily homogenous.

```typescript
type StrNumPair = [string, number];
type Nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 0];
```

A big change in TypeScript 4.0 is the long-awaited possibility to spread generic types in tuples. Generic types will be instantiated according to the context:

```typescript
type PushElem<T extends unknown[], E> = [...T, E];

type PushElemExample = PushElem<[1, 2, 3], 42> // [1, 2, 3, 42]
```

This is convenient because functions operating on variadic tuples don't need to have multiple signature overloads defined. More on tuples [in the handbook][ts-40].

## Playing with tuples

Now let's define common operators on tuple types, just as we would do on ordinary lists. I return `never` for impossible cases.

```typescript
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;

type HeadEx1 = Head<[1,2,3]>; // 1
type HeadEx2 = Head<[]>; // never

type Tail<T extends unknown[]> = T extends [unknown, ...infer U] ? U : never;

type TailEx1 = Tail<[1,2,3]>; // [2, 3]
type TailEx2 = Tail<[]>; // never

type Cons<X, XS extends unknown[]> = [X, ...XS];

type ConsEx = Cons<"a", ["b"]>; // ["a", "b"]
```

Defining `concat` is also pretty straightforward, just spread both lists. Before TS 4.0 we could only spread concrete types at the end of tuple.

```typescript
type Concat<XS extends unknown[], YS extends unknown[]> = [...XS, ...YS];

type ConcatEx = Concat<[1, 2, 3], ["a", "b", "c"]>; // [1, 2, 3, "a", "b", "c"]
```

We see that defining such operators is fairly easy with current TypeScript features. Let's look at the following example:

```typescript
type Split<T extends unknown[]> = SplitAux<[], T, []>;
type SplitAux<L extends unknown[], R extends unknown[], A extends [unknown[], unknown[]][]> =
    R["length"] extends 0
        ? [[L, []], ...A]
        : SplitAux<[...L, Head<R>], Tail<R>, [[L, R], ...A]>;

// [[[1, 2, 3], []], [[1, 2], [3]], [[1], [2, 3]], [[], [1, 2, 3]]]
type SplitEx = Split<[1,2,3]>;
```

This operator computes all possible array splits into two parts. The auxiilary operator takes two lists representing current division and accumulator for results, which is initially empty. With each recursive step, one element from the right list is pushed to the left one and current state is saved in the accumulator. Recursion is complete when all elements end up in the left list.

## Curry'ing functions

Given a particular split and type `RT`, we can define a function type:

```typescript
type MakeFuncType<L extends unknown[], R extends unknown[], RT> =
     (...p1: L) => (...p2: R) => RT

// (p1_0: string, p1_1: number) => (p2_0: boolean) => number
type MakeFuncTypeExample = MakeFuncType<[string, number], [boolean], number>;
```

Here we created a type of function from `string` and `number` to function from `boolean` to `number`. Now we can generate all possible function types based on splits:

```typescript
type MakeAllFuncTypes<ArgsSplit extends [unknown[], unknown[]][], RT> =
    ArgsSplit extends [[ [...infer L], [...infer R] ], ...infer XS]
        ? XS extends [unknown[], unknown[]][]
            ? [MakeFuncType<L, R, RT>, ...MakeAllFuncTypes<XS, RT>]
            : []
        : [];
```

This peculiar use of `[...infer L]` is intended because when typing `infer L` instead, TypeScript forgets that it satisfies `unknown[]` in the `MakeFuncType` call. This is also the reason I artificially added `XS extends [unknown[], unknown[]][]` condition before recursive call.

How do we construct a type that can represent all functions of the generated types at the same time? In other words, some type `F'` representing a function that can be partially applied once based on a function of some type `F`? We need to define an operator that will generate intersection type from all types in a given tuple.

```typescript
type Intersect<T extends unknown[]> = T extends [infer X, ...infer XS]
    ? X & Intersect<XS>
    : unknown;

// (() => string) & ((x: number) => number)
type IntersectExample = Intersect<[() => string, (x: number) => number]>;
```

Note that at the bottom of the recursion, I return `unknown`, TypeScript top type.


Putting it all together, we can now define `SemiCurry` operator:

```typescript
type SemiCurry<F> = F extends (...p: infer P) => infer RT
    ? Intersect<MakeAllFuncTypes<Split<P>, RT>>
    : never;


type FuncExample = (p1: number, p2: number, p3: string) => number[];
type FuncExample2 = () => string;

type SemiCurryExample = SemiCurry<FuncExample>;
let f: SemiCurryExample;

f(1,2,"3")();
f(1, 2)("3");
f(1)(2, "3");
f()(1, 2, "3");
```

The reason I call it `SemiCurry` is because it allows partial application only once, but I believe with slight modifications it could be used to type Ramda's curry-like functions, which are fully curried but also maintain their natural usage in JS. Instead, I'll finish with my `TotalCurry` example, which converts a function type to its curried equivalent just as we'd expect in functional languages.

```typescript
type TotalCurry<F extends (...p: any[]) => any> = Parameters<F> extends [infer P, ...infer PS]
    ? PS extends []
        ? (p: P) => ReturnType<F>
        : (p: P) => TotalCurry<(...ps: PS) => ReturnType<F>>
    : F;

type TotalCurryExample = TotalCurry<FuncExample>;

let g: TotalCurryExample;
let res = g(1)(2)("foo"); // number[]
```

## Summary

Here I described basic ways to manipulate tuples with advanced features of TypeScript type system, wich can be fun itself or be used to define complex types like curried function types. For more advanced types you can explore projects like [ts-toolbelt][ts-toolbelt] or take some challenge like in this [collection of TS challenges][ts-challenges].

In the next post I'll explore template literal types and natural numbers.

Useful links:
- [TS is Turing Complete][ts-turing-complete]
- [Typescript toolbelt][ts-toolbelt]
- [Advanced types handbook][advanced-types]
- [Typescript 4.0][ts-40]
- [Typescript challenges][ts-challenges]

[ts-turing-complete]: https://github.com/Microsoft/TypeScript/issues/14833
[ts-toolbelt]: https://millsp.github.io/ts-toolbelt/
[advanced-types]: https://www.typescriptlang.org/docs/handbook/2/types-from-types.html
[ts-40]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-0.html
[ts-challenges]: https://github.com/type-challenges/type-challenges
