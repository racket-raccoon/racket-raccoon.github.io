+++
title = "Named Let and Mutual Recursion"
date = 2025-01-19
[taxonomies]
tags = ["let", "letrec", "default arguments", "mutual recursion", "trace"]
+++

A quick introduction to the *named* `let`, why you might want to use `letrec`, and what *"mutually recursive functions"* are. To learn more about the `let` family, consult [The Racket Guide](https://docs.racket-lang.org/guide/let.html "let") and [The Racket Reference](https://docs.racket-lang.org/reference/let.html).

<!-- more -->

The syntax of named let initially felt alien to me, requiring multiple visits to the reference page, but it's actually quite simple. Named `let` is syntactic sugar that allows us to write a function and immediately call it in-place — perfect for recursion and basic `while` loops. Let's examine the classic SICP factorial example:

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

The only modification here is that instead of explicitly calling `fact-iter` as `(fact-iter 1 1 n)`, we've made those arguments [optional](https://docs.racket-lang.org/guide/lambda.html#%28part._.Declaring_.Optional_.Arguments%29 "optional arguments") by providing default values. This makes it look very similar to the named `let` syntax:

```Racket
(define (factorial n)
  (let fact-iter ([product 1] [counter 1] [max-count n])
    (if (> counter max-count)
        product
        (fact-iter (* counter product)
                   (+ counter 1)
                   max-count))))
```

Some programmers prefer using `let` over `define` for local bindings, even without a compelling reason — it's often considered a matter of style. However, the general rule is to reduce code indentation when possible and use `define` unless you need `let`-specific features. The Racket refactoring tool [resyntax](https://docs.racket-lang.org/resyntax/index.html "resyntax") can automatically refactor unnecessary `let` expressions, as shown in its very first documentation example.

Notably, attempting to rewrite the first example with raw `let` and `lambda` fails because the identifier created by a regular `let` can't be recursive. `fact-iter` isn't available in the `lambda` body — you must use `letrec`:

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

While `let*` only allows using identifiers from previous `[]` clauses, `letrec` enables both recursive use of an identifier and referencing identifiers from previous and subsequent clauses, enabling [mutually recursive functions](https://en.wikipedia.org/wiki/Mutual_recursion). A classic example is the `is-even?` and `is-odd?` functions defined in terms of each other. Let's use the `racket/trace` package to visualize the call stack — a valuable debugging tool, particularly for recursive calls:

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

Now you can see these functions playing ping-pong, calling each other until reaching a base case. Finally, here's the same example rewritten using `letrec`, which allows mutual recursion even though `is-odd?` is referenced before being defined:

```Racket
(letrec ([is-even?
          (λ (x)
            (if (zero? x) #t (is-odd? (sub1 x))))]
         [is-odd?
          (λ (x)
            (if (zero? x) #f (is-even? (sub1 x))))])
  (is-even? 6))
```

