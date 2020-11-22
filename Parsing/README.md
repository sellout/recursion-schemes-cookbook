# How can I parse with recursion schemes?

(That’s not a very good question for the book’s structure.)

```haskell
type Parser t a = [t] -> [(a, [t])]
-- t -> Compose [] (a,) [t]
```
A parser is a coalgebra. What does that mean, though? It means that parsing might not terminate. The existance of zero-length parsers illustrates this. If, given 0 characters, we can always extract an expression, then a poor grammar could run forever without consuming anything.

Even if we didn’t have zero-length parsers, the type above works with an infinite stream of tokens (which you can think of as characters, for simplicity). So, we might still parse an infinite input stream forever.

I feel like that latter case is _nicer_ than the former case – the former gives us a sense that we just can’t tell, but the latter implies that if we can ensure the input is finite, then the parse will terminate, and if it’s not, well, that’s a risk we have to take. At least the caller can control it.

But, you never get a finitary guarantee from the type in either case.

```haskell
data Foo a = Meh | Bar a | Lit Int

coproduct
    :: (Parser a -> Parser f a)
    -> (Parser a -> Parser g a)
    -> Parser a -> Parser (Coproduct f g a)
coproduct f g = uncurry (<|>) (fmap InL . f &&& fmap InR . g)

nu :: (Parser a -> Parser (f a)) -> Parser (Nu f)
nu f = 

foo :: Parser a -> Parser (Foo a)
foo a = 
```

This looks vey coalgebraic, right? `a -> f a`, but both sides wrapped in `Parser`
