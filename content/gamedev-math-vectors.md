+++
title = "Math for Gamedev 101: Vectors"
date = 2026-01-25
[taxonomies]
tags = ["math", "gamedev"]
+++

Recently I bought [Simon's](https://simondev.io/about) course on [Math for Game Developers](https://simondev.io/lessons/math) and as I finally found some time to go through the first couple of  chapters I would like to share my lecture notes here.
I used Racket for mathematical functions and comments for plain English, a lot of stuff is also borrowed from the wiki page on [Euclidean vectors](https://en.wikipedia.org/wiki/Euclidean_vector).

<!-- more -->

```Racket
#lang racket

(require rackunit)

;; Euclidean space is the mathematical formalization of the "flat"
;; geometry we learned in school where for example:
;; * parallel lines never meet
;; * angles in triangle sum to 180°
;; * Pythagorean theorem holds.. etc

;; Cartesian coordinate system is the standard way to represent
;; Euclidean space numerically using perpendicular axes x y z
;; to define positions in space

;; Simple "number line" is way to represent a one-dimensional Euclidean space

;; Point / Position / Position Vector in gamedev all denote the same idea:
;; representation of a specific location in the virtual space

;; Vector = Magnitude (length or size) + Direction
;; Points are written with just letters; Vectors have a little arrow on top
;; Numerically they are represented exactly the same but mean different things

(test-case "Point vs Vector"
           ;; thermometer example: one-dimensional space
           (define current 5)  ; current temperature (point, initial state)
           (define forecast 2) ; "2 degrees warmer" change (vector)
           ;;; point + vector = point (also point - vector = point)
           (check-equal? (+ current forecast) 7)
           ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
           (define A 10) ; temperature now (point)
           (define B 15) ; temperature later (point)
           ;;; point - point = vector
           (check-equal? (- B A) 5)
           ;;; (point + point) doesn't make much sense
           )

;; Two vectors are considered equal
;; if they have the same magnitude and direction = if their coordinates are equal
(test-case "Equality"
           (define a '(1 2 3))
           (define b '(1 2 3))
           (check-equal? a b))

;; contracts from now on should help to distinguish
;; points/vectors (numlist?) from scalars (number?)
;; those are more strict contracts for two- and three-dimensional vectors
(define numlist? (listof number?))
(define vector2/c (list/c number? number?))
(define vector3/c (list/c number? number? number?))

;; To add the vectors we add the corresponding components from each vector
(define/contract (add a b)
  (-> numlist? numlist? numlist?)
  (map + a b))

(define/contract (sub a b)
  (-> numlist? numlist? numlist?)
  (map - a b))

(test-case "Addition and subtraction"
           (define a '(1 2 3))
           (define b '(1 2 3))
           (check-equal? (add a b) '(2 4 6))
           (check-equal? (sub a b) '(0 0 0)))

(define (random-vector3) (list (random 10) (random 10) (random 10)))

;; (a + b) + c = a + (b + c) 
(test-case "Associativity"
           (define a (random-vector3))
           (define b (random-vector3))
           (define c (random-vector3))
           (check-equal? (add (add a b) c) (add a (add b c))))

;; a + b = b + a 
(test-case "Commutativity"
           (define a (random-vector3))
           (define b (random-vector3))
           (check-equal? (add a b) (add b a)))

;; Vector subtraction is neither associative nor commutative!

;; Vector length with Pythagorean theorem
;; Vector's v magnitude is denoted as ‖v‖
(define/contract (magnitude v)
  (-> numlist? number?)
  (sqrt (foldl + 0 (map (curryr expt 2) v))))

;; Here 'from' and 'to' are not vectors but points!
(define/contract (distance from to)
  (-> numlist? numlist? number?)
  (magnitude (sub to from)))

(test-case "Distance"
           (check-equal? (magnitude '(2 0)) 2)
           (check-equal? (magnitude '(0 3)) 3)
           (check-equal? (magnitude '(3 4)) 5)
           
           (check-equal? (distance '(3 4) '(0 0)) 5)

           (check-equal? (magnitude '(3 4 12)) 13))

;; Scalar multiplication of vector V and scalar S written as SV
(define/contract (scale v s)
  (-> numlist? number? numlist?)
  (map (curry * s) v))

;; Hadamard Product, written as a ∘ b
;; just an element-wise multiplication
;; could be used for non-uniform scale!
(define/contract (hadamard a b)
  (-> numlist? numlist? numlist?)
  (map * a b))

;; Both scale functions (scalar and component-wise) are
;; Commutative and associative - order doesn't matter
(test-case "Vector scaling"
           ;; multiplying with a scalar changes magnitude, same direction
           (check-equal? (scale '(3 4) 4) '(12 16))
           ;; negative scalar reverses direction 
           (check-equal? (scale '(1 2) -1.5) '(-1.5 -3.0))

           ;; RGB color can be represented as three-dimensional vector!
           ;; Hadamard Product could be used to tint color by other color
           (define/contract (tint-red color)
             (-> vector3/c vector3/c)
             (define red-mask '(1 0 0))
             (hadamard color red-mask))
           
           (check-equal? (scale '(1 2 3) 5) '(5 10 15))

           (check-equal? (hadamard '(3 4) '(1 0)) '(3 0))
           (check-equal? (hadamard '(5 6) '(5 5)) '(25 30))
           (check-equal? (hadamard '(1 2 3) '(1 -1 1)) '(1 -2 3)))

;; Unit vector is any vector with a length of one, indicates direction
;; denoted by a lowercase letter with a circumflex or "hat" ^
;; Vector can be normalized or divided by its length to create a unit vector
(define/contract (normalize v)
  (-> numlist? numlist?)
  (scale v (/ 1 (magnitude v))))

(test-case "Normalized Vectors"
           (check-equal? (normalize '(3 4)) '(3/5 4/5))
           (check-equal? (normalize '(0.75 0.25)) (normalize '(3 1))))

;; Dot Product, Scalar projection, Scalar product
;; also ‖a‖ ‖b‖ cosθ where θ is an angle between a and b
;; if both normalized gives us a way to say how similar vectors are
(define/contract (dot a b)
  (-> numlist? numlist? number?)
  (foldl + 0 (hadamard a b)))

(test-case "Dot Product"
           ;; Orthogonal! Perpendicular to each other, θ = pi/2, cosθ = 0
           (check-equal? (dot '(1 0) '(0 1)) 0)
           (check-equal? (dot '(1 0) '(0 -1)) 0)
           ;; Opposite direction
           (check-equal? (dot '(1 0) '(-1 0)) -1)
           ;; Same direction
           (check-equal? (dot '(1 0) '(1 0)) 1)

           ;; dot product of a vector with itself is equal to
           ;; the square of the magnitude
           (define a (random-vector3))
           (check-= (dot a a) (expt (magnitude a) 2) 0.1))

;; Vector (a × b) is perpendicular to both a and b and the plane formed by them
;; Its magnitude represents the area of the parallelogram spanned by a and b
(define/contract (cross a b)
  (-> vector3/c vector3/c vector3/c)
  (match-let ([(list ax ay az) a]
              [(list bx by bz) b])
    (list (- (* ay bz) (* az by))
          (- (* az bx) (* ax bz))
          (- (* ax by) (* ay bx)))))

(test-case "Cross product"
           (check-equal? (cross '(1 0 0) '(0 1 0)) '(0 0 1))
           ;; Order matters! Changing order reverses the direction
           (check-equal? (cross '(0 1 0) '(1 0 0)) '(0 0 -1))
           ;; For parallel vectors a x b results in the zero vector
           (check-equal? (cross '(1 1 1) '(2 2 2)) '(0 0 0))
           ;; If either a or b is the zero vector, a × b is also the zero vector
           (check-equal? (cross '(1 2 3) '(0 0 0)) '(0 0 0))
           (check-equal? (cross '(0 0 0) '(1 2 3)) '(0 0 0)))
```

