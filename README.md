# Define Recursive Functions

In this assingment we will add `letrec` to our `Calc` language so that recursive
functions can be defined more easily.

Please use the [implementation of `Calc`](https://github.com/sbunivedu/calc-lang-sol/blob/let/sol.zip)
(without `letrec`) as your starter code and study the code base along with the test cases to review
the design and implementation of the language.

## Update environment
First, we will switch to use the "ribcage" representation of the environment as
described on [this page](https://github.com/sbunivedu/environment/blob/main/README.md)
so that we can define multiple mutually recursive functions in a single `letrec` expression.

This ribcage implementation is almost a drop-in replacement for the current "list of lists"
implementation except that the `extend-env` function name needs be changed and the parameter
list needs to be reordered.

You also need to update the initial environment creation code in `evaluator.rkt` as follows:
```scheme
(define env
  (extend-env
   (list 'pi 'e)
   (list 3.141592653589793 2.718281828459045)
   (extend-env
    (list '+ '- '*  'sum 'list)
    (list + - * + list)
    (extend-env
     (list '= '< '> 'avg)
     (list (lambda (a b) (if (= a b) 1 0))
           (lambda (a b) (if (< a b) 1 0))
           (lambda (a b) (if (> a b) 1 0))
           (lambda lst (/ (apply + lst) (length lst))))
     (empty-env)))))

; Next, we extend-env with those things that can use the previous things:

(set! env
      (extend-env
       '(min max)
       (list
        (evaluator (parser '(func (a b) (if (< a b) a b))) env)
        (evaluator (parser '(func (a b) (if (> a b) a b))) env))
       env))
```

## Update parser

The parser needs to be updated to support the new concrete syntax:
```
<expression> ::= (letrec (({<identifier> <expression>})*) <expresion>)
                 letrec-exp (vars vals body)
```

Update the `define-datatype` expression for `calc-exp` and the `parser`
function with the following clauses respectively so that `letrec` expressions
can be parsed correctly:
```scheme
(letrec-exp
  (vars (list-of symbol?))
  (vals (list-of calc-exp?))
  (body calc-exp?))
```
```scheme
((eq? (car exp) 'letrec)
 (letrec-exp (map car (cadr exp))
             (map parser (map cadr (cadr exp)))
             (parser (caddr exp))))
```

## Update evaluator
Modify the `evaluator` to handle `letrec-exp` in the abstract syntax tree
similar to `let-exp` except that we call `extend-env-rec` to extend the
environment.

In `extend-env-rec`, we first create a new extended environment with
a list of `syms` (IDs) and a list of `vals` (evaluated `calc` expressions)
as usual. Then, we iterate through the "closures" in `vals` while
replacing each of the enclosed environment with the newly extended environment.

```
(define (extend-env-rec syms vals env)
  (let ((new-env (extend-env syms vals env)))
    (update-env (caar new-env) (cdar new-env) new-env)))
```

Your job is to implement this `update-env` function, which takes a list of
symbols for IDs and a vector of values, if a "value" is a "closure" (`closure-exp`),
replace its `env` field with the `new-env` (3rd parameter to `update-env`),
and return an updated environment with closures that points to this updated
environment itself (circular).

You will need to deal with [Scheme vector](https://docs.racket-lang.org/reference/vectors.html).
Here are some examples:
```scheme
(define v (vector 'a 'b 'c))

(vector->list v)
; => '(a b c)

(list->vector '(1 2 3))
; => '#(1 2 3)

(vector-ref v 1)
; => 'b

(vector-set! v 1 'd)

v
; => '#(a d c)
```

## Test your solution

Here are a few test programs:
```scheme
(letrec ((fact (func (n)
                     (if n
                         (* n (fact (- n 1)))
                         1))))
  (fact 5))
; => 120

(letrec ((even? (func (n)
                      (if n
                          (odd? (- n 1))
                          1)))
         (odd? (func (n)
                     (if n
                         (even? (- n 1))
                         0))))
  (even? 42))
; => 1
```
