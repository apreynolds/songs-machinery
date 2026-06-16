# Song sheets — project reference & log

Reference material kept out of CLAUDE.md for lookup as needed: package issues,
resolved problems, implemented-feature mechanism internals, and tooling/debugging
notes. CLAUDE.md holds the live authoring contracts and design goals; this file
is the "when you actually touch that code" detail. Scope is the whole song-sheet
project, not just the `leadsheets` package.

## Reminders / how to investigate

- **System package source:** `/usr/local/texlive/2025/texmf-dist/tex/latex/leadsheets/`
  - chord placement: `leadsheets.library.chords.code.tex`
  - song/verse environments: `leadsheets.library.songs.code.tex`
  - music symbols (bars etc.): `leadsheets.library.musicsymbols.code.tex`
- **PR#46 fix** lives in `~/repos/1sys/tex/songs/` and shadows the system copy via
  `TEXINPUTS="$HOME/repos/1sys/tex//:"` (set in `~/.config/zsh/.zprofile`). When
  compiling test files by hand outside that shell, export it explicitly:
  `export TEXINPUTS="$HOME/repos/1sys/tex//:"`.
- **Isolating a behaviour:** a bare `\documentclass{leadsheet}` + a single
  `\begin{verse}` (no `MyLeadsheets`) removes our own formatting as a variable.
- **Probing expl3 internals:** patch a command with `\cs_set_protected:Npn`
  inside `\ExplSyntaxOn ... \ExplSyntaxOff` in the preamble and `\iow_term:x` /
  `\typeout` the value of interest. Note the verse body is processed **twice**
  (a measuring pass + the real pass), so expect probe output to print twice.

## Issues

### `empty-chord-dim` is a no-op (TeX Live 2025)

**Symptom:** Setting `empty-chord-dim` to anything (0pt, 2cm, 5cm, …) has *zero*
visible effect; output is byte-identical regardless of value.

**Cause (confirmed by isolated tests):** When a chord sits over blank lyric text
(a space immediately follows the chord, e.g. `^{Em} |`), the splitter
`\__leadsheets_set_chord:nwn` (chords.code.tex:65) takes the blank branch
(line 72) and emits the dimension as **horizontal glue**:
`\skip_horizontal:N \l__leadsheets_empty_chord_dim` (≈ `\hskip <dim>`). That glue
becomes the *entire bottom cell* of the chord/text `tabular` in
`\leadsheets_place_above` (line 172). But LaTeX's `l`/`c`/`r` column templates
wrap every cell as `\ignorespaces <content> \unskip`, and **`\unskip` strips a
trailing glue item** — which is exactly what this cell is. So the width is
removed every time, for any value.

Demonstrated with a minimal `tabular`, same 4cm in every row:

| bottom cell                  | result        |
|------------------------------|---------------|
| `\hskip4cm` (glue, trailing) | **no gap**    |
| `\rule{4cm}{0pt}` (rigid)    | full 4cm gap  |
| `\makebox[4cm]{}` (rigid)    | full 4cm gap  |
| `\hskip4cm.` (glue, not last)| full 4cm gap  |

Rigid widths survive `\unskip`; trailing glue does not. The package only ever
emits the trailing-glue form, so the key is structurally dead.

Unknown whether this is a regression (e.g. from a LaTeX kernel change to how
`l/c/r` columns / `\unskip` behave — possibly the same era as the PR#46
breakage) or has been latent since written. Not tested against older versions.

