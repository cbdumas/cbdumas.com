---
title: Church-encoded datatypes in Haskell, part 2
date: December 27, 2015
---

[Last time](./church_encoding.html) we covered Haskells Maybe type encoded as a funciton.
This post will introduce two recursively defined structures, lists and natural numbers. Here I give
the normal ADT representation of a list, and its corresponding Church encoding.

```haskell
data List a = Nil | Cons a (List a)

newtype ListC a = ListC {unList :: forall r. r -> (a -> r -> r) -> r}
```

Just like MaybeC, ListC is a funciton that takes a default value and another function. In the case
of an empty list, it returns the default value. To understand the recursive (non-empty) case it is
helpful to note the similarity between the type of ListC and the type of foldr on regular lists,
and then to see how cons is implemented for Church lists: 

```haskell
foldr :: (a -> r -> r) -> r -> [a] -> r
```
In face the types of foldr and ListC are so similar because Church encoded lists are exactly
functions which fold over the lists they encode. To see how, observe the cons function given here:

```haskell
-- nil is a fold that does nothing but return the zero value supplied to it
nil :: ListC a
nil = ListC $ \z f -> z

-- cons take an element to add and a list, and build a new function which
-- folds over the new (longer) list
cons :: a -> ListC a -> ListC a
cons head (ListC tail) = ListC $ \z f -> head `f` (tail z f)
```

To give a concrete example, lets look at the list [1, 2] and use some equational reasoning to figure
out how its Church encoding works.

```haskell
-- Ordinary representation, then with explicit (:) calls
list :: [Int]
list = [1, 2]
     = 1 : (2 : [])

-- Then the Church encoding, using substitution to expand the definitions
-- newtype wrapping and unwrapping removed for clarity
listC = ListC Int
listC = 1 `cons` (2 `cons` nil)
      -- using the definition of cons:
      = 1 `cons` (\z f -> 2 `f` (nil z f))
      -- by def. of nil, with capture avoiding substitution:
      = 1 `cons` (\z f -> 2 `f` ((\z' f' -> z') z f))
      -- by applying the inner lambda:
      = 1 `cons` (\z f -> 2 `f` z)
      -- by def. of cons:
      = \z f -> 1 `f` ((\z' f' -> 2 `f'` z') z f
      -- by applying the inner lambda:
listC = \z f -> 1 `f` (2 `f` z)
```

As the last line shows, the Church encoded version is a function which folds over the list from the
right. 

And we can observe the parallels between these constructors and the definition of the foldr function
for lists in [the Prelude](https://hackage.haskell.org/package/base-4.8.1.0/docs/src/GHC.Base.html#foldr):

```haskell
foldr k z = go
        where
          go []     = z --nil
          go (y:ys) = y `k` go ys --cons
```


So if we want to define the Foldable instance for ListC, we need only define foldr as follows:
```haskell
instance Folable ListC where
    foldr f z (ListC l) = l z f

-- and a few other instances are easy to do given 'foldr':
instance Functor ListC where
    fmap f = foldr (cons . f) nil

instance Show a => Show (ListC a) 
    show = foldr (\head tail -> show head ++ "," ++ tail) "End"
```

In order to concatenate two lists we want to replace the end of one list with an entire new one. The
foldr function allows this to be expressed very concisely:

```haskell
(<++>) :: ListC a -> ListC a -> ListC a
xs <++> ys = foldr cons ys xs
```

Defining Nat
============

Having seen how ListC works to encode a recursive type, we can define our first primitive type
(rather than a type constructor like ListC or MaybeC), NatC. As usual, we'll look at the (recursive)
ADT and the Church encoding side by side:

```haskell
data Nat = Zero | Succ Nat

newtype NatC = NatC {unNat :: forall r. r -> (r -> r) -> r}
```

This looks exactly like the ListC type signature, but missing the type that goes inside the list.
Looking at the constructors for NatC:

```haskell
zero :: NatC
zero = NatC $ \x f -> x

succ :: NatC -> NatC -> NatC
succ n = NatC $ \x f -> f (n x f) 
```

And we can definte operations like fromInt, addition, multiplication from these constructors. However we can take
a slight shortcut and use the observation above that ListC and NatC are almost identical to reuse
some of our ListC machinery for NatC. This shortcut will utilize the following, alternative
definition to NatC:

```haskell
newtype NatC = NatC (ListC ())
```

Which is saying that our NatC type can be represented as a list of units. The length of the list
corresponds to the value of the number. Sum can the be given by list concatenation, 
