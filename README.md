# Call Stack Elimination

## Objectives:
- Explore continuations
- Write functions in Continuation-Passing Style (CPS)
- Eliminate the need for a call stack

## Motivation

Consider the factorial function written with standard recursion in Python:
```python
def fact(n):
    if n == 1:
        return 1
    else:
        return n * fact(n - 1)
```
[code](https://repl.it/GXAb/0)

The factorial function in Scheme:
```scheme
(define fact
  (lambda (n)
    (if (= n 1)
        1
        (* n (fact (- n 1))))))
```

But this type of code will fail in languages like C and Java ([code example](https://repl.it/@lubaochuan/BoringSpatialInsurance)) for a completely different reason: you will **overflow** the bits that a number can contain. Both Scheme and Python do a smart thing here: they automatically switch from Integers to BigNums (a data structure-based number representation).

If we call `fact(1000)` in Python it will fail (maximum recursion depth exceeded). This is the problem that we are exploring with continuations and CPS.

Let's take a look at the trace:
```scheme
#lang scheme

(require racket/trace)

(define fact
  (lambda (n)
    (if (= n 1)
        1
        (* n (fact (- n 1))))))

(trace fact)

(fact 4)
```

We can use a derivation to model a calculation with fact:
```
(fact 4)
= (* 4 (fact 3))
= (* 4 (* 3 (fact 2)))
= (* 4 (* 3 (* 2 (fact 1))))
= (* 4 (* 3 (* 2 (* 1 (fact 0)))))
= (* 4 (* 3 (* 2 (* 1 1))))
= (* 4 (* 3 (* 2 1)))
= (* 4 (* 3 2))
= (* 4 6)
= 24
```

We see that a value is returned, and then that result is used further. In other words, we have to multiply the result after the recursive call. The easiest way to handle this is to have a "call stack":
1. save where you are in the computation (push onto stack)
2. go off and compute (recursive) function
3. return, pop off the stack where you were
4. continue

When we use functions from other languages to implement `Calc` functions, we use a stack of function calls. Of course, for those languages that have a limited stack, this is a problem if we want to have the kinds of semantics/behavior that `Scheme` has.

Can we avoid the call stack? How to implement Scheme in a language that uses a call stack for functions?

The Big Idea: What if we could abstract all of the **computation that is left to do**? Then, we could do that computation without having to use a stack. Here, we introduce the idea of a continuation.

## Continuation
To use the idea of a continuation, we will write functions in continuation-passing style (CPS). If we can write the evaluator in CPS, then we might be able to avoid the call stack problem.

To write and use functions in CPS:
1. add an extra parameter, k (for continuation), to each function in CPS
2. in the function, whenever you get a result, "pass" it to the continuation (call the 3. continuation with the result)
4. when you call the function, if you need to do some computation after the function call, pass in a new continuation, and do the rest of the computation in the continuation, which includes "pass" the result to the received continuation.

**A continuation will be a function that takes a result, does anything that it needs to do to it, and passes that to the previous continuation.** This moves all recursion into the [tail call position](https://en.wikipedia.org/wiki/Tail_call).

## CPS Examples
Follow these examples by reading the comments:
```scheme
(define fact-cps
  (lambda (n k) ;; first, add k as a parameter
    (if (= n 1)
        (k 1) ;; if you get a result, pass it to the continuation
        (fact-cps (- n 1) ;; since there is computation to do after the call
           (lambda (v) ;; create a continuation, and do the rest
             (k (* v n))))))) ;; pass the result to the continuation
```

To start off the computation, we need to supply an initial continuation. This toplevel continuation doesn't have any further work to do, so it just returns the value. In other words, the initial continuation is the **identity** function.
```scheme
(fact-cps 5 (lambda (i) i))
```
Let's see what the trace looks like:
```scheme
#lang scheme

(require racket/trace)

(define fact-cps
  (lambda (n k)
    (if (= n 1)
        (k 1)
        (fact-cps (- n 1)
           (lambda (v)
             (k (* v n)))))))

(trace fact-cps)

(fact-cps 4 (lambda (i) i))
```
```
>(fact-cps 4 #<procedure>)
>(fact-cps 3 #<procedure>)
>(fact-cps 2 #<procedure>)
>(fact-cps 1 #<procedure>)
<24
24
```

This trace is very different from the regular fact trace. Notice that when we return a value, there is nothing left to do, and no need for a "call stack"---there is no computation left to do after the return.

This is called "tail call optimization" (TCO). In many languages, including C in special circumstances, tail calls that just return can be recognized, and then optimized into assembly language that doesn't push the call onto the stack... it is realized that the function can just return what the recursive function returns.

Let's consider `member?`, and write it in CPS:

```
#lang scheme

(require racket/trace)

(define fact-cps
  (lambda (n k)
    (if (= n 1)
        (k 1)
        (fact-cps (- n 1)
           (lambda (v)
             (k (* v n)))))))

(trace fact-cps)

(fact-cps 4 (lambda (i) i))
```

This works, but is not interesting.

### Problem 1
Why isn't `member?` interesting? Why didn't we create a continuation?

Let's write `my-length` as a regular recursive function and in CPS:
```scheme
(define my-length
  (lambda (lst)
    (cond
     ((null? lst) 0)
     (else (+ 1 (my-length (cdr lst)))))))

(my-length '(1 2 3 4))

(define my-length-cps
  (lambda (lst k)
    (cond
      ((null? lst) (k 0))
      (else (my-length-cps
             (cdr lst)
             (lambda (v)
               (k (+ 1 v))))))))

(my-length-cps '(1 2 3 4) (lambda (v) v))
```

Let's write Fibonacci function in CPS. First, the regular `fib` function and then the CPS version.
```
(define fib
  (lambda (n)
    (cond
     ((= n 0) 1)
     ((= n 1) 1)
     (else (+ (fib (- n 1)) (fib (- n 2)))))))

(define fib-cps
  (lambda (n k)
    (cond
     ((= n 0) (k 1))
     ((= n 1) (k 1))
     (else (fib-cps
            (- n 1)
            (lambda (v1)
              (fib-cps (- n 2)
                       (lambda (v2)
                         (k (+ v1 v2))))))))))
```

This is interesting because it has two recursive calls, both of which need to be put into tail call positions. We do that by putting the second one in the continuation of the first.
