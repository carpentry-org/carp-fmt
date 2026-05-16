# carp-fmt style

`carp-fmt` is a Wadler-style pretty printer with a small, fixed table of
rules. It walks the parsed source as a document tree, decides per-group
whether to render flat or broken, and emits text. The output target is 80
columns with 2-space indent. Both the page width and the rule table are
hard-coded; the formatter is opinionated, not tunable.

What follows describes what the formatter will do to your source. The
inverse (or "what your source has to look like before carp-fmt accepts
it") is the same thing, since carp-fmt's output is idempotent, i.e. feeding its
output back in is a fixed point.

## Function calls

A list whose head isn't a known special form is treated as a function
call. `carp-fmt` picks one of two layouts based on the width of the first
argument.

When the first argument is short (i.e. its flat rendering fits within 40 columns)
subsequent arguments hang under it:

```clojure
(and (= a 1)
     (> b 3))
```

When `arg0` is wide, such as a multi-line lambda or a complex compound
expression, the operator goes alone on the head line and all arguments
flow at an indent:

```clojure
(Array.reduce
  &(fn [a s] ...)
  init
  coll)
```

This is inspired by [zprint](https://github.com/kkinnear/zprint)'s hang/flow
heuristic: hanging produces nicer output when the alignment column is shallow,
but pushes arguments into a too-narrow column when the first argument is already
large.

## Special forms

For known forms `carp-fmt` has an explicit head-line count (HLC) rule.
HLC says how many leading items stay on the head line. The remaining
items become the indented body.

| HLC | Forms |
|---|---|
| 1 | `do`, `cond` |
| 2 | `def`, `defdynamic`, `defmodule`, `deftype`, `fn`, `let`, `let-do`, `when`, `when-do`, `unless`, `unless-do`, `if`, `for`, `while`, `while-do`, `match`, `match-ref`, `doc`, `hidden`, `private`, `implements`, `load`, `register`, `register-type`, `deftest`, `assert-equal`, `assert-true`, `assert-false`, `with-test` |
| 3 | `defn`, `defn-`, `defndynamic`, `defn-do`, `defmacro` |

The HLC=3 entries are the forms whose name appears between the
operator and the first body element (`(defn name [args] body)`). HLC=2
covers the bulk of the special forms, i.e. anything that takes one
significant argument (binding vector, condition, etc.) and then
a body. HLC=1 is reserved for forms where every body item carries
equal weight (`do`) or has its own paired-body discipline (`cond`).

```clojure
(defn add [x y]
  (+ x y))

(let [a 1
      b 2]
  (+ a b))

(do
  (foo)
  (bar))
```

### Paired bodies

`match`, `match-ref`, and `cond` have an additional rule: their body
items aren't independent, instead they pair as `(pattern result)`. Each pair
forms its own group. A pair stays on one line when it fits. If it
doesn't, the pattern goes on its own line and the result is indented.

```clojure
(match x
  (Maybe.Just v) v
  (Maybe.Nothing) default)

(cond
  (= x 0) "zero"
  (positive-result-that-doesnt-fit-inline x)
    (other-branch))
```

## Pair-aware arrays

Three constructs in Carp use arrays whose elements semantically pair:
`let`-style binding vectors (`[k1 v1 k2 v2 …]`), plain-`deftype` and
`register-type` field arrays (`[name type name type …]`), and `Dict`
literals (`{k1 v1 k2 v2 …}`). `carp-fmt` renders all three the same way:
a key and its value stay on one line, pairs separated by soft breaks
that align under the first key when broken.

```clojure
(let [first 1
      second (some-expr)
      third (another-expr)]
  body)

(deftype URI
  [scheme (Maybe String)
   host (Maybe String)
   port (Maybe Int)])

{:name "carp"
 :version "0.6"}
```

When a bindings array contains an inline comment, the comment stands
on its own line between pairs and the pairing of surrounding entries is
preserved:

```clojure
(let-do [pos 0
         ; skip past first \r\n
         headers (parse-headers s)
         body @""]
  body)
```

## Bracketed forms

For plain arrays (`Form.Arr`, `Form.StaticArr`) and any `Form.Dict`
without pair-able contents, carp-fmt opens the bracket and puts the
first child on the same line. Subsequent children align under it:

```clojure
[1
 2
 3]

$[byte1
  byte2
  byte3]
```

This avoids the "dangling bracket" style (`[\n  1\n  2\n]`) that
some Lisp formatters produce by default.

## Auto-inlining

When the flat-rendered form fits within the remaining page width,
carp-fmt collapses it to a single line. This makes short calls and
short bindings stay compact regardless of how the source was written:

```clojure
; source
(defn id [x]
  x)

; output
(defn id [x] x)
```

There are four exceptions. `carp-fmt` keeps the form multi-line when:

- it contains an inline comment,
- it has a blank line between siblings,
- it's a `deftype` with two or more `Lst` variant items (a sum type), or
- it's a `defmodule` with two or more body forms.

The first two are about preserving structure the author put there. The
last two are about visual grouping: a sum type's variants belong
one-per-line, and a module that defines multiple things isn't a one-liner
even when it would fit.

## Reader macros

Synthetic `(ref x)` / `(copy x)` / etc. forms produced by the reader
are rendered back as their reader macro:

| Form | Prefix |
|---|---|
| `(ref x)` | `&x` |
| `(copy x)` | `@x` |
| `(deref x)` | `~x` |
| `(quote x)` | `'x` |
| `(quasiquote x)` | `` `x `` |
| `(unquote x)` | `%x` |
| `(unquote-splicing x)` | `%@x` |

## Values

`Double` values are emitted as plain decimal. The
formatter walks increasing fractional precisions (`%g` first, then
`%.17g`, then `%.Pf` for P up to 330) and picks the shortest
representation that round-trips to the same `Double`. A trailing `.0`
is appended when the rendered value otherwise has no decimal point,
so it parses as `Double` and not `Int`.

`String` literals preserve their content byte-for-byte. CRLF
(`\r` followed by `\n`) is emitted as the escape pair `\r\n`.
Standalone `\n` is left as a raw newline so multi-line doc strings
stay legible.

## Whitespace

Blank lines between top-level forms are preserved, collapsed to at most
one. Blank lines between siblings inside a compound form are also
preserved. Trailing whitespace on lines is removed.
