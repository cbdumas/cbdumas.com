---
title: Church-encoded datatypes in Haskell
date: December 19, 2015
---

Expressions in  λ-calculus are very tersely defined. Per [Wikipedia][lambda-definition]:

[lambda-definition]: https://en.wikipedia.org/wiki/Lambda_calculus#Definition


> The set of lambda expressions, Λ, can be defined inductively:
>
> 1. If x is a variable, then x ∈ Λ
> 2. If x is a variable and M ∈ Λ, then (λx.M) ∈ Λ
> 3. If M, N ∈ Λ, then (M N) ∈ Λ

Practical programming languages almost always define data primitives *and* function primitives, but
lambda calculus provides only unnamed functions. The Church encoding is one way of defining data types
in lambda calculus, and this post will explore what it means to treat functions as data with the
Church encoding in Haskell.

The first Church encoded type that really clicked for me was Maybe. Starting with Maybe
is a bit of a cop out, because it's only useful if you have some other data to wrap in it, but bear
with me!

Maybe is given here with its usual algebraic data type notation and its corresponding Church-encoding:

```haskell
-- As defined in the Prelude:
data Maybe a = Nothing | Just a

-- And an equivalent type, represented as a function:
newtype MaybeC a = MaybeC { runMaybe :: forall r. r -> (a -> r) -> r }
```

From the definition of Maybe we know that `Maybe a` is either wrapping an `a` or it's nothing. Inspecting the type
of MaybeC the intuition is that it takes a function `f :: (a -> r)` and a default value, and returns a value of `r`, 
either by applying `f` to the wrapped `a`, or returning the default. This is possible without knowledge of the type `r`,
 thus the universal quantification. The signature again, with the parts labelled:

```haskell
--                       output
--                 f       |
--      default    |       |
--        |        |       |
forall r. r -> (a -> r) -> r
```


Translating this intuition into Haskell gives the data constructors:

```haskell
-- When we do have a value, apply the given function to it
just :: a -> MaybeC a
just val = MaybeC $ \_ f -> f val

-- In the case where we have no value to work with, return the default
nothing :: MaybeC a
nothing = MaybeC $ \x _ -> x
```


And all of the usual instances for Maybe can be defined for MaybeC as well. For playing with this
type in GHCi it is especially useful to have an instance of Show (it is also useful to remember to
enable RankNTypes to get it to compile). Instances for Show and Functor can be defined
as follows:

```haskell
instance Show a => Show (MaybeC a) where
    show (MaybeC m) = m "Empty" (\x -> "Contains: " ++ show x)

instance Functor MaybeC where
    fmap f (MaybeC m) = m nothing (just . f)
```

The Show instance provides a good example of how to use a MaybeC value by passing in a
function and a default value. Here the default is the string "Empty", and the the function prints
the value insdie the MaybeC otherwise. Defining the Applicative and Monad instances is left as an
exercise for the interested reader.

In order to prove that Maybe and MaybeC are ismorphic, we first write the functions that map
between them:

```haskell
churchToMaybe :: MaybeC a -> Maybe a
churchToMaybe (MaybeC m) = m Nothing Just

-- Going in the other direction, note that this function can also be
-- written by slightly rearranging the arguments to `maybe` in Data.Maybe
maybeToChurch :: Maybe a -> MaybeC a
maybeToChurch Nothing = nothing
maybeToChurch (Just x) = just x
```

Isomorphism further requires that those functions are mutually inverse. That is:

```haskell
churchToMaybe . maybeToChurch = id
-- AND that
maybeToChurch . churchToMaybe = id
```

Which can be proved by equational reasoning, but I'll not write that out here. The next installment
will introduce the Church numerals, and Church encoded lists, and then we can really start to use
Church encoded data for something interesting!




