# What skein needs from twill

skein is written in twill and it runs: `twill test tests` is 11 files, 11
passed, and `twill run examples/pipeline.tw` exits 0 on twill 1.7.1. This file
started as the reason it did not. It is now the record of what this library
asked the language for, entry by entry, and which of those arrived.

Each entry says one of three things. **Delivered and used** means twill ships it
and skein calls it. **Delivered, not wired up** means twill ships it and skein
still runs the workaround, which is skein's debt and not the language's. **Open**
means the language still does not have it.

It is a work queue for the language, not a complaint. Every entry was reached by
writing real code and hitting the wall, which is the only way a list like this
is worth anything. Where skein has a workaround the workaround is described,
because how ugly a workaround is measures how badly the feature is wanted.

| # | Entry | State |
| --- | --- | --- |
| 1 | Unicode character database | open |
| 2 | `Dict` membership and iteration | delivered, iteration not wired up |
| 3 | Sorting with a caller-supplied comparison | open |
| 4 | Nested containers | delivered and used |
| 5 | `Res[T, E]` | delivered and used |
| 6 | A function type | delivered, not wired up |
| 7 | Integer division and modulo | delivered |
| 8 | `f64_log` and the float conversions | delivered and used |
| 9 | `str()` on `I64`, string concatenation | delivered and used |
| 10 | `i64_of_str` | delivered, the `Opt` not fully used |
| 11 | `chr(n)` | delivered and used |
| 12 | `enum` with payloads and exhaustive `match` | delivered and used |
| 13 | A priority queue for BPE training | skein's own debt |
| 14 | Unigram training and not only estimation | skein's own debt |
| 15 | `continue` | delivered, not wired up |
| 16 | A test runner | delivered and used |
| 17 | A tensor from an `Arr[I64]` | delivered; a dtype seam is left |

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64` with bitwise operations, `Str` with length and byte
indexing, `Bytes`, `Arr[T]`, `Dict[Str, V]` and `Dict[I64, V]`, and `struct`
with reference semantics. Everything below is on top of that.

Several entries duplicate an entry in twill's own `docs/needs.md`. Where they
do, the twill number is named and the entry says what skein adds to it, if
anything. Duplicates are kept rather than deleted: a second repository hitting
the same wall for a different reason is evidence about priority, and deleting
the entry throws that evidence away.

## Was blocking: skein could not run at all without these

### 1. A Unicode character database, or an honest replacement for one

**Needs:** canonical decomposition and composition mappings, canonical
combining classes and composition exclusions, and general category data
**Used by:** `src/normalize.tw` (`apply`, `fold_latin1`, `latin1_fold`),
`src/pretok.tw` (`is_punct`)
**Status:** OPEN. Nothing in the language or the standard library on 1.7.1;
there is no `nfc` and no character-database builtin. `std/text.tw` has
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
**Status:** DELIVERED, and the iteration half is NOT WIRED UP. The lookup half
was resolved in 1.6 (NEEDS-22) and skein uses it. Iteration (NEEDS-8) arrived
too: 1.7.1 has `dict_keys(d)`, which returns the keys as an `Arr`. There is
still no `for` over a `Dict` itself. No file in `src/` calls `dict_keys`, so
every parallel key array described below is still there and is now skein's debt
rather than the language's.

`dict_get` returns `Opt[V]`, which is what this entry said would be better than
`dict_has` plus an index. `vocab.id_of` and `bpe.rank_of` are both one
`dict_get` now. That removed their -1 returns, and it also halved their hashing:
`dict_has` followed by an index hashed the key twice, and those two functions
are the innermost lookups in the package.

Iteration is worked around below, and the workaround is no longer forced.

Iteration is worked around by keeping a parallel `Arr` of the keys next to every
`Dict`: `vocab.Counts` has `keys`, `vocab.Vocab` has `toks`. That doubles the
memory of a frequency table and it is a real invariant to maintain, because the
array and the dictionary can disagree and nothing checks that they do not.
`vocab.observe` is the only function allowed to write either, which is how the
invariant is kept, and it is kept by discipline rather than by the type.

Removing the parallel arrays is not a mechanical substitution of `dict_keys`,
which is why it has not happened yet. The arrays carry insertion order, and
`vocab.build` assigns ids from that order so that a vocabulary is a function of
the data alone; `dict_keys` is not documented to return keys in insertion order
and skein has not established that it does. Whoever does this has to check that
first, and `tests/vocab_test.tw` has the assertions that would catch it.

### 3. Sorting

**Needs:** a sort over `Arr[T]` with a caller-supplied comparison
**Used by:** `src/vocab.tw` (`sort_index`, called by `build`),
`src/sequence.tw` (`sort_by_length`, called by `bucket`)
**Status:** OPEN. 1.7.1 has `sort`, which refuses a list whose elements are not
strings ("sort on a list expects every element to be a string") and takes no
comparison function; there is no `sort_by`. NEEDS-23 asked for sorting an
`Arr[Str]` and that is what shipped. **This entry duplicates NEEDS-23** and
widens it, and the widening is the part still missing: skein needs a comparison
over two parallel arrays, not over the elements of one, so neither of the two
sorts below can be deleted.

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
**Status:** DELIVERED AND USED. `Arr[Arr[Str]]` and an `Arr` of structs both
work on 1.7.1, `bpe.train`, `encoding.Batch` and `sequence.bucket` are written
against them as described, and `tests/bpe_test.tw`, `tests/encoding_test.tw` and
`tests/sequence_test.tw` exercise all three. NEEDS-72 is closed as far as skein
is concerned. The rest of this entry is the record of why it mattered.

There was no workaround. BPE training holds every distinct word as its current
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
**Status:** DELIVERED AND USED, for everything this entry named (2026-08, on
twill 1.7). The out-parameter pattern is gone from the package entirely; a plain
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

**Nothing is left.** The nine that had no value to return alongside the error --
`encoding.assert_shape`, `register_extra`, `pad_to`, `pad_batch_longest`,
`pad_batch_fixed`, `unigram.set_logp`, `fit`, `validate` and `embed.check_ids`
-- have moved too, along with `embed.check_encoding` and `vocab.add_reserved`,
which the earlier list had missed. Every fallible function in skein returns a
`Res` now, and the empty-string-means-success convention is gone from the
package.

Two functions still return `""` and are not this: `encoding.source_text` and
`vocab.token_of` return a string as their answer, and `token_of` returning `""`
for an out-of-range id is a deliberate choice with its reason in the comment
above it -- a model emitting a nonsense argmax during early training should
produce a visibly blank token rather than kill the process.

**One thing the conversion itself found.** `pad_to` opened with

    let need: I64 = width - length(e)
    if need <= 0 { return "" }

which is an early *success* -- already at least this wide, nothing to do -- and
under the old convention it was the same two bytes as the success at the bottom
of the function. Nothing in the type or the text said which it was, and a
mechanical reading of "returns a Str, empty on success" gets it wrong in the
direction that turns a no-op into a failure. It is `Ok(unit)` now and says so.
That the distinction was invisible is the argument for the whole entry.

### 6. A function type, for the tokeniser seam

**Needs:** a function value passed as a parameter, and a spelling for its type
**Used by:** nothing today, and `src/encoding.tw` (`build_single`,
`build_pair`) wants it
**Status:** DELIVERED, NOT WIRED UP. The question this entry asked is answered:
on 1.7.1 a systems-mode function takes a function value and the type is spelled
`Fn(I64) -> I64`, which I checked by writing and running one. loom's
`docs/needs.md` entry 3 asked the same question for its step function.

No file in `src/` declares an `Fn` parameter, so the seam below is unbuilt and
the mismatch it describes is still silent. This is skein's work now.

skein's five models all produce `Pieces` and all have the same shape, and the
natural design is for `build_single` to take the model's encode function.
Instead every call site calls the model itself and passes the resulting
`Pieces`, which works and means the driver cannot enforce that the pre-tokeniser
a model was trained with is the one it is being encoded with. That mismatch is
silent and produces subtly wrong tokenisation.

## Was blocking: features the source assumes exist

### 7. Integer division and modulo on `I64`

**Used by:** `src/vocab.tw` (`to_hex` uses `shr` and `and` instead, which is why
they are written that way), `src/sequence.tw` (`bucket`)
**Status:** DELIVERED. On 1.7.1, `/` on two `I64` is integer division (`7 / 2`
is `3`) and `%` is the remainder (`7 % 2` is `1`). Untyped literals are a
separate matter: a bare `7 / 2` at the top of a systems-mode file prints `3.5`,
so the annotations matter. NEEDS-24 and NEEDS-44 are
closed. **This entry duplicated NEEDS-24 and NEEDS-44**, which were themselves
duplicates of each other.

Neither call site has been changed and neither should be. `vocab.to_hex`
extracts nibbles with `shr` and `and` rather than `/ 16` and
`% 16`. That is arguably better code and it was not a choice: with a power of
two the shift is available and correct, and if the base were ten it would not
be. `sequence.bucket` steps an index by the batch size and compares, which is
the loop it would write anyway.

### 8. `f64_log`, and the float conversions in systems mode

**Used by:** `src/unigram.tw` (`fit`, and `UNREACHABLE` as a literal)
**Status:** DELIVERED AND USED. `F64`, `f64_of_i64` and `f64_log` all work in
systems mode on 1.7.1. `src/unigram.tw` calls both in `fit` and
`tests/unigram_test.tw` passes. NEEDS-40 and NEEDS-68 are closed as far as
skein is concerned. **This entry duplicated NEEDS-40 and NEEDS-68.**

The unigram model is log probabilities and it could not be written without a
logarithm and without `f64_of_i64`. There was no workaround.

`UNREACHABLE` is `-1e18` written out as a decimal literal, standing in for
negative infinity. A real negative infinity would be better and the literal is
sound: a path of a million pieces at -100 each sums to -1e8, so nothing can
reach the sentinel from below.

### 9. `str()` on `I64`, and string concatenation

**Used by:** every error message in the package, `src/vocab.tw` (`save`,
`pair_key` in `src/bpe.tw`)
**Status:** DELIVERED AND USED. `str()` on `I64` and `+` on `Str` both work;
every error message and `bpe.pair_key` depend on them and the suite passes.
NEEDS-45, NEEDS-35 and NEEDS-99 are closed. **This entry duplicated NEEDS-45,
NEEDS-35 and NEEDS-99.** What is left below is skein's own performance note,
not a language request.

`bpe.pair_key` builds a dictionary key as `str(len(a)) + ":" + a + b`, which is
the length-prefixed encoding that makes the key unambiguous when a symbol can
contain any byte. It is on the hot path of BPE training, once per adjacent pair
per word per merge pass, so the allocation matters and a structured key would be
better. There is no way to spell one.

### 10. `i64_of_str`

**Used by:** `src/vocab.tw` (`load`)
**Status:** DELIVERED AND USED, including the part this entry cared about.
NEEDS-19 shipped, and it shipped returning an `Opt`: `i64_of_str("12")` is
`Some(12)`, so a parse failure and a legitimate zero are distinguishable, which
is what this entry said was missing beyond the function itself. **This entry
duplicated NEEDS-19.**

skein has not taken all of it. `vocab.load` goes through a helper `i64_or(s,
fallback)` that matches the `Opt` and substitutes a fallback, which is -1 for a
count or an id and is caught by the bounds checks that follow, and 0 for a
frequency. So the frequency column still cannot tell `banana` from `0`, exactly
as this entry described, and it is now a choice in `src/vocab.tw` rather than a
missing feature. Making `load` refuse a non-numeric frequency by line number is
a small change and `tests/vocab_test.tw` is where the assertion goes.

### 11. `chr(n)`

**Used by:** `tests/normalize_test.tw`, `tests/pretok_test.tw`,
`tests/simple_test.tw`, `tests/vocab_test.tw` and `tests/bpe_test.tw`, to write
individual bytes in test fixtures; `examples/pipeline.tw`
**Status:** DELIVERED AND USED. NEEDS-34 shipped; `chr(65)` is `A` on 1.7.1.
`examples/pipeline.tw` builds its accented fixture as
`chr(195) + chr(169)` and the tests named above build theirs the same way.
**This entry duplicated NEEDS-34.**

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
**Status:** DELIVERED AND USED. Resolved in 1.6, which shipped NEEDS-3.

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
**Status:** DELIVERED, NOT WIRED UP. NEEDS-12 shipped; `continue` works in a
`while` on 1.7.1. No file in `src/` uses it, so all three functions named above
still have the shape this entry complained about. **This entry duplicated
NEEDS-12.**

`strip_control` is a three-deep if-else where two of the branches only advance
the cursor. With `continue` it is a guard and a body. The nesting is the kind
that survives review and then acquires a fourth case.

### 16. A test runner

**Would improve:** `tests/`
**Status:** DELIVERED. `twill test tests` exists on 1.7.1 and is exactly what
this entry asked for: it collects the `*_test.tw` files, runs each one and
reports once.

```
11 file(s): 11 passed, 0 failed
```

It is what CI runs and what the README tells a reader to run, and no test file
is invoked individually anywhere any more.

`tests/harness.tw` is still here and is still the third byte-for-byte copy of
the same file in this ecosystem. The runner replaced the invocation, not the
assertions: `check`, `equal_str` and the rest are skein's own and `twill test`
does not provide them. Deleting the three copies needs assertions in the
toolchain, which is a smaller ask than this entry made and should be its own
entry when someone writes it. loom's entry 15 and spool's are the same
situation.

### 17. A tensor built from an `Arr[I64]`

**Needs:** a conversion from `Arr[I64]` to a 1-D integer `Tensor`, or a
`gather` that accepts an `Arr[I64]` directly
**Used by:** `src/embed.tw` (`embed`, `batch_ids`)
**Status:** DELIVERED, and the assumption is now checked. An `Arr[I64]` is
accepted where `std/nn`'s `embedding(p, ids)` wants row indices, and there is
also a `tensor(a)` that turns an `Arr[I64]` into a 1-D integer tensor
(`tensor([1, 2, 3], shape=[3])`), which is the conversion this entry asked for
in case it was not. Twill's NEEDS-62 (`Nested`) and NEEDS-25 (a foreign call
into the native tensor core) were the nearest existing entries.
**This entry overlapped NEEDS-62 and NEEDS-25.**

skein passes an `Arr[I64]` and it is accepted: `tests/embed_test.tw` passes and
`examples/pipeline.tw` embeds a real batch and prints its shape. This was the
one interface in the package that had not been checked against a real
implementation of anything, and it has been.

Since this was written, the numeric half grew narrow dtypes (twill
`docs/dtypes.md`), which sharpens what the conversion should produce rather than
changing what is missing. The id and type-id columns are row indices and want
i32, the dtype twill gives every index; the mask is a per-token 1-or-0 and wants
bool. So the conversion this entry asks for is specifically `Arr[I64]` to an i32
(or bool) tensor, not to f64, and `src/encoding.tw` now documents the dtype of
each column at the boundary so a consumer builds them narrow. The embedding
output is a separate matter and needs nothing: `embed` is dtype-transparent, the
table's dtype flows straight through gather, so a bf16 table returns bf16 rows
with no widening. What is left of this entry is dtype and not acceptance:
`src/encoding.tw` documents the intended dtype of each column at the boundary,
and nothing in skein asserts it, because skein hands over an `Arr[I64]` and does
not choose the dtype the consumer builds from it. That is a documented seam
rather than a missing feature, and it is the reason this entry stays open at
all.
