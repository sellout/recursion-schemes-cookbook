# How can I more easily think in terms of recursion schemes?

## Don’t.

I have found the easiest way to think about recursion schemes is to avoid thinking about recursion schemes. I know, like all of recursion schemes, that sounds contradictory and obtuse. But the primary thing you need to do is break down your recursive structure into a pattern functor. Once you do that, you can write “plain old functions” without thinking about recursion schemes.

Here’s an example. Suppose we want to define `size` to count the number of elements in a list.
```haskell
length :: List a -> Natural
```
First, we swap out recursive bits for their pattern functors
```haskell
length' :: XNor a _ -> Natural
```
Now, much like we often think when defining recursive functions, what else do we need in order to calculate the result? In this case, we need the size of the tail of the list, right?
```haskell
length' :: XNor a Natural -> Natural
```
In this case, we already have an algebra
```haskell
length' :: Algebra (XNor a) Natural
```

In other cases, it’s very easy to wind up with functions that _don’t_ look like algebras. That’s not a problem, though. When you’re defining these functions, you should do so as simply as possible. If you need additional arguments, pass them in. If you need to return in a `Monad`, do so.
```haskell
foo :: Natural -> XNor a Text -> Either Error Text
```
That looks nothing like an algebra, right? don’t let that bother you. Just write what makes sense. The effort of turning a function (or set of functions) into an algebra should be done at the point of folding.
```haskell
lowerAlgebra (uncurry foo <=< sequenceA)
  :: Algebra (Compose (Either Error) (XNor a)) (Natural, Text)
```
We use four additional operations there (`lowerAlgebra`, `uncurry`, `<=<`, and `sequenceA`) to get the original function into the shape of an Algebra. But look at the algebra – would you rather implement the original `foo` or `foo'` below?
```haskell
foo' :: Compose (Either Error) (XNor a) (Natural, Text) -> (Natural, Text)
```
As a programmer, you should be trying to eliminate 

**NB**: Recursion scheme libraries often have a `size` algebra, which is different from `length` – it gives the number of _nodes_ in the structure, and since `Neither` is a node, the `size` is always one greater than the `length` of the list. However, `size` is definied for any tree with a `Foldable` pattern functor, so it’s very general.
