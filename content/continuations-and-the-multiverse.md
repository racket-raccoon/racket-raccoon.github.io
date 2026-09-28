+++
title = "Continuations and the Multiverse"
date = 2026-07-26
draft = true
[taxonomies]
tags = ["racket", "continuations", "control flow"]
+++

I'm trying to be less of a larper and learn more about the basic building blocks and core concepts of the Lisp/Scheme world and functional programming in general.
This time I'm trying to teach myself a thing or two about **continuations**.

<!-- more -->

## Part One: `prompt` and `control`

My brain is completely cooked by years of imperative programming, and my attention span is so bad that I can hardly learn anything from definitions like the one in [The Racket Guide](https://docs.racket-lang.org/guide/conts.html):

> A continuation is a value that encapsulates a piece of an expression’s evaluation context.

So I'm going to explain it in the order that finally made it click for me.

My somewhat controversial take is that maybe it's better to start discussing continuations with `prompt` and `control`, which are available through `(require racket/control)`.
One of the first questions that comes up when someone tells you that "a continuation captures the rest of the computation" is what *rest* means and how large this *rest* is.
I feel like having a `prompt` form that creates a visible boundary around a region of code helps a lot.

I only need one example for this, but it is a slightly silly one: a tiny D&D-inspired battler.
Hopefully, this code will be easy enough for anyone to understand.

```Racket
#lang racket

;; both the player and her enemies are game entities
(struct entity (name hp attack) #:transparent)

;; factory function for entity creation
(define (create-entity type)
  (case type
    [(player) (entity "player" 15 5)]
    [(orc) (entity "orc" 15 3)]
    [(troll) (entity "troll" 35 4)]
    [else (error "unknown entity")]))

;; roll a die!
(define (d8) (random 1 9))

;; an attack deals base damage + a d8 roll
(define (attack who dmg)
  (define new-hp (- (entity-hp who) dmg (d8)))
  (struct-copy entity who [hp new-hp]))

;; an entity is considered dead if it has no HP left
(define (dead? who)
  (<= (entity-hp who) 0))

;; control detailed battle logging
(define verbose? (make-parameter #f))

;; the player and an enemy take turns attacking each
;; other until one of them drops dead
(define (fight player enemy [turn #t])
  (when (verbose?)
    (displayln player)
    (displayln enemy)
    (displayln "===================="))

  (cond
    [(dead? player) player] ; defeat
    [(dead? enemy) player] ; victory
    [turn (fight player (attack enemy (entity-attack player)) (not turn))]
    [else (fight (attack player (entity-attack enemy)) enemy (not turn))]))
```

The client code for the battle simulation might then look like this:

```Racket
;; this is our timeline, the version of the universe our hero ends up in
(random-seed 811)

;; our humble hero has to fight an orc and then a troll!
(define player (create-entity 'player))
(parameterize ([verbose? #t])
  (set! player (fight player (create-entity 'orc)))
  (set! player (fight player (create-entity 'troll))))
(if (dead? player) 'defeat 'victory)
```

`random-seed` deterministically seeds the current pseudo-random number generator, so the same seed produces the same sequence of die rolls and can serve as a repeatable timeline.

Unfortunately, our hero finds herself in a grim situation.
She manages to beat the orc, but the dungeon master rolls the dice again, and she is defeated!

```
...
====================
#(struct:entity player 6 5)
#(struct:entity troll 29 4)
====================
#(struct:entity player 0 5)
#(struct:entity troll 29 4)
====================
'defeat
```

Well, our chances were slim anyway.
Look how huge this troll is.
35 hit points?
You've got to be kidding me.
Did we even have a chance here?
*([Earth-811](https://marvel.fandom.com/wiki/Earth-811)? Damn you, Sentinels!)*

What if we could cheat a little bit?
What if we could stop time and peek into all the other timelines?
After all, Doctor Strange examined 14,000,605 possible futures and saw only one in which the heroes won.
Let's consider this version of the simulation instead:

```Racket
(require racket/control)

;; we don't quite have Doctor Strange's patience; 10,000 timelines is enough
(define (all-winning-timelines k [n 10000])
  (for/list ([i (in-range n)]
             #:do [(define died? (eq? (k i) 'defeat))]
             #:unless died?)
    i))

(prompt
 (random-seed
  (control k
           (define timelines (all-winning-timelines k))
           (displayln timelines)
           (cond
             [(empty? timelines) 'escaped]
             [else
              (parameterize ([verbose? #t])
                (k (first timelines)))])))

 (define player (create-entity 'player))
 (set! player (fight player (create-entity 'orc)))
 (set! player (fight player (create-entity 'troll)))
 (if (dead? player) 'defeat 'victory))
```

Here is a useful way to think about this `prompt` block.
First, let's split its contents into two parts: the `control` expression itself and everything else surrounding it.
The next trick is to imagine that we have a **"hole"** in place of that `control` expression.
It would look something like this:

```Racket
(random-seed 🕳️)
(define player (create-entity 'player))
(set! player (fight player (create-entity 'orc)))
(set! player (fight player (create-entity 'troll)))
(if (dead? player) 'defeat 'victory)
```

Here, the code around the hole is the "rest of the computation" we mentioned before.
It starts at the hole and ends at the nearest enclosing `prompt`.
Can we save it and use it later?
At this point, your intuition might kick in and tell you that it looks a bit like a function.
In this example, we can think of the **captured continuation** as a function that takes one argument: an integer seed.
Calling it plugs that seed into the hole and runs the captured context; because the continuation is composable, the result then returns to the caller.
Let's call the function `k` (as in Kontinuation, obviously) and put it into a `let`.
The body of `control`, which performs our timeline search, becomes the `let` body:

```Racket
(let ([k
       (λ (seed)
         (random-seed seed)
         (define player (create-entity 'player))
         (set! player (fight player (create-entity 'orc)))
         (set! player (fight player (create-entity 'troll)))
         (if (dead? player) 'defeat 'victory))])
  (define timelines (all-winning-timelines k))
  (displayln timelines)
  (cond
    [(empty? timelines) 'escaped]
    [else
     (parameterize ([verbose? #t])
       (k (first timelines)))]))
```

This reads much better to my eyes.

> **_NOTE_** The rewrite above is only a useful mental model, not code that Racket literally creates at runtime.
More precisely, the reduction retains the outer `prompt`; omitting it makes no difference in this example.
Likewise, the captured `k` is a composable continuation, not *really* a simple lambda, but this abstraction is useful enough to reason about the code.

Let's run the code.
It seems like we managed to snatch victory from the jaws of defeat after all.
Only two of the first 10,000 timelines end in victory: `(1788 3003)`.
We just barely survive with one hit point!

```
...
====================
#(struct:entity player 1 5)
#(struct:entity troll 11 4)
====================
#(struct:entity player 1 5)
#(struct:entity troll 0 4)
====================
'victory
```

We replayed our continuation for all 10,000 seed values and collected the winning timelines.
If we discovered that the situation was hopeless, we would still have the option to retreat and regroup.

There are a few things I left out for magical effect, though:

* First of all, time doesn't magically stop as I might have led you to believe.
If the captured computation performs side effects, those effects are repeated for every timeline we inspect and once more when we replay the winning timeline.

* This is why we create the player and enemies after `control`, inside the captured computation: every replay creates fresh entities.
The `entity` structure is also immutable, and `attack` uses `struct-copy` to return a new entity instead of modifying an existing one.
This makes accidental sharing between timelines less dangerous.
I still used some ugly local mutation with `set!` because I thought it would better highlight how the continuation can capture a whole sequence of expressions.

* One last important point is that continuation capture happens dynamically.
We should only imagine making the mental `let` rewrite when evaluation reaches the `control` form, because the active evaluation context determines what gets captured.
This can be especially confusing if you have `control` hidden inside a loop or a conditional.

To reinforce that last point, I want to change the logic a bit.
Instead of collecting all the winning timelines, let's grab the first one that leads to victory and stop early.

```Racket
(define (first-winning-timeline k [n 10000])
  (for/first ([i (in-range n)]
              #:do [(define died? (eq? (k i) 'defeat))]
              #:unless died?)
    i))

(prompt
 (random-seed
  (control k
           (define timeline (first-winning-timeline k))
           (if timeline
               (parameterize ([verbose? #t])
                 (k timeline))
               'escaped)))

 (define player (create-entity 'player))
 (set! player (fight player (create-entity 'orc)))
 (set! player (fight player (create-entity 'troll)))
 (if (dead? player) 'defeat 'victory))
```

`all-winning-timelines` uses `for/list`, while `first-winning-timeline` uses `for/first`.
`for/first` returns `#f` if nothing is found and short-circuits the loop, giving us an early return.
Funnily enough, we now know enough to achieve the same behavior with `prompt` and `control`.
Take a look at this version:

```Racket
(define (first-winning-timeline k [n 10000])
  (prompt
   (for ([i (in-range n)])
     (unless (eq? (k i) 'defeat)
       (control k1 i)))
   #f))
```

If `control` is never reached because every timeline ends in `'defeat`, evaluation simply falls through, and the final `#f` becomes the result of the `prompt`.
However, the first time we reach that `control` form, it captures the rest of the loop as `k1`.
If we perform the rewrite mentally, we can see that `k1` is ignored and the expression returns `i` instead.
Very roughly, it looks like this (if that makes any sense):

```Racket
(let ([k1 (λ (v) ...)]) i)
```

> **_NOTE_** One reason continuations are so tricky to learn is that it is hard to find a genuinely practical example. You rarely need continuations in day-to-day programming, and this is definitely not a best-practice example of how to return early in Racket.


## Part Two: `call-with-composable-continuation`

Now that we have an intuitive understanding of what `prompt` is and how `prompt` and `control` work together, we can take the next step.
You may have noticed that we had to explicitly use `(require racket/control)` earlier.
That suggests that the operators we used are higher-level convenience forms implemented in terms of some lower-level primitives.
Let's see those primitives in action.
We'll use the same example to explore `call-with-continuation-prompt` and `call-with-composable-continuation` and see how they differ from the `prompt` and `control` we already know and love.

```Racket
(call-with-continuation-prompt
 (λ ()
   (random-seed
    (call-with-composable-continuation
     (λ (k)
       (define winning (first-winning-timeline k))
       (displayln winning)
       (if winning winning 0))))

   (define player (create-entity 'player))
   (set! player (fight player (create-entity 'orc)))
   (set! player (fight player (create-entity 'troll)))
   (if (dead? player) 'defeat 'victory)))
```

This is our first naive attempt.
The structure is very close to what we've seen before, but the API has changed a bit.
Both procedures now take another procedure as their first argument, so we have to wrap our logic in lambdas before passing it in.
Here, `k` itself is literally the same kind of composable continuation that `control` exposes!
This block might look the same at first glance, but if you look closely, you'll see the important difference.
When we used `prompt` and `control`, we had to invoke the continuation manually and pass it the winning seed before the captured computation could proceed.
In other words, the `control` form captured and suspended the following computation; the simulation could proceed only when we invoked `k` explicitly.
Here, however, the value returned by our lambda (the winning seed) is plugged directly into the hole, after which the program continues as if nothing happened.
As a consequence, we seemingly lose the ability to escape the battle if none of the seeds we tested wins, so we simply give up and return zero.
Luckily, there is a way to trigger this abort manually.

```Racket
(call-with-continuation-prompt
 (λ ()
   (random-seed
    (call-with-composable-continuation
     (λ (k)
       (define winning (first-winning-timeline k))
       (unless winning
         (abort-current-continuation
          (default-continuation-prompt-tag)
          (λ () 'escaped)))
       winning)))

   (define player (create-entity 'player))
   (set! player (fight player (create-entity 'orc)))
   (set! player (fight player (create-entity 'troll)))
   (if (dead? player) 'defeat 'victory)))
```

`abort-current-continuation` discards the current continuation up to the nearest prompt, and the prompt's handler executes the callback we provided instead.
This is the first time we have encountered the concept of a *continuation prompt tag*, so let's quickly address it as well.
`call-with-continuation-prompt` takes an optional `prompt-tag` argument.
This tag must be obtained from either `default-continuation-prompt-tag` or `make-continuation-prompt-tag`.
When you capture a continuation with `call-with-composable-continuation`, you also select a prompt tag, either explicitly or implicitly.
The capture extends only to the nearest prompt carrying that specific tag.
So when I said that `abort-current-continuation` discards the current continuation up to the nearest prompt, I really meant the nearest prompt carrying the same tag.
I hope that makes sense.
The following example works exactly the same way, but it uses an explicit custom tag instead of the default one.

```Racket
(define checkpoint (make-continuation-prompt-tag 'checkpoint))
(call-with-continuation-prompt
 (λ ()
   (random-seed
    (call-with-composable-continuation
     (λ (k)
       (define winning (first-winning-timeline k))
       (unless winning
         (abort-current-continuation
          checkpoint
          (λ () 'escaped)))
       winning)
     checkpoint))

   (define player (create-entity 'player))
   (set! player (fight player (create-entity 'orc)))
   (set! player (fight player (create-entity 'troll)))
   (if (dead? player) 'defeat 'victory))
 checkpoint)
```

I feel like that huge wall of text and all those silly examples should finally make [this passage from the Racket Guide](https://docs.racket-lang.org/guide/conts.html) click.

> Other Scheme systems traditionally support a single prompt at the program start, instead of allowing new prompts via call-with-continuation-prompt.

> Continuations as in Racket are sometimes called delimited continuations, since a program can introduce new delimiting prompts, and continuations as captured by call-with-composable-continuation are sometimes called composable continuations, because they do not have a built-in abort.

## Part Three: `call/cc`