**Workaround / takeaway:** Treat `empty-chord-dim` as non-functional — do not set
it or rely on it in `MyLeadsheets.sty`. To put a chord over a deliberately wide
blank beat, use *rigid* space in the `.song` (`\rule`, `\quad`, `\makebox`), not
this key. (A patch making `place_above` emit `\rule` instead of
`\skip_horizontal` would revive it, but that means editing package internals —
against the project's design goals — so avoid unless specifically wanted.)

## Resolved

- PR#46 fix for `leadsheets.library.songs.code.tex` placed in `~/repos/1sys/tex/songs/`,
  shadowing the unpatched system copy via `TEXINPUTS`
- `\prop_new:c` hack removed from `MyLeadsheets.sty` (no longer needed with PR#46 fix)
- vimtex log parsing: resolved — `aux_dir` set in `vimtex.vim`
  dynamically (`~/.cache/latexmk/{dirname}-{jobname}/`) so vimtex knows where its log is

## Mechanism reference

Internals of the IMPLEMENTED compile-time-transpose features. The *condensed*
authoring contracts (what to type in a `.song`) live in CLAUDE.md's "Output
variants" section; this is the full how-it-works, output mapping, and rationale
for when the mechanism itself needs touching.

### Capo variants

Working end-to-end for `samples/Uberlin.song` (concert source, `key={Gm}`,
`capo=3`): `build-pdfs` produces `Uberlin.pdf` (concert) and `Uberlin-CAPO.pdf`
(Em shapes + "capo 3" note), both showing key Gm. Leans on leadsheets' *native*
capo handling rather than hand-driven transposition — far simpler than the earlier
`\CapoSetup`/`\SongCapo`/5-variant-table sketch it replaces.

**How it works:**

- **`transpose-capo = true` is set globally** in `MyLeadsheets.sty`. With a
  `capo=N` property present, leadsheets transposes every chord by `12 − N`
  semitones = **down N** (mod octave) — exactly the shapes fretted with the capo
  on (`\__leadsheets_check_capo:`, `songs.code.tex`). Chords transpose only when a
  `key` property also exists, so **every song must set `key=`** (concert pitch) or
  leadsheets warns per chord. A non-capo song is unaffected: `capo=0` gives a
  full-octave transpose = identity (verified — no enharmonic drift).
- The fret `N` lives **only in `capo=N`** (the native property). There is no
  `%! capo: N` number and no `\SongCapo` macro any more.
- The **default build is therefore the capo-shapes sheet with a "capo N" note and
  needs no machinery at all** — even a bare vimtex hand-compile is correct.
- The **displayed `key=` stays concert pitch** because the title template prints
  `\songproperty{key}` *raw*, never through the chord transposer. (Keep it raw —
  the stock leadsheets template *would* transpose the key.) `\fixedchord` is only
  needed to render a ♯/♭ glyph in a key like `F#m`, and is still unported.

**The concert-pitch sibling** (e.g. for a bass player) is produced by
`build-pdfs` when the song carries `%! capo-pair`. With it the script builds
**two** PDFs: `Song.pdf` = concert and `Song-CAPO.pdf` = capo shapes. The concert
one injects `\def\ConcertVariant{true}` before `\input` (a `-pdflatex` override
**forced with `latexmk -g`**, since the injected `\def` doesn't change the source
and latexmk would otherwise keep a stale PDF); the capo-shapes one is just a
native build under `-jobname=Song-CAPO`.

**Division of labour:**

- **`capo=N` property** — single source of truth for the fret (and, alone, gives
  the capo-shapes sheet via pure leadsheets — no script needed).
- **`%! capo-pair` magic comment** — boolean; asks `build-pdfs` for the
  concert sibling too.
- **`.sty`** (the "Capo variants" block) — `\providecommand\ConcertVariant{false}`
  + `\newif\ifMyLSconcert`; on the concert build (`\ConcertVariant` = `true`) it
  emits `\setleadsheets{transpose-capo=false}` (chords stay concert) and the
  title template's `\ifMyLSconcert\else … \notebox{capo \songproperty{capo}} …`
  suppresses the note. *Gotcha:* the flag is compared with `\ifx`, so the helper
  `\MyLSconcertflag` must be a plain `\def` (not `\newcommand`, which is `\long`
  and won't `\ifx`-match the injected non-long `\def`).
- **`build-pdfs`** — `wants_capo_pair()` reads the comment; `compile_concert()`
  does the injected `-g` build; `build_song()` routes plain `Song.pdf`→concert and
  `Song-CAPO.pdf`→native shapes.

**Output mapping & policy.** Policy: always transcribe in concert pitch; a capo
song is just a concert song that additionally carries `capo=N`. Two motivating
cases both collapse to that: (1) a song transcribed with no capo (say concert B)
can still emit a `-CAPO` sheet whose chords are the capo shapes; (2) a song
fretted with a capo (concert Gm, sections fingered in Em) can emit both the
Em-shapes sheet and a concert sheet (e.g. for a bassist). The mapping:

    capo=N                  -> Song.pdf       (capo shapes + "capo N")
    capo=N, %! capo-pair    -> Song.pdf       (concert)
                               Song-CAPO.pdf  (capo shapes + "capo N")

Suffix conventions: `-CAPO` (single hyphen) marks *different musical content*;
`--chords`/`--lyrics` (double hyphen) marks a *different presentation of the same
content*.

**Constraint — never `transpose=M` in a capo song.** The capo axis owns
transposition (via `capo=N` + `transpose-capo`); a literal `[transpose=M]`
combines with it (`net_steps = M − N`) and corrupts both sheets. If a baseline
transpose is ever needed alongside a capo, bake it into the authored chords.

**Two independent flag families** decide which PDFs are produced — the
chords/lyrics axis (`%! variants:`) and the capo axis (`capo=N` + optional
`%! capo-pair`). A song uses either alone or both; keep them mentally separate.
Still pending: the `\fixedchord` port for accidental-bearing concert keys (`F#m`
etc.), and folding the chords/lyrics axis together with the capo axis (a chords
variant of *each* pitch).

### Transpose variants

Original-key sibling, parallel to capo but for a *real* key change (the chords'
sounding pitch moves, unlike capo's fingering shapes). Working for
`samples/TransposeDemo.song` (transcribed in C, `transpose=2`): `build-pdfs`
produces `TransposeDemo.pdf` (chords in D) and `TransposeDemo-OriginalKey.pdf`
(chords in C).

- The amount `N` lives in `\providecommand\SongTranspose{N}`; the song option is
  `transpose=\SongTranspose` (a **macro**, never a literal `transpose=2`). l3keys
  passes the value to `\int_set`, which expands the macro — so `transpose=\macro`
  *does* work (unlike a macro standing for a whole `key=value`, which does not).
- **Why the macro indirection is mandatory:** the original-key build must *undo*
  the transpose, and a literal `transpose=2` (parsed last, inside the song group)
  can't be undone. With `\providecommand`, an injected `\def\SongTranspose{0}`
  *before* `\input` wins and the in-song `\providecommand` becomes a no-op — so the
  amount gets an independent off-switch, the separation capo gets from its
  `transpose-capo` bool.

**Division of labour:**

- **`\providecommand\SongTranspose{N}` + `transpose=\SongTranspose`** — the amount
  and its use; the `.song` owns the default (a bare/vimtex compile = transposed).
- **`%! transpose-pair`** — boolean magic comment; asks for the original-key
  sibling too.
- **`build-pdfs`** — `wants_transpose_pair()`; default PDF = native transposed
  build; `compile_original_key()` injects `\def\SongTranspose{0}` (forced `-g`) for
  `SONG-OriginalKey.pdf`. **No `.sty` change at all** — leadsheets does the
  transposition.

**Naming:** `SONG.pdf` (transposed, the default) + `SONG-OriginalKey.pdf` —
single hyphen like `-CAPO`, marking different musical content.

**Capo vs transpose — keep them straight.** Capo changes the fingering shapes,
not the sounding key (label stays concert), driven by `capo=N` + `transpose-capo`.
Transpose changes the sounding key (label should follow), driven by
`transpose=\SongTranspose`. Independent axes; a song uses at most one.

### Deferred — the key label following transposition

The title template prints `\songproperty{key}` raw, so a transposed `SONG.pdf`
currently shows the *authored* key (e.g. "C") even though its chords are in D; the
original-key PDF happens to read correctly. Making the key follow the
transposition is the same conditional flagged for capo — the stock template uses
`\writechord{\songproperty{key}}` (`songs.code.tex:481`), which transposes — so:
branch the key print on `\ifsongproperty{capo}` (capo → raw concert key; else →
`\writechord`, which transposes for `transpose=` and is identity otherwise). The
reference `.sty`'s `\fixedchord` (vestigial, unported) may be relevant here.

## Tooling / debugging

### Vimtex quickfix behaviour

Vimtex's log parser only surfaces `Package X Warning:` patterns into the quickfix
window — `Class X Warning:` messages (e.g. KOMA's own warnings about incompatible
packages) appear in the raw log but are invisible in the quickfix. When diagnosing
warnings, check the raw log at `~/.cache/latexmk/{dirname}-{jobname}/` (vimtex
compilations) or `~/.cache/latexmk/${project_name}_$hash/` (Generate* commands).

## build-pdfs — status

`build-pdfs` (repo root, `~/repos/1sys/tex/songs/build-pdfs`) compiles each
`.song` to its default PDF and reads magic comments in the **leading** comment
block (first non-blank, non-`%` line ends the block, mirroring the TeX-engine
magic comment) to build siblings. The script carries full inline comments; this
is just the orientation and the live status. See *Output variants* in CLAUDE.md
for the authoring contracts and *Mechanism reference* above for the internals.

Magic comments (each axis independent):

    %! variants: chords, lyrics   chords/lyrics axis -> Song--<variant>.pdf
    %! capo-pair                  capo axis -> concert + Song-CAPO.pdf
    %! transpose-pair             transpose axis -> Song-OriginalKey.pdf

**Built:**
- Default PDF for each `.song` (always produced).
- `-h`/`--help`; no args = every `*.song` in cwd, else the named files. A per-song
  failure is reported and skipped; non-zero exit if any failed.
- Aux files to `~/.cache/latexmk/{parent-dir}-{jobname}/` via `-auxdir`, sharing
  vimtex's incremental-build database; the PDF lands next to the source. No
  `latexmk -c` (the `.fdb_latexmk` database is worth keeping).
- **capo axis** — `wants_capo_pair()` + `compile_concert()` injects
  `\def\ConcertVariant{true}` (forced `-g`); `build_song()` routes `Song.pdf`→
  concert, `Song-CAPO.pdf`→native shapes.
- **transpose axis** — `wants_transpose_pair()` + `compile_original_key()` injects
  `\def\SongTranspose{0}` (forced `-g`) for `Song-OriginalKey.pdf`.
- **chords variant** — `compile_variant()` injects `\def\Variant{chords}` (forced
  `-g`), producing `Song--chords.pdf`. The `.sty` honours `\Variant` via two
  parallel families: every lyric verse type (`verse`, `chorus`, …) has a **chart
  twin** `<type>chords` (`versechords`, `choruschords`, …), each a real
  `\newversetype` whose default `template=` is its counterpart's coloured box. In
  the chords build the lyric types are suppressed and the twins render; in every
  other build the twins are suppressed and the lyric types render. Suppression is
  the same xparse `+b` swallow as before (no throwaway box, no warnings). **Both**
  lists in the `.sty` must stay complete — a missing lyric type leaks lyrics into
  the chords sheet, a missing twin leaks a chart into the lyric sheet. Because the
  twin is a real verse type it reproduces its counterpart's colour *and* label
  (`V1:`/`C1:`/`In:`) automatically. Verified on `samples/ChordsVariantDemo.song`.
- **lyrics variant** — same `compile_variant()` path (the `case` arm is just
  `chords | lyrics`), injecting `\def\Variant{lyrics}` for `Song--lyrics.pdf`. The
  `.sty` flips `print-chords=false` for this variant only; the family selection is
  the same as the default build (lyric bodies print, chart twins swallowed). See
  *Variants from one `.song`* item 2 for the `^`/`_` detail.

**Pending:** the `phone` variant; folding the chords/lyrics axis together with the
capo/transpose axes (e.g. `Song--chords-CAPO.pdf` — not wired
up, since `read_variants` runs off the default-pitch source after the
capo/transpose branch). The `-CAPO`/`-OriginalKey` (single-hyphen, *different
musical content*) vs `--chords`/`--lyrics` (double-hyphen, *different presentation*)
suffix split is the convention to preserve when that cross-product is built.

## Future work — output variants from one `.song`

The reference produced several PDFs from one source via a tangle of `\newtoggle`
flags that cross-set each other (`LyricsWithChords`, `LyricsNoChords`,
`ChordProgression`, `ChordsAndLyrics`, `SmallLyrics`, `Structure`) and four
content-gating macros (`\chords`, `\lyrics`, `\lyricswithchords`, `\structure`),
each `\iftoggle{…}{#1}{\@gobble{#1}}`. The cost landed in the `.song`: the same
material was typed 3–4× per section, once per variant. The overhaul should reach
the same outputs with far less duplication. Planned variants:

1. **Default — lyrics + chords.** The `^{C}word` lyric body. Baseline. *Done.*
2. **Lyrics-only — DONE, the `lyrics` variant.** The **same** `^{C}word` body with
   leadsheets' `print-chords=false`. No duplication — one body serves both 1 and 2.
   Selected by `\def\Variant{lyrics}`; renders the same family as the default build
   (lyric types print, chart twins swallowed) and only flips `print-chords`. Key
   detail: `print-chords=false` suppresses **only** `^{C}word` chords (`^` =
   `\getorprintchord`, gated by `print-chords` in `\leadsheets_place_above`); the
   `_{C}` chord-only tokens in `\measures` go through `\writechord` and print
   regardless (`songs.code.tex:143-144`). So lyric sections become words-only while
   instrumental sections (a `\measures` line of `_{C}` chords, no `^{}word`) keep
   their progression — the sensible outcome, and it falls out of the documented
   `^`/`_` split with no extra code. Verified on `samples/ChordsVariantDemo.song`.
3. **Chart-only (no lyrics) — DONE, the `chords` variant.** A **separate** body
   built with `\measures` inside per-type chart twins `\begin{versechords}` …
   (see *build-pdfs — status*).
   The `.song` carries a chord skeleton in addition to the lyric body — i.e. some
   "duplication", **but that's a feature, not a bug**: "hide the lyrics" is both
   technically difficult *and* would produce garbage. There is no `print-lyrics`
   key in leadsheets, and chords routinely change **mid-bar and mid-word**
   (~80% of the cover corpus, e.g. `M^{B7b9}E`), so the chord position over the
   lyric carries information that can't be cleanly stripped. Auto-extraction
   would be a fragile parser over lyric token-soup (`\ul{}`, `\ssp`, math, bare
   letters); the reference hand-wrote the chart for exactly this reason. The
   chord skeleton is short, and it's the representation a band actually wants.
4. **Phone-format.** *Pending.* Long-paper geometry (tall narrow page), otherwise
   the default lyrics + chords layout. (Reference: `\Longpaper`.)

Variant selection is a single `\def\Variant{…}` command-line flag (injected by
`build-pdfs`) reduced to **one** clean variant state in the `.sty`, not the
reference's six cross-set toggles — realised for `chords` and `lyrics` (each a
small `\newif`/`\ifx` block in the "select which family renders" section, plus the
`print-chords` switch at the file end); `phone` extends the same lever.

The **Reference's `Structure` option is abandoned** — not being carried forward.

Possible **class option for grayscale** for any variant (phone excepted, but no
strong reason either way) — for B&W printing, mirroring the reference's
`\BlackAndWhite` branch that swapped tinted section boxes for ruled ones.

### `.song` API — parallel section environments (DONE)

**Implemented.** The `chords` variant uses **per-type parallel environments**: the
lyric body stays in `verse`/`chorus`/…, and the chart goes in a parallel twin
`<type>chords` (`versechords`, `choruschords`, `introchords`, …). The active
variant prints exactly one family; the other's bodies are swallowed (xparse `+b`).
Usage:

    \begin{verse}[template=versebox]
      ... ^{C}word lyric body — prints in lyrics+chords and lyrics-only ...
    \end{verse}
    \begin{versechords}
      ... \measures chart body — prints in chart-only ...
    \end{versechords}

Each twin is a real `\newversetype` whose **default** `template=` is its
counterpart's coloured box (`versechords` → `versebox`), so it needs **no
`[template=]`** in the `.song` and reproduces the lyric section's colour *and*
label (`V1:`/`C1:`/`In:`) automatically — the labels line up section-for-section
because the sections are authored in parallel. `choruschords` deliberately drops
chorus's italic (it takes the global upright/bold `verses-format`). Defined in the
`.sty`'s "Output variant: chords" block: the `\newversetype` table, a
`\setleadsheets` block mirroring the labels, and the two suppression `clist`s.
(Naming chosen `<type>chords` over the earlier `chartverse` sketch.)

**Naming convention** — adding a section type to the corpus now means adding *both*
the lyric type (top of `.sty`) and its `<type>chords` twin (chords block), and
listing each in its suppression `clist`. `\RenewDocumentEnvironment` errors on a
missing name, so a half-added type fails loudly rather than silently leaking.

**Per-section `minmeasure=` — DONE.** `\begin{versechords}[minmeasure=3cm] …`
sets *that* section's measure floor, overriding the global `\minmeasure`, then
reverts at `\end` (a leadsheets section is a TeX group, and the key's `\setlength`
runs through the environment's `\keys_set` inside that group). It's the clean
declarative lever for roomier chart bars — preferred over the per-measure `\mw`
hack below. Implemented in the `.sty`'s "Per-section `\minmeasure` override" block
as a `minmeasure` key `\keys_define`d onto the `leadsheets/<type>` path of every
managed section type (lyric types *and* chart twins, so it works on an
`\measures`-bearing intro/outro too, not just charts). Wider content still keeps
its natural width — the key moves only the *minimum*. Verified on
`samples/ChordsVariantDemo.song` (`versechords[minmeasure=3cm]` widens V1's bars;
C1/Out revert).

A third case rounds this out: **instrumental sections that should appear
identically in both the lyric and chart outputs** — a solo, intro, outro, or
break whose body is just a chord progression with no lyrics. Rather than
duplicate them into a `solo` + `solochords` pair (as the demo currently does for
`intro`/`introchords`), mark the single environment to appear in both:

    \begin{solo}[both]
      \measures{ ... }   % shown in lyrics+chords AND in the chart
    \end{solo}

This works precisely *because* such a body is already chart-shaped (a
`\measures` line of `_{C}` chords, no stacked `^{C}word` lyrics), so the same
content renders correctly in both contexts. `[both]` is therefore meaningful
**only** for lyric-free, chart-compatible bodies — it is *not* a way to show a
lyric verse in the chart (that's the whole reason charts are a separate body).

Two things to decide when building `[both]`:

- **Behaviour in the lyrics-only variant.** A chord-only solo has nothing left
  once chords are suppressed. Options: omit the section, leave it blank, or show
  a short note (the reference used `\notebox{8 bar solo}` here).
- **The token name.** With three variants, `[both]` presumes the
  lyrics+chords/chart pair specifically; a clearer token (e.g. `[shared]`, or an
  explicit `[in=lyrics,chart]` form that scales to any subset) may read better.

### Chart-only body — desired capabilities

The chart builder (extending `\measures`) should support:

- **Repeat / beat markers centered in the bar**, e.g. `*` or `/` for "play the
  same chord on this beat" (a bar like `| Em / / / |`).
- **Multi-chord bars** with beat positions, e.g. `| Em * Am Bm |` where `*`
  marks beat 2, etc. — i.e. several chords laid out across the bar's beats.
- **Barline alignment across lines** wherever possible: bar N at the same
  horizontal position on every line of a section. This is the harder
  fixed-width / `tabular`-column feature already flagged in *Known issues* (the
  current `\measures` guarantees only a *minimum* spacing, not an aligned grid).
  For a chart specifically, an aligned grid is more important than for the lyric
  body, so this is where that feature would first earn its complexity.
  - The proper, *non-hacky* route is a real `tabular`/`array` grid for the
    section: each measure a cell, bars drawn inside cells (since repeat/final
    bars vary per row, `tabular`'s fixed column rules can't do them). The column
    algorithm then sizes each column to its **widest cell across all lines** —
    note this is *better* than "match line 1", which would clip a later line
    whose bar happens to be wider. Real complications to expect: per-row bar
    styles, ragged measure counts (`\multicolumn`), and "min-width but grow"
    columns (a `p{}` floor + auto-grow needs a `varwidth`/measuring trick).

- **Per-measure width override `\mw{<dim>}` — implemented, tested clean, then
  reverted** (a bit hacky/un-tex-y for routine use; kept here as a proven escape
  hatch). Used inside a measure group to stretch one bar past `\minmeasure`,
  e.g. `{\mw{2.4cm}_{G}}` or `{\mw{2\minmeasure}_{G}}`. Still a *minimum* (wider
  content keeps natural width). Mechanism if revived: `\NewDocumentCommand\mw{m}`
  sets a **global** flag + dim (global so it survives the `\hbox_set` group that
  measures the content) and emits no ink; `\__myls_measurebox` resets the flag
  before measuring, then uses the override as the box's min target when set. It's
  the manual lever for one-off stretches; the per-section `minmeasure=` option
  and a real grid (above) are the cleaner answers for the systematic case.

