+++
title = "Leetcode #844: Backspace String Compare"
date = 2025-05-23
[taxonomies]
tags = ["leetcode", "strings", "stack"]
+++

I'm gonna try to trick my brain into doing some leetcode challenges over the course of this year, so let's start with something simple like ["Backspace String Compare"](https://leetcode.com/problems/backspace-string-compare/description/).

<!-- more -->

First thing that comes to mind is to build two normalised strings separately and then just compare those with `equal?`.
Every non-backspace character is pushed into a stack-like buffer and for every `#` we pop one of those out - clean and simple.

```Racket
(define (build str)
  (let loop ([lst (string->list str)] [result '()])
    (cond
      [(null? lst) result]
      [(char=? (car lst) #\#)
       (if (null? result)
           (loop (cdr lst) result)
           (loop (cdr lst) (cdr result)))]
      [else (loop (cdr lst) (cons (car lst) result))])))

(define (backspace-compare s t)
  (equal? (build s) (build t)))
```

Unfortunately, we can't call it a day yet; now we have to talk about complexity, and I always have a hard time analysing those.
Time-wise, we have to iterate over both strings once to build their normalised representation.
Meaning, if the size of `s` is `A` and the size of `t` is `B`, then we'll have to go through `A + B` elements just to build those.
`equal?` is basically a loop that is going to iterate through the resulting strings: worst-case scenario would be if `(length (build s)) == A` and `(length (build t)) == B`, meaning that iteration goes up to `(min A B)`.
Probably worth noting that we could speed up this check by keeping track of the stack buffer size e.g. increment on push and decrement on pop.
This way we could compare their sizes and fail fast if they aren't the same; doesn't affect the worst-case scenario so doesn't change our complexity analysis.

That means that time complexity would be either `O(2A+B)` or `O(A+2B)`, smart people would just wave their hands and say that it's the same as `O(A+B)` as `2A` is linearly proportional to `A` so we don't really care about constant multipliers.
We can even simply say that it's `O(N)` implying `N = A + B` as a good generalisation.

Obviously, this challenge is all about space complexity so let's estimate it: we allocate two additional lists to store normalised strings.
Making the same point about the worst-case scenario and string sizes, it's clear that space complexity is `O(A+B)` or `O(N)` and it can be improved upon.

We would like to avoid creating those buffers altogether, pursuing space complexity of `O(1)`, and apparently the trick is to iterate the strings in reverse.
Going from right to left, we need two pointers to iterate both strings in parallel.
I'm gonna use those little closures to achieve that:

```Racket
(define (next str)
  (define it (sub1 (string-length str)))
  (lambda ()
    (let loop ([skips 0])
      (if (negative? it)
        'end
        (let ([ch (string-ref str it)])
          (set! it (sub1 it))
          (cond
            [(char=? ch #\#) (loop (add1 skips))]
            [(zero? skips) ch]
            [else (loop (sub1 skips))]))))))

(define (backspace-compare s t)
  (define next-s (next s))
  (define next-t (next t))
  (let loop []
    (match (list (next-s) (next-t))
      [(list 'end 'end) #t]
      [(or (list 'end _) (list _ 'end)) #f]
      [(list a a) (loop)]
      [else #f])))
```

Every time we meet `#` we have to increment the `skips` count and we can only return the next character if `skips` count is zero.
Not only do we avoid allocating unnecessary buffers, but we also fail fast as iteration stops as soon as we find a difference between those sequences.

I foolishly used `null` instead of `'end` when I wrote it first and fell head on into a trap related to the way `match` works.
Matching a two-element list against `(list null null)` apparently might not perform as you would expect and both Claude and ChatGPT initially don't seem to be aware of that either.
You should think about `null` in Racket as a regular identifier like `(define null '())` rather than some magic *literal*.
Here is how ChatGPT describes it if you turn on the "deep research" mode for the request "evaluate this racket expression `(match (list #\p #\p) [(list null null) #t] [else #f])`":
> Since the first pattern matches (list #\p #\p), the overall expression returns #t. In summary, (match (list #\p #\p) [(list null null) #t] [else #f]) evaluates to #t because null in the pattern is a wildcard variable (not the empty-list), and the two characters are equal so the match succeeds

Very last point I wanted to make is related to functions like `next` that have an internal state and return the next value in a sequence every time you call them.
There is a nice convenience function `in-producer` that creates a wrapper around such functions and allows you to iterate through those sequences with a regular `for` loop.
Here is an example of how you could write an adapter for the `next` function to make it return a list straight away - very similar to what the `build` function did for us:
```Racket
(define (build str)
  (for/list ([ch (in-producer (next str) 'end)]) ch))
```

