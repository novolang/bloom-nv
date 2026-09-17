# Changelog

All notable changes to bloom-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `blbloom` — the load-bearing promise, kept by the type: `false` is
  certain and `true` is probable, and there are no false negatives
  ever. The filter is sized from the two numbers a caller can reason
  about — how many items, what rate — and `false_positive_rate`
  answers the rate at the CURRENT fill rather than the target, because
  a filter past its capacity goes on working while quietly becoming
  useless. `union` is exact and `intersection` is not, and the doc
  comment says which number stops meaning anything after one.
- `blcounting` and `blcuckoo` — the two filters that can forget, and
  every operation that could turn the one-sided error into a false
  negative answers a `Result`: an increment that would wrap a counter,
  a remove whose counters prove the item was never added, and a cuckoo
  insert that ran out of relocations. A Bloom filter degrades
  gracefully and a cuckoo filter fails abruptly; both are said out
  loud rather than discovered.
- The hash is a `fn(Bytes) -> Int` parameter, and one hash is enough:
  the `k` positions come from the two halves of a 64-bit hash by
  Kirsch and Mitzenmacher's construction. The filter carries the
  caller's LABEL for the hash and refuses a union across two different
  labels, which would otherwise answer `false` for an item one of them
  holds.
- `blfault` — eight refusals, with `is_capacity_fault` separating
  "the filter is full, build a bigger one" from "your code is wrong".

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  bloom-nv.<module>.<fn>`.
- **One residual way to produce a false negative, and nothing can
  detect it.** Removing an item that was never added, whose positions
  collide with items that were, corrupts a counting or cuckoo filter.
  The refusals catch every case where the filter has proof; this one
  leaves none. The README states the rule: only remove what you know
  you added.
- **No scalable or partitioned filter**, which grows by chaining
  filters as it fills. It needs a policy for when to add one, and the
  policy is the caller's.
- **No stable Bloom filter, and no quotient filter.**
- **No wire compatibility with another implementation's serialised
  form.** `to_bytes` is this package's own, carrying the parameters a
  union has to check.
