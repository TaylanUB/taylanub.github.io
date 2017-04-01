% What are macros?

Some languages have a special syntax like the following:

    try {
      blah
    } catch (FooException e) {
      blub
    } finally {
      blip
    }

which executes `blah`, handles exceptions of type `FooException` with
`blub`, and always calls `blip` to clean-up stuff at the end (no
matter if an exception was raised or not), like closing a port that
was opened in `blah`.  (If we had called `close()` on that port at the
end of `blah`, it might not be executed if an exception interrupts the
flow of code.)

Now consider that your language does *not* have that `finally{}`
syntax feature, and you're forced to do the following each time:

    try {
      blah
    } catch (AnyException e) {
      if ( isFooException(e) ) {
        blub
      }
      blip
      if ( ! isFooException(e) ) {
        throw(e)
      }
    }

This is incredibly annoying.  Normally one can write functions to
encapsulate common patterns of code, but can you write a function to
encapsulate the above pattern?  Difficult, eh?

(If your language supports first-class functions, you could actually
write a `tryCatchFinally` function which takes three functions as its
arguments, so this is not the best example (and should remind us that
*many* situations where we think "yay I'll use a macro!" might be
better served with a higher-order function, i.e. a function that takes
other functions as arguments), but let's ignore that for a moment;
there are often cases where even a higher-order function cannot
encapsulate a common pattern of code.)

In a language without macros, you're forced to write repetitive code
in many cases if the language doesn't have firsthand *syntactic*
support to encapsulate a common coding pattern.  But when you have
macros, you can encapsulate those patterns.  Macros don't take values
as arguments, instead they take unexecuted *code snippets* (variable
names, expressions, statements, etc.) as their arguments, and can
inject code around those snippets to build common code patterns.

Macro definitions often use pattern-matching on the syntax of the
language; the following pseudocode uses such a system, where the
formal parameters of the `tryFin` macro definition are not necessarily
identifiers (like the variable names in function parameters), and
instead may be code templates:

    defineMacro tryFin(tryBody,
                       <optional> 'catch' (ExcType excVar) { catchBody },
                       'finally' { finBody }) {
      try {
        tryBody
      } catch (AnyException e) {
        if ( isType(ExcType, e) ) {
          excVar = e
          catchBody
        }
        finBody
        if ( ! isType(ExcType, e) ) {
          throw(e)
        }
      }
    }

(Assume that our macro system is so intelligent that it automatically
deletes out the two if statements when the optional code-snippet
argument has not been provided, because those if statements refer to
identifiers that only appear in the optional parameter.)

(In this pseudocode I used `'singleQuoted'` identifiers to express
those which are expected to appear verbatim as *keywords* in the
argument snippets while calling the macro, instead of serving as
template variables that get bound to (some part of) an argument
snippet.)

Now you can write:

    tryFin({
      blah
    }, catch (FooException e) {
      blub
    }, finally {
      blip
    })

Ain't that neat?
