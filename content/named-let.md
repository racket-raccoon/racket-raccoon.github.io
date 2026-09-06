+++
title = "Named Let and Mutual Recursion"
date = 2025-01-19
[taxonomies]
tags = ["let", "letrec", "default arguments", "mutual recursion", "trace"]
+++

A quick introduction to the *named* `let`, why you might want to use `letrec`, and what *"mutually recursive functions"* are.
To learn more about the `let` family, consult [The Racket Guide](https://docs.racket-lang.org/guide/let.html "let") and [The Racket Reference](https://docs.racket-lang.org/reference/let.html).

<!-- more -->

Named `let` looked backwards to me at first; I had to return to the reference page more than once.
It is basically a local recursive function that gets called immediately, which makes it handy for recursion and little `while`-style loops.
Here is the SICP factorial example I used to make sense of it:

```Racket
(define (factorial n)
  (define (fact-iter [product 1] [counter 1] [max-count n])
    (if (> counter max-count)
        product
        (fact-iter (* counter product)
                   (+ counter 1)
                   max-count)))
  (fact-iter))
```

The only modification here is that instead of explicitly calling `fact-iter` as `(fact-iter 1 1 n)`, we've made those arguments [optional](https://docs.racket-lang.org/guide/lambda.html#%28part._.Declaring_.Optional_.Arguments%29 "optional arguments") by providing default values.
This makes it look very similar to the named `let` syntax:

```Racket
(define (factorial n)
  (let fact-iter ([product 1] [counter 1] [max-count n])
    (if (> counter max-count)
        product
        (fact-iter (* counter product)
                   (+ counter 1)
                   max-count))))
```

I used to reach for `let` for local bindings even when an internal `define` left the code with less indentation.
These days I mostly choose whichever version is easier to scan.
The Racket refactoring tool [resyntax](https://docs.racket-lang.org/resyntax/index.html "resyntax") can perform this particular cleanup automatically; it is the first example in its documentation.

Notably, attempting to rewrite the first example with raw `let` and `lambda` fails because the identifier created by a regular `let` can't be recursive.
`fact-iter` isn't available in the `lambda` body — you must use `letrec`:

```Racket
(define (factorial n)
  (letrec ([fact-iter
            (λ ([product 1] [counter 1] [max-count n])
              (if (> counter max-count)
                  product
                  (fact-iter (* counter product)
                             (+ counter 1)
                             max-count)))])
    (fact-iter)))
```

`let*` lets each clause refer to earlier clauses.
`letrec` lets the bindings refer to one another, including themselves, which is what we need for [mutually recursive functions](https://en.wikipedia.org/wiki/Mutual_recursion).
The usual example is `is-even?` and `is-odd?`.
`racket/trace` makes their little game of ping-pong visible:

```Racket
(define (is-even? x)
  (if (zero? x) #t (is-odd? (sub1 x))))

(define (is-odd? x)
  (if (zero? x) #f (is-even? (sub1 x))))

(require racket/trace)
(trace is-even? is-odd?)

(is-even? 6)
```

```
>(is-even? 6)
>(is-odd? 5)
>(is-even? 4)
>(is-odd? 3)
>(is-even? 2)
>(is-odd? 1)
>(is-even? 0)
<#t
#t
```

They keep calling each other until `x` reaches zero.
Here is the same example with `letrec`, where `is-even?` can refer to `is-odd?` before its definition appears:

```Racket
(letrec ([is-even?
          (λ (x)
            (if (zero? x) #t (is-odd? (sub1 x))))]
         [is-odd?
          (λ (x)
            (if (zero? x) #f (is-even? (sub1 x))))])
  (is-even? 6))
```

