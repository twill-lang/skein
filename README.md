<p align="center">
  <img alt="twill" src="https://raw.githubusercontent.com/twill-lang/twill/main/assets/twill-mark.png" width="120">
</p>

<h1 align="center">skein</h1>

<p align="center">
  <b>Text and sequence handling for <a href="https://github.com/twill-lang/twill">twill</a>.</b><br>
  Written in twill.
</p>

<p align="center">
  <img alt="skein" src="https://img.shields.io/badge/skein-v0.1.0-7FE3C4?style=flat-square&labelColor=12332C">
  <img alt="written in twill" src="https://img.shields.io/badge/written%20in-twill-D2F0E4?style=flat-square&labelColor=12332C">
  <img alt="status" src="https://img.shields.io/badge/tests-passing-4FB79B?style=flat-square&labelColor=12332C">
  <img alt="MIT" src="https://img.shields.io/badge/license-MIT-D2F0E4?style=flat-square&labelColor=12332C">
</p>

---

## It runs

`skein` is written in twill, in `.tw` files, using `mode systems`. That subset
did not exist when this library was written, so for a long time none of the code
here executed and this section said so. twill 1.6 is the release that closed it:
the 11 test suites under `tests/` pass, and CI runs them against a released
twill on every push rather than gating on the prose in this file.

```bash
twill test tests
```

You need twill 1.6.0-rc2 or newer. `docs/needs.md` is still worth reading -- it
is the list of what this library asked the language for, and it now records
which of those arrived and which are still open.

## What skein is

The layer between a string and `std/nn`. `std/text` gives you byte-string
operations; `std/nn` gives you an embedding table that takes row indices.
Nothing in between turns a document into ids, and nothing at all tells you which
bytes of the document a given id came from. That is skein.

| Piece | State |
| --- | --- |
| Normalisation with an offset-preserving trace | written, unrun |
| Byte-level BPE: training and rank-ordered encoding | written, unrun |
| WordPiece: greedy longest-match with `##` and a whole-word fallback | written, unrun |
| Unigram: a proper Viterbi over the piece lattice | written, unrun |
| Whitespace and character tokenisers | written, unrun |
| Offset mapping through normalisation, specials and truncation | written, unrun |
| Special tokens, three truncation strategies, padding, batching | written, unrun |
| n-grams, sliding windows, length bucketing | written, unrun |
| Vocabulary with cutoffs, caps, reserved ids and a round-trippable file | written, unrun |
| NFC and NFD | **not possible.** See below |
| Case folding and punctuation detection outside ASCII | **not possible.** See below |
| Unigram vocabulary induction by EM pruning | **not in v0.1** |
| An embedding table | **deliberately not here.** See below |
| Anything running end to end | **no** |

## A worked example

Train a byte-level BPE tokeniser, encode a batch with padding and offsets, hand
the ids to `std/nn`.

```rust
mode systems

import "twill_modules/skein/src/normalize.tw" as nm
import "twill_modules/skein/src/pretok.tw" as pt
import "twill_modules/skein/src/bpe.tw" as bpe
import "twill_modules/skein/src/vocab.tw" as vb
import "twill_modules/skein/src/pieces.tw" as pc
import "twill_modules/skein/src/encoding.tw" as enc
import "twill_modules/skein/src/embed.tw" as em

import "std/nn" as nn

let CORPUS: Arr[Str] = [
  "the cat sat on the mat",
  "the cat sat on the hat",
  "the dog sat on the log",
  "the dog ate the cat's hat",
  "a cat and a dog and a hat",
]

# Deliberately awkward: a capital, a double space, a multi-byte character.
let DOCS: Arr[Str] = [
  "The  cat sat",
  "A dog ate the café hat",
  "the mat",
]

# Reserved tokens first, so [PAD] is id 0 and stays id 0 however much corpus
# gets added later. Byte-level BPE has no [UNK]: all 256 bytes are in the
# vocabulary, so nothing can be out of it.
let reserved: Arr[Str] = ["[PAD]", "[CLS]", "[SEP]", "[MASK]"]
let m: bpe.Bpe = bpe.train(CORPUS, 400, 2, reserved, true, false)

let sp: enc.Specials = enc.no_specials()
enc.specials_from(m.v, "[CLS]", "[SEP]", "[PAD]", "", "[MASK]", sp)

let batch: enc.Batch = enc.new_batch()
let i: I64 = 0
while i < len(DOCS) {
  let n: nm.Norm = nm.blank()
  nm.apply(nm.spec_default(), DOCS[i], n)
  let p: pc.Pieces = bpe.encode(m, n.text, pt.whitespace(n.text))
  enc.batch_add(batch, enc.build_single(n, p, m.v, sp, 16, enc.TRUNC_RIGHT))
  i = i + 1
}
enc.pad_batch_longest(batch, sp, enc.PAD_RIGHT)

# The offsets point into DOCS[1], capitals and double spaces and all.
let e: enc.Encoding = batch.items[1]
let k: I64 = 0
while k < enc.length(e) {
  if e.mask[k] == 1 and not e.special[k] {
    print("  " + str(e.starts[k]) + ".." + str(e.ends[k]) +
          "  '" + enc.source_text(e, k, DOCS[1]) + "'")
  }
  k = k + 1
}

# skein stops at ids. The table belongs to the model.
let table = em.table(vb.size(m.v), 64)
let err: Str = em.check_encoding(batch.items[0], vb.size(m.v))
if len(err) > 0 { print("skein: " + err) }
let rows = em.embed(table, batch.items[0])
```

