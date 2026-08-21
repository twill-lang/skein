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

You need twill 1.7.0 or newer. Get a binary from the twill releases; there is
no build step, because there is nothing here but twill source.

```bash
curl -fsSL -o twill \
  https://github.com/twill-lang/twill/releases/download/v1.7.1/twill-v1.7.1-linux-amd64
chmod +x twill
./twill --version
```

The asset name is `twill-v1.7.1-<os>-<arch>`, and the assets that exist for
v1.7.1 are `linux-amd64`, `linux-arm64`, `darwin-amd64`, `darwin-arm64` and
`windows-amd64.exe`. Substitute yours.

Then, from the project root:

```bash
twill test tests
```

```
ok    tests\bpe_test.tw
ok    tests\embed_test.tw
ok    tests\encoding_test.tw
ok    tests\normalize_test.tw
ok    tests\pieces_test.tw
ok    tests\pretok_test.tw
ok    tests\sequence_test.tw
ok    tests\simple_test.tw
ok    tests\unigram_test.tw
ok    tests\vocab_test.tw
ok    tests\wordpiece_test.tw

11 file(s): 11 passed, 0 failed
```

Then the example, which is the whole library end to end:

```bash
twill run examples/pipeline.tw
```

```
vocabulary: 278 tokens, 18 merges
batch: 3 rows of 14, 17 padding positions

row 1, normalised: a dog ate the café hat
row 1, token spans against the source:
  0..1  'A'
  1..5  ' dog'
  5..6  ' '
  6..8  'at'
  8..9  'e'
  9..13  ' the'
  13..15  ' c'
  15..16  'a'
  16..17  'f'
  17..18  '�'
  18..19  '�'
  19..23  ' hat'

row 1 in windows of 6 with an overlap of 2: 3 windows

embedded row 0: 14 positions of 64
mask: 42 entries
```

Both output blocks are pasted from a run of twill 1.7.1 on Windows, which is
where the path separator in the first one comes from. CI runs the same two
commands on linux-amd64 on every push.

The two tokens at `17..18` and `18..19` are the two bytes of `é` printed one at
a time, so what you actually see there is whatever your terminal does with half
a character; mine showed the replacement character twice. The section below says
why they arrive separately. `docs/needs.md` is still worth reading --
it is the list of what this library asked the language for, and it records
which of those arrived and which are still open.

## What skein is

The layer between a string and `std/nn`. `std/text` gives you byte-string
operations; `std/nn` gives you an embedding table that takes row indices.
Nothing in between turns a document into ids, and nothing at all tells you which
bytes of the document a given id came from. That is skein.

Every row below that says "runs" names the test file that runs it. Nothing in
this table is a claim about code that has not executed.

| Piece | State |
| --- | --- |
| Normalisation with an offset-preserving trace | runs. `tests/normalize_test.tw` |
| Byte-level BPE: training and rank-ordered encoding | runs. `tests/bpe_test.tw` |
| WordPiece: greedy longest-match with `##` and a whole-word fallback | runs. `tests/wordpiece_test.tw` |
| Unigram: a proper Viterbi over the piece lattice | runs. `tests/unigram_test.tw` |
| Whitespace and character tokenisers | runs. `tests/simple_test.tw` |
| Offset mapping through normalisation, specials and truncation | runs. `tests/encoding_test.tw` |
| Special tokens, three truncation strategies, padding, batching | runs. `tests/encoding_test.tw` |
| n-grams, sliding windows, length bucketing | runs. `tests/sequence_test.tw` |
| Vocabulary with cutoffs, caps, reserved ids and a round-trippable format | runs. `tests/vocab_test.tw`. The format is a `Str`; nothing here writes it to disk |
| The handoff to `std/nn` | runs. `tests/embed_test.tw`, and `examples/pipeline.tw` embeds a real batch |
| Everything above, end to end | runs. `examples/pipeline.tw`, output above |
| NFC and NFD | **not possible.** See below |
| Case folding and punctuation detection outside ASCII | **not possible.** See below |
| Unigram vocabulary induction by EM pruning | **not in v0.1** |
| An embedding table | **deliberately not here.** See below |

## A worked example

