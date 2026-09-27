# matchers-nv

A matcher is a value that both judges another value and says, in words,
what it was looking for. An assertion written with one fails with
`expected a list of length 3, but was a list of length 2` rather than
with `expected true`. The vocabulary and the sentence shape are
[Hamcrest](https://hamcrest.org)'s, and the matchers over a `Result` and
an optional follow Rust's `assert_matches`. This package brings both to
novo-lang.
[httpmock-nv](https://novo-lang.org/packages/httpmock-nv) is built on it:
a header or body expectation there is a `Matcher<Str>` from here.

## What it is

A **matcher** over a type `T` is a `Matcher<T>`: a function that judges
one value of that type, together with the sentence that says what it
wants. `matchvalue.equal_to(42)` is a `Matcher<Int>` whose sentence is
`equal to 42`. Every function in this package that produces a matcher is
called a **combinator**, whether it builds one from nothing or from other
matchers.

Judging a value answers a **`MatchOutcome`**: whether it matched, and if
it did not, a clause saying what was found. The two halves are written by
the matcher, because the matcher is the only thing that knows both.
`matchers.failure_text` joins them into the sentence a failure prints.

| Part of the sentence | Where it comes from | Example |
| --- | --- | --- |
| the subject | `assert_that_named`'s first argument | `the total` |
| `expected …` | the matcher's own description | `expected equal to 42` |
| `but …` | the outcome of judging this value | `but was 41` |

Combinators compose. `matchcombine.all_of` takes a list of matchers over
one type and makes a matcher that requires all of them.
`matchcoll.every_item` takes a matcher over an element and makes a
matcher over a list of them. `matchcombine.field_of` takes a function
that reads a field and a matcher over that field, and makes a matcher
over the whole structure.

A matcher is a struct holding a function, not a trait. A caller with a
rule this package does not have writes it with `matchers.matcher` in
three lines, and it then composes with everything here.

Nothing in this package reads a file, opens a socket or prints. Judging a
value builds a string, and the assertion hands that string to `std.test`.

## Install

```
novo pkg add matchers-nv
```

## Example

```novo
use std.test
use matchers
use matchcoll
use matchcombine
use matchvalue

@test
fn test_the_ports_are_usable()
    let ports = [80, 443, 70000]

    // A matcher for one port: at least 1 and at most 65535.
    let usable: Matcher<Int> = matchcombine.all_of([matchvalue.at_least(1),
                                                    matchvalue.at_most(65535)])

    // Lift it over the list, so every element must satisfy it.
    let all_usable: Matcher<[Int]> = matchcoll.every_item(usable)

    // Assert, naming the subject so the failure says what was wrong.
    matchers.assert_that_named("the ports", ports, all_usable)
    // the ports: expected every item at least 1 and at most 65535,
    // but element 2 was 70000
```

Build and test with `novo pkg build` and `novo test`.

## What the package contains

| Module | Contents |
| --- | --- |
| `matchers` | `Matcher<T>`, the outcome of a judgement, the two assertions, the three Hamcrest readers, the sentence they are reported as, and the escape hatch for a matcher of your own. |
| `matchcombine` | Matchers built from other matchers: all, any, negation, the two-argument forms, and the two that judge a part of a value reached through a function you supply. |
| `matchvalue` | Equality, ordering, a range, closeness for floating point, the two booleans, and membership of a fixed set. |
| `matchtext` | Prefix, suffix, substring, case-insensitive and trimmed equality, blankness, length, and the two regular-expression matchers with a check for what is portable. |
| `matchcoll` | Length, emptiness, membership, exclusion, exact equality, relative order, the two quantifiers over elements, one element by index, and a matcher over the length. |
| `matchresult` | `Ok` and `Err` with or without a matcher over the payload, `Some` and `None`, and the one that judges the value inside an `Ok` and fails on an `Err`. |

## How to choose an entry point

**`matchers.assert_that` is the ordinary call.** It judges the value and
fails the test with the sentence if the matcher did not match.

**`matchers.assert_that_named` puts a subject in front of the sentence.**
Use it in a table-driven test, where forty rows run the same assertion
and the message alone cannot say which row failed.

**`matchers.check_that` judges without asserting.** It answers the
outcome, for a validation pass that collects every reason rather than
stopping at the first.

**`matchers.matches`, `.describe` and `.describe_mismatch` are Hamcrest's
three methods.** They are derived from `check_that` and are here for code
written against that vocabulary.

**`matchers.matcher` is how you add one.** Give it a noun phrase and a
judging function, and the result composes with every combinator in the
package.

## The rules a user needs

1. **A matcher's description is a noun phrase; a mismatch is a clause.**
   The description completes "expected …", so `"an even number"`, not
   `"is even"`. The mismatch completes "but …", so `"was 4"`, with no
   leading capital and no full stop. Both rules matter only when you
   write your own matcher with `matchers.matcher`.
2. **A matcher that matched has no mismatch text.** `MatchOutcome`'s
   `mismatch` is absent when `ok`, rather than being an empty string. "It
   matched" and "it did not match and the matcher had nothing to say" are
   different facts.
3. **Membership, order and equality are three questions.**
   `matchcoll.contains_all` says the elements are present and says
   nothing about where. `matchcoll.in_order` says they appear in that
   order with anything allowed between. `matchcoll.equal_list` says the
   list is exactly that list. A test that used the strictest when it
   meant the loosest breaks when somebody changes a sort, and it breaks
   looking like a real failure.
4. **`matchresult.is_err` is the weak assertion, and `is_err_with` is
   not.** `is_err` passes for any failure, so a function that starts
   failing for a new reason still satisfies it. Use `is_err_with` with a
   matcher over the error when the test is about a specific refusal.
5. **`matchtext.matches_regex` is the whole string; `contains_regex` is
   any substring.** They are two functions rather than one with a flag,
   because which one a test means is the most common mistake with a
   regular-expression assertion.
6. **A pattern that does not compile matches nothing and says so.** A
   matcher is built before anything is judged and has nowhere to put a
   refusal, so the mismatch names the engine's own error.
   `matchtext.is_valid_pattern` checks a pattern up front.
7. **A portable pattern stays inside I-Regexp, and this package reports
   rather than refuses.** I-Regexp ([RFC 9485](https://www.rfc-editor.org/rfc/rfc9485))
   is literals, `.`, character classes, `|`, `?`, `*`, `+`, `{n,m}` and
   grouping, with no anchors, no backreferences, no lookaround and no
   lazy quantifiers. `matchtext.unportable_construct` answers nothing for
   a pattern inside that set, and names what took it outside otherwise:
   `"anchor"`, `"backreference"`, `"shorthand class"`, `"lookaround"`,
   `"group option"`, `"lazy quantifier"`, `"word boundary"` or
   `"POSIX class"`.
   A suite that will only ever run on this toolchain may use everything
   the standard library's engine has.
8. **Two names differ from Hamcrest's, and the language is the reason.**
   `matchcombine.is_not` rather than `not`, because `not` is the boolean
   operator. `matchtext.contains_text` rather than `contains`, because
   `matchcoll.contains` is the list one.
9. **`matchcombine.field_of` takes a function, not a field name.** The
   language has no reflection, so a matcher cannot get from a string to a
   field of an arbitrary type. Passing the accessor costs one short
   function per field and is checked by the compiler: a misspelled field
   name is a build error here and a run-time failure in Hamcrest.
   `matchcombine.after` is the same shape with no label, for a projection
   that is a computation rather than a field.
10. **A matcher whose type appears only in its answer needs a typed
    position.** `matchcoll.has_len(3)` and `matchresult.is_ok()` take no
    argument of the judged type, so the type closes from an annotated
    binding, `let m: Matcher<[Int]> = matchcoll.has_len(3)`, or another
    concrete position, and not from another generic call's argument.
    Error `E2016`'s hint lists the positions.
11. **A matcher prints the value it judged through `Debug`.**
    SPEC section 3.8.1 gives `Debug` to every struct and enum whose
    fields have it, so a matcher over your own type prints that type
    without your writing a formatter.
12. **`matchers.described_as` replaces the sentence and leaves the
    judging alone.** Use it when a composed matcher's generated
    description is accurate and unreadable, and the test knows a better
    name for what it is checking.
13. **Asserting declares no effects.** `test.fail` and
    `test.assert_false` declare none in the standard library's own
    signatures, so `matchers.assert_that` declares none either, and a
    `@test` that uses it needs no effects it did not already have.

## What is not included

- **A test runner.** `std.test` in the standard library is the harness,
  and it is available to every program without a dependency.
- **A trait for matchers.** A trait bound carries an effect argument and
  never a type argument (SPEC section 3.6), so a combinator generic over
  the value type cannot be written against a trait. A trait object is
  charged the effects of every implementation of that trait in the
  program (SPEC section 5.6), which a package that declares no effects
  cannot accept. `Matcher<T>` is a struct holding a function for those
  two reasons.
- **A fixed set of matcher kinds.** One enum per value kind would be
  closed: a matcher over your own type would have to be added to this
  package. `matchers.matcher` is the alternative.
- **Looking a field up by name at run time.** See rule 9.
- **A regular-expression dependency.** `matchtext` runs the standard
  library's engine, which is available to every program, declares no
  effects and is linear in time, so there is no catastrophic
  backtracking. Its dialect is a Perl-flavoured subset with
  leftmost-first alternation, closest to Go's `regexp`.
- **A formatting dependency.** A matcher renders through `Debug`.
  Requiring `Display` would mean a matcher over your own type does not
  compile until you write a formatter.
- **A dependency on a property-testing package.** proptest-nv is a
  consumer of this package rather than the other way round: a property's
  body asserts, and these are the assertions it makes.
- **Running on a microcontroller.** No such claim is made. The failure
  message is the point of this package, and the embedded runtime does not
  link strings.

## Related packages

- [httpmock-nv](https://novo-lang.org/packages/httpmock-nv) is a stub
  HTTP server for tests. A header or body expectation there is a
  `Matcher<Str>` from here, so the failure when a request does not match
  reads like every other assertion in the suite.
- [proptest-nv](https://novo-lang.org/packages/proptest-nv) runs a
  property over many generated inputs. A property body can judge with
  `matchers.check_that` and pass the description to
  `propcheck.holds_if`.
- [snapshot-nv](https://novo-lang.org/packages/snapshot-nv) compares a
  value against a stored copy of itself. Take that package when the
  expected value is too large to write out, and this one when the test
  is about a named property of the value.
- `std.test` in the standard library is where a novo-lang `@test`
  reports. This package calls `test.fail` inside `assert_that` and
  `assert_that_named`, and nowhere else. Every assertion here is
  therefore an ordinary test failure.

## Tests

```bash
novo test --isolate tests/matchers_tests.nv      # 10 tests: the representation and the sentence
novo test --isolate tests/matchsubject_tests.nv  # 10 tests: the matchers over each kind of value
```

The oracle is Hamcrest's own vocabulary: `equal_to`, `all_of`, `any_of`,
`close_to`, `starts_with`, the quantifiers and `describedAs`, and the
"expected …, but …" sentence they produce. Where a name had to change,
the suite records why.

What is asserted is mostly the text. A matcher library's claim is that
its failures read, so the tests check what `describe` says, what
`describe_mismatch` says, and what `failure_text` makes of the two. A
matcher that judged correctly and described itself badly would pass a
suite that only checked `matches`. The suite also asserts that a whole
match and a search are different assertions, that an unportable pattern
is reported and not refused, that the list matchers name the element that
failed, that membership, order and equality stay three questions, that a
`Result` matcher tells a failure from a wrong value, and that the
optional matchers print what was there instead.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
