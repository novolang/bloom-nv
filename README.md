# bloom-nv

A **membership filter** answers one question — "have I seen this?" —
in a few bits per item, by allowing a bounded rate of wrong answers in
one direction only. The
[Bloom filter](https://dl.acm.org/doi/10.1145/362686.362692) (Burton
Bloom, 1970) is the original; the **counting** variant can remove an
item, and the
[cuckoo filter](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)
(Fan, Andersen, Kaminsky and Mitzenmacher, 2014) can remove one in less
space. This package brings all three to novo-lang. The reference
implementations are the Rust crate
[bloomfilter](https://docs.rs/bloomfilter) and the Python library
[pybloom](https://github.com/jaybaird/python-bloomfilter).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a membership filter is

A Bloom filter is an array of bits and a number of hash positions. To
add an item, set the bits at its positions. To test one, look at those
bits: if any is zero the item was **definitely never added**; if all
are one it **probably was**.

The error is one-sided. A `false` answer is certain. A `true` answer is
wrong with at most the probability the filter was sized for — a **false
positive**. There are no false negatives at all, which is what makes
the structure useful: a caller can skip an expensive lookup whenever
the filter says `false`, and be sure it skipped nothing real.

Two numbers fix everything else: how many items are expected, and what
false-positive rate is acceptable.

| Items | Rate | Bits each | Hash positions |
| --- | --- | --- | --- |
| Any | 10% | 4.8 | 3 |
| Any | 1% | 9.6 | 7 |
| Any | 0.1% | 14.4 | 10 |
| Any | 0.01% | 19.2 | 13 |

A tenfold better rate costs about five more bits per item, not ten
times the memory.

A plain Bloom filter cannot remove an item: clearing its bits would
clear bits other items need, and the filter would start giving false
negatives. The two variants here can.

| Filter | Removes | Memory | Fails by |
| --- | --- | --- | --- |
| Bloom | No | 1 bit per position | The rate climbing quietly |
| Counting Bloom | Yes | 4 bits per position | A counter saturating |
| Cuckoo | Yes | Fingerprints, less than Bloom below 3% | An insert being refused |

All three need a 64-bit hash. **That hash is the caller's**, passed in
as a function. One is enough: the several positions each item needs are
derived from the two halves of one hash value (Kirsch and
Mitzenmacher, 2006).

## Install

```
novo pkg add bloom-nv
```

## Example

```novo
use blbloom

// The caller's 64-bit hash. In a real program this is xxhash-nv's
// XXH3, or whatever the program already hashes its keys with.
fn key_hash(b: Bytes) -> Int
    bytes.len(b) * 2654435761

fn main() [io]
    // Room for 100_000 items, wrong once in a hundred `true` answers.
    match blbloom.new(100_000, 0.01, "key_hash")
        Err(e) => println("bad sizing: ${e.message()}")
        Ok(empty) =>
            println("${blbloom.bit_count(empty)} bits, ${blbloom.hash_count(empty)} positions")

            let seen = blbloom.add(empty, bytes.from_str("ada"), key_hash)

            // `false` is certain: skip the expensive lookup.
            if blbloom.contains(seen, bytes.from_str("grace"), key_hash)
                println("grace may be known — check the database")
            else
                println("grace was definitely never added")

            // Watch the fill: past its capacity the rate climbs silently.
            if blbloom.is_saturated(seen)
                println("rebuild larger: now ${blbloom.false_positive_rate(seen)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: bloom-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `blfault` | The eight ways a filter or an operation can be refused. |
| `blbloom` | The Bloom filter: sizing, add, test, the current rate, union and intersection. |
| `blcounting` | The counting Bloom filter, which can remove, and the two refusals that keep it honest. |
| `blcuckoo` | The cuckoo filter: fingerprints, relocation, removal, and the load factor to watch. |

## How to choose an entry point

**Take `blbloom` unless you need to remove items.** It is the smallest
and the simplest, and nothing about it can fail at run time.

**Take `blcuckoo` when items must be removed** and the target rate is
below about 3%: it uses less space than a Bloom filter there, and much
less than a counting one. Its cost is that an insert can be refused.

**Take `blcounting` when removals are frequent and inserts must never
fail.** It is sixteen times the memory of a plain filter, and it
degrades the way a Bloom filter does.

**`add_hashed` and `contains_hashed` are for a caller that already
hashed the item** for another reason, and should not pay twice.

## The rules a user needs

1. **`false` is certain and `true` is probable.** A filter never gives
   a false negative — unless rule 8 is broken.
2. **The hash is yours, and it must be the same one every time.** A
   filter fed through two hashes gives false negatives.
3. **Name the hash.** The label passed to `new` is compared on every
   union and interpreted nowhere.
4. **Size from the item count and the rate**, not from the memory.
   `new(capacity, target, …)` does the arithmetic:
   `m = -n ln p / (ln 2)^2` bits and `k = (m / n) ln 2` positions.
   `with_bits` is for matching another implementation.
5. **A target rate is in `0.0 ..< 1.0`, exclusive.** Zero would need an
   exact set; one is a filter that says yes to everything.
6. **Past its capacity, a Bloom filter goes on working and gets
   worse.** Nothing is refused and nothing complains.
   `is_saturated` and `false_positive_rate` are the two calls that tell
   you, and the second reports the rate at the current fill rather than
   the target.
7. **A plain Bloom filter cannot remove.** There is no `remove` in
   `blbloom`, deliberately.
8. **Only remove what you know you added.** Removing an item that was
   never added decrements counters, or clears a fingerprint, that
   belongs to items that were. Both filters refuse when they have proof
   — a zero counter, an absent fingerprint — and there is no proof in
   the case where the item collides with real ones. That case is the
   only way to make these structures report a false negative, and
   nothing can detect it.
9. **An item added twice must be removed twice.**
10. **A counting filter's counters hold 0 to 15.** An add that would
    overflow one is refused, because a wrapped counter turns a later
    removal into a false negative.
11. **A cuckoo insert can fail.** When both candidate buckets are full
    the filter relocates existing fingerprints, and after five hundred
    relocations it gives up with `BlCuckooFull`. A filter above about
    0.95 `load_factor` is full in practice.
12. **A cuckoo insert takes the random numbers it may need.** Which
    fingerprint to evict is a random choice, and this package draws
    nothing. `uniforms_needed` says how many are enough.
13. **A union is exact; an intersection is not.** The union of two
    filters is the filter that would have had both streams added. An
    intersection answers `true` for items whose bits are set in both
    for unrelated reasons, so its rate is not bounded by either
    input's, and `false_positive_rate` stops meaning anything for it.
14. **Union and intersection need identical shapes and hash labels.**
    `blfault.is_capacity_fault` separates "the filter is full" from
    "this call is wrong".

## Running on a microcontroller

Every module in this package builds for a microcontroller: no function
performs any input or output, reads a clock or draws a random number. A
device deciding whether it has already seen a packet identifier sizes a
filter once and keeps it in a fixed byte array.

Memory is exactly the sizing: one bit per position for `blbloom`, four
for `blcounting`, and `fingerprint_bits` per slot for `blcuckoo`. The
rate arithmetic uses `Float`; the filters themselves are integer
operations over bytes.

## What is not included

- **A hash function.** See rule 2. A caller that already has one should
  not get a second, and pinning one here would pin every consumer.
- **A random number generator.** See rule 12.
- **A scalable or partitioned filter**, which chains new filters as it
  fills. When to add one is a policy, and the policy is the caller's.
- **A stable Bloom filter and a quotient filter.** Both are
  worthwhile; neither is one of the three shapes most callers need.
- **Wire compatibility with another implementation's serialised form.**
  `to_bytes` is this package's own, carrying the parameters a union
  checks.
- **Counting how many times an item was seen.** A Bloom filter cannot;
  `blcounting.min_counter` is an upper bound over four-bit counters and
  saturates at fifteen. See sketch-nv.

## Related packages

- [sketch-nv](https://novo-lang.org/packages/sketch-nv) answers the
  questions these do not: how many distinct items, how often one
  appeared, and what the 99th percentile was. It takes its hash the
  same way.
- [xxhash-nv](https://novo-lang.org/packages/xxhash-nv) is a fast
  64-bit hash to pass in. So is crypto-nv's SHA-256, when the input is
  adversarial and a collision would be someone's doing.
- [lru-nv](https://novo-lang.org/packages/lru-nv) is the other half of
  a cache: a filter says whether to look, and a cache holds what was
  found.

## Tests

```bash
novo test tests/blbloom_tests.nv     # the one-sided error, the sizing, union and intersection
novo test tests/bldelete_tests.nv    # the two filters that forget, and their refusals
```

The normative sources are Bloom's 1970 paper for the filter and its
sizing, Kirsch and Mitzenmacher (2006) for deriving `k` positions from
one hash, and Fan et al. (2014) for the cuckoo filter. The reference
implementations are `bloomfilter` and `pybloom`. The suite asserts that
an item that was added is always found, that 1000 items at 1% gives
9586 bits and 7 positions, that a target of zero or one is refused,
that a filter past its capacity reports a current rate above its
target, that a union across two shapes or two hash labels is refused,
that removing an item whose counters are zero is refused, and that a
cuckoo filter's `BlCuckooFull` is a capacity fault rather than a
mistake in the caller.

The tests compile today and fail at run, each on the
`not implemented: bloom-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `blfault.BlFault`, `blbloom.BloomFilter`, `blcounting.BlCountingFilter`, `blcuckoo.BlCuckooFilter` | the types are declared |
| `blfault.code`, `.is_capacity_fault`, `BlFault.message` | no |
| `blbloom.new`, `.with_bits`, `.add`, `.add_hashed`, `.contains`, `.contains_hashed` | no |
| `blbloom.false_positive_rate`, `.target_rate`, `.is_saturated` | no |
| `blbloom.bit_count`, `.hash_count`, `.inserted_count` | no |
| `blbloom.union`, `.intersection`, `.to_bytes`, `.from_bytes` | no |
| `blcounting.new`, `.add`, `.remove`, `.contains`, `.min_counter` | no |
| `blcounting.present_count`, `.false_positive_rate`, `.hash_count`, `.to_bytes`, `.from_bytes` | no |
| `blcuckoo.new`, `.add`, `.contains`, `.remove`, `.uniforms_needed` | no |
| `blcuckoo.stored_count`, `.capacity_of`, `.load_factor`, `.false_positive_rate` | no |
| `blcuckoo.to_bytes`, `.from_bytes` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
