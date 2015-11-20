# Arc Global Variables in an Arc Table

It turns out to be a surprisingly small change to Arc<sub>3.1</sub> to
implement Arc global variables using a plain Arc table:

https://github.com/awwx/arc-globals-in-table/compare/f67c9fb...3a08d9f

A reference to a global variable `a` then means the same thing as
`globals!a`, and `(= a 42)` means the same thing as
`(= globals!a 42)`.

I extended `eval` and `load` to accept a second optional argument,
the table to use for the global variables for the compiled code.

    arc> (eval '(plus 3 4) (obj plus +))
    7

Since the globals variable is a plain Arc table, you can make a
independent copy of the current set of global variables simply by
using `copy` to copy the table:

    arc> (let g (copy globals)
           (= g!+ 'plus)
           (eval '(+ 3 4) g))
    Error: "Function call on inappropriate object plus (3 4)"
    arc> +
    #<procedure:+>

If you want to "export" a variable from one namespace to another you
can simply copy it:

    (= g!foo h!foo)


## ar-globals

`ar-globals` is set to a table of the globals created by the Arc
runtime in `ac.scm`, before `arc.arc` or the other Arc libraries are
loaded.  Thus if you want to load some different version of `arc.arc`,
you could say:

    (= myarc (copy ar-globals))
    (load "myarc.arc" myarc)


## "Undefined" Global Variables

Since Arc tables don't distinguish between a value in a table being
`nil` and not being present at all, I was worried that this might make
calls to misspelled functions too confusing.

In Arc<sub>3.1</sub> refering to an undefined variable produces an
error:

    arc> foo
    Error: "_foo: undefined;\n cannot reference undefined identifier"

But since referring to a "missing" value in an Arc table simply
returns `nil`, this means that refering to any "undefined" global
variable acts like it has been defined with a value of `nil`:

    arc> foo
    nil

However, while Arc lists of at least one element can be called:

    arc> ('(a b c d) 2)
    c

Empty lists (`nil`) cannot:

    arc> ('() 2)
    Error: "Function call on inappropriate object nil (2)"

So a call to an "undefined" global still gives an error:

    arc> (foo)
    Error: "Function call on inappropriate object nil ()"

An alternative of course would be to change Arc tables to distinguish
between a key not being present and a key being present with a value
of `nil`.  But since calling an "undefined" function still produces a
recognizable error, I think this can be left to be an independent
decision.


## Implementation

Arc code is compiled by `ac` with a reference to the globals made
available to that compiled code.  The globals value becomes embedded
in the compiled code.

For example, a reference to a global variable `a` is compiled into
`(ar-funcall1 {globals} 's)`, where {globals} is the globals value
(the globals value itself, not a variable referring to the globals
value).

Thus the only way to have code use a different globals namespace is to
compile it again, giving a different globals value to the compiler.

In Arc code `globals` is compiled to the globals value, giving Arc
code a reference to its own globals value.

    arc> globals!do
    #(tagged mac #<procedure: do>)

Note that `globals` isn't a variable and can't be assigned to; it's
compiled specially by the compiler.

To embed the "globals" value into compiled code, we need to pass to
`ac` the "globals" value to use for the code.

I could have done this by adding `globals` as a parameter to every
`ac-` function, but that would have been tedious.  Instead I make
`globals` a parameter (a.k.a. a dynamic variable, or an implicit
variable as I like to call it), which makes the "globals" value
available to the `ac-` functions without having to pass it as a
parameter to each one.

Note that globals isn't parameterized in the runtime; this is just
for the convenience of the compiler.

`eval` now needs to be implemented in `arc.arc`, so that it has a
reference to the current globals to use as the default global
namespace to eval in.

In Arc<sub>3.1</sub> `eval` is implemented in the Scheme runtime:

    (xdef eval (lambda (e)
                  (eval (ac (ac-denil e) '()))))

However the runtime doesn't have a reference to `globals` to use as a
default.  (There's one runtime for all compiled code, regardless of
which globals value is being used for that code).

Putting `eval` in `arc.arc` gives it a reference to the current
globals to use as the default:

    (def eval (expr (o into globals))
      (ar-eval expr into))

And `eval` in `ac.scm` becomes `ar-eval`:

    (xdef ar-eval (lambda (e globals)
                    (eval (arc-compile (ac-denil e) globals))))

`load` also takes the second optional parameter, the "globals"
namespace to load the file into:

    (def load (file (o into globals))
      (w/infile f file
        (w/uniq eof
          (whiler e (read f eof) eof
            (eval e into)))))

Similarly `bound` also needs to be moved into `arc.arc`, so that it
knows which global namespace to look in to see if the variable is
bound or not:

    (def bound (name)
      (isnt globals.name nil))


After these changes `ac.scm` no longer uses Racket's
`namespace-variable-value` and `namespace-set-variable-value!` at all.
