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
