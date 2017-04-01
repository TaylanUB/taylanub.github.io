Why syntax-case
===============

Exposing the s-expressions making up the code to the procedural macro
system seems like a no-brainer, upon which higher abstractions would
be built.

I would contend though that it's a mistaken perspective.  I don't
expect to absolutely change the reader's mind, but let me try. I'll
argue my position from both a theoretical and practical side.
Starting with the theory.

While Scheme code is made up of s-expressions at the lowest level,
these sexprs gain additional meaning within the semantics of Scheme.
As we know, every sexpr is evaluated in a given environment.  For
instance in R7RS, this environment is determined either by the series
of imports at the start of a top-level R7RS program, the import
declarations of a library definition, or the explicit environment
object passed to the eval procedure.  Thus, every sexpr that makes up
Scheme code to be evaluated also has a context, an evaluation
environment, attached to it.  Therefore, a raw sexpr is not the
correct reification of code when passed to or returned from a
metaprogramming system such as procedural macros.  Under syntax-case,
the reified syntax objects correctly consist of a sexpr plus its
context information.  The ones passed to the macro system have the
context already bundled, and the macro system can create further such
syntax objects with appropriate context bundled to them via 'syntax',
'quasisyntax', and 'datum->syntax'.  Raw sexprs can be extracted from
syntax objects via 'syntax->datum'.

Now, one could say that an equally valid reification is to pass the
sexprs and context as separate objects.  Semantically, I suppose this
is equivalent.  It is indeed what Syntactic Closures (SC), Reverse
Syntactic Closures (RSC), and Explicit Renaming (ER) macro systems do
in different ways, though they also all do something special/implicit
when a raw sexpr is returned from a macro procedure:

- The SC system passes procedural macros a sexpr and an env object
  corresponding to the environment from which the macro is called (the
  caller environment).  The programmer uses make-syntactic-closure to
  bundle the caller environment to an identifier; otherwise the
  identifier will be bundled with the environment in which the macro
  was defined (the definition environment) when it's returned from the
  macro procedure.

- The RSC system passes procedural macros a sexpr and the definition
  environment.  The programmer uses make-syntactic-closure to bundle
  the definition environment to an identifier; otherwise the
  identifier will be bundled with the caller environment when it's
  returned from the macro procedure, though this is pretty much the
  same thing as what happens with 'define-macro', which is that the
  identifier is simply evaluated at the macro call site as if it
  appeared there normally.  Therefore you could say RSC macros are
  "unhygienic by default".

- The ER system passes procedural macros a "rename" procedure which
  bundles the definition environment to an identifier when the
  procedure is called on it.  Other identifiers are implicitly bundled
  with the caller environment upon return.  Essentially, ER is a thin
  wrapper around RSC.

This might be subjective, but I would say these are somewhat lazy
and/or clumsy abstractions.  SC and RSC are pretty low-level; why
burden the programmer with the task of manually bundling identifiers
with environments?  SC at least bundles the definition environment to
bare identifiers implicitly on return, but that in turn limits some
usages I will touch upon later.

As for the IR system, it's the least bad: from the sexpr (tree) it
passes to procedural macros, all the leaves are already bundled with
the calling environment, and bare identifiers are bundled with the
definition environment upon return.  This is a fairly fine
abstraction, but has the clumsiness of implicit bundling, which I will
now explain.

The problem with implicit bundling of the definition environment is,
it doesn't let you compose syntax objects stemming from a variety of
environments.  I learned the usefulness of this through experience,
making good use of syntax-case's capabilities in my Bytestructures
library to offer an optional macro API to users who need maximum
performance; via this, one can offload all offset calculation of
structs to compile-time, essentially giving you something like C's
type system for Scheme (it's indeed for FFI purposes, among more).
This works thanks to the possibility of every bytestructure descriptor
object (essentially a "type" object) being able to return its own
partial syntax object --bundled to its own definition environment-- to
a larger subsystem at compile time, that composes these syntax objects
together to create the final piece of code that references bindings
from all across the different modules in which different descriptors
are defined.

Note that while the bundling of a definition environment to an
identifier is automatic under syntax-case, it's still explicit.  You
use 'syntax' or 'quasisyntax' to have a whole sexpr bundled to the
environment in which it appears; you don't have to call a procedure on
each identifier appearing in the leaves of that sexpr.

The only annoyance with syntax-case is that bundling doesn't
necessarily happen at the level of identifiers.  When you bundle a
whole expression like '(cons foo bar)' with an environment, you'll get
a single syntax object containing that list of symbols as its payload,
rather than a list of three syntax objects that are individually
bundled with the given environment.  The 'syntax-case' special form
lets one destructure such syntax objects easily though, transparently
working both with syntax objects containing lists, and lists
containing syntax objects.

External resources on SC, RSC, ER, and IR systems:
- http://www.gnu.org/software/mit-scheme/documentation/mit-scheme-ref/Macros.html
- https://wiki.call-cc.org/man/4/Macros
- https://wiki.call-cc.org/explicit-renaming-macros

Bytestructures:
- https://github.com/TaylanUB/scheme-bytestructures


A concrete example
------------------

Look into 'bytestructures/body/numeric.scm', and note how the 'uint8'
descriptor ends up having (when the macros are expanded) a getter
procedure like:

    (define (getter syntax? bytevector offset)
      (if syntax?
          #`(bytevector-u8-ref #,bytevector #,offset)
          (bytevector-u8-ref bytevector offset)))

Note the bit where 'bytevector-u8-ref' appears within a quasisyntax.
This captures the identifier within this module, turning it into a
syntax object; one that bundles the symbol 'bytevector-u8-ref' with
the information that it's to be looked up in this module, where that
identifier is bound (though imported from '(rnrs bytevectors)').
Effectively, it's an absolute reference to the 'bytevector-u8-ref'
procedure from '(rnrs bytevectors)', in our beloved first-class
functions fashion.  Just this it's a compile-time reference to the
procedure instead of a run-time reference.

Meanwhile in another module, a procedural macro is defined in an
environment where 'bytevector-u8-ref' isn't bound, only 'uint8'.  When
the macro is called, it calls the getter procedure of uint8 (with the
syntax? flag set to true) as a helper, which returns the above seen
snippet within quasisyntax, which contains the reference to the
'bytevector-u8-ref' procedure.  When the macro returns this snippet
(or it could mix it into a larger syntax object as a subform), that
reference still functions; the macro ends up emitting code that
references 'bytevector-u8-ref' even though that was never bound in the
environment the macro was defined in.

Under IR, the procedural macro could have only emitted references to
bindings that are in the same module the procedural macro was defined
(its definition environment).  More precisely, every symbol the macro
returns would have been automatically bound to the environment in
which the macro was defined, with no chance of us telling the system
to look up some of them in another module.

Only in the syntax-case system, thanks to the fact that syntax objects
can be created and juggled around arbitrarily, we are able to call a
helper procedure from another module that gives us a piece of code
containing references to bindings in *that* module and not ours.  Thus
we get composable, modular procedural macros.


Actually, in SC and RSC...
--------------------------

I noticed this after writing the first part of this post.

The low-level MIT/GNU Scheme macro system that offers SC and RSC
macros actually has a capability that makes it equally powerful as
syntax-case: the 'capture-syntactic-environment' operator.  With this,
we can essentially imitate 'quasisyntax'; just call that operator to
get the current module environment, then use 'make-syntactic-closure'
to bundle a symbol with it, and you have the equivalent of a syntax
object.

I see no sense in burdening programmers with such low level facilities
however, and prefer the relative simplicity of syntax-case.