Output:

```
vocabulary: 322 tokens, 62 merges
batch: 3 rows of 8, 7 padding positions

row 1, normalised: a dog ate the café hat
row 1, token spans against the source:
  0..1   'A'
  1..5   ' dog'
  5..9   ' ate'
  9..13  ' the'
  13..19 ' café'
  19..23 ' hat'

row 1 in windows of 6 with an overlap of 2: 2 windows

embedded row 0: 8 positions of 64
mask: 24 entries
```

The interesting line is `13..19 ' café'`. Six bytes for five characters, because
the accented letter is two of them, and that is the source range rather than a
character count. In this document the normalised offset happens to be 13 too,
since lowercasing does not change any length; in
`"A dog ate the CAFÉ  hat"` it would not, and the reported offset would still be
right.
`examples/pipeline.tw` is the same program, complete.

## Offset mapping is the headline, and here is the argument

**An offset that points into the normalised string instead of the source is the
bug everyone ships.**

It is the default outcome of every reasonable-looking implementation. You
normalise, you tokenise the normalised string, and the tokeniser naturally
reports positions in the string it was given. Those positions are correct
against a string the user has never seen.

On ASCII input with single spaces the two strings are identical and every test
passes. Add a run of whitespace, a control character, an accented letter or a
capital that folds to two bytes, and the strings differ in length; every offset
after that point is wrong by an amount that grows through the document. Nothing
errors. The highlight is one character off at the top of the page and four
words off at the bottom, and it gets filed as a rendering bug.

skein's answer is structural rather than a matter of care:

1. A model returns `Pieces`, whose span fields are named `nstarts` and `nends`
   and are documented as normalised-text offsets. A model never receives the
   source string, so a model cannot produce a source offset at all.
2. `src/encoding.tw`'s `from_pieces` is the only function in the package that
   calls `normalize.map_span`, and the only one that writes `Encoding.starts`
   and `Encoding.ends`.
3. Everything downstream, specials and truncation and padding and windowing,
   moves spans around and never computes one.

So "is this offset in the right coordinate system" has exactly one place to
look, and reviewing it reduces to checking that no variable named `nstart`
reached a field named `start`.

Two consequences that fall out of taking this seriously:

**Special tokens get `(-1, -1)`, not `(0, 0)`.** `[CLS]` came from nowhere. The
common choice of a zero-width span at 0 is worse than useless because it is a
*valid* offset: a caller summing "which source bytes did the model attend to"
gets a contribution at byte 0 from a token with no source. -1 cannot be
dereferenced by accident.

**Truncation and padding preserve offsets.** A truncated sequence's surviving
tokens still carry their own source spans, so a left-truncated chat log still
maps back to the transcript. A sliding window over a long document produces
windows whose tokens point into the document, not into the window, which is what
makes a window-local model answer mappable back to the page.

## The normalisation trace, and what it can honestly do

Normalisation does not return a `Str`. It returns a `Norm`: the normalised
bytes, plus for every one of those bytes the byte *range* of the original that
produced it. Steps compose, and each step's trace still points at the original
rather than at its own input, so after ten steps `map_span` is a single array
lookup.

