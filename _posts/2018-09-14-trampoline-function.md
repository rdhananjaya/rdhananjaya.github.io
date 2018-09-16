---
layout: post
title:  "Understanding trampoline function: example in Java"
summary: "Here I try to understand the design and execution of trampoline concept using Java as example language."
date:   2018-09-14 22:01:36 +0530
categories: design, CS
comments: true
page_id: trampoline
---

After numerous attempts at understanding the trampoline concept
I found [this](https://stackoverflow.com/questions/189725/what-is-a-trampoline-function) StackOverflow question that helped me understand it better, well better than my other attempts. In this post I'm elaborating it a bit so that I can refer to it later.

```java
import java.math.BigInteger;

// Data structure used to hold data used for trampolining.
class Trampoline<T>
{
    public T get() { return null; }
    public Trampoline<T>  run() { return null; }

    T execute() {
        Trampoline<T>  trampoline = this;

        while (trampoline.get() == null) {
            trampoline = trampoline.run();
        }

        return trampoline.get();
    }
}

public class Factorial
{
    public static Trampoline<BigInteger> factorial(final int n, final BigInteger product)
    {
        if(n <= 1) {
            return new Trampoline<BigInteger>() { public BigInteger get() { return product; } };
        }
        else {
        // tramplining algorithm implementation for use case at hand.
            return new Trampoline<BigInteger>() {
                public Trampoline<BigInteger> run() {
                    return factorial(n - 1, product.multiply(BigInteger.valueOf(n)));
                }
            };
        }
    }

    public static void main( String [ ] args )
    {
        // invocation of the trampoline function.
        System.out.println(factorial(100000, BigInteger.ONE).execute());
    }
}
```

Trampolining is used mostly to avoid excessive stack allocations. That is in languages that does not support trail call optimization. Because recursive function calls with large enough inputs will lead to stack overflow.
We can use trampolining technique to avoid those stack allocations. Although this will help us avoid stack allocation this might not help us with the overall memory consumption.

Let's consider the naive factorial function, using BigInteger result type.

```java
BigInteger factorial(long n) {
    if (n <= 1) {
        return BigInteger.ONE;
    }
    return BigInteger.valueOf(n).multiply(factorial(n-1))
}
```


The name trampoline is because in this mechanism the execution jumps to the trampoline function and immediately jumps out to some arbitrary execution and then again fall into the trampoline and jumps back and so forth.

In the example `factorial` program, `run()` is this arbitrary execution step (or the thunk). In our case run() will execute one step of the algorithm and jump back to the trampoline. Then the while loop will schedule the next step of the execution, depending on what the previous execution of `run()` returned.

The `factorial` function in the trampolining code looks at the inputs and return one of the two instances of the Trampoline instances.


