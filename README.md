# matchers-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Hamcrest's shape for novo-lang: **assertion combinators whose failures
read**.

```
expected a list of length 3, but was a list of length 2
the total: expected equal to 42, but was 41
element 1 was -1
```

rather than

```
expected true
```

A matcher knows two things a bare `test.assert(a == b)` does not: what
it was looking for, and what it found instead.  Every failure in this
package is those two joined into one sentence.

Six modules.

| surface | module | reach for it when |
| --- | --- | --- |
| the **assertion** | `matchers` | you are asserting, or writing a matcher of your own |
| the **combinators** | `matchcombine` | you are saying several things about one value |
| a **value** | `matchvalue` | equality, ordering, closeness |
| a **string** | `matchtext` | prefixes, substrings, patterns |
| a **list** | `matchcoll` | length, membership, order, quantifiers |
| a **Result or an optional** | `matchresult` | you have a `Result<T, E>` or a `?T` |

## Adding it, and checking it

```bash
novo pkg add matchers-nv        # into your novo.toml
novo pkg build                  # type- and effect-check the package
novo test --isolate tests/matchers_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: matchers-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use std.test
use matchers
use matchcombine
use matchvalue

@test
fn test_the_port_is_usable()
    let port = 70000
    let m: Matcher<Int> = matchcombine.all_of([matchvalue.at_least(1),
                                               matchvalue.at_most(65535)])
    matchers.assert_that_named("the port", port, m)
    // : the port: expected at least 1 and at most 65535, but was 70000
```

## The representation, and why it is a struct

```novo norun:pseudo
pub struct Matcher<T>
    judge: fn(T) -> MatchOutcome
    expected: Str
```

A function held in a struct, beside the sentence that says what the
function is looking for.  Every combinator in the package is a function
that returns one.

**Why not a trait**, which is what hamcrest has.  Two things in this
language stop it:

1. **A trait bound cannot carry a type argument.**  SPEC § 3.6's grammar
   is `trait_bound ::= TYPE_IDENT [ '[' ident ']' ]` — the optional
   bracket is the *effect* argument.  So
   `fn assert_that<T, M: Matches<T>>(…)` is a parse error, and a
   combinator generic over the value type cannot be written against a
   trait at all.
2. **`dyn Trait` costs the union of every impl in the program**
   (SPEC § 5.6).  A `core` package whose budget is `[]` cannot hold a
   trait object, because one effectful impl of that trait *anywhere* in
   the assembly puts an effect on it — and an assertion library is
   exactly the package a program implements that trait for in a dozen
   places.

**Why not one enum per value kind** — `MatchInt`, `MatchStr`,
`MatchList`, which is the other obvious answer.  A caller cannot extend
it: a matcher over a caller's own `Invoice` would have to be added to
this package.  With a closure in a field, `matchers.matcher` lets a
caller write one in three lines and compose it with everything here:

```novo norun:pseudo
let even: Matcher<Int> = matchers.matcher("an even number", judge_even)
matchers.assert_that(n, matchcombine.both(even, matchvalue.at_least(10)))
```

It is the same arrangement proptest-core-nv's `PropStrategy<T>` has, for
the same two reasons — which is worth more than either decision on its
own, because a person who has read one package can read the other.

**Three methods become one call.**  hamcrest has `matches`, `describe`
and `describeMismatch`, and the third is only ever called after the
first returned false.  Splitting them means judging the value twice,
which in a language with ownership means handing it over twice.  So the
stored operation is one — `judge` answers a `MatchOutcome` carrying both
the verdict and, when it is no, the reason — and `matchers.matches`,
`matchers.describe` and `matchers.describe_mismatch` are public
functions derived from it.  The hamcrest surface is all there; only the
storage is one field instead of three methods.

## `has_field` takes an accessor, because the language has no reflection

hamcrest's `hasProperty("total", equalTo(42))` looks a field up by
string at run time.  This language has no reflection and no row
polymorphism, so `fn has_field<T>(name: Str, …) -> Matcher<T>` cannot be
written at all — there is no way for a body to get from a `Str` to a
field of an arbitrary `T`.

What can be written is the same idea with the projection named instead
of the field:

```novo norun:pseudo
let m: Matcher<Invoice> =
    matchcombine.field_of("total", invoice_total, matchvalue.equal_to(42))
```

It costs one short accessor per field, and it is **checked**: a typo in
`"totl"` is a run-time failure in hamcrest and a compile error here.
`matchcombine.after` is the same shape with no label, for a projection
that is a computation rather than a field.