Train a byte-level BPE tokeniser, encode a batch with padding and offsets, hand
the ids to `std/nn`. Abridged, and written as a consumer would write it, with
the vendored `twill_modules/` import paths. `examples/pipeline.tw` is the same
program complete, with its `fn main`, its named constants, its windowing step
and imports relative to this repository, and it is the file that was run to
produce the output above.

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

# Deliberately awkward: a capital, a double space, a multi-byte character. The
# accented letter is built with `chr` rather than written literally, so the
# fixture does not depend on this file's own encoding surviving a diff tool.
let DOCS: Arr[Str] = [
  "The  cat sat",
  "A dog ate the caf" + chr(195) + chr(169) + " hat",
  "the mat",
]

# Reserved tokens first, so [PAD] is id 0 and stays id 0 however much corpus
# gets added later. Byte-level BPE has no [UNK]: all 256 bytes are in the
# vocabulary, so nothing can be out of it.
let reserved: Arr[Str] = ["[PAD]", "[CLS]", "[SEP]", "[MASK]"]
let m: bpe.Bpe = bpe.train(CORPUS, 400, 2, reserved, true, false)

let sp: enc.Specials = match enc.specials_from(m.v, "[CLS]", "[SEP]", "[PAD]", "", "[MASK]") {
  Ok(built) => built,
  Err(msg) => { print("skein: " + msg); return },
}

let batch: enc.Batch = enc.new_batch()
let i: I64 = 0
while i < len(DOCS) {
  let n: nm.Norm = match nm.apply(nm.spec_default(), DOCS[i]) {
    Ok(norm) => norm,
    Err(msg) => { print("skein: " + msg); return },
  }
  let p: pc.Pieces = bpe.encode(m, n.text, pt.whitespace(n.text))
  enc.batch_add(batch, enc.build_single(n, p, m.v, sp, 16, enc.TruncRight))
  i = i + 1
}
match enc.pad_batch_longest(batch, sp, enc.PadRight) {
  Ok(_) => unit,
  Err(msg) => { print("skein: " + msg); return },
}

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
let rows = match em.embed(table, batch.items[0], vb.size(m.v)) {
  Ok(r) => r,
  Err(msg) => { print("skein: " + msg); return },
}
```

The output is in the section above, since that is the run of this program.

The interesting part is the tail of the accented word. This corpus is five
sentences, so BPE learns 18 merges and none of them touch `café`, which does not
appear in the corpus at all; ` café` comes out as five tokens rather than one.
That is a fact about a toy corpus and not about the offsets. The spans still
tile: `13..15`, `15..16`, `16..17`, `17..18`, `18..19`, which is bytes 13 to 19
of the source, six bytes for five characters, because the accented letter is two
of them. The last two tokens are the two halves of that letter, which is why
each prints as a replacement character on its own. In this document the
normalised offset happens to be 13 too, since lowercasing does not change any
length; in `"A dog ate the CAFÉ  hat"` it would not, and the reported offset
would still be right.

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

To run the tests or the example, clone the repository and point a twill 1.7
binary at it; the section at the top has the commands and their output. There is
no build step.

To use skein from another project, via spool:

```
spool add skein https://github.com/twill-lang/skein
```

I have not run that command against this repository, so treat it as spool's
documented usage rather than as something verified here.

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
docs/needs.md       what this library asked the language for, and what arrived
```

## Dependencies

twill, and nothing else. No third-party twill packages. `std/text`, `std/nn` and
the tensor builtins are the whole surface skein builds on, and `std/text` does
the byte-string work rather than skein duplicating it: `find`, `slice`,
`lines`, `char_starts`, `char_size`, `char_code`, `from_byte`, `push_str`,
`is_space`, `is_alnum` and the rest are used throughout and are not
reimplemented anywhere here.

## Contributing

The most useful contributions right now are the three entries in
[`docs/needs.md`](docs/needs.md) marked "delivered, not wired up": twill has
`dict_keys`, function values and `continue`, and skein still runs the workaround
for each. Entry 10 is a fourth of the same shape, smaller.

After that, a correction to `docs/needs.md` itself: a feature listed there that
the language already has, a workaround that is worse than described, a
cross-reference to a twill `NEEDS-N` that is wrong or missing, or a new entry
found by reading the source. Then the offset-mapping argument above, which is
the part most worth arguing with.

## License

MIT. See [LICENSE](LICENSE).
