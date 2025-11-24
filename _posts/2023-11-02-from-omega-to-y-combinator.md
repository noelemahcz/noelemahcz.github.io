---
title: "From Omega to Y combinator"
categories: [Functional Programming, Haskell]
tags: [Haskell, Y Combinator]
---

## Ω Combinator

In λ-calculus, the **Ω combinator** is defined as:

```
Ω = (λx.x x) (λx.x x)
```

To implement this in Haskell, we first need a recursive type that allows self-application:

```haskell
-- (Rec a) and (Rec a -> a) are isomorphic,
-- and with newtype, they also share the same runtime representation.
newtype Rec a = Rec (Rec a -> a)

{-# NOINLINE unRec #-}
unRec :: Rec a -> Rec a -> a
unRec (Rec f) = f
```

> The `NOINLINE` pragma is necessary here to prevent GHC's optimizer from inlining functions that take recursive types as parameters, see [Known bugs in GHC][1].
{: .prompt-warning }

Now, we can define the **Ω combinator** to mirror its λ-calculus form:

```haskell
omega =
  (\x -> (unRec x) x)
  $ Rec
  (\x -> (unRec x) x)
```

Of course, `omega` isn't very useful on its own, as it results in an infinite loop that does nothing.

Let's add some observable behavior to it:

```haskell
omegaPrint =
  (\x -> putStrLn "Ω" >> (unRec x) x)
  $ Rec
  (\x -> putStrLn "Ω" >> (unRec x) x)
```

Alternatively, we can write it in a more concise form:

```haskell
omegaPrint = f (Rec f)
  where f x = putStrLn "Ω" >> (unRec x) x
```

Evaluating `omegaPrint` will result in an infinite loop, printing "Ω" repeatedly.

## Y Combinator

If we generalize the `omegaPrint` definition by abstracting the printing behavior:

```haskell
omegaWithExtraBehavior f =
  (\x -> f $ (unRec x) x)
  $ Rec
  (\x -> f $ (unRec x) x)

print x = putStrLn "Ω" >> x

omegaPrint = omegaWithExtraBehavior print
```

As it turns out, the `omegaWithExtraBehavior` function is the famous **Y Combinator** (or Fixed-point Combinator), implemented in Haskell.

```haskell
-- The Y Combinator
y f =
  (\x -> f $ (unRec x) x)
  $ Rec
  (\x -> f $ (unRec x) x)

-- An example function to use with the Y combinator
printNTimes f n = when (n > 0) $ putStrLn "Y" >> f (n - 1)

-- Apply the Y combinator
yPrintNTimes = y printNTimes
```

This corresponds to the following definition in λ-calculus:

```
Y = λf. (λx.f (x x)) (λx.f (x x))
```

[1]: https://downloads.haskell.org/ghc/latest/docs/users_guide/bugs.html#bugs-in-ghc
