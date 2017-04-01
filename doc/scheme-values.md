% Novel value passing semantics for Scheme

Note: Remember the equivalence between procedure calls, and value
returns to continuations, while reading this document.  Returning
values means calling the current continuation with those values as
arguments.  Consider the following implementation of `values`:

    (define (values . vals)
      (call/cc (lambda (cont) (apply cont vals))))

It literally applies the current continuation to a list of values!

Thus, "value" and "argument" are pretty much synonyms, although both
are used in this document because sometimes one is clearer.


Zero value returns
------------------

Scheme often says that an expression returns an unspecified value when
it doesn't make sense for it to return a value.  This applies, for
example, to procedures called only for side-effect.

It would be cleaner to specify such expressions to return zero values.

Scheme also specifies that the continuation of a non-tail expression
in a `<body>` takes an arbitrary number of values and discards them.

It would be cleaner to specify such continuations to take zero values.

***

This would prevent a class of programmer errors where a procedure's
return value is ignored accidentally.  For instance Guile's `system*`
procedure returns an integer to report the exit status of the process
that was executed; it's easy to forget that no exception will be
raised, and a possible failure of the executed process has to be
tested as per this integer return value instead.

This means that code such as

    (begin
      (copy-file x y)
      (do-something))

cannot possibly "leak" any values returned by `copy-file`; an error
will be raised if `copy-file` returns more than zero values.

When returned values are desired to be discarded, an `ignore` special
form could be used, which accepts one operand that is an arbitrary
expression, ignores all values resulting from the evaluation of that
expression, and returns zero values.

    (begin
      (ignore (system* "false"))  ;ignore exit status
      (do-something))

***

Another problem it would prevent is where programmers accidentally
rely on a procedure, whose return value is unspecified, to return a
non-false value.

This happened in many GNU Guix package recipes, where every build
phase procedure is expected to return a Boolean value, but many
programmers wrote procedures that end in a call to a procedure with an
unspecified return value (typically, one called for side effects),
which works only because GNU Guile consistently returns a special
`*unspecified*` value which is non-false, although its own user manual
generally says that procedures return an unspecified value, and not
the special `*unspecified*` value.

Now either Guile will have to constrain itself to return this value
consistently where a return value was really meant to be unspecified,
or many Guix recipes will break.  If such procedures returned zero
values instead, then said build phase procedures would fail because
they don't return the expected Boolean, and the programmers would
write them properly to begin with.

(This would, however, not solve the problem where the numeric return
value of `system*` is interpreted as true, and therefore successful,
when it really was an integer indicating failure.  This brings us to
the question of whether it wouldn't be better to raise a type mismatch
error in cases where a non-Boolean value is used as a Boolean, but
that is another topic.)

***

This safety net has some small necessary overhead on every procedure
call in a non-tail position in a body (of a begin, lambda, etc.).  An
implementation could provide an optimization to remove the safety net.


Passing optional values
-----------------------

A well-known extension to procedure call semantics is to allow
procedures to specify optional parameters.  Thus the call-site does
not need to provide all possible arguments to the procedure; they take
default values when unspecified.

What's missing is support for the call-site to provide values that are
optional to handle by the called procedure.

***

This would allow, for instance, writing an interface that takes
user-defined procedures and calls them with a number of values they
must handle and a number of values which might be but are not
necessarily useful, and users of this interface could pass any
procedure that handles at least the mandatory values.

The programmer would otherwise need to wrap some procedures to ignore
some of their arguments and pass on the others, or write procedures
with ignored parameters that aren't referenced in the procedure body.

***

Perhaps a better example is not when calling a procedure, but when
returning values.  Some "search" or "lookup" procedure (for a hash
table, database, etc.) returns one mandatory-to-handle value, which is
the search result or a "not found" token; and also returns an optional
Boolean which explicitly says whether a result was or was not found.
This way users of the procedure can distinguish between the case where
no result was found, and the case where the "not found" token just
happened to be the actual result of the search.  But they can also
ignore the extra Boolean and conflate those cases if they want.

Currently most systems fulfill this use-case by making superfluous
return values always be ignored, or by providing an alternative
procedure, which raises an exception, calls a given procedure, or does
something else in the case where no result was found.

***

