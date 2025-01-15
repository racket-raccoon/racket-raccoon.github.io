+++
title = "Memoization: A Primer"
date = 2025-01-10
[taxonomies]
tags = ["Memoization", "Cache", "Double lookup"]
+++

Anyways, *function memoization*, amiright? "Top-down" dynamic programming. Before attempting to calculate the function result for a particular set of parameters, we look up the table first to avoid doing the same work twice.

<!-- more -->

Super useful in many situations, especially when we want to speed up an intuitive but inefficient recursive solution without re-writing it into an iterative "bottom-up" one. A quick google shows that there are a couple of libraries that can do the heavy lifting for us, e.g., the [memo](https://docs.racket-lang.org/memo/index.html "memo") package. Here is an example provided in the docs:

```Racket
(require memo)

(define/memoize (fib n)
  (if (< n 2)
      1
      (+ (fib (sub1 n)) (fib (- n 2)))))

(fib 100)  ; 573147844013817084101
```

That should work fine for most situations; however, `define/memoize` by default uses `eq?` instead of `equal?` to compare cache keys, which might become an unpleasant surprise for you.

> This package provides macros for defining memoized functions. A memoized function stores its results in a **hasheq table**.

> The hash procedure creates a table where keys are compared with equal?, hashalw creates a table where keys are compared with equal-always?, **hasheq procedure creates a table where keys are compared with eq?**, hasheqv procedure creates a table where keys are compared with eqv?.

This is not the end of the world, of course. As the documentation says, the `#:hash` parameter specifies a hash table to use for the memoized data, and accessing the cache is done by calling the function without any arguments. Here is a quick demo:

```Racket
(define/memoize (display/test s)
  (displayln s))
...
> (display/test "groot")
groot
> (display/test (string #\g #\r #\o #\o #\t))
groot
> (display/test)
'#&#hasheq(("groot" . #<void>) ("groot" . #<void>))
```

```Racket
(define/memoize (display/test s) #:hash hash
  (displayln s))
...
> (display/test "groot")
groot
> (display/test (string #\g #\r #\o #\o #\t))
> (display/test)
'#&#hash(("groot" . #<void>))
```

Just to have another tool in your Racket toolbox, here is how to write the simplest memoizer from scratch. Following the implementation given in ["Racket Programming the Fun Way"](https://nostarch.com/racket-programming-fun-way "Racket Programming the Fun Way"), we would write something like this:

```Racket
(define fib
  (let ([h (make-hash)])
    (define (fib n)
      (cond
        [(< n 2) 1]
        [(hash-has-key? h n) (hash-ref h n)]
        [else
         (let ([f (+ (fib (sub1 n)) (fib (- n 2)))])
           (hash-set! h n f)
           f)]))
    fib))
```

That's a neat way of creating a closure. First, we always check for the base case, then we look up the table and return early if the result is already there. Otherwise, we perform the computation and save the value into the table. The only annoying thing here (probably coming from my C++ background) is the classic double lookup pitfall, one of the reasons why `std::map` didn't even have a `contains` method until C++20. Not sure what would be the cleanest way to avoid double lookup in this case, but I opt for something like this:

```Racket
(define fib
  (let ([h (make-hash)])
    (define (fib n)
      (if (< n 2)
          1
          (let ([cached (hash-ref h n 'miss)])
            (if (eq? cached 'miss)
                (let ([f (+ (fib (sub1 n)) (fib (- n 2)))])
                  (hash-set! h n f) f)
                cached))))
    fib))
```
