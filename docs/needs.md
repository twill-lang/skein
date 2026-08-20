# What skein needs from twill

skein is written in twill and does not run yet. This file is the reason: the
language, runtime and data features the source uses that twill does not provide
today, with the file and function that needs each one, and what skein does in
the meantime.

It is a work queue for the language, not a complaint. Every entry was reached by
writing real code and hitting the wall, which is the only way a list like this
is worth anything. Where skein has a workaround the workaround is described,
because how ugly a workaround is measures how badly the feature is wanted.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64` with bitwise operations, `Str` with length and byte
indexing, `Bytes`, `Arr[T]`, `Dict[Str, V]` and `Dict[I64, V]`, and `struct`
with reference semantics. Everything below is on top of that.

Several entries duplicate an entry in twill's own `docs/needs.md`. Where they
do, the twill number is named and the entry says what skein adds to it, if
anything. Duplicates are kept rather than deleted: a second repository hitting
the same wall for a different reason is evidence about priority, and deleting
the entry throws that evidence away.

## Blocking: skein cannot run at all without these

### 1. A Unicode character database, or an honest replacement for one

**Needs:** canonical decomposition and composition mappings, canonical
combining classes and composition exclusions, and general category data
**Used by:** `src/normalize.tw` (`apply`, `fold_latin1`, `latin1_fold`),
`src/pretok.tw` (`is_punct`)
**Status:** nothing in the language or the standard library. `std/text.tw` has
UTF-8 decoding (`char_starts`, `char_size`, `char_code`) and no tables at all,
and says so in its own header.

skein does the part that is expressible without a table and refuses the part
that is not. `NormSpec.nfc` returns a message rather than approximating, because
a normaliser that silently performs 3% of NFC fails on exactly the inputs nobody
has in their test corpus. Accent folding covers the Latin-1 Supplement block,
U+00C0 to U+00FF, which is 64 entries written out in `latin1_fold` and is a
table small enough to review rather than a database. Case folding is ASCII only.
Punctuation detection in `pretok.is_punct` is ASCII only, so a typographic quote
or an ideographic full stop is treated as part of the word it touches.

What the workaround costs: a corpus in any script that uses combining marks
normalises inconsistently, so the same word written two ways gets two
vocabulary entries and half the training signal each. That is the single largest
correctness gap in the repository and it cannot be closed from twill source.

This is not a duplicate of a twill entry. Twill's NEEDS-61 asks for `trim_space`
over Unicode rather than ASCII, which is one predicate from the same missing
data, and NEEDS-15 covers non-ASCII whitespace in the lexer. Both are the small
end of this.

### 2. Membership and iteration on a `Dict`

**Needs:** `dict_has(d, k) -> Bool`, and a way to iterate a `Dict`'s keys
**Used by:** `src/vocab.tw` (`id_of`, `contains`, `observe`, `count_of`,
`load`), `src/bpe.tw` (`rank_of`, `count_pairs`, `load_merges`)
**Status:** the lookup half is RESOLVED in 1.6 (NEEDS-22). Iteration (NEEDS-8)
is still open.

`dict_get` returns `Opt[V]`, which is what this entry said would be better than
`dict_has` plus an index. `vocab.id_of` and `bpe.rank_of` are both one
`dict_get` now. That removed their -1 returns, and it also halved their hashing:
`dict_has` followed by an index hashed the key twice, and those two functions
are the innermost lookups in the package.

Iteration is still worked around below, unchanged.

Iteration is worked around by keeping a parallel `Arr` of the keys next to every
`Dict`: `vocab.Counts` has `keys`, `vocab.Vocab` has `toks`. That doubles the
memory of a frequency table and it is a real invariant to maintain, because the
array and the dictionary can disagree and nothing checks that they do not.
`vocab.observe` is the only function allowed to write either, which is how the
invariant is kept, and it is kept by discipline rather than by the type.

### 3. Sorting

**Needs:** a sort over `Arr[T]` with a caller-supplied comparison
**Used by:** `src/vocab.tw` (`sort_index`, called by `build`),
`src/sequence.tw` (`sort_by_length`, called by `bucket`)
**Status:** twill's NEEDS-23 asks for sorting an `Arr[Str]`. **This entry
duplicates NEEDS-23** and widens it: skein needs a comparison over two parallel
arrays, not over the elements of one.

skein has written two sorts. `vocab.sort_index` is a bottom-up merge sort over
an index array, because a vocabulary is tens of thousands of entries and it has
to be stable so that the id assignment is a function of the data alone.
`sequence.sort_by_length` is an insertion sort, because it sorts pools of a few
hundred and insertion sort is faster and simpler at that size.

That is the third and fourth hand-rolled sort in this ecosystem. The cost is not
the code, it is that stability is now a property four separate implementations
have to keep, and a vocabulary built by an unstable sort silently reassigns ids
between runs.

### 4. Nested containers

**Needs:** `Arr[Arr[Str]]`, and an `Arr` whose element type is a struct
**Used by:** `src/bpe.tw` (`train`, `count_pairs`, the `words` inventory),
`src/encoding.tw` (`Batch.items` is `Arr[Encoding]`), `src/sequence.tw`
(`bucket` takes `Arr[Encoding]` and returns `Arr[Batch]`)
**Status:** twill's NEEDS-72 covers nested containers. **This entry duplicates
NEEDS-72.**

There is no workaround. BPE training holds every distinct word as its current
symbol list and rewrites the lists in place on each merge pass; flattening that
into one array with an offset table is possible and would make `apply_merge`
rewrite the whole corpus rather than one word. Batching is an array of
encodings by definition.

### 5. `Res[T, E]`, or multiple returns

**Used by:** every fallible function in the package. `src/normalize.tw`
(`apply`), `src/vocab.tw` (`add_reserved`, `load`), `src/bpe.tw`
(`load_merges`), `src/unigram.tw` (`set_logp`, `fit`, `validate`),
`src/encoding.tw` (`specials_from`, `register_extra`, `pad_to`,
`pad_batch_longest`), `src/embed.tw` (`check_ids`, `batch_ids`, `batch_mask`)
**Status:** done for everything this entry named (2026-08, on twill 1.7). The
out-parameter pattern is gone from the package entirely; a plain
empty-string-on-success return survives in places where nothing is returned
alongside it, and that is a different and much smaller thing.

`Res[T, E]`, `Opt[T]` and postfix `?` all exist. The `Opt` half went first,
where a function had one thing to return or nothing: `vocab.id_of`,
`vocab.from_hex` (which also absorbed `hex_ok`, so its callers each lost a line
and the file is read once instead of twice), `bpe.rank_of`, `encoding.lookup`
and the `Specials` ids.

Then the six that had to return a value *and* an error, which is what forced the
out-parameter:

| was | is |
| --- | --- |
| `normalize.apply(spec, s, out: Norm) -> Str` | `Res[Norm, Str]` |
| `vocab.load(s, out: Vocab) -> Str` | `Res[Vocab, Str]` |
| `bpe.load_merges(s, out: Bpe) -> Str` | `Res[Bpe, Str]` |
| `encoding.specials_from(v, .., out: Specials) -> Str` | `Res[Specials, Str]` |
| `embed.batch_ids(b, out: Arr[I64]) -> Str` | `Res[Arr[I64], Str]` |
| `embed.batch_mask(b, out: Arr[I64]) -> Str` | `Res[Arr[I64], Str]` |

Every one of those made the caller allocate a value before knowing whether the
call would succeed, and then read it without being made to check. `blank()` and
the other empty constructors that existed only to feed them are deleted.

`embed` was the sharpest case and is also fixed. It returned a tensor and so
could not also return an error, which made the check a separate
`check_encoding` call the caller had to remember -- and this entry called that
"exactly the API shape this library exists to argue against elsewhere". It is
`embed(p, e, vocab_size) -> Res[Tensor, Str]` now and does the check itself.
`check_encoding` stays, for a caller validating a batch before embedding any of
it.

**What is left, and why it is not the same problem.** `encoding.register_extra`,
`pad_to`, `pad_batch_longest`, `pad_batch_fixed`, `unigram.set_logp`, `fit`,
`validate`, `embed.check_ids` and `encoding.assert_shape` still return a `Str`
that is empty on success. None of them has a value to return alongside it: they
mutate in place or they are pure checks, so there is no out-parameter and no
half-built value for a caller to read. Moving them to `Res[Unit, Str]` is
tidier and is worth doing, but it buys a type rather than removing a way to be
wrong, which is what the six above did.

### 6. A function type, for the tokeniser seam

**Needs:** a function value passed as a parameter, and a spelling for its type
**Used by:** nothing today, and `src/encoding.tw` (`build_single`,
`build_pair`) wants it
**Status:** functions are values in numeric twill; whether a systems-mode
function may take one, and how the type is spelled, is not stated anywhere.
loom's `docs/needs.md` entry 3 asks the same question for its step function.

skein's five models all produce `Pieces` and all have the same shape, and the
natural design is for `build_single` to take the model's encode function.
Instead every call site calls the model itself and passes the resulting
`Pieces`, which works and means the driver cannot enforce that the pre-tokeniser
a model was trained with is the one it is being encoded with. That mismatch is
silent and produces subtly wrong tokenisation.

## Blocking: features the source assumes exist

### 7. Integer division and modulo on `I64`

**Used by:** `src/vocab.tw` (`to_hex` uses `shr` and `and` instead, which is why
they are written that way), `src/sequence.tw` (`bucket`)
**Status:** twill's NEEDS-24 and NEEDS-44 both ask for this. **This entry
duplicates NEEDS-24 and NEEDS-44**, which are themselves duplicates of each
other.

`vocab.to_hex` extracts nibbles with `shr` and `and` rather than `/ 16` and
`% 16`. That is arguably better code and it was not a choice: with a power of
two the shift is available and correct, and if the base were ten it would not
be. `sequence.bucket` steps an index by the batch size and compares, which is
the loop it would write anyway.

### 8. `f64_log`, and the float conversions in systems mode

**Used by:** `src/unigram.tw` (`fit`, and `UNREACHABLE` as a literal)
**Status:** twill's NEEDS-40 covers `F64` in systems mode with the conversions,
and NEEDS-68 covers the transcendental primitives. **This entry duplicates
NEEDS-40 and NEEDS-68.**

The unigram model is log probabilities and it cannot be written without a
logarithm and without `f64_of_i64`. There is no workaround; the entry is here
because a reader who assumes the numeric half of the language is available in
systems mode will be surprised, and because it is a second repository asking.

`UNREACHABLE` is `-1e18` written out as a decimal literal, standing in for
negative infinity. A real negative infinity would be better and the literal is
sound: a path of a million pieces at -100 each sums to -1e8, so nothing can
reach the sentinel from below.

### 9. `str()` on `I64`, and string concatenation

**Used by:** every error message in the package, `src/vocab.tw` (`save`,
`pair_key` in `src/bpe.tw`)
**Status:** twill's NEEDS-45 covers `str()` on `I64`, NEEDS-35 and NEEDS-99
cover `Str` concatenation with `+`. **This entry duplicates NEEDS-45, NEEDS-35
and NEEDS-99.**

`bpe.pair_key` builds a dictionary key as `str(len(a)) + ":" + a + b`, which is
the length-prefixed encoding that makes the key unambiguous when a symbol can
contain any byte. It is on the hot path of BPE training, once per adjacent pair
per word per merge pass, so the allocation matters and a structured key would be
better. There is no way to spell one.

### 10. `i64_of_str`

**Used by:** `src/vocab.tw` (`load`)
**Status:** twill's NEEDS-19 asks for it. **This entry duplicates NEEDS-19.**

The vocabulary file's frequency column and its two header numbers are parsed
with `i64_of_str`. What is missing beyond the function itself is a way to tell a
parse failure from a legitimate zero: `i64_of_str("banana")` and
`i64_of_str("0")` must be distinguishable, and with no `Opt` they are not. A
corrupt frequency currently loads as zero, which does not break anything today
because frequencies are informational after a vocabulary is built, and would
break silently the moment anything reads them.

### 11. `chr(n)`

**Used by:** `tests/normalize_test.tw`, `tests/pretok_test.tw`,
`tests/simple_test.tw`, `tests/vocab_test.tw` and `tests/bpe_test.tw`, to write
individual bytes in test fixtures; `examples/pipeline.tw`
**Status:** twill's NEEDS-34 asks for `chr(n)` for a single byte. **This entry
duplicates NEEDS-34.**

Every test in this repository that covers non-ASCII behaviour builds its fixture
byte by byte, because a `.tw` source file containing a literal accented
character would depend on the file's own encoding surviving every editor and
every diff tool between here and CI. `text.from_byte` exists and does the same
job inside the library; `chr` is what the tests use because they should not
import the library under test to build their inputs.

## Not blocking, but the source is worse without them

### 12. `enum` with payloads and exhaustive `match`

**Would improve:** `src/encoding.tw` (`TRUNC_RIGHT`, `TRUNC_LEFT`,
`TRUNC_LONGEST_FIRST`, `PAD_RIGHT`, `PAD_LEFT`, and the if-chains in
`build_single` and `build_pair`), `src/pretok.tw` (the choice of pre-tokeniser)
**Status:** RESOLVED in 1.6, which shipped NEEDS-3.

The five constants are now two enums, `Trunc` and `Pad`. The four copies of
`if strategy == TRUNC_LEFT { left } else { right }` collapsed into one
`truncate_one`, which holds the file's single exhaustive `match` on `Trunc`, and
`pad_to` dispatches through a `match` on `Pad`.

One thing this did not fix, and it is worth recording because the entry
predicted it would. `build_single` given `TruncLongestFirst` still right-
truncates: there is only one sequence and nothing for longest-first to balance
against. What changed is that this is now a written-out arm with a reason beside
it rather than an `else` nobody chose, and a fourth strategy makes
`truncate_one` a compile error instead of quietly joining the same `else`.

`src/pretok.tw`'s choice of pre-tokeniser is still not an enum; callers select
one by calling `whitespace`, `word_punct` or `whole` directly, so there is no
tag to dispatch on and nothing to make exhaustive.

### 13. A priority queue, or a language reason not to want one

**Would improve:** `src/bpe.tw` (`train`, `count_pairs`)
**Status:** not a language feature. This is skein's own debt and it is recorded
here because the file points at this entry.

BPE training rebuilds the full pair-count table on every merge pass. The
standard implementation keeps the counts incrementally and adjusts only the
pairs touched by the merge, using a heap with decrease-key, and it is roughly an
order of magnitude faster on a real corpus. It is not written because its
correctness argument is subtle (a merge changes the counts of up to four
neighbouring pairs, and getting the boundary cases wrong produces a merge list
that is almost right) and a first version should not be where that is first
attempted.

### 14. Unigram training, and not only unigram estimation

**Would improve:** `src/unigram.tw` (`fit`)
**Status:** skein's own debt, recorded because `fit` names this entry.

`fit` computes relative frequencies over an existing vocabulary. Real unigram
training starts from a large seed vocabulary and alternates a segmentation of
the corpus with a re-estimation of the probabilities, pruning the pieces whose
removal costs the least likelihood, until the target size is reached. That loop
is what makes a unigram vocabulary better than a frequency-ranked one. skein has
the Viterbi it needs and not the loop, so a skein unigram model is a Viterbi
over someone else's vocabulary.

### 15. `continue`

**Would improve:** `src/normalize.tw` (`strip_control`), `src/wordpiece.tw`
(`encode_word`), `src/encoding.tw` (`is_special_id`)
**Status:** twill's NEEDS-12 asks for it. **This entry duplicates NEEDS-12.**

`strip_control` is a three-deep if-else where two of the branches only advance
the cursor. With `continue` it is a guard and a body. The nesting is the kind
that survives review and then acquires a fourth case.

### 16. A test runner

**Would improve:** `tests/`
**Status:** none. `tests/harness.tw` is a hand-rolled counter and `report` calls
`exit(1)`. loom's entry 15 and spool's say the same thing.

Every test file is a program that has to be run individually. `tests/harness.tw`
is now the third byte-for-byte copy of the same file in this ecosystem, which is
two more than there should be, and each copy is a place the assertion set can
drift. A `twill test` that collected `*_test.tw`, ran each in a fresh
interpreter and reported once would delete all three.

### 17. A tensor built from an `Arr[I64]`

**Needs:** a conversion from `Arr[I64]` to a 1-D integer `Tensor`, or a
`gather` that accepts an `Arr[I64]` directly
**Used by:** `src/embed.tw` (`embed`, `batch_ids`)
**Status:** `std/nn`'s `embedding(p, ids)` is documented as taking "a 1-D tensor
or list of integer row indices"; whether an `Arr[I64]` is one of those in
systems mode is not stated. Twill's NEEDS-62 (`Nested`) and NEEDS-25 (a foreign
call into the native tensor core) are the nearest existing entries.
**This entry overlaps NEEDS-62 and NEEDS-25.**

skein passes an `Arr[I64]` and assumes it is accepted. If it is not, every
tokenised batch needs a copy into a tensor, which is one allocation per batch
and is fine, and it needs a way to spell that copy, which does not exist. This
is the single point where skein touches the numeric half of the language and it
is the one interface in the package that has not been checked against a real
implementation of anything.

Since this was written, the numeric half grew narrow dtypes (twill
`docs/dtypes.md`), which sharpens what the conversion should produce rather than
changing what is missing. The id and type-id columns are row indices and want
i32, the dtype twill gives every index; the mask is a per-token 1-or-0 and wants
bool. So the conversion this entry asks for is specifically `Arr[I64]` to an i32
(or bool) tensor, not to f64, and `src/encoding.tw` now documents the dtype of
each column at the boundary so a consumer builds them narrow. The embedding
output is a separate matter and needs nothing: `embed` is dtype-transparent, the
table's dtype flows straight through gather, so a bf16 table returns bf16 rows
with no widening. The gap is still the one interface unchecked against a real
tensor implementation; it is just better specified now.