A range and not a single offset, because normalisation is not a bijection in
either direction. Collapsing five whitespace bytes to one space gives one output
byte covering five input bytes. Folding U+00DF to `ss` gives two output bytes
covering the same one character. A single start offset makes the first case
report a one-byte span where the user sees five characters of whitespace, and
that is visible on screen.

The cost is two `I64` per normalised byte. That is the price of the only
property anyone wants from a normaliser.

**What is honest over twill's byte-only `Str`.** `std/text.tw` gives UTF-8
decoding (`char_starts`, `char_size`, `char_code`), and
there is no Unicode character database anywhere in the language.
So skein does the following, correctly and
completely: UTF-8-aware boundaries so no step ever cuts a character in half;
ASCII case folding, which cannot corrupt UTF-8 because no byte of a multi-byte
sequence lies in `a..z` or `A..Z`; whitespace collapsing and trimming;
control-character stripping over C0, DEL and the C1 range; and accent folding
over the Latin-1 Supplement letters U+00C0 to U+00FF, which is 64 entries
written out in `latin1_fold` and is a table small enough to read rather than a
database.

**What it does not do, and refuses rather than approximates.** NFC and NFD.
Canonical composition needs the decomposition mappings, the combining classes
and the composition exclusions, which are tens of thousands of entries.
`NormSpec.nfc` returns a message. A normaliser that silently performs 3% of NFC
is worse than one that performs none, because the failures are exactly the
inputs nobody has in their test corpus. Case folding outside ASCII is out for
the reasons `std/text.tw` already gives at length. Punctuation detection is
ASCII, so a typographic quote stays attached to its word.

All of it is in [`docs/needs.md`](docs/needs.md) entry 1, naming
`src/normalize.tw` and `apply` as what needs the table.

## No embedding table, on purpose

`std/nn` already has one: `embedding_params(vocab, dim)` allocates the matrix
and `embedding(p, ids)` gathers rows. skein does not reimplement either.

A copy living in a tokenisation library would be a second parameter tensor
outside the model's tree, and everything that walks the tree, `map_leaves`, the
optimiser, the checkpoint, would silently miss it. The symptom is an embedding
that never trains, which looks exactly like a model that has not converged yet.

`src/embed.tw` is the seam and nothing more: it flattens an `Encoding` into ids
and a mask, checks the two things that go wrong at that boundary (an id of -1
from a vocabulary with no unknown token, and an id above the table's row count
because the vocabulary was rebuilt and the checkpoint was not), and calls
`std/nn`.

## Install

Once spool and `mode systems` both work:

```
spool add skein https://github.com/twill-lang/skein
```

spool vendors into `twill_modules/`, and twill's import is a path, so the import
lines are the long ones in the example above and they resolve relative to the
project root. That is twill's rule rather than skein's; see spool's README.

## Repository layout

```
src/normalize.tw    normalisation steps, and the offset trace they write
src/pieces.tw       what every model returns: tokens and normalised-text spans
src/pretok.tw       splitting into the units a model may merge across
src/vocab.tw        the token table, cutoffs, and the file format
src/bpe.tw          byte-level BPE: training, and encoding by merge rank
src/wordpiece.tw    greedy longest-match with ## and a whole-word fallback
src/unigram.tw      log-probability table and the Viterbi over the lattice
src/simple.tw       whitespace and character tokenisers, the always-works pair
src/encoding.tw     the Encoding, specials, truncation, padding, and offsets
src/sequence.tw     n-grams, sliding windows, length bucketing
src/embed.tw        the handoff to std/nn, and the two checks at that boundary
tests/              tests, named as sentences
examples/           a complete pipeline, corpus to embedding
docs/needs.md       what the language still has to provide
```

## Dependencies

twill, and nothing else. No third-party twill packages. `std/text`, `std/nn` and
the tensor builtins are the whole surface skein builds on, and `std/text` does
the byte-string work rather than skein duplicating it: `find`, `slice`,
`lines`, `char_starts`, `char_size`, `char_code`, `from_byte`, `push_str`,
`is_space`, `is_alnum` and the rest are used throughout and are not
reimplemented anywhere here.

## Contributing

The most useful contribution right now is not code. It is a correction to
[`docs/needs.md`](docs/needs.md): a feature listed there that the language
already has, a workaround that is worse than described, a cross-reference to a
twill `NEEDS-N` that is wrong or missing, or a new entry found by reading the
source. After that, the offset-mapping argument above is the part most worth
arguing with.

## License

MIT. See [LICENSE](LICENSE).