## Where the type inference shows

Two shapes need an annotated binding today, and both are visible in the
examples above:

- **A named function passed where `fn(T) -> F` is expected closes
  neither.**  `matchers.matcher(…, judge_even)` and
  `matchcombine.field_of(…, invoice_total, …)` need
  `let m: Matcher<Int> = …`.  That is filed against the toolchain, with
  a three-line repro; the signatures here are the ones they should be
  once it is fixed.
- **A type parameter that appears only inside a nested generic** —
  `has_len<T>() -> Matcher<[T]>`, `is_ok<T, E>() -> Matcher<Result<T, E>>`
  — closes from a *concrete* position and not from another generic
  call's parameter.  That one is documented behaviour rather than a
  defect: `E2016`'s own hint lists the positions a parameter closes
  from.

Neither changes what a matcher does; both change how a call site is
written, so every example in this package is written the way it has to
be written today.

## The regex is `std.regex`, and what that commits a test to

`matchtext.matches_regex` and `matchtext.contains_regex` run the
standard library's engine, which is prepended to every program,
effect-free at every tier, and **linear in time by construction** — so
they cost no dependency and no catastrophic backtracking.  The dialect
is a Perl-flavoured subset with leftmost-first alternation; the closest
relative is Go's `regexp`.

What a **portable** test should stay inside is smaller, and it has a
name: **I-Regexp (RFC 9485)** — literals, `.`, character classes, `|`,
`?`, `*`, `+`, `{n,m}` and grouping, with no anchors, no
backreferences, no lookaround, no lazy quantifiers and no POSIX classes.
jsonpath-nv scopes its `match()` to exactly that set, because a JSONPath
query crosses a wire and has to mean the same thing on the other side.

**A test does not**, so this package reports rather than refuses:
`matchtext.unportable_construct` answers `None` for a pattern inside
I-Regexp and names what took it outside otherwise — `"anchor"`,
`"lazy quantifier"`, `"word boundary"`, `"POSIX class"`.  A suite being
moved to another implementation can be told which of its patterns will
not move with it, and a suite that will only ever run here may use
everything `std.regex` has.

`matches_regex` is the whole string and `contains_regex` is any
substring, and they are two functions rather than one with a flag
because which of the two a test means is the most common mistake with a
regex assertion — and a boolean at a call site does not say which is
which.

## Three questions a list assertion can ask, kept apart

`matchcoll.contains_all` says the elements are there and says nothing
about where.  `matchcoll.in_order` says they appear in that order, with
anything allowed between.  `matchcoll.equal_list` says the list is
exactly that list.

A test that used the strictest when it meant the loosest breaks when
somebody changes a sort, and it breaks looking like a real failure.
Three names, three assertions, and the README saying so is most of the
protection there is.

## `assert_that` is effect-free, which looks wrong and is not

`test.fail` and `test.assert_false` declare no effects in the standard
library's own signatures — they write one runtime global that the
harness reads afterwards — so a pure function that asserts stays pure,
and a `@test fn` needs no effect row it did not already have.  That is
the same reason `test.case` can be called from a pure test.

## What this package does not depend on

**Not proptest-nv.**  It is a consumer of this package rather than a
dependency of it: a property's body asserts, and these are the
assertions it makes.

**Not a regex package**, for the reason above.

**Not a formatting package.**  A matcher renders the value it judged
through `Debug`, which SPEC § 3.8.1 gives to every struct and enum whose
fields have it — so a matcher over a caller's own type prints that type
without the caller writing anything.  `Display` would have been the
prettier bound and the wrong one: requiring it means a matcher over a
caller's type does not compile until the caller writes a formatter,
which is a bad first minute with the one library whose job is a readable
message.

## Naming

Every public type in this package starts `Match`, and every module file
starts `match`.  Struct and enum identity is keyed by **name** across a
whole assembly, dependencies included, so two packages that both declare
`Matcher` cannot be used by one program — and `Matcher`, `Assertion` and
`Outcome` are names some other package will want.

Two functions are not called what hamcrest calls them, and the reason is
the language rather than taste:

- **`is_not`, not `not`** — `not` is the boolean operator and a function
  cannot take the name.
- **`contains_text`, not `contains`** — `matchcoll.contains` is the list
  one, and a name that means two things is how a reader ends up
  asserting something they did not mean.

## The reference implementation

[hamcrest](https://hamcrest.org), whose vocabulary and whose
"expected …, but …" sentence this ports, and Rust's `assert_matches`
for the `Result` and optional shapes.

Apache-2.0.
