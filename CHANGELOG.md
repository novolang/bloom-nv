# Changelog

All notable changes to bloom-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-09-27

The first implementation of the interface published as 0.0.1.

### Changed

- A filter is changed in place.  `blbloom.add` and `.add_hashed` take
  the filter as a `var` parameter and answer nothing.
  `blcounting.add`, `.remove`, `blcuckoo.add` and `.remove` take it as
  a `var` parameter and answer `Result<Unit, BlFault>`.  In 0.0.1 each
  answered a new filter, and under reference semantics that is a copy
  of the whole bitmap on every add.  A program keeps a filter in a
  `var` binding.  A refused add or remove changes nothing.
- `BloomFilter` has a `target` field, which `blbloom.target_rate`
  reads, and its `bitmap` and `inserted` fields are `var`, as are the
  counters, the entries and the counts of the other two filters.
- `blbloom.with_bits` refuses a bit count or a hash count below one as
  `BlCapacityNotPositive`, and sets the capacity to `m ln 2 / k`.
- The counting filter is four times the memory of a plain one.  The
  README said sixteen.
- The README no longer says the package builds for a microcontroller.
  The filters hold `Bytes` and a `Str`, and a device build admits
  neither.

### Added

- The serialised layouts, documented at each `to_bytes`: `NVBF`,
  `NVCB` and `NVCF`, a version byte, big-endian fields, the hash label
  and the payload.
- `tests/layout_tests.nv`, positions and bytes worked out by hand, and
  `tests/rate_tests.nv`, the false-positive rate measured over seeded
  keys.
- `tests/coverage.sh`, which merges the suites' line coverage over
  `src/`.

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
