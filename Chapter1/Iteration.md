# While We're Here

Our first encounter with iteration usually starts with a while loop -- unless you were lucky enough
to be programming with assembly language. The idea of a while loop is that we want to execute a
series of statements until we meet some condition. To illustrate, we have some C like code:

```c
while condition {
    statement1;
    statement2;
    ...
}
```

We generally think of this program having steps. At each step we have some statements we want to
execute, whether it's updating some variables or making some calculations. We then check if our
condition has changed since the previous step and exit once it's satisfied.

Let's take a more concrete example of the Fibonacci sequence -- the exemplar of recursion. The
Fibonacci sequence is defined mathematically as:

```
fib n | n == 0 = 0
fib n | n == 1 = 1
fib n = fib (n - 1) * fib (n - 2)
```

The first two cases here are what are known as base cases. They are the cases of the function that
define the starting point, since `fib(0)` will give us `0`, fib(1)` will give us `1`, then `fib(2)`
can be broken down into `fib(1) + fib(0)` which gives us `1`, and so on.

**Exercise:** Using pen and paper what is the value of `fib(5)`? What about `fib(12)`?

Our goal here is to see how this translates into our iterative model of while loops. For while loops
there is usually some state that we carry and we update it as we go along. For Fibonacci this state
will be based on our input number `n` and two variables `a` and `b`.
To start, we define the function with our stateful input. The starting values for `a` and `b` will
be our bases cases `0` and `1`.

```c
fn fib(n: int) {
    a: int = 0;
    b: int = 1;
}
```

We then need to introduce our loop to handle the recursive cases as well as the condition for when
to stop. Since our input `n` will be asking for the `n`th Fibonacci number, we will be decrementing
the input and checking we haven't gone past `0`.

```c
fn fib(n: int) {
    a: int = 0;
    b: int = 1;

    while (n > 0) {
        n = n - 1;
    }
}
```

The final step for our loop is to calculate the updated value of our iteration. This is where step
reasoning comes in. We think about what the state of our values are at each step. As it stands we
only have two values `a` and `b`. So let's start with deciding what happens when `n` is `0`. In this
case we never enter the loop, so our resulting value should be `0`.

**Exercise:** Which of `a` or `b` should we return?

```c
fn fib(n: int) {
    a: int = 0;
    b: int = 1;

    while (n > 0) {
        n = n - 1;
    }

    return a;
}
```

In the case where `n` is `1` we need some way to make `a` equal to `1`. Looking back our
mathematical definition we know that we `fib(1) = 1` so we could just assign `a = b`.

```c
fn fib(n: int) {
    a: int = 0;
    b: int = 1;

    while (n > 0) {
        a = b;
        n = n - 1;
    }

    return a;
}
```

But then what happens when `n` is `2`? We need to be able to calculate `fib(0) + fib(1)`. In our
case, that's the calculation of `a + b`. Let's assign this value to a new variable `x`.

```c
fn fib(n: int) {
    a: int = 0;
    b: int = 1;

    while (n > 0) {
        x: int = a + b;
        a = b;
        n = n - 1;
    }

    return a;
}
```

But this leaves `b` permanently assigned to `1`. So if we reason what's happening at each step we
get a table that looks like the following:

| n = 2 | a = 0 | b = 1 | x = 1 |
| n = 1 | a = 1 | b = 1 | x = 2 |
| n = 0 | a = 1 | b = 1 | x = 2 |

So our final step is to update `b`, by assigning it the brand new Fibonacci number that is held in
`x`.

```c
fn fib(n: int) {
    a: int = 0;
    b: int = 1;

    while (n > 0) {
        x: int = a + b;
        a = b;
        b = x;
        n = n - 1;
    }

    return a;
}
```

To make sure we're doing this right, we can draw out another table like above, but let's denote
`a`'s value at the start of the loop step as `a0` and `a`'s value at the end of a loop step as `a1`,
similarly for `b`.

| n = 5 | a0 = 0 | b0 = 1 | a1 = 1 | b1 = 1 |
| n = 4 | a0 = 1 | b0 = 1 | a1 = 1 | b1 = 2 |
| n = 3 | a0 = 1 | b0 = 2 | a1 = 2 | b1 = 3 |
| n = 2 | a0 = 2 | b0 = 3 | a1 = 3 | b1 = 5 |
| n = 1 | a0 = 3 | b0 = 5 | a1 = 5 | b1 = 8 |

This means when `n` reaches `0` we have the value that's left in `a` to be returned which is `5`. If
we lay out the Fibonacci sequence here: 0, 1, 1, 2, 3, 5 we can see that the 5th value in the
sequence is `5` (TODO(finto): Classic computer science -- off by one error).

**Exercise:** Factorial is also an overused recursion example and this book won't be any different!
Given the mathematical definition of factorial:

```
fac n | n == 0 = 1
fac n | n == 1 = 1
fac n = n * fac(n - 1)
```

Reason about it in the same way above, defining a while loop, and create some tables to see if your
reasoning matches up.

# For Each Step
