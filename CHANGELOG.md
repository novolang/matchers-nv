# Changelog

Every published version, newest first.  This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0 — 2026-09-28

The first implementation of the interface published as 0.0.1.

### Added

- `matchers`: the outcome, the three Hamcrest readers derived from one
  judgement, the two assertions and the sentence they fail with,
  `described_as`, `anything` and `nothing_at_all`.
- `matchcombine`: `all_of`, which stops at the first clause that fails;
  `any_of`, which judges every clause and reports each distinct
  mismatch; `is_not`, `both`, `either`, `field_of` and `after`.
- `matchvalue`: equality, the four orderings, `between`, `close_to`
  with an inclusive tolerance, the two booleans and `one_of`.
- `matchtext`: prefix, suffix, substring, ASCII case-insensitive and
  trimmed equality, blankness, byte length, the two regular-expression
  matchers, `is_valid_pattern` and `unportable_construct`.  A subject
  longer than 64 bytes is shown as its first 64 bytes and its length.
- `matchcoll`: length, emptiness, membership, exclusion, exact
  equality, relative order, `every_item`, `any_item`, `item_at` and
  `has_length_that`.  A failure names the element or the index.
- `matchresult`: `is_ok`, `is_ok_with`, `is_err`, `is_err_with`,
  `is_some`, `is_some_with`, `is_none` and `unwrapped`.

### Behaviour the interface left open

- `between` with its bounds swapped matches nothing, and its
  description says the bounds are swapped.
- `unportable_construct` also names a backreference, a shorthand class,
  a lookaround and a group option.

### Toolchain

- The toolchain floor is 0.14.0.  The bodies target novo 0.14.0 and
  carry no workaround for a compiler defect.

### Tests

- 20 tests in two suites.  `tests/coverage.sh` merges the suites'
  line coverage over `src/`.

## 0.0.2 — 2026-09-16

README rewritten to the package README style guide
(docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Six modules.  `matchers` is the representation and the assertion,
  `matchcombine` the combinators, and `matchvalue`, `matchtext`,
  `matchcoll` and `matchresult` the matchers over a value, a string, a
  list and a `Result` or optional.
- **A matcher is a struct with a function-typed field**, not a trait,
  and the README argues it: SPEC § 3.6's trait bounds carry an effect
  argument and never a type one, so a combinator generic over the value
  type cannot be written against a trait at all — and a `core` package
  cannot hold a `dyn Trait`, which costs the union of every impl in the
  program.  It is proptest-core-nv's arrangement for the same two
  reasons.
- **Three hamcrest methods become one stored call.**  `describeMismatch`
  is only ever reached after `matches` returned false, and splitting
  them means handing the value over twice in a language with ownership.
  So `judge` answers a `MatchOutcome` carrying both, and `matches`,
  `describe` and `describe_mismatch` are derived from it — the hamcrest
  surface intact, with one field instead of three methods.
- **The failure is a sentence**, which is the row on the grid taken
  literally: `expected a list of length 3, but was a list of length 2`
  rather than `expected true`.  `failure_text` is where the two halves
  are joined so that every matcher's failure has one shape, and
  `assert_that_named` puts a subject in front for a table-driven test.
- **`has_field` takes an accessor and not a field name**, because the
  language has no reflection and no row polymorphism.  It costs one
  short function per field and it is checked by the compiler, which the
  string form never is.
- **`matches_regex` runs `std.regex` and reports rather than refuses.**
  The portable subset is I-Regexp (RFC 9485) and
  `matchtext.unportable_construct` names what takes a pattern outside
  it — jsonpath-nv refuses because a query crosses a wire, and a test
  does not.
- **`assert_that` is effect-free**, because `test.fail` is: a pure
  function that asserts stays pure.
- `tests/` holds nineteen API tests across two files, every one red.
  The suites assert the SENTENCES as much as the judgements, because a
  matcher that judged correctly and described itself badly would pass a
  suite that only checked `matches`.