Specifying optional arguments in a procedure call might work with a
special token marking the beginning of the optional values, such as:

    (procedure foo bar #!optional alpha beta)

This token does not evaluate; it is only valid as part of a
procedure-call expression and special-handled by `eval`.

***

Optional values could be returned by, well, calling the current
continuation with optional values:

    (call/cc (lambda (k) (k foo bar !optional alpha beta)))

but that's not very nice.  The `values` procedure, which is already
used to call the current continuation with multiple values, could be
used for this purpose:

    (values foo bar #!optional alpha beta)

Being implemented as a primitive, it would intercept the optional
status of its arguments and return them again as such.

***

When a procedure with optional parameters is called also with optional
values, the behavior would be as follows:

The mandatory and optional parameters of the procedure are
conceptually appended to create a full parameter list.  The mandatory
and optional values passed to the procedure are appended as well,
creating a full value list.  Each value in the value list fills the
corresponding slot in the parameter list until either list is
exhausted.  It is an error if there remain unfilled mandatory
parameter slots, or unused mandatory values.

***

When a procedure takes a rest-arguments list, the optional status of
values is discarded.


Optional keyword arguments
--------------------------

The concept of passing optional values can be extended to include
keyword arguments (or "keyword values"), which are another well-known
extension to procedure call semantics.

In GNU Guix recipes, it is extremely common to see a formal parameters
specification such as `(#:key inputs outputs #:allow-other-keys)`,
meaning the keyword arguments `inputs` and `outputs` are bound to
these identifiers, and any other keyword arguments are ignored.  This
`#:allow-other-keys` token would be unnecessary if the call-site could
specify that most or all keyword arguments are optional to handle.

The `#!optional` token could be re-used for this:

    (procedure foo bar #!optional alpha beta
               #:keyword-x value-x #:keyword-y value-y
               #!optional
               #:keyword-a value-a #:keyword-b value-b)

***

The `values` procedure would again be used to call the current
continuation with given values:

    (values foo bar #!optional alpha beta
            #:keyword-x value-x #:keyword-y value-y
            #!optional
            #:keyword-a value-a #:keyword-b value-b)


Keyword arguments and rest arguments
------------------------------------

Improper formal parameter lists don't combine well with keyword
arguments.  Keeping with s-expression structure, one would have to put
the dotted tail of the list after all keyword parameters, and it would
not be obvious what it denotes.  I would propose to support improper
parameter lists for backwards compatibility only, and offer a `#!rest`
token for new code, which would also work in combination with the
`#!keyword` token:

    (lambda (x y #!optional a b #!rest rest
             #!keyword alpha beta #!rest keyword-rest)
      ...)

Using an old-style rest argument like:

    (lambda (x y . rest) ...)

would be equivalent to:

    (lambda (x y #!rest rest) ...)

meaning it errors when mandatory keyword values are received, but
ignores optional keyword values.

***

Sometimes one wants to wrap a procedure in another, remaining agnostic
to what arguments it accepts.  In that case one normally uses not a
formal parameter list, but a single identifier.  We don't want to
break compatibility too much so that would keep functioning as a rest
arguments list, and a new syntax would be allowed to intercept
received values as an object of a new "values" type:

    ((lambda (#!values values) values)
     foo bar #!optional alpha beta
     #:keyword-x value-x #:keyword-y value-y)

         => #<values mandatory: (foo bar)
                     optional: (alpha beta)
                     keyword-mandatory:
                     ((keyword-x . value-x) (keyword-y . value-y))
                     keyword-optional:
                     ((keyword-a . value-a) (keyword-b . value-b))>

This type would be accepted by `apply` as an alternative to lists.
There could also be a full API for this new type, to create instances
of it, access their components, etc.

Any other use of the #!values token would be an error.  I.e. it cannot
be mixed in any way with any other formal parameter list; it must
always occur as the first element of a two-element formal parameter
list, the second element being the variable to which the values object
will be bound.


Automatic handling of wrong number of values
--------------------------------------------

A common complaint about multiple-value returns is that they don't
compose well with the rest of Scheme because they are not first-class.

A possible solution is to automatically wrap wrong numbers of values
in values objects.

This way, code such as `(foo (bar))` and `(apply foo (bar))` works no
matter how many values are returned (hence first-class); the former
might just eventually raise a type error if it didn't expect to get a
values object.

However, this has some serious problems.

Firstly, this generally delays error reporting (leading to an object
of a wrong type, a values object, being passed to some place instead
of immediately erroring).  Worse, it hinders error reporting much more
severely in some cases, notably conditionals: a values object of zero
values or several false values will presumably be true unless the
implementation complicates its conditionals to special-handle these,
and even then their semantics are not obvious.

So it's probably least problematic to generally error on values
mismatch, and only allocate values objects when a formal parameter
specification is a single identifier.  We accept the problem of second
class status of multiple values, and live with it.


Reference implementation in GNU Guile
-------------------------------------

A "reference implementation" of all above ideas should be under the
same directory as this file, under the name scheme-values.scm (or with
an additional .txt at the end to satisfy certain systems).  This
implements a trivial meta-circular evaluator; it doesn't add the
described semantics to Guile itself.

Fin.
