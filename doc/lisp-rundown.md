% A Rundown of Lisp

Lisp is a *family* of programming languages.  The most popular are
Common Lisp, Scheme, Emacs Lisp, and recently Clojure.  This text
might have a slight Scheme flavor.

Lisps brought a lot of innovation, but nowadays mainly share one
feature that sets them apart from other languages: their code is
written in s-expressions.


Symbolic Expressions
--------------------

First you must learn about *symbolic expressions*, AKA
*s-expressions*, AKA *sexprs*, AKA *sexps*.  S-expressions are a
recursively defined data format (like JSON, and a bit like XML):

> A sexpr is either 1. an atom, or 2. a pair of sexprs.

- An atom is a single datum like a number, string, etc.
- A pair is written `(x . y)`, where `x` and `y` are sexprs.
  (Recursion!)

(Instead of the arrays and dictionaries in JSON, you have pairs.)

(The parentheses and dot in the pair are just syntax, like the commas
in arrays or colons in dictionaries in JSON.  The dot *must* be
separated with spaces though, unlike commas or colons in JSON.)

In addition to usual atomic data types, sexprs support an atomic type
called *symbol*.  Symbols are restricted strings that don't need
quotation.  They are useful for things like tags, keywords,
identifiers, etc.  In some implementations they're case-insensitive.

Examples of valid sexprs:

    "A string."
    a-symbol
    a.Really!@$Really%^&Weird*=+Symbol<>/?
    2.5  ;a number (oh and comments start with a semicolon)
    ("foo" . 3)  ;a pair of two atoms
    (("foo" . bar) . 5)  ;nested pairs
    ((-1 . 1) . (a . z))  ;likewise

Note that `'foo'` is not a string; only `"foo"` is.  The apostrophe
has a special meaning that we will learn later.

We can represent many data structures with pairs that are aligned in
such a way as to reflect the structure.  The most common is the list,
which is represented as a chain of pairs:

    (first-element . (second-element . (third-element . NIL)))

Here we used the symbol `NIL` to mark the point at which the list
ends.  Some implementations use a special atomic value instead, which
is called *the empty list* or *null* and is written `()`.  This empty
list object is also of a separate data type.  No other object than the
empty list is of that type.

    (first-element . (second-element . (third-element . ())))

That's neat and makes sense because now we can say:

> A list is either the empty list, or a pair with the first element of
> the list in the left position, and the rest of the list in the right
> position.

Writing lists with pairs is obviously ugly, so all implementations
support a short version:

    (first-element second-element third-element)

When read by a sexpr reader (parser), this is internally transformed
to pairs, ending with the implementation's preferred list-ending
object (`nil` or `()`).

You'll also see mixed syntax; the following two are equivalent:

    (foo bar baz . bat)
    (foo . (bar . (baz . bat)))

Note that that's not a proper list; in fact it's called an *improper
list* because it's almost a list but doesn't end with the list-ending
object but some other atomic object.  It's valid data though, whereas
the following is simply invalid syntax:

    (foo . bar baz)

When transmitting data, lists would be used where in JSON you would
use arrays.  For dictionaries, you would use an *association list* AKA
*alist*; these are simply a list of key-value pairs:

    ((key1 . value1)
     (key2 . value2)
     (key3 . value3))

Note that we used mixed syntax; the outer structure is a list that
doesn't use the dotted pair notation, but the individual elements are
simply pairs and not lists so they use the dotted pair syntax.

The keys could be of any type (unlike in JSON where they must be
strings) but most often they're symbols.

That's all there is to sexprs!  They're a simple data format that
could be used in place of JSON.


Evaluation
----------

Lisp code is written in sexprs.  When you pass a sexpr to a lisp
*evaluator* (interpreter or compiler), it expects the sexpr to conform
to certain rules, so that it makes sense as program code.  The
evaluation rules are as follows.

***

Atoms other than symbols *evaluate to themselves*.  So for example the
sexpr number `5` evaluates to the number 5 in memory, the sexpr string
`"foo"` evaluates to the string "foo" in memory, etc..  They just
stand for themselves in the code.  They're *literals*.

Symbols are normally variable references, evaluating to the value of
the variable.

