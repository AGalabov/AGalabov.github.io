This year I decided to take on the [Advent of Code](https://adventofcode.com/) challenge along with some of my colleagues and it took me only a day to take on an interesting challenge of my own. You can look at my progress here: [AGalabov/advent-of-code-2021](https://github.com/AGalabov/advent-of-code-2021)

# About the problem

Advent of Code is an yearly challenge counting down the days until Christmas. Every day participants are given two "puzzles" to solve. You only need to get to the right solution, so you are free to use any language, framework or technology. I decide that since this is my first go at the challenge, I'd stick to what I know best - _Typescript_ and occasionally _C++_.

## Day 1, puzzle 1

The first puzzle came up. There was a whole narrative created that would be followed throughout the event, but essentially the task could be described as simple as this:

Given an array of whole positive numbers, find how many times did the numbers rise compared to their previous one. For example:

```
199 200 208 210 200 207 240 269 260 263
```

the desired outcome would be `7`, the numbers that we counted being `200 208 210 207 240 269 263`.

As expected from day 1 this was not hard task. The solution I came up with took no more than a few minutes and it gave me the expected results both on sample data and on the actual input provided. It looked like this:

```ts
input
  .map((x, i) => (i > 0 ? Number(x > input[i - 1]) : 0))
  .reduce((x, y) => x + y);
```

Then I went on and solved the other puzzle for the day and that was that. Or so I thought.

## Typescript's Typesystem

The day after me and a colleague started discussing the event. He mentioned that knowing _Typescript_ is in fact _Turing compete_ it is fully possible to solve this issue using the typesystem only. At first I was confused, but then he explained his approach to me, but also that he didn't manage to finish it because of the depth limit TS enforces (as of TS 4.3 it is 50). So that's when I decided it's game time!

# The solution

## What I wanted to achieve

When it comes to coding something that you are not completely sure how to implement, I find it easier if you start with the interface. So I ask myself

> "What I want to achieve?"

```ts
type Sample = [199, 200, 208, 210, 200, 207, 240, 269, 260, 263];

type Solution = NumberOfIncreases<Sample>; // Solution should be of the type 7 (the exact literal 7)
```

## Building blocks

To start with we need a type representation of a number that would allow me to do simple arithmetics. Looking at the possible types that _Typescript_ provides, we have simple values (`boolean`, `number`) that just won't make it - we cannot expand them. We need something that keep multiple values (like an array) but makes a difference between having 2,3 or 10 values. Enter tuples!

A tuple is essentially an array but one of a definite size, where elements at particular positions have to be of a definite type. For example `type StringAndNumber = [string, number];` defines a type. Any variables of the type need to be an array of exactly 2 elements, first of which being a string, and the second - a number.

The important thing about tuples is that as every array, there is a `length` property that can be used.

```ts
type Length = StringAndNumber['length']; // Length would be of type 2
```

To generalize it we can make `Length` accept a generic argument:

```ts
type Length<T extends any[]> = T extends { length: infer L }
  ? L extends number
    ? L
    : Length<[]>
  : Length<[]>;
```

What that code does is as simple as:

1. Check if the provided type `T` has a property `length`
2. If it does and it is a `number` - return it (`L`)
3. If not - return the length of an empty array (or by transition - `0`)

So now we know that tuples can give us the fundamental building block for numbers. So let's define some types:

```ts
type Tuple = any[];
type Zero = [];
```

For what is worth, a tuple is still an array of elements. Since we only care about their length we can go with `any` (in any other case, please provide meaningful types).

We can revisit our `Length` definition now to make it a little more semantically appealing:

```ts
type Length<T extends Tuple> = T extends { length: infer L }
  ? L extends number
    ? L
    : Length<Zero>
  : Length<Zero>;
```

So now we have our representation of numbers. But how do we actually "represent" numbers as tuples. Again if we think about the usage first we would want something like `BuildTuple<5>` to create a tuple like `[any, any, any, any, any]`. We can do this recursively:

```ts
type BuildTuple<N extends number, T extends Tuple = Zero> = T extends {
  length: N;
}
  ? T
  : BuildTuple<N, [...T, any]>;
```

Again this code my look a little scary to someone but it can be read as:

1. We take a number `N` and an accumulated value `T` (which by default is our `Zero`)
2. We check if the `length` property of `T` is exactly the desired number `N`
3. If it - then `T` is our tuple representation
4. If not - we recursively call `BuildTuple` much like a function with the new accumulated value being T with an additional `any`

Just to check that everything works as expected, let's apply `Length` on a type created from `BuildTuple`

```ts
type Five = Length<BuildTuple<5>>>; // Five is actually of type 5.
```

### Limitations

As I already mentioned there are recursion depth limitation when it comes to _Typescript_:

```ts
type UnderLimit = BuildTuple<46>; // this would be infered correctly
type AboveLimit = BuildTuple<47>; // this one results in "Type instantiation is excessively deep and possibly infinite"
```

## Arithmetics:

Now that we have our concept of `Zero` and can create any other number as a tuple representation, how do we apply arithmetics.

### Addition

Let's start with defining `PlusOne`:

```ts
type PlusOne<N extends number> = Length<[...BuildTuple<N>, any]>;
```

This is self explanatory:

1. Given a number `N` we create a tuple from it using `BuildTuple`
2. We then create another tuple, expanding the current tuple and adding one more type element (`any`).
3. Finally we use `Length` so that we go back to a number constant number type.

As a result we have:

```ts
type Six = PlusOne<5>; // Six is of type 6
```

The way to define `Plus` for two numbers is pretty obvious now:

```ts
type Plus<A extends number, B extends number> = Length<
  [...BuildTuple<A>, ...BuildTuple<B>]
>;
type Twelve = Plus<4, 8>; // Twelve is of type 12
```

### Subtraction

Doing the opposite we can define `MinusOne`:

```ts
type MinusOne<N extends number, T = BuildTuple<N>> = T extends Zero
  ? Length<Zero>
  : T extends [any, ...infer Rest]
  ? Length<Rest>
  : Length<Zero>;
```

What we do here is the following:

1. Given a number `N` we create a tuple from it using `BuildTuple` and save it as `T`
2. We check if `T` is `Zero` and if so we return `Length<Zero>` (which is 0)
3. If not then, we check if `T` can be represented as a single type `any` and some values `Rest`
4. If we can, then `Length<Rest>` is the result of the subtraction (we have removed one element)
5. If not - then we have a `0`

As a result we have:

```ts
type Thirteen = MinusOne<14>; // Thirteen is of type 13
```

And just like with `Plus` it's quite easy to define a `Minus` now:

```ts
type Minus<
  A extends number,
  B extends number,
  T = BuildTuple<A>
> = T extends Zero
  ? Length<Zero>
  : T extends [...infer Rest, ...BuildTuple<B>] // just like with the any, we just define expand a tuple of the size to subtract
  ? Length<Rest>
  : Length<Zero>;

type Seven = Minus<13, 6>; // Seven is of type 7
```

## Comparisons

In order to find the numbers of increases we would need to be able to compare two numbers. Let's do it:

```ts
type GreaterThan<
  A extends number,
  B extends number
> = BuildTuple<B> extends Zero
  ? BuildTuple<A> extends Zero
    ? false
    : true
  : GreaterThan<MinusOne<A>, MinusOne<B>>;
```

Let's go line by line:

1. We create a tuple from B and check if it is `Zero`
2. If it is then we create one from `A` as well and check it agains `Zero`.
3. If it is then `A` and `B` are equal, hence we return `false`
4. If `A` isn't, then `A` is greater than `B` and we return `true`
5. If `B` is not `Zero` then we recursively call `GreaterThan` subtracting one from both `A` and `B`

The results is as expected:

```ts
type Greater = GreaterThan<5, 3>; // Greater is of type true
type Equal = PlusOne<5, 5>; // Equal is of type false
type Smaller = GreaterThan<3, 5>; // Smaller is of type false
```

## The actual solution

Let's take a look at it from a developer's standpoint and try to find a recursive solution (since this is how TS works):

```ts
function numberOfIncreases(
  numbers: number[],
  lastNumber: number | null = null,
  accumulator: number = 0
): number {
  if (numbers.length === 0) {
    return accumulator;
  }

  const [number, ...rest] = numbers;

  if (lastNumber && number > lastNumber) {
    return numberOfIncreases(rest, number, accumulator + 1);
  }

  return numberOfIncreases(rest, number, accumulator);
}
```

The bottom of our recursion is when we have gone through all elements and we return the accumulated value. If we have a `lastNumber` (any call apart from the first one) and the current `number` is greater we call the function on the `rest` of the numbers with an increased `accumulator`. Else we just call it on `rest` without increasing it.

Now let's "translate" that into a type definition:

```ts
type NumberOfIncreases<
  Input extends number[],
  Previous extends number | null = null,
  Accumulator extends number = Length<Zero>
> = Input extends []
  ? Accumulator
  : Input extends [infer Curr, ...infer Rest]
  ? Previous extends number
    ? GreaterThan<Curr, Previous> extends true
      ? NumberOfIncreases<Rest, Curr, PlusOne<Accumulator>>
      : NumberOfIncreases<Rest, Curr, Accumulator>
    : NumberOfIncreases<Rest, Curr, Accumulator>
  : Accumulator;
```

Now sadly the direct "translation" does not work. Typescript infers `Curr` and `Rest` as `unknown` instead of as `number`. To fix this we could:

1. Use `Input[0]` instead of `Curr` to have proper inference
2. Define a type `RestFromArray` that would infer them properly

Now for `RestFromArray`, the intuitive solution would be:

```ts
type RestFromArray<T extends Tuple> = T extends [any, ...infer Rest]
  ? Rest
  : [];
```

It would work in cases of `RestFromArray<[1, 2, 3]>` and would result in `[2, 3]`. But in more complex cases as in the `NumberOfIncreases` type definition `RestFromArray<Input>` would once again result in `unknown[]`.

To fix this we can use a trick, that uses function arguments, since they are represented by actual tuples:

```ts
type RestFromArray<T extends Tuple> = ((...args: T) => void) extends (
  first: any,
  ...rest: infer Rest
) => void
  ? Rest
  : [];
```

Now the final implementation would then look like this:

```ts
type NumberOfIncreases<
  Input extends number[],
  Previous extends number | null = null,
  Accumulator extends number = Length<Zero>
> = Input extends []
  ? Accumulator
  : Previous extends number
  ? GreaterThan<Input[0], Previous> extends true
    ? NumberOfIncreases<RestFromArray<Input>, Input[0], PlusOne<Accumulator>>
    : NumberOfIncreases<RestFromArray<Input>, Input[0], Accumulator>
  : NumberOfIncreases<RestFromArray<Input>, Input[0], Accumulator>;
```

This is a very good example how actual logic can be written with types.

And now after all of this we get to the moment of truth. Let's test it out? Are you excited? Let's go:

```ts
type Sample = [199, 200, 208, 210, 200, 207, 240, 269, 260, 263];

type Solution = NumberOfIncreases<Sample>;
```

You type that and `ts` starts to complain about `"Type instantiation is excessively deep and possibly infinite"` error once again. But actually that shouldn't come as a surprise to anyone right now. We already discovered using the current implementation even running `BuildTuple<47>` reaches the maximum depth of calculation. So we cannot expect to create one for `199`, `200` and so on and then make computations on top of them.

But that doesn't necessarily mean that the type definition does not work. Let's test it with smaller numbers:

```ts
type Sample = [1, 2, 4, 3, 6, 5, 7, 8, 5, 6];

type Solution = NumberOfIncreases<Sample>; // Solution is correctly of type 6! :yay:
```

So the "algorithm" actually works just fine!

# Conclusion

We did manage to implement an algorithm that uses types only to solve an Advent of Code level puzzle.

> Is it useful? - **No!**\
> Is it something that you need to do in your life? - **Probably no!**\
> Was it fun? - **For me, yes!**\
> Was it worth it? - **YES!**

## Takeaway points

In the process of implementing the building blocks, arithmetics, comparisons and the actual algorithm:

1. I found out some new capabilities of Typescript
2. I reminded myself of some maths fundamentals
3. I wrote my first and hopefully not last blog post
