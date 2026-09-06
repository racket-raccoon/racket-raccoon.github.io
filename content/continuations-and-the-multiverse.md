+++
title = "Continuations and the Multiverse"
date = 2026-07-26
[taxonomies]
tags = ["racket", "continuations", "control flow"]
+++

I'm trying to be less of a larper and learn a bit more about some basic building blocks and core concepts from the Lisp/Scheme world, as well as functional programming in general.
This time I'm trying to teach myself a thing or two about **continuations**.

<!-- more -->

My brain is completely cooked by years of imperative programming, and my attention span is so bad that I can hardly learn anything from definitions like the one in [The Racket Guide](https://docs.racket-lang.org/guide/conts.html):

> A continuation is a value that encapsulates a piece of an expression’s evaluation context.

So I'm going to explain it in the order that finally made it click for me.

My somewhat controversial take is that maybe it's better to start discussing continuations with `prompt`/`control`, which are available through `(require racket/control)`.
One of the first questions that comes up when someone tells you that "a continuation captures the rest of the computation" is what *rest* means and how large this *rest* is.
I feel like having a `prompt` form that creates this visible boundary around a region of code helps a lot.

I only need one example for this, but it is a slightly silly one: a tiny D&D-inspired battler.
This code should be easy enough for anyone to understand.

```Racket
#lang racket

;; both player and her enemies are game entities
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

;; print detailed battle log
(define verbose? (make-parameter #f))

;; player and enemy take turns attacking each
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

;; this is our timeline, the version of the universe our hero ends up in
(random-seed 811)

;; our humble hero has to fight an orc and then a troll!
(define player (create-entity 'player))
(parameterize ([verbose? #t])
  (set! player (fight player (create-entity 'orc)))
  (set! player (fight player (create-entity 'troll))))
(if (dead? player) 'defeat 'victory)
```

`random-seed` initializes Racket's pseudo-random number generator deterministically, so the same seed produces the same sequence of die rolls and can stand for a repeatable timeline.

Unfortunately, it looks like our hero finds herself in a grim situation.
We managed to beat the orc, but the dungeon master rolls the dice again and our hero is defeated!

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

Well, the chance was slim anyway.
Look how huge this troll is.
35 hit points?
You've got to be kidding me.
Did we even have a chance here?
*(Earth-811 was doomed.)*
*(Damn you, Sentinels!)*

What if we could cheat?
What if we could stop time and peek into all the other timelines?
After all, Doctor Strange had to examine 14,000,605 possible futures to find the one successful outcome.
Let's consider this code:

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

Here is a useful intuition for reading this `prompt` block.
First, let's split its contents into two parts: the `control` expression itself and the computation surrounding it.
The next trick is to imagine that we have a **"hole"** in place of the whole `control` expression.
It would look something like this:

```Racket
(random-seed 🕳️)
(define player (create-entity 'player))
(set! player (fight player (create-entity 'orc)))
(set! player (fight player (create-entity 'troll)))
(if (dead? player) 'defeat 'victory)
```

In this program, this is the "rest of the computation" we mentioned before.
It starts at the hole and ends at the nearest enclosing `prompt`.
Is there a way to save it somehow?
Here your intuition might kick in and say that it looks a bit like a function.
In this example, we can think of the **captured continuation** as a function of one parameter.
Its argument fills the hole—in our case with an integer seed—and calling it runs everything from that hole to the surrounding `prompt`.
Let's call the function `k`, as in Kontinuation, obviously, and put it into a `let`.
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

Hopefully now it reads better.
For this example, the rewrite is a useful mental model, not new code that Racket literally creates at runtime.
The real `k` is a composable continuation, but it behaves like the function above in the ways we use here.
So we actually managed to snatch the victory from the jaws of defeat.
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

So we replayed our continuation over and over again, feeding in different seed values until we found a winning seed.
If the game was rigged or we knew that this was hopeless (or we just didn't have the time to go through more than, say, 1,500 timelines), we would know that it was wise to retreat and regroup.

There are some things I left out for magical effect and have to disclose, though.

* First of all, time doesn't magically stop as I might have led you to believe.
If the captured computation performs side effects, they happen once for every timeline we inspect, and once more when we replay the winning timeline.

* This is why we create the player and enemies after `control`, inside the captured computation: every replay creates fresh entities.
  The `entity` structure is also immutable, and `attack` uses `struct-copy` instead of modifying an existing entity.
  This makes accidental sharing between timelines less dangerous.
  I still used some ugly local mutation with `set!` because I thought it highlighted how the continuation can capture a sequence of expressions.

* The last, quite important thing is that continuation capture happens dynamically.
  We should only imagine making the mental `let` rewrite when evaluation reaches the `control` form, because the active evaluation context determines what gets captured.
  This can be especially confusing if you have `control` hidden inside a loop or a conditional.