Plain pieces of data or plain variable references aren't very
interesting of course; the real actions happen in lists.  When a list
is evaluated, the first element determines what happens:

- It could be the name of a function.  In that case the list
  represents a function call, AKA *application*, and the rest of the
  list give the arguments to the function.

So where in another language you might write

    do_something(foo, bar)

in a Lisp you would write

    (do-something foo bar)

- It could also be the name of a *syntax keyword*.  In that case the
  special rules of that syntax apply to the rest of the list, which
  could be anything.  For example `if` is a syntax keyword.

In another language the `if` syntax might look like:

    if (foo) bar; else baz;

But in Lisp it would be represented as a list:

    (if foo bar baz)

Or that other language might use `{ }` for code blocks:

    if (foo) { bar; baz; } else { qux; fux; }

In Scheme you would use the `begin` syntax for that:

    (if foo (begin bar baz) (begin qux fux))

(In Common Lisp and Emacs Lisp the `begin` syntax is called `progn`
instead, for historical reasons.)

But let's go back to function application first.

Some implementations use separate namespaces for variables and
functions.  In those, the symbol which is the first element of a list
must simply be the name of a function.  In other implementations, the
symbol is just a variable reference, and the variable must hold a
function object, or *procedure* as called in Scheme.

Lisps that have separate function and variable namespaces are called a
*lisp-2*, because they have two namespaces.  Others are called a
*lisp-1*.  (Actually both also have a syntax keyword namespace, but
whatever.)  Lisp-2s can have a function and a variable of the same
name, so the code `(list list)` would call the `list` function on the
value of the `list` variable.  In a lisp-1 you would be calling the
`list` procedure on itself, because both instances of `list` in that
code are just variable references evaluating to the `list` procedure.

***

What happens if the first element of a list is another list?  Some
implementations simply don't allow this.  In those that do, the inner
list is just evaluated normally but must result in a procedure object,
and that procedure is then applied to the rest of the list as usual.
For example:

    ((give-me-a-procedure foo) bar)

That first applies `give-me-a-procedure` to `foo`, which gives us a
procedure; then that procedure is applied to `bar`.

***

By the way, the order in which arguments in a function application are
evaluated might be unspecified in a lisp implementation, so the code

    (do-whatever (print-foo-and-return-whatever)
                 (print-bar-and-return-whatever))

might print bar before printing foo.

***

That's all really.  Now you know how to write data in sexprs, and you
know how to write sexpr data which makes sense as lisp code.  But
you'll need to learn many important syntax keywords before you can
actually write meaningful lisp code; the most important ones are
explained in the examples below.


