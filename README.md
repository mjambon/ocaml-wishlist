OCaml wishlist
==============

Smarter record and variant type inference
-----------------------------------------

Converting from one record type to another happens a lot
and is illustrated by the example below:

```ocaml
module Foobar =
struct
  type xy = { x : int; y : int }
end

module Barfoo =
struct
  type xyz = { x : int; y : int; z : int }
end

open Foobar
open Barfoo

let b_of_a (a : Foobar.xy) =
  {
    x = a.x;
    y = a.y;
    z = 0;
  }
```

fails in OCaml 3.12.0 with the following error message:

```
File "record.ml", line 16, characters 8-9:
    x = a.x;
        ^
Error: This expression has type Foobar.xy
       but an expression was expected of type Barfoo.xyz
```

Knowing that `a` was annotated by the user with type `Foobar.xy`,
the compiler should allow the expression `a.x`. Instead,
it looks for the latest definition of field `x`, which is provided
via `open Barfoo` and is the wrong type.

Currently the fix consists in writing:

```ocaml
let b_of_a a =
  {
    x = a.Foobar.x;
    y = a.Foobar.y;
    z = 0;
  }
```

which is pretty unreadable. Unfortunately this happens all the time in large 
applications. The same problem exists for variants.

Suggested semantics for `a.x`:

* if the type of `a` is already constrained to a record type, then 
  `.x` will be checked for compatibility with the type of `a` first.
  Optionally, the compiler could check that the module where the field comes
  from is open.
* if the type of `a` is not known to be a record type, then
  `.x` will determine the type of `a` using the current algorithm
  based on name of the field.


Features coming with the proposed changes:

* no need to define fields with an artificial prefix that stands for the type
  of the record
* a field or variant name could occur multiple times within the same module
  signature and still be usable (`id`, `name`, `value`, `x`, `String`,
  `And`, `Node`, ...)
* backward compatibility


### Further ideas to consider

The type of a record expression could be inferred from the set of 
  field names rather than field by field:

```ocaml
type p2 = { x : int; y : int }
type p3 = { x : int; y : int; z : int }

(* This, instead of having to place create2 before the definition of p3 *)
let create2 x y = { x; y }
let create3 x y z = { x; y; z }
```

Error messages could be adapted and improved. These typical errors occur:

* missing field names: if the fields are a subset of fields from 
  existing record types,
  then the missing fields based on these types should be suggested.
* extra field names: if the fields are a superset of fields from
  existing record types,
  then the removal of the extra field names should be suggested
  based on these types.
* misspelled/renamed field names: if some fields do not exist in any visible
  record type, then it is checked for types that contain all the remaining
  fields, and the missing fields are suggested.
* plain confusion: if some fields exist in different record types,
  all these possible record types should be suggested.
* no field exists: explicit module path syntax or `open` should be suggested
  (even better: suggest the actual modules that provide a compatible
  record type)

Similar error-reporting rules can be established for record patterns
and simple field access with the dot notation, taking into account that
only a subset of the fields is given.


Namespaces
----------

Authors of different libraries sometimes end up using identical module
names. This happens and when it happens, the user of both libraries
needs to copy and locally modify one of the two.

This shouldn't happen with namespaces, which allow to put modules into
different namespaces without touching the libraries
that provide the modules.

[OCamlPro/ocaml-namespaces](https://github.com/OCamlPro/ocaml-namespaces)
looks like it implements just what we need and we are all waiting impatiently
for it to make it into standard OCaml.


Generic print function
----------------------

We would like a function that takes any OCaml value and prints it like in the
toplevel.

```ocaml
val to_string : ?full: bool -> 'a -> string
  (** Produce a human-readable representation of an OCaml value.
      @param full if true, the value must be printed in full.
                  By default, long values may be truncated using [...]
  *)

val print : ?full: bool -> 'a -> unit
  (** [print x] is the same as [print_endline (to_string x)]. *)
```

Details:

* it would print functions as `<fun>`
* it would print abstract values as `<abstract>`
* polymorphic values such as the elements of an `'a list`
  could be either rejected at compile time or printed as `<poly>`.

In terms of implementation, we understand that a
representation of the type should be available at runtime, which
is not currently the case. Since we don't want to modify the current runtime,
the representation of the type would not be a subfield of the value
to be printed.
As a consequence, `to_string` and `print` would require a special treatment
by the compiler. Much like `assert` expands into something containing
its location, `to_string x` would expand into something like
`unsafe_to_string (typeof x) x`. The special `typeof` construct 
may or may not be made available to the user for further applications.
