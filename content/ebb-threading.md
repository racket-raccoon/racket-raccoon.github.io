+++
title = "Same Threading Syntax but Less Allocation"
date = 2026-05-24
[taxonomies]
tags = ["threading", "ebb", "benchmarks"]
+++

This is a small braindump on why you might want to use `ebb/threading` as a drop-in replacement for `threading-lib`.
Same familiar `~>` shape, but with `ebb/forms` it can avoid some intermediate list allocation.

<!-- more -->

Have I already told you how much I admire the Racket community?
It's so friendly, full of ridiculously smart people, and every now and then someone casually demos a thing that makes me feel like I need to go read three papers and then lie down for a bit.

Trying to be a tiny bit less of a lurker, I started attending monthly Racket virtual meetups.
On one of those meetups, [Dominik](https://racket.discourse.group/u/dominik.pantucek/summary) gave a great demo of an optimization in [Qi](https://docs.racket-lang.org/qi/index.html) that immediately grabbed my attention.

The problem itself is very easy to understand.
If we write a pipeline of list transformations, Racket normally has to build intermediate lists between those steps:

```Racket
(define (filter-map xs)
  (map sqr (filter odd? xs)))
```

First we traverse `xs` to produce the result of `filter`, then we traverse that intermediate list to produce the result of `map`.
Qi can optimize this kind of pipeline and avoid constructing those intermediate lists.
The docs call this [stream fusion or deforestation](https://docs.racket-lang.org/qi/When_Should_I_Use_Qi_.html#%28part._.Don_t_.Stop_.Me_.Now%29), which is a very cool name for "please stop allocating all this stuff I don't actually need".

I was obviously very impressed.

So I asked if something similar could work with the [threading syntax I already use](/threading/) from Alexis King's threading [library](https://docs.racket-lang.org/threading/index.html).
I just threw the idea out there because it seemed like an amazing feature: keep the familiar `~>` syntax, but make it possible to get Qi-style deforestation as a drop-in replacement.

Of course, "drop-in replacement" is the kind of phrase users say right before maintainers start quietly suffering, this turned out to be "harder than it sounds"(tm).

However, on the next meetup Dominik showed his new `ebb` package doing exactly that!

I mean.. you ask a random speculative question, and one month later there is an actual package exploring the idea, how cool is that!
This is exactly the kind of thing that makes me want to keep showing up to those meetups.

Here is the threading style I already use:

```Racket
(require threading)

(~> xs
    (map add1 _)
    (filter odd? _)
    (map (lambda (x) (* x x)) _)
    (foldl + 0 _))
```

With `ebb` package the same exact code should work if you use `(require ebb/threading)` instead of plain `(require threading)`.
This by itself is already nice because existing code can keep the same general shape but the more interesting part starts when we also require `ebb/forms`.
Now `map`, `filter`, and `foldl` are no longer just arbitrary Racket function calls placed inside a threading macro.
They are forms `ebb` can recognize, which gives it enough information to fuse the pipeline:

- `ebb/threading` gives us familiar `~>` syntax
- `ebb/forms` gives `ebb` optimizable pipeline steps

So just to make a point we can turn this example into a tiny benchmark, let's start with a big list, `map`, `filter`, `map` again, then consume everything with `foldl`.
On my machine, with `n = 1000000` and five rounds, one run looked like this:

```text
threading-lib: cpu=316ms real=316ms gc=141ms result=166666666666500000
ebb/threading: cpu=285ms real=285ms gc=128ms result=166666666666500000
ebb/forms: cpu=34ms real=34ms gc=0ms result=166666666666500000
```

Not a real benchmark suite, obviously, but still a very satisfying sanity check.
The plain `ebb` threading version is in the same general area as `threading-lib`.
The `ebb` forms version is much faster and spends basically no time in GC, which is exactly what we would hope to see if intermediate lists are not being allocated.
(this approach gets us into the same kind of territory as writing the traversal manually with `for/fold` and clauses like `#:when`, `#:unless`, and `#:do`)

> **_NOTE_** There is one thing that is probably obvious for anyone who used Qi before but came as a surprise to me.
When you make use of `ebb/forms` inside an `ebb` pipeline, `map`, `filter`, and `foldl` are not normal Racket list functions anymore, they are stream steps i.e. transformers and a consumer.
Apparently those stream steps actually don't need `_` placeholders because the list/stream argument is already implicit.

First implication of that is that you can get rid of those placeholders altogether, in fact `ebb` does just that under the hood and treats it as

```Racket
(~> xs
    (map add1)
    (filter odd?)
    (map (lambda (x) (* x x)))
    (foldl + 0))
```

Second implication is that something like this would work using plain `threading-lib` but won't work with `ebb/forms`

```Racket
(~> add1
    (map _ (list 1 2 3)))
;; => '(2 3 4)
```
