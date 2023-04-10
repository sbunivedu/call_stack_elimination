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
1. add an extra parameter, k (for continuation), to each **function** in CPS
2. in the **function**, whenever you get a result, "pass" it to the continuation (call the continuation with the result)
3. when you call the **function**, if you need to do some computation after the **function** call, pass in a new continuation, and do the rest of the computation in the continuation, which includes "pass" the result to the received continuation.

**A continuation will be a function that takes a result, does anything that it needs to do to it, and passes that to the previous continuation.** This moves all recursion into the [tail call position](https://en.wikipedia.org/wiki/Tail_call).

We could implement the `fact` function using tail recursion as follows:
```scheme
#lang scheme

(require racket/trace)

(define fact
  (lambda (n)
    (fact-tail n 1)))

(define fact-tail
  (lambda (n acc)
    (if (= n 1)
        acc
        (fact-tail (- n 1) (* n acc)))))

(trace fact-tail)
```

## CPS Examples
Follow these examples by reading the comments:
```scheme
(define fact-cps
  (lambda (n k)                ;; first, add k as a parameter
    (if (= n 1)
        (k 1)                  ;; if you get a result, pass it to the continuation
        (fact-cps (- n 1)      ;; since there is computation to do after the call
           (lambda (v)         ;; create a continuation, and do the rest
             (k (* v n)))))))  ;; and pass the result to the continuation
```

To start off the computation, we need to supply an initial continuation. This toplevel continuation doesn't have any further work to do, so it just returns the value. In other words, the initial continuation is the **identity** function.
```scheme
(fact-cps 4 (lambda (i) i))
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

This trace is very different from the regular `fact` trace. Notice that when we return a value, there is nothing left to do, and no need for a "call stack"---there is no computation left to do after the return.

This is called "tail call optimization" (TCO). In many languages, including C in special circumstances, tail calls that just return can be recognized, and then optimized into assembly language that doesn't push the call onto the stack... it is realized that the function can just return what the recursive function returns.

Let's consider `member?`, and write it in CPS:

```scheme
#lang scheme

(require racket/trace)

(define member?
  (lambda (item lst)
    (cond
     ((null? lst) #f)
     ((eq? item (car lst)) #t)
     (else (member? item (cdr lst))))))


(define member?-cps
  (lambda (item lst k)
    (cond
     ((null? lst) (k #f))
     ((eq? item (car lst)) (k #t))
     (else (member?-cps item
                        (cdr lst)
                        (lambda (v)
                          (k v)))))))

(trace member?-cps)

(member?-cps 'a '(c b a) (lambda (i) i))
```

```
>(member?-cps a (c b a) #<procedure>)
>(member?-cps a (b a) #<procedure>)
>(member?-cps a (a) #<procedure>)
<#t
#t
```

This works, but is not interesting.

### Problem 1
Why isn't `member?` interesting? Why didn't we create a continuation?

Let's write `my-length` as a regular recursive function and in CPS:
```scheme
#lang scheme

(require racket/trace)

(define my-length
  (lambda (lst)
    (if (null? lst)
        0
        (+ 1 (my-length (cdr lst))))))

(define my-length-cps
  (lambda (lst k)
    (if (null? lst)
        (k 0)
        (my-length-cps (cdr lst)
                       (lambda (v)
                         (k (+ 1 v)))))))

(trace my-length-cps)

(my-length-cps '(1 2 3 4) (lambda (i) i))
```
```
>(my-length-cps (1 2 3 4) #<procedure>)
>(my-length-cps (2 3 4) #<procedure>)
>(my-length-cps (3 4) #<procedure>)
>(my-length-cps (4) #<procedure>)
>(my-length-cps () #<procedure>)
<4
4
```

Let's write Fibonacci function in CPS. First, the regular `fib` function and then the CPS version.
```
#lang scheme

(require racket/trace)

(define fib
  (lambda (n)
    (if (or (= n 1) (= n 2))
        1
        (+ (fib (- n 1) (- n 2))))))

(define fib-cps
  (lambda (n k)
    (if (or (= n 1) (= n 2))
        (k 1)
        (fib-cps (- n 1)
                 (lambda (v)
                   (fib-cps (- n 2)
                            (lambda (w)
                              (k (+ v w)))))))))

(trace fib-cps)

(fib-cps 6 (lambda (i) i))
```
```
>(fib-cps 6 #<procedure>)
>(fib-cps 5 #<procedure>)
>(fib-cps 4 #<procedure>)
>(fib-cps 3 #<procedure>)
>(fib-cps 2 #<procedure>)
>(fib-cps 1 #<procedure>)
>(fib-cps 2 #<procedure>)
>(fib-cps 3 #<procedure>)
>(fib-cps 2 #<procedure>)
>(fib-cps 1 #<procedure>)
>(fib-cps 4 #<procedure>)
>(fib-cps 3 #<procedure>)
>(fib-cps 2 #<procedure>)
>(fib-cps 1 #<procedure>)
>(fib-cps 2 #<procedure>)
<8
8
```

This is interesting because it has two recursive calls, both of which need to be put into tail call positions. We do that by putting the second one in the continuation of the first.

### Problem 2
Re-write 6 different recursive functions in CPS form. Make sure that these are not trivial (like `member?`) but are functions that require the creation of a continuation (e.g.
`map`). Test them to make sure that they work.

Wait! This doesn't help!

This doesn't yet help... we are still using recursion, and the host language (e.g., Python, Java, C, etc.) still must have a call stack.

But, now that we have abstracted the "computation left to do", let's see if we can "factor out" the recursive function calls. That is, we move the recursive function calls out from inside the CPS functions.

Let's make these changes:
* define `make-cont` to wrap the procedure
* define `apply-cont` to apply a continuation to a result

```
#lang eopl

;; tag a continuation, like define-datatype does:
(define make-cont
  (lambda (f)
    (list 'continuation f)))

;; is it a tagged continuation?

(define continuation?
  (lambda (item)
    (and (list? item)
         (> (length item) 0)
         (eq? (car item)
              'continuation))))

;; apply a continuation to a result like before:
(define apply-cont
  (lambda (k r)
    (if (continuation? k)
        ((cadr k) r)
        (eopl:error 'apply-cont "invalid continuation: ~a" k))))

(define my-length-cps
  (lambda (lst k)
    (if (null? lst)
        (apply-cont k 0)
        (my-length-cps (cdr lst)
                       (make-cont (lambda (v)
                                    (apply-cont k (+ 1 v))))))))

(my-length-cps '(1 2 3 4) (make-cont (lambda (i) i)))
```

That is nice, but it didn't change anything!

However, rather than applying the continuation to the result, let's just return the function representing the rest of the computation and the result in a list:
```scheme
(define apply-cont
  (lambda (k v)
    (if (continuation? k)
        (list k v) ;; WE CHANGED THIS LINE!
        (eopl:error 'apply-cont "invalid continuation: ~a" k))))
```

Now, when we attempt to run the function, we get a partial result, and a continuation representing what is left to do:
```
> (my-length-cps '(1 2 3 4) (make-cont (lambda (i) i)))
((continuation #<procedure>) 0)
```

### Problem 3
Explain what we have at this point.

We could then apply that continuation to the result:
```
(define result (my-length-cps '(1 2 3 4) (make-cont (lambda (v) v))))

result

((cadar result) (cadr result))
```

We could continue until there are no continuations left.

### Problem 4
Keep applying the continuations until you get an answer.

### Problem 5
Re-write the 6 functions to use your new continuation functions. Test them all to make sure that they work.

What can you do with these continuations?

At this point, you could:
* save the continuation (say, to a file), and continue the continuation later. How would that be useful?
* do the computation again; why?
* create a debugger; how?
* go back in time; how?
* interleave the continuations from different computations in a particular order (prioritize some, or evenly execute); why? what would this be useful for?

### Problem 6
Explain each of these ideas and answer the related questions. Can you think of other ideas you could use continuations for?

But how does this help eliminate the call stack?

To get rid of the call stack completely, we introduce the idea of a trampoline. A trampoline is a method where continuations are bounced inside a loop, rather than recursive calls that go deeper and deeper.

Here is a version of a trampoline for our continuations:
```
(define trampoline
  (lambda (result)
    (let loop ((result result))
      (if (and (list? result)
               (> (length result) 0)
               (continuation? (car result)))
          (loop ((cadar result) (cadr result)))
          result))))

(trampoline (my-length-cps '(1 2 3 4) (make-cont (lambda (i) i))))
```
This version uses ["named let"](https://stackoverflow.com/questions/31909121/how-does-the-named-let-in-the-form-of-a-loop-work) expression,
which is equivalent defining a tail recursive internal function as follows:
```scheme
(define trampoline
  (lambda (result)
    (letrec
        ((loop (lambda (x)
                 (if (and (list? x)
                          (> (length x) 0)
                          (continuation? (car x)))
                     (loop ((cadar x) (cadr x)))
                     x))))
      (loop result))))
```

### No call stack!

### Problem 7
Explain how we could use these ideas in our language Calc.

## Resources
* http://en.wikipedia.org/wiki/Continuation-passing_style
* http://2ality.com/2012/06/continuation-passing-style.html
* https://stackoverflow.com/questions/47948928/javascript-change-recursive-function-with-trampoline-version
* https://medium.com/@b.essiambre/continuation-passing-style-patterns-for-javascript-5528449d3070
* http://raganwald.com/2013/03/28/trampolines-in-javascript.html
* https://eli.thegreenplace.net/2017/on-recursion-continuations-and-trampolines/
* https://dzone.com/articles/asynchronous-programming-and
* https://jsbin.com/nicohazaqo/edit?js,console
* http://matt.might.net/articles/by-example-continuation-passing-style/
* http://exploringjs.com/es6/ch_async.html
* https://jlongster.com/Whats-in-a-Continuation