Examples
--------

    ;; The rest of this file is all syntactically valid lisp code.
    ;; The examples use a lisp similar to Scheme (a lisp-1).


    (add 5 (divide 6 2))  ; => 8

    ;; The symbol 'add' is not the name of a syntax keyword, so it
    ;; gets evaluated as a variable reference (or looked up in the
    ;; function namespace); this yields a procedure that does
    ;; addition.  The other elements are also evaluated; 5 just
    ;; evaluates to 5, and the list (divide 6 2) evaluates to 3.
    ;; Finally the addition procedure is applied to 5 and 3, yielding
    ;; 8.  (The comment with "=> 8" is just to say that the whole
    ;; thing evaluates to 8.)

    ;; Actually '+' and '/' are also symbols, so they could be used
    ;; too:
    (+ 5 (/ 6 2))  ; => 8


    (if (equal? a b)
        (print "a and b are equal")
        (print "a and b are not equal"))

    ;; The symbol 'if' names a syntax keyword.  The rules of this
    ;; syntax say: first evaluate the second element of the list (the
    ;; one after the 'if'), then if the resulting value is true, only
    ;; evaluate the third sexpr, otherwise only evaluate the fourth
    ;; sexpr.  Note that the question mark in 'equal?' is just a part
    ;; of that symbol.

    ((if (= 5 5) double half) 10)  ; => 20
    ((if (= 4 6) double half) 10)  ; => 5

    ;; 'if' not only executes the then-branch or else-branch as code;
    ;; it also returns any value coming from the branch it evaluated.
    ;; So we conditionally either apply the doubling procedure, or the
    ;; halving procedure.  (Of course '=' is just the variable holding
    ;; the numeric equality comparison procedure.)


    (define power-level (+ 9000 1))

    ;; 'define' is also syntax.  The rules say: the second element
    ;; must be a symbol that names a new variable; the third element
    ;; is evaluated and yields the initial value for that variable.


    (lambda (foo bar) body)

    ;; The Greek letter "lambda" is used in maths to denote the
    ;; creation of a so-called "lambda abstraction", which is the
    ;; mathematical model of lisp functions.  So 'lambda' is used as a
    ;; syntax keyword to create procedures.  The second element is the
    ;; parameter list of the procedure.  The rest of the list is one
    ;; or more sexprs that make up the procedure's body.  A procedure
    ;; always returns the value of the last evaluation in its body.


    ((lambda (x) (print "adding 5 to my argument") (+ x 5))
     10)  ; => 15

    ;; The first element is a list that evaluates to a procedure by
    ;; using the lambda syntax.  The procedure is then applied to 10.


    (define double (lambda (x) (* x 2)))
    (print (double 5))

    ;; Binds the variable 'double' to a procedure, then uses it.


    (define (double x) (* x 2))
    (print (double 5))

    ;; The define syntax is smart and understands this format as a
    ;; shortcut for defining procedures, so it's the same as the
    ;; previous example.


    (let ((x (calculate-x))
          (y (calculate-y)))
      (+ x y))

    ;; The let syntax binds some variables and then evaluates its
    ;; body.  The above is entirely equivalent to:

    ((lambda (x y) (+ x y))
     (calculate-x)
     (calculate-y))


    ;; That was a big chunk of the fundamentals of lisp.  In fact the
    ;; last one is not fundamental because it's equivalent to using
    ;; lambda (but looks nicer), and in lisp you can actually define
    ;; new syntax keywords, so you could implement 'let' yourself in a
    ;; library!  All implementations provide it as a built-in though,
    ;; because it's so useful.  It's the normal way in which you
    ;; create *local* variables in lisp, where 'define' is more used
    ;; for global ones.



Quotation
---------

    ;; Now let's see how we can use sexprs as actual data in a lisp
    ;; program (which itself is already sexpr data).  This is a bit
    ;; tricky at first, because we write down a single sexpr, but when
    ;; a lisp evaluator goes over it, most of it turns into program
    ;; logic while some parts of it remain pure data.


    (define foo (quote some-sexpr))

    ;; Quote is a syntax keyword; this syntax expects exactly one
    ;; other element in the list, and returns it as verbatim sexpr
    ;; data, cancelling the lisp evaluator's effect on it.  In this
    ;; case we bind 'foo' to the symbol 'some-sexpr'.

    ;; We have shortcut syntax for quote:

    (define foo 'some-sexpr)  ;Any 'x automatically becomes (quote x).

    ;; (There really is only one apostrophe on the left side.)


    (define list1 '(some list))
    (define list2
      (quasiquote (foo (unquote list1) bar (unquote (+ 3 2)))))

    ;; Quasiquote is the same as quote, except that the sexpr is
    ;; scanned for lists that start with 'unquote'.  Those lists must
    ;; have only one other element, for which evaluation is resumed
    ;; again.  So now 'list2' holds the sexpr '(foo (some list) bar
    ;; 5)'.

    ;; Note that 'unquote' is not a syntax keyword; it's just scanned
    ;; for by 'quasiquote'.

    ;; We also have shortcut syntax for quasiquote and unquote:

    (define list2 `(foo ,list1 bar ,(+ 3 2)))


    (define list1 '(bar baz))
    (define list2 `(foo (unquote-splicing list1) quux))

    ;; Quasiquote also scans for lists starting with
    ;; 'unquote-splicing'.  Those parts are also unquoted (evaluated),
    ;; but they *must* evaluate to a list.  The elements of that list
    ;; are "spliced" (inserted) into that position in the surrounding
    ;; list.  So in the above example, 'list2' gets bound to '(foo bar
    ;; baz quux)'.

    ;; Again, we have shortcut syntax for unquote-splicing:

    (define list2 `(foo ,@list1 quux))

    ;; Fin.
