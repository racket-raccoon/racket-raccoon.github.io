+++
title = "Threading and Function Composition"
date = 2025-01-15
[taxonomies]
tags = ["threading", "Procedures", "Clojure"]
+++

One of the features I started using extensively this year is a threading macro. The word "thread" in this context means passing a value through a pipeline of functions and has nothing to do with concurrent threads of execution. Also, I wanted to give an honorable mention to some higher-order functions we have out of the box from Racket.

<!-- more -->

## TLDR; It's just syntactic sugar to flatten nested function calls for readability. With this macro, you can rewrite `(add1 (exact-floor (log num 10)))` as `(~> num (log 10) exact-floor add1)`.

I first bumped into this concept while doing some [Clojure](https://clojure.org/guides/threading_macros "clojure threading") coding challenges on [Exercism](https://exercism.org/ "exercism"). Those macros are part of the core Clojure language, so you don't need to install any libraries. In Racket, unfortunately, you have to install the [threading-lib](https://docs.racket-lang.org/threading/index.html "threading-lib") package first, e.g. `raco pkg install threading`. That actually was one of the reasons why I hadn't used it for a long time, just tried to avoid third-party packages as much as possible.

First of all, the package documentation is extremely well written, so please just [check it out](https://docs.racket-lang.org/threading/introduction.html "docs"). Here is one more quick example to give you a taste of it. In functional programming, we often push our data through a pipeline of functions, transforming it step-by-step to get the desired result. Here is one of the countless ways to transform a string into its acronym; comments show the state of the data at every pipeline stage going left to right:

```Racket
(require threading)

(define (acronym str)
  (~> str                           ; "Complementary metal-oxide semiconductor"
      string-upcase                 ; "COMPLEMENTARY METAL-OXIDE SEMICONDUCTOR"
      (string-split #px"[\\s_-]+")  ; '("COMPLEMENTARY" "METAL" "OXIDE" "SEMICONDUCTOR")
      (map (curryr string-ref 0) _) ; '(#\C #\M #\O #\S)
      list->string))                ; "CMOS"
```

Now let's take a step back and talk about some tools Racket gives us out of the box to combine functions together. The function provided in TLDR uses some [arithmetic operations](https://www.youtube.com/watch?v=uESjbE1jUxo "explained") to calculate the number of digits in a given number. We could use `compose` or `compose1` functions here to combine those building blocks together. Let's note some of the differences between those two approaches:

```Racket
(define digits.v1 (compose1 add1 exact-floor (curryr log 10)))
(define digits.v2 (λ~> (log 10) exact-floor add1))
```

Not only does `compose` apply those functions in reverse order, but we also had to use `curryr` to provide an adapter for the `log` function. The threading macro will always try to plug the value into the first parameter hole but also provides that magic `_` that allows you to specify where to put it, as we did in the line `(map (curryr string-ref 0) _) ; '(#\C #\M #\O #\S)` of our acronym example. `λ~>` or `lambda~>` here demonstrates how we can obtain a function that represents the pipeline to pass it into another higher-order function, e.g. `map` or `filter`.

A lot of those functions could save you from writing yet another lambda or currying a function. I want to finish this text by mentioning a couple of [functions](https://docs.racket-lang.org/reference/procedures.html#%28part._.Additional_.Higher-.Order_.Functions%29) provided by `racket/function` and `racket` but not `racket/base`. Let's give an honorable mention to the `identity` function, which is surprisingly useful for a function that just returns back whatever you passed in. Here is a simplified example from SICP - functions `sum-cubes` and `sum-integers` are implemented in terms of a higher-order `sum` function:

```Racket
(define (sum term a b)
  (if (> a b)
      0
      (+ (term a)
         (sum term (add1 a) b))))

(define (sum-cubes a b)
  (sum (curryr expt 3) a b))

(define (sum-integers a b)
  (sum identity a b))
```

As a bonus, `identity` could be used to filter/count all the non-#f elements of the list:

```Racket
(filter identity '(#t 'foo #f #f 42 #f #f "bar")) ; '(#t 'foo 42 "bar")
```

Sometimes we need to combine predicates in place during filtering - `conjoin`, `disjoin`, and `negate` are for the rescue. We can express "give me a function that checks if a given value satisfies ALL of those predicates" as `(conjoin pred1? pred2? pred3?)`. Similarly, you can ask if a value satisfies AT LEAST one of the given predicates with `(disjoin pred1? pred2? pred3?)`. In this example, we have a two-dimensional grid map like in a classic rogue, `#\.` is a walkable floor tile while `#\#` is a wall that blocks the way. Combining `in-bounds?` and `is-wall?` together with `conjoin` and `negate` allows us to get a list of all available steps from a given tile.

```Racket
(define N 3)

;; dungeon map with size N x N
(define A (vector #( #\# #\# #\# )
                  #( #\. #\. #\. )
                  #( #\# #\# #\# )))

;; access cell at (cons row col)
(define (at posn)
  (match-define (cons i j) posn)
  (vector-ref (vector-ref A i) j))

(define (in-bounds? posn)
  (match-define (cons i j) posn)
  (and (>= i 0) (>= j 0) (< i N) (< j N)))

(define (is-wall? posn)
  (char=? (at posn) #\#))

;; list of steps from current cell
;; one step in each direction
(define (wasd from)
  (match-define (cons i j) from)
  (list (cons (sub1 i) j)       ; up
        (cons (add1 i) j)       ; down
        (cons i (sub1 j))       ; left
        (cons i (add1 j))))     ; right

;; list all the available steps from cell (1 . 0)
;; you can't move through the walls or leave the map
(filter (conjoin in-bounds? (negate is-wall?)) (wasd (cons 1 0)))
; with the same result as
(~> (cons 1 0)
    wasd
    (filter in-bounds? _)
    (filter-not is-wall? _))
```

