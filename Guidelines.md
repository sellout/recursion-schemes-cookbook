# General Guidelines

## prefer algebras over folds

The key benefit of working with recursion schemes is being able to decompose things that otherwise seem too intertwined.

First, we can think in terms of individual steps of a tree – given just this one node, what do we need from the surrounding context in order to produce a value? This is what an algebra attempts to encapsulate.

In fact, even thinking in terms of “algebra” is overly restrictive. We think of F-algebras as having a very specific shape, `φ :: f a -> a`, and trying to produce this up-front can re-introduce some of the problems that we hope to eliminate with recursion schemes.

For example, if we’re trying to use a monadic algebra, writing it directly would force us to do `φ :: f (m a) -> (m a)` and handle the chaining of the monad ourselves. But we often simply want to do `φ' :: f a -> m a`. And we should be able to. We can then use `φ = φ' <=< sequence` when we need to actually apply the fold.

So, in order to simplify our operations the most, we shouldn’t even be thinking about algebras, but simply writing the functions that we need in the simplest way possible. We can then convert these functions into algebras when we fold them.

```haskell
-- | Convert a function that isn’t an F-algebra into a generalized F-algebra
--   that can be used with `zygo`. Which makes it clear that we need another
--  `f b -> b` in order to create the context.
zygoize :: (f b -> a) -> f (b, a) -> a
zygoize f = f . fmap fst
```

`zygoize`, above, is another case that shows that we don’t have to force ourselves into the pattern of a (generalized) F-algebra. If we don’t need to use the inherited value, then we shouldn’t even bother to ignore it – we shouldn’t have it there in the first place.

The “downside” of this is that it doesn’t _eliminate_ the complexity. It transfers it to where the fold is defined. Rather than simply doing `cata φ`, you may need to do a lot of massaging to combine the functions in various ways, like 
```haskell
gcata (distTuple (zipAlgebras g h) (zygoize f <=< sequence)
```
And it can be _way_ more complicated than that. However, there is no _other_ complexity when applying a fold, so better to shift this complexity there and away from your business logic, where you want to minimize the noise.

So, even though the complexity in this case isn’t eliminated, it’s moved away from the code you need to be very consicous of and careful about. Also, business logic usually includes rules that aren’t easy to encode in types, so it’s extra-important to be careful there, whereas the complexity with aligning algebras is apparant, but largely handled by following the types.

## prefer functors over fixed-points

