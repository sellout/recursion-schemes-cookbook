# Proto-Algebras

In the introduction, we discussed how "algebras" are a fundamental component of recursion schemes. This is absolutely true, but when using recursion schemes in practice, I want to suggest that you forget about algebras. Instead, we'll work with what I call "proto-algebras". Proto-algebras are nothing special, they're just any old function. So, then why come up with a new name if it doesn't denote anything special? The name is meant to indicate that these are functions that we'll _turn into_ algebras.

Algebras always look like `f a -> a`. Some common proto-algebras look like `f a -> m a` or `f (w a) -> a` or `b -> f a -> a`. If you squint a litte, you can see the underlying algebra, but it's not _yet_ an algebra. As I said, proto-algebras are just any old function, so some of them look much less like algebras than the functions above, but the "proto-algebra" label indicates our _intent_ to use them like algebras. Also, some proto-algebras can be made algebras in multiple ways, so we can save work by writing it as a proto-algebra and converting to each of the ways we might want to use it on demand.

The purpose of using proto-algebras is to allow us to write functions that more clearly declare what we're trying to accomplish. Recursion schemes inherently allow us to separate recursion from our logic, and once we've done that, we have the freedom to write functions that do exactly what we want.

For our first proto-algebra, `alg :: f a -> m a`, we can combine it with `sequenceA :: f (m a) -> m (f a)` to define `alg <=< sequenceA :: f (m a) -> m a`, which _is_ an algebra.

The second one `f (w a) -> a` is a little more complicated (so we capture it in the definition of `gcata` for you).  But we end up with
```haskell
lowerAlgebra ::
  (forall x. f (w x) -> w (f x)) ->
  (f (w a) -> a) ->
  f (w a) -> w a
lowerAlgebra dist alg = fmap alg . dist . fmap duplicate
```
where `dist` will be different depending on the concrete type of `w`.

Finally, `alg :: b -> f a -> a` can be turned into an algebra in at least two different ways. One is similar to the previous case, where uncurry `alg :: (b, f a) -> a` can be turned into an algebra with a similar approach. But the other is to just treat `b` as some constant, so, `alg 3 :: f a -> a` gives you an algebra.

So, the takeaway here is that you should write the function you need, not the function that fits into `cata`.

In some cases, you can even turn the same function into either an algebra or a coalgebra. `natTrans :: forall a. f a -> g a` is called a natural transformation. Since you can't know what the `a` is when you define a function of that type, it means that it could be anything. So, we can turn this into an algebra via `embed . natTrans :: f (Mu g) -> Mu g`, but you can just as easily turn it into a coalgebra `natTrans . project :: Nu f -> g (Nu f)`.

We've seen a few examples, but we'll cover a whole lexicon of approaches for turning your functions into algebras, and expect to find the more common ones provided in libraries.

Let's have an example, this one is a bit complex, but I think we can make it work.
```haskell
displayExpr ::
  MyType ->
  MyVar ->
  MyExpr (MyType, MyVar) ->
  Either ExprFailure String
```
Does this look like an algebra to you? It's not really, but also, it's not too far off.
Looking at this, we can see that in order to _use_ this proto-algebra, we're also going to need a way to get MyType and MyVar for various nodes. Let's write proto-algebras that do those things, too.
```haskell
deduceType :: MyExpr MyType -> MyType
```
that says that given a node with the types of its children, we can figure out the type of the node. This is already an algebra. How nice for us.
```haskell
nextVar :: State MyVar MyVar
```
This looks _nothing_ like an algebra, right? It's not even a function.

Ignoring the fact that these things don't look right yet, let's accept that they're at least proto-algebras, and that they let us completely independently define how we get each piece of data that we need.

Building a single comprehensive algebra can be a little complicated, but that complication is _already_ in the code, it's just that when you define one operation that does everything, it's littered throughout the code. This sifts the clean logic, leaving the tangly bits behind.
