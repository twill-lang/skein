# Changelog

## v0.1.0 (unreleased)

### Changed

- **The assertions are `std/test`, and the pin is twill 1.12.0.** twill 1.11
  ships the assertions the test runner already assumed, which is what
  `docs/needs.md` entry 16 said would delete skein's copy of them. Every suite
  imports `std/test` as `t` and calls the same names it did before. `report`
  returns the status instead of calling `exit`, and prints its summary in the
  shape `twill test` reads, so the runner shows the counts beside each file:
  371 assertions across eleven suites, where before it showed none.
  `tests/harness.tw` keeps one formatter, `opt_str`, and nothing that counts.
  A helper importing `std/test` gets its own counter, so a failure it recorded
  would never reach the suite's `report`; that was measured with a probe that
  came back green with a failure in it, and the five suites that compare an
  `Opt[I64]` define `equal_opt_i64` locally on top of `t.equal_str` for that
  reason. `spool.toml`, CI and the README's install line move to 1.12.0.

First cut of skein, the text and sequence library for twill, written in twill.

It runs on twill 1.7. `twill test tests` is 11 files, 11 passed, 0 failed, and
`twill run examples/pipeline.tw` exits 0. Everything below has executed. See
`docs/needs.md` for what the language still owes this library and `README.md`
for the status table.

Added:

- Normalisation as a traced transform. `Norm` carries the normalised bytes plus,
  per byte, the source byte range that produced it, so a token span maps back to
  the original text through any number of steps. Control stripping (C0, DEL and
  C1), ASCII case folding, whitespace collapsing, trimming, and accent folding
  over the Latin-1 Supplement block. NFC is refused with a message rather than
  approximated.
- Offset mapping computed in exactly one function, `encoding.from_pieces`.
  Models return spans in the normalised text and never see the source, so a
  model cannot produce a wrong source offset. Offsets survive special tokens,
  truncation, padding and windowing.
- Byte-level BPE. All 256 bytes as the base alphabet, so nothing is ever out of
  vocabulary. Training records the ordered merge list; encoding replays it by
  rank rather than matching longest-first, and the file says why the difference
  matters.
- WordPiece, greedy longest-match-first with a `##` continuation prefix and an
  all-or-nothing unknown fallback per word.
- Unigram, with a real Viterbi over the lattice of every piece matching at every
  character boundary, plus a single-character unknown edge so no word is ever
  unsegmentable.
- Whitespace and character tokenisers, so a caller always has something that
  works before any model is trained.
- A vocabulary with reserved ids at the front, frequency counting, a minimum
  frequency cutoff, a maximum size, and a deterministic hex-encoded file format
  that round trips tokens containing spaces, newlines and byte 0x00.
- Special token registry, three truncation strategies that account for the
  specials' own budget, left and right padding to a fixed width or to the
  longest row in a batch, and the attention mask that results.
- n-grams, sliding windows with a configurable stride or overlap, and length
  bucketing with a pool so batches are length-homogeneous without the batch
  order becoming a function of the lengths.
- A handoff to `std/nn`'s embedding that checks the two things that go wrong at
  that boundary and reimplements nothing.

Known gaps, deliberate for v0.1:

- No Unicode character database, so no NFC, no case folding outside ASCII and no
  punctuation detection outside ASCII. `docs/needs.md` entry 1.
- Unigram estimation but not unigram vocabulary induction: no EM pruning loop.
  `docs/needs.md` entry 14.
- BPE training rebuilds the pair counts on every merge pass rather than
  adjusting them incrementally. `docs/needs.md` entry 13.
