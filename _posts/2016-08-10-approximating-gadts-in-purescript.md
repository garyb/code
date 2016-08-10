---
layout: post
title: Approximating GADTs in PureScript
categories: code
---

Before getting into this, I should probably mention there’s a paper that describes everything I’m about to in more detail and probably more eloquently too: [GADTless Programming in Haskell 98](https://www.cs.ox.ac.uk/files/3060/gadtless.pdf) by Martin Sulzmann and Meng Wang.

## The problem

In Haskell, with the `-XGADTs` extension enabled, we could write a little DSL for arithmetic and comparison expressions something like this:

``` haskell
data Expr a where
  Val :: Int -> Expr Int
  Add :: Expr Int -> Expr Int -> Expr Int
  Mult :: Expr Int -> Expr Int -> Expr Int
  Equal :: Expr Int -> Expr Int -> Expr Bool
  Not :: Expr Bool -> Expr Bool

eval :: Expr a -> a
eval (Val x) = x
eval (Add x y) = eval x + eval y
eval (Mult x y) = eval x * eval y
eval (Equal x y) = eval x == eval y
eval (Not x) = not (eval x)
```

We can throw various things at our `eval` function and see that it returns values of the appropriate type:

```
> eval (Val 42)
42

> eval (Add (Val 1) (Val 2))
3

> eval (Equal (Val 0) (Val 1))
False

> eval (Not (Equal (Mult (Val 10) (Val 1)) (Add (Val 0) (Val 1))))
True
```

Without GADTs it seems like this `eval` function would be impossible to write, since we wouldn’t have the information to determine the result type that matching on a data constructor of a GADT gives us. Here’s the naïve attempt to do so in PureScript:

``` haskell
data Expr a
  = Add (Expr Int) (Expr Int)
  | Mult (Expr Int) (Expr Int)
  | Equal (Expr Int) (Expr Int)
  | Not (Expr Boolean)
  | Val Int

eval :: forall a. Expr a -> a
eval (Add x y) = eval x + eval y
-- ... no point going any further, there’s already a problem:
--
--   Could not match type
--
--     Int
--
--   with type
--
--     a0
--
```

This is happening because we’re attempting to return a value of `Int`, but according to the type signature, we should be returning a value of type `a`. Oh dear. At this point `a` is like an existential type, there’s no way we can magic up a value that will satisfy it.

To avoid the type variable return we could write the evaluator in pieces, something like this:

``` haskell
evalI :: Partial => Expr Int -> Int
evalI (Val x) = x
evalI (Add x y) = evalI x + evalI y
evalI (Mult x y) = evalI x * evalI y

evalB :: Partial => Expr Boolean -> Boolean
evalB (Equal x y) = evalI x == evalI y
evalB (Not x) = not (evalB x)

eval :: Either (Expr Int) (Expr Boolean) -> Either Int Boolean
eval (Left expr) = Left (unsafePartial evalI expr)
eval (Right expr) = Right (unsafePartial evalB expr)
```

But this is not at all as useful as the original `eval` since you have to deal with the `Either` result every time. It also makes extending the DSL difficult, as if we wanted to add expressions that dealt with a type other than `Int` or `Boolean` we’d have to split the `Either`s yet again... not to mention the `Partial` issues.

So, is there some other way we can do this and still have a generic `eval` function without GADTs?

## Type equality

The thing that we need from GADTs to make a function like `eval` possible is some notion of type equality.

In the GADT-defined `Expr`, the final part of the type annotation for each data constructor, `... -> Expr SomeType`, is essentially saying “`a ~ SomeType` for this constructor”. We use `~` for equality rather than `=` when talking about types.

When a constructor of a GADT is pattern-matched, the typechecker gains knowledge of this equality and then lets us return a value of the appropriate type, rather than being stuck with the rigid type variable `a`.

We can’t directly encode type equalities using the type system in PureScript, but we can take advantage of “Leibniz equality” [[1]](#ref1) and encode it in a value instead:

``` haskell
newtype Leibniz a b = Leibniz (forall f. f a -> f b)

infix 4 type Leibniz as ~

symm :: forall a b. (a ~ b) -> (b ~ a)
symm = ...

coerce :: forall a b. (a ~ b) -> a -> b
coerce = ...
```

The above is from [purescript-leibniz](https://github.com/paf31/purescript-leibniz). I’ve omitted the implementation of `symm` and `coerce` here because `unsafeCoerce` is used for efficiency’s sake, but it is possible to implement both [without cheating](https://github.com/garyb/purescript-leibniz-proof).

As the name `coerce` suggests, this allows us to coerce one type to another as long as we have a `Leibniz` value for proof that the types are equal. Since at some point we knew the types were equal, this means the coercion is totally safe.

The `symm` function is used when we have a type equality that we need to “turn around” so that we can `coerce` in either direction based on a provided `Leibniz` value. This is possible thanks to the fact that equality relations are symmetric (for every `a` and `b`, if `a = b` then `b = a`), and is where the name `symm` comes from.

## The solution

First we extend each of our data constructors with an extra argument to carry a `Leibniz` value as a proof of what their `a` type variable should be:

``` haskell
data Expr a
  = Add (Expr Int) (Expr Int) (a ~ Int)
  | Mult (Expr Int) (Expr Int) (a ~ Int)
  | Equal (Expr Int) (Expr Int) (a ~ Boolean)
  | Not (Expr Boolean) (a ~ Boolean)
  | Val Int (a ~ Int)
```

If you squint a bit you can see that it’s starting to look more like the original GADT version of the definition now too.

We can now write `eval` using the `coerce` function to safely cast the result type back to `a` in each case:

``` haskell
coerceSymm :: forall a b. (a ~ b) -> b -> a
coerceSymm = coerce <<< symm

eval :: forall a. Expr a -> a
eval (Val x proof) = coerceSymm proof x
eval (Add x y proof) = coerceSymm proof (eval x + eval y)
eval (Mult x y proof) = coerceSymm proof (eval x * eval y)
eval (Equal x y proof) = coerceSymm proof (eval x == eval y)
eval (Not x proof) = coerceSymm proof (not (eval x))
```

We could avoid the need for `symm`/`coerceSymm` here by swapping the arguments to `Leibniz` in the data constructors (`Int ~ a`) but purely for aesthetic reasons I prefer to put the type variable first.

This isn’t quite as elegant as if we had real GADTs as we need to coerce the result in each case, but it’s not so bad - certainly a lot better than the `Either` based version!

The only thing we haven’t covered is how to actually construct a `Leibniz` value in the first place. What possible function could we provide that satisfies `forall f. f a -> f b` where `a` equals `b`? The identity function! `Leibniz` also has a `Category` instance, so actually we can just use `id` directly:

``` haskell
value = Val 1 id

--   The inferred type of value was:
--
--     Expr Int
```

To make constructing our `Expr` values a bit more pleasant, we probably want to define a function for each constructor so we can omit the `id` each time:

``` haskell
val :: Int -> Expr Int
val x = Val x id

add :: Expr Int -> Expr Int -> Expr Int
add x y = Add x y id

mult :: Expr Int -> Expr Int -> Expr Int
mult x y = Mult x y id

equal :: Expr Int -> Expr Int -> Expr Boolean
equal x y = Equal x y id

not :: Expr Boolean -> Expr Boolean
not x = Not x id
```

Oh look, the exact type signatures from the GADT constructors are here now.

And finally, we can run our original example cases again:

```
> eval (val 42)
42

> eval (add (val 1) (val 2))
3

> eval (equal (val 0) (val 1))
false

> eval (not (equal (mult (val 10) (val 1)) (add (val 0) (val 1))))
true
```

So we can indeed get a pretty good approximation of GADTs in PureScript with this technique, at the expensive of some boilerplate.

## Notes

1. <a name="ref1"></a>As far as I can tell it’s named such as it’s an encoding of [Leibniz](https://en.wikipedia.org/wiki/Gottfried_Wilhelm_Leibniz)’s principle of the [identity of indiscernibles](https://en.wikipedia.org/wiki/Identity_of_indiscernibles).
