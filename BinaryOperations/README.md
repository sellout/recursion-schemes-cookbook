# How do I fold over two structures at once?

There are many operations where we want to look at two structures. The first one that comes to mind is often equality:
```haskell
equal :: (a, a) -> Bool
```
This is something we end up implementing all the time (or often deriving), but when we’re working with recursion schemes, how can we do this with an algebra?

I’ll spoil the trick up-front: We write an algebra over the _first_ parameter, where the [carrier](../glossary.md#carrier) is a function from the second parameter to the result, like so
```haskell
equal' :: Projectable t f => f (t -> Bool)

cata equal' :: (Recursive t f, Projectable t f) => t -> t -> Bool
```

The type on `cata equal'` there applies the fold to the _first_ `t`, and the resulting `t -> Bool`  makes it look exactly like the `equal` operation we want.

This shape of algebra is an [attribute grammar](../AttributeGrammars/README.md). But this specific case is for when both parameters are recursive structures that we want to align from the _root_.

## How to Implement

## The Shortcut

These algebras we’ve implemented are a bit ugly. Since part of recursion schemes is meant to _simplify_ our logic by extracting the recursion from it, this seems like a loss. Yes, we have many of the other benefits of recursion schemes still, but writing `equal` in a directly-recursive style seems much more natural. How would we _want_ to write an algebra that deals with multiple recursive structures?

It’d be nice if we could see _both_ functors at once, like
```haskell
type Algebra2 f a = (f a, f a) -> a

cata2 :: Recursive t f => Algebra2 f a -> (t, t) -> a
```
And there’s a helpful type that can turn `Algebra2` into a “regular” algebra:
```haskell
type :*: f g a = (f a, g a)

instance Recursive (t, t) (f :*: f) where
  cata :: Algebra (f :*: f) a -> (t, t) -> a
  cata φ = undefined
```
But how can we implement `cata` in this case? We can start by `project`ing both `t`s, which gives us `(f t, f t)`. But there are two new problems now – first we need to know how to “align” the `t`s, giving us some collection of `(t, t)` that we can recursively call `cata` on. And … that’s the second problem, even if we _can_ align them, we need to recurse on `cata` explicitly, ignoring the particular `Recursive` structure of `t`.

Let’s tackle the first one of these first. Is there some generic way we can combine the elements of `f` and `g`? If `f` is `List`, we can probably `zip` them – but what if the lengths are different? If `f` is `Maybe` … we can pair the `Just`s if they’re both `Just` – but what’s the correct behavior when one is `Just` and the other is `Maybe`? There doesn’t seem to be any reasonable generic choice.

Well, we can always punt – there is a little known structure called “Day convolution”
```haskell
data Day f g c = forall a b. Day (a -> b -> c) (f a) (g b)
```
This is similar to `:*:`, but has two differences
1. `f` and `g` can have different element types and
2. it takes a function that can combine the elements of `f` and `g` into a `c`.

If we use `Day` instead of `:*:`, `cata` no longer needs to figure out how to combine the `t`s – it can ask the `Day` structure.
```haskell
cata :: Projectable (t, u) (Day f g) => Algebra (Day f g) a -> (t, u) -> a
cata φ = . project
```

## tl;dr

```haskell
elementAt :: Algebra (Day Optional (XNor a)) (Maybe a)
elementAt = \case
    -- we’re at the last index, so return the value
    Day _ Nothing  (Both h _) -> Just h
    -- we’ve finished the list, but it’s not long enough
    Day _ _        Neither    -> Nothing
    -- we’re still going, so use the provided function to combine the remaining
    -- parts of both
    Day f (Just n) (Both _ t) -> f n t

cata2 :: (Recursive t f, Projectable u g) => Algebra (Day f g) a -> t -> u -> a
cata2 = cata . lowerDay

cata2 elementAt :: Natural -> [a] -> Maybe a -- overconstrained to illustrate
```

**NB**: With this approach, the _first_ argument to `cata` needs to have a `Recursive` instance, which the other one is more lax, so consider this when deciding what order the parameters should be in. E.g., with `elementAt`, as defined above, `Natural` needs to be finite, while `[a]` can be lazy/infinite. This is reasonable, because taking an element from a finite position of an infinite structure makes sense. But if we had reversed the functors, then we wouldn’t be able to support that case (the list would have to be finite), and we’d also have to “fail” (return `Nothing`) if we ever tried to get the element at `infinity = ana Just`.
