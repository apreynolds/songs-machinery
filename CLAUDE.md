# Song sheet creation using LaTeX

This repo contains custom LaTeX `.sty` files, based on the `leadsheets` package
(https://github.com/cgnieder/leadsheets) which uses KOMA-Script under the hood
(`leadsheet.cls` loads `scrartcl`). The package is no longer maintained. The
functional package lives at
`/usr/local/texlive/2025/texmf-dist/tex/latex/leadsheets/`. Note the file
`leadsheets.library.songs.code.tex`, which carries a fix for an issue caused by a
more general LaTeX update; see https://github.com/cgnieder/leadsheets/pull/46.

**`MyLeadsheets.sty`** is the main file — a clean, KOMA-native rebuild (replacing
an older package hacked together over years without KOMA knowledge). The repo is
now in **maintenance/tweak mode**: the features and formats I wanted are in place,
and work now means small adjustments to `MyLeadsheets.sty` or the `build-pdfs`
script. There is no active feature backlog; the design history and a couple of
low-priority reference items live in NOTES.md.

See **`NOTES.md`** for reference detail kept out of these instructions: package
issues, resolved problems, implemented-feature mechanism internals, design
history, and tooling/debugging notes. This file holds the live authoring
contracts and conventions; NOTES.md is the "when you actually touch that code"
detail.

## Design goals / principles

- **Compile completely cleanly** — no warnings in the log, not just no errors
- **Minimize hackiness** — prefer KOMA-native solutions over third-party packages
  that fight with KOMA; use `\setkomafont`, `\KOMAoptions`, etc. instead of
  `titlesec`, `parskip`, etc.
- **Prefer well-understood, maintainable code** — if uncertain about a change,
  be cautious and test carefully; this project is not time-pressured
- **Avoid `leadsheets` internals** — prefer documented, standard-LaTeX solutions
  built on the package's public API. Reaching into private internals (e.g.
  double-underscore `\l__leadsheets_…` variables) is a last resort, justified
  only when there's no documented path — as with the PR#46 fix.
- **expl3 is allowed but not preferred** — willing to use LaTeX3/expl3 when
  genuinely necessary, but a documented plain-LaTeX route wins when one exists.
  Example: the coloured section boxes are wired up as each part type's *default*
  template in the `.sty` (`\setleadsheets{verse/template=versebox, …}`) rather
  than via an expl3 helper, so a bare `\begin{verse}` in the `.song` is already
  boxed.
- **Keep both `.song` and `.sty` as clean as possible** — these two goals often
  conflict (a cleaner `.song` can push complexity into the `.sty`, and vice
  versa). Expect case-by-case judgement calls; there's no fixed rule.
- **`geometry` is intentionally kept** — KOMA's typearea is designed for
  typographic layouts; song sheets use very tight custom margins (0.5cm) that
  are deliberately non-typographic. KOMA does not warn about `geometry`.

## Song-file (`.song`) conventions / `.sty` API

Quick orientation for reading a `.song` (details and rationale are in
`MyLeadsheets.sty`'s section comments):

- **Song header** — set entirely from the `\begin{song}[]{…}` properties; the
  title template prints one line — `[arrangement] Title (Band) Key [capo] …
  [genre][difficulty]` — plus an optional second `Tuning: …` line. In a songbook
  the *same* heading is reused as the TOC entry (so the TOC carries the rich
  formatting), **except** the tuning line, which is deliberately kept off the TOC,
  the PDF bookmark, and the running head. There is **no `tempo`** in the header
  (dropped in the redesign). Properties:
  - `title`, `band` — leadsheets built-ins. `key` prints **raw** (concert pitch,
    never transposed); wrap an accidental key in `\fixedchord{…}` so the sharp/flat
    glyph renders, e.g. `key={\fixedchord{Gm}}`. **Write accidentals as `\sharp` /
    `\flat`, never a literal `#`** — e.g. `key={\fixedchord{F\sharp m}}`,
    `key={\fixedchord{B\flat}}`. A raw `#` compiles standalone but breaks in a
    songbook: the heading is reused as the TOC entry and a literal `#` doubles to
    `##` in the `.toc` under hyperref, erroring on read-back. `\sharp`/`\flat` are
    the exact glyphs `\writechord` converts `#`/`b` into, so output is identical
    (see `\fixedchord` in `MyLeadsheets.sty`). `capo=N` adds a `\notebox{capo N}`
    label after the key (fingering shapes, *not* a key change — see "Output
    views and arrangements").
  - `istransposed` — a **purely cosmetic** marker that the song has been moved off
    its written key (e.g. dropped a few semitones to sing comfortably), so it shows
    at a glance without reaching for the original. It changes **nothing musical** —
    unlike leadsheets' built-in `transpose=` (which moves the sounding pitch) or
    `capo=` (fingering shapes). Prints a small TikZ arrow plus an optional amount,
    wrapped in coloured parentheses, **between the key and the capo**:
    `istransposed={up 3}` → `(↑3)`, `{down 2}` → `(↓2)`, `{up}` → `(↑)`. The first
    word picks the arrow; the rest prints after it. The arrow is sized to the
    **height of a digit in the current font** (measured per-draw, so it matches the
    amount beside it and tracks the heading/TOC font); its baseline is its
    bounding-box south, so the up-arrow's tail and the down-arrow's tip sit on the
    text baseline. `\transup`/`\transdown` remain as standalone glyph macros and
    still work as a legacy value (`istransposed={\transup 2}`). Knobs:
    `\MyLStransposescale` (default `1` — multiplier on the measured digit height),
    `\MyLStransposecolor` (default `BrickRed`, tints the arrow, amount, and parens).
    Survives into a songbook TOC like the capo note (robust `\MyLStransposed`, see
    `MyLeadsheets.sty`).
  - `genre` and `difficulty` share the **right margin**, in that order (genre as
    green text, then the difficulty glyph). All four combinations work (neither /
    genre / difficulty / both). `difficulty` takes one of three glyph macros:
    `{\easy}` (green ●), `{\moderate}` (blue ▲), `{\difficult}` (black ◆).
  - `arrangement` — a short monospace tag (`arrangement={pno.}`, `{gtr.}`, `{cp.}`)
    printed bracketed and blue to the **left** of the title (`[pno.]`); handy for
    telling wrapper siblings of one song apart (piano vs guitar arrangement). The
    slot has a **6-char minimum width** so tagged titles line up in a songbook TOC;
    `arrangement={}` (empty) reserves the blank box to align an untagged sibling,
    while an *absent* `arrangement` prints nothing. Restyle with
    `\MyLSarrangementfont` / `\MyLSarrangementcolor` / `\MyLSarrangementraise`.
  - `tuning` — rare; prints the `Tuning: …` second line below the heading
    (accidentals rendered, never transposed, via `\MyLSformattuning`). Restyle /
    reposition with `\MyLStuningfont` / `\MyLStuningraise`. Mechanism internals (the
    min-width box, the TOC/bookmark safety, the post-`\section` tuning line) are in
    `NOTES.md`.
  - `tempo` (a leadsheets built-in) is accepted but rendered **nowhere** — not in
    the header (dropped in the redesign) and, since the Remarks-block decoupling,
    not there either. Setting `tempo=` is harmless but has no visible effect.
    (There is no longer a `remarks` *property*; "remarks" now means only the
    environment below.)
- **Remarks block** — a `remarks` environment, written **after** `\end{song}`
  (outside the song env, so prose has normal catcodes and no song-body bolding),
  appends a block **just below the song's last section**: a header `Remarks`,
  then the environment's own free-text body (set in `\small` to subordinate it).
  A bare `\begin{remarks}\end{remarks}` prints just the header. The block reads
  **nothing** from the song (pure free text — no title, tempo, or property pulled
  in), so it has no song-id plumbing and no coupling to the preceding song. It is
  separated from the body by a fixed (slightly rubber) `\addvspace`, **not** pushed
  to the page bottom — an earlier `\vfill` design left the block marooned in
  mid-page whitespace on short songs; if it doesn't fit it flows to the **top** of
  the next page. It travels into a songbook like any post-`\end{song}` body.
  Restyle via the `\MyLSremarks*` knobs (`…label`, `…headfont`, `…rule`,
  `…bodyfont`, and `\setlength\MyLSremarksskip{…}` for the gap above).
  - **View filtering** — an optional comma-list of view names limits the block to
    those views: `\begin{remarks}[chords]` prints only in the `chords` view,
    `\begin{remarks}[chords,full]` in those two. A bare `\begin{remarks}` (empty
    list) prints in **every** view. The `debug` view is **exempt** — it always shows
    every remarks block (it proofs everything on one sheet). Mechanism in `NOTES.md`.
- **Coloured section boxes** are the *default* for every part type, so a bare
  `\begin{verse}` … `\begin{chorus}` … is already boxed in its own colour — no
  `[template=]` needed. The named templates (`versebox`, `chorusbox`, `solobox`,
  `introbox`, `outrobox`, `interludebox`, `bridgebox`, `liftbox`, `prechorusbox`,
  `postchorusbox`, `custompartbox`) remain only as per-section *overrides*: pass
  `[template=chorusbox]` to borrow another type's box, or `[template=itemize]`
  (leadsheets' built-in) for a plain, un-boxed section. The defaults are set in
  the `.sty`'s "make each box the part type's DEFAULT template" block.
- **Section labels** auto-number with a short prefix (`V1:`, `C1:`) for
  verse/chorus, or show a short name (`In:`, `S:`, `Out:`, …). Override any one
  section with `[name=…]` (and `[named=…,numbered=…]`).
- **Chords**: `^{C}` sets a chord above the following word; `_{C}` is a
  chord-only token (no lyric). Rendered brick-red, bold, sans, left-aligned.
- **`\measures`** — the music-line builder giving a *minimum* bar spacing:
  `\measures[<style>]{ {m1}{m2}{m3} }` draws one line of measures, each in a
  left-aligned box of at least `\minmeasure` (default 2.5cm) with `\measurepad`
  (default 0.35em) on each side of the content, so sparse measures (a lone
  chord) don't collapse and glyphs don't butt the barlines. `<style>` is empty
  (normal bars), `repeat` (`|: … :|`), or `final` (`… |||`); add more styles via
  the `\str_case` in the `.sty`. Lines with no measures are written plain (not
  wrapped). NB: leadsheets' `empty-chord-dim` is dead (see `NOTES.md`), so
  `\measures` is the spacing mechanism, not chord padding.
- **Per-section `minmeasure=`** — any section type accepts a `minmeasure` option
  that overrides the global `\minmeasure` floor for *that* section only, then
  reverts at `\end` (the section is a TeX group): `\begin{versechords}[minmeasure=3cm]
  … \end{versechords}`. Most useful on chart (chords) sections, where wider bars
  align better; also valid on lyric types with `\measures` (e.g. an intro). Still
  a *minimum* — wider content keeps its natural width.

## Output views and arrangements (authoring contracts)

These were both called "variants" once; they are now two distinct words. A song
yields several PDFs via two **different** mechanisms, which compose:

1. **Views — presentation axis (one `.song`, magic comment).** Different
   *renderings* of the same content — chords vs lyrics — come from a single
   `.song`. `build-pdfs` reads a magic comment in the **leading** comment block
   (before `\documentclass`, mirroring the TeX-engine magic comment) and builds
   the siblings.
2. **Arrangements — musical-content axis (separate wrapper `.song`s, no magic comment).**
   Different *content* — capo shapes vs concert, a real key change, or any number
   of keys — comes from **separate wrapper files** that each `\input` a shared
   body. No magic comment and no `build-pdfs` cleverness: each wrapper is a normal
   standalone `.song` that compiles to its own PDF.

Mechanism internals and rationale are in `NOTES.md`.

- **Views — `%! views: chords`.** Selects which *body* prints.
  Every section is authored as a **pair**: the lyric body in `verse`/`chorus`/…
  as usual, and the chart body in a parallel **chart twin** named `<type>chords`
  — `\begin{versechords} … \measures … \end{versechords}`,
  `choruschords`, `introchords`, `outrochords`, `interludechords`,
  `bridgechords`, `solochords`, `prechoruschords`, `postchoruschords`,
  `liftchords`, `custompartchords`. Each twin is a real verse type whose default
  template is its counterpart's coloured box, so — like the lyric types — it
  needs **no `[template=]`** and renders in the **same colour and same label**
  (`V1:`/`C1:`/`In:`) as its lyric section. The **`full` view** — the default
  build, the suffixless `Song.pdf` (`\View` defaults to `full`) — prints the lyric
  bodies (chords over words) and swallows the twins; the
  `chords` view (`Song--chords.pdf`) prints the twins and suppresses every
  lyric verse type. The **`lyrics` view** (`Song--lyrics.pdf`) prints the same
  lyric bodies as the `full` view but with `print-chords=false`, so `^{C}word`
  chords are stripped to leave the words; `_{C}` chord-only tokens in `\measures`
  still print (the documented `^`=print-or-place / `_`=write split), so an
  instrumental section keeps its progression. The **`debug` view**
  (`Song--debug.pdf`) prints **both** families in one sheet, for proofing line
  lengths and spacing without flipping between PDFs. The **`phone` view**
  (`Song--phone.pdf`) prints the **same content as `full`** (lyric bodies, chords
  over words) but on a taller page — 8.5×22in instead of letterpaper, *same width,
  same 0.3cm margins, same font* — so a song that fits the usual "two letter pages"
  soft target lands on a single tall "phone page" read as one scroll (longer songs
  flow onto a second phone page). Two songbook behaviours change in this view:
  `\clearoddpage`/`\bookinclude` skip their odd-page padding (page parity is moot
  on the tall layout, and the blanks would be very long), and the back-to-contents
  link is enlarged (`\large` vs the other views' `\footnotesize`). The only knobs
  are `\MyLSphoneheight` (default `22in`) and `\MyLStoclinksize` (set to `\large`
  under phone). The **grayscale modifier** is *orthogonal* to the content axis:
  it switches **all colour off** for plain printing and composes with full/chords/
  lyrics as the view tokens **`full-gray`**, **`chords-gray`**, **`lyrics-gray`**
  (plus bare `gray` = `full-gray`). It is implemented as a true modifier — the
  build strips the `-gray` part, sets `\ifMyLSgray`, and runs the **base** content
  view unchanged — so e.g. `lyrics-gray` strips over-word chords exactly like
  `lyrics`. Colour is killed two ways: `xcolor` loads with the `monochrome` option
  (every coloured glyph — chord symbols, noteboxes, `\bg` text, difficulty/genre/
  arrangement markers — falls back to black ink) and the coloured section boxes are
  forced to a **white** background (no tint, no border added). Filenames:
  `chords-gray`/`lyrics-gray` keep their name (`Song--chords-gray.pdf`), but
  `full-gray` (and `gray`) both output `Song--gray.pdf` (the `full-` is dropped).
  Deliberately minimal — there are no knobs.
  - *Shared instrumental sections.* A lyric-free, chart-shaped section (an
    intro/outro/solo that is just a `\measures` line) has **identical** content in
    both the lyric type and its twin. There is no `[both]` option; instead, to
    keep the two from drifting, define the body once as a macro and reference it
    in both wrappers: `\newcommand{\IntroChords}{\measures{…}}` then
    `\begin{intro}\IntroChords\end{intro}` and
    `\begin{introchords}\IntroChords\end{introchords}`. (The genuinely *different*
    lyric-vs-chart sections — verse/chorus — can't be shared this way and aren't
    meant to be.)
- **Arrangements — wrapper `.song`s sharing a body.** For *any* arrangement
  whose musical content differs (capo shapes, concert pitch, a transposed key,
  several keys at once): extract the song body — the `\begin{verse}…` sections,
  **no preamble**, not `\begin{song}` — into `Song--input.song`, then write thin
  wrapper files that each set their own `\begin{song}[…]{…}` options and
  `\input{Song--input.song}`:
  - **Concert vs capo.** Transcribe in concert pitch; every wrapper sets `key=`
    (concert — required, or leadsheets warns per chord). `Song.song` omits `capo`
    → concert sheet; `Song-CAPO.song` carries `capo=N` → capo shapes plus a "capo
    N" note. The key label stays concert pitch in both (the note keys off
    `\ifsongproperty{capo}`). **Never put both `capo=N` and `transpose=M` in one
    wrapper** — they combine and corrupt that sheet.
  - **Key change (transpose).** A *real* key change (the sounding pitch moves,
    unlike capo): the wrapper carries `transpose=N` — now a **plain literal**
    (the old `\SongTranspose` macro indirection is gone, since nothing has to undo
    it). `Song.song` with `transpose=2`, `Song-OriginalKey.song` with no
    `transpose=`.
  - Name wrappers for the PDF you want (`Song.song`, `Song-CAPO.song`,
    `Song-OriginalKey.song`, `Song-Dmaj.song`, …). `build-pdfs` compiles each to
    one PDF and **skips `*--input.song`** (it has no preamble). Because each
    wrapper is a full standalone song, dropping different versions into one
    songbook is trivial (`\bookinclude` each). Filename gotcha: the shared file
    **must** use `--`, not `__` — `_` is active inside `song` (chord syntax) and
    mangles an `\input`-ed underscore filename (see `NOTES.md`).

The two mechanisms **compose**: a wrapper (one arrangement) may itself carry
`%! views: chords` to get the chords/lyrics views of that particular key/capo.

Capo vs transpose: capo changes the fingering shapes only (label stays concert);
transpose changes the sounding key (label should follow). A single wrapper uses
at most one. Suffix convention: `--chords`/`--lyrics` (double hyphen) marks a
*different presentation* of the same content; `-CAPO`/`-OriginalKey` (single
hyphen) marks *different musical content*; `--input` (double hyphen) marks the
shared body include, never compiled directly.

Transposed-chord **spelling**: leadsheets picks a sharp/flat lean from the *transpose
step count*, not the resulting key, so a capo (or a from-non-C transpose) can spell a
section with the accidental-heavy enharmonic — e.g. capo 1 lands a Db-area section on
`C#/D#m/F#`. Force the lean with `enharmonic=sharp/flat`; if that pushes into double
accidentals (capo 1 + `enharmonic=flat` → `Dbb` for a C), add **`\simplifyaccidentals`**
to collapse the theoretical spellings to conventional names (`Dbb→C`, `F##→G`, `Cb→B`,
`E#→F`, …). It's an opt-in switch, `\setleadsheets`-scoped (whole sheet / rest-of-song /
one section, `\simplifyaccidentalsoff` to undo), and composes with `enharmonic=` — the
usual capo pairing is `enharmonic=flat` + `\simplifyaccidentals`. Don't use it when you
genuinely want a theoretical key (C# major's `E#/B#`). Mechanism in `NOTES.md`.

## Fitting — `\resize`

Songs aim (within reason) to fit two pages, and song parts are **atomic** — a
verse/chorus never splits across a page, so an overflowing part bumps wholesale
to the next page, often leaving a blank tail. That whitespace is a feature, not
a bug. But when a part *just barely* overflows, one knob can buy it back.

- **`\resize[<factor>]` — per-document vertical tightener.** Put it before *or*
  after `\begin{document}`; it scales the three **vertical**-space contributors
  by one factor, leaving **font size, page geometry, and every horizontal measure
  untouched** (each song still feeds a shared songbook, so type stays uniform;
  chords stay `\normalsize`). The three levers: (1) interline gap inside a section
  (`\openup\mysonglinesep`), (2) section-box internal padding (`top`/`bottom`/
  `boxsep`), (3) the gap *between* consecutive boxes (`beforeafter skip`).
  `\resize[1]` is an **exact, pixel-verified no-op**; `<1` tightens, `>1` loosens;
  the default arg is `0.95` (a gentle nudge — pass `0.9`/`0.85` to smush harder).
  The factor multiplies **on top of** the base length knobs (`\mysonglinesep`, the
  2mm/1mm box padding stay the per-quantity levers), so the two compose. It is the
  right tool only for a *marginal* bump: a song whose whole last third is on the
  overflow page is a genuine multi-pager and would need an unflatteringly small
  factor. Mechanism (which tcolorbox defaults it restates, why placement is free)
  in `NOTES.md`.
  - **In a songbook, put `\resize` in the song *body* (after `\begin{document}`),
    not the preamble** — `\includeleadsheet` gobbles each song's preamble (same as
    a left-in `\ForceEvenPages`), so a preamble `\resize` is silently ignored in a
    book (it still works standalone). Body placement works **both** standalone and
    in a book, so it is the portable spot. A body `\resize` is **automatically
    scoped to its own song**: `\bookinclude` wraps the include in a group and
    `\resize` is a local `\renewcommand`, so it reverts for the next song — no
    leak (pixel-verified). A `\resize` in the *songbook's* preamble instead sets a
    **book-wide default** that any song may locally override and revert from.

## Songbooks & even-page output

These PDFs are read on screen in two-page (spread) mode, where page parity
matters. All three tools live in `MyLeadsheets.sty` ("Even-page / odd-start
padding" + "Songbook back to contents link" sections); internals in NOTES.md.

- **`\ForceEvenPages` — standalone song.** Put it in a song's preamble to pad the
  PDF to an even page count (one blank page appended at `\AtEndDocument` when the
  content is odd). Off by default. No effect inside a songbook — `\includeleadsheet`
  gobbles a song's preamble, so a left-in `\ForceEvenPages` is simply ignored there.
- **`\clearoddpage` — songbook.** Use in place of `\newpage` before an
  `\includeleadsheet` to start the next song on an *odd* page (pads the preceding
  song / the TOC to even, inserting a blank only when needed). KOMA's native
  `\cleardoubleoddpage` won't do this in this single-sided class (it gates the
  blank behind `\if@twoside`).
- **`\bookinclude{file}` — songbook (supersedes `\clearoddpage\includeleadsheet`).**
  Clears to an odd page, includes the song, then drops a **fit-tested
  back-to-contents hyperlink** in the bottom-right of the song's *last* page — but
  only when it fits without forcing a new page (a chock-full last page gets none).
  Songs need no edits and a standalone compile never sees it. Put `\MyLStoctarget`
  just before `\tableofcontents` as the jump destination; the songbook must load
  `hyperref`. Restyle via `\renewcommand\MyLStoclinktext{…}`; opt one song out of
  the link by reverting that line to plain `\clearoddpage\includeleadsheet`.

## Behaviour notes & gotchas

- **`\measures` min-width is not a column grid (by design).** Wide (lyric-heavy)
  measures expand past `\minmeasure`, so barlines are *not* aligned vertically
  across the lines of a section — they're only guaranteed *no closer* than the
  minimum. This is accepted, not a TODO; an aligned grid (bar N at the same x on
  every line) would be a separate, harder feature and is not planned.

- **`\repeatbar` — ported and live.** The TikZ repeat glyph (two brick-red dots +
  a slash) is now in `MyLeadsheets.sty` (the "Inline markers" block), so legacy
  raw-pipe `.song` files that use `$\repeatbar$` compile. It is a **standalone
  macro**, not wired into the `\measures`/chart builder; whether repeat marks
  should instead be a `\measures` style remains an open design question, but the
  port doesn't pre-empt it (the builder already has the `repeat` *style*, giving
  `|: … :|` barlines, which is a different thing from the inline glyph). A few
  legacy raw-pipe files (`| ^{Gm}word |`, hand `\hspace`) remain unconverted to
  `\measures` but still compile.

- **`\notebox` — promoted; `\mydot` — superseded by `\beat`/`\beatdot`.** Both were
  ported verbatim from the reference into `MyLeadsheets.sty` (the "Inline markers"
  block), depend only on already-loaded packages (TikZ via `tcolorbox`, colours via
  `xcolor`), and predate the clean rebuild. Status now: **`\notebox` is
  load-bearing** — the capo header label (`\notebox{capo N}`, title template) uses
  it, so it stays. **`\mydot` is now a DEPRECATED alias** for the bare `\beatdot`
  (one-time warning), kept only so legacy archive `.song` files compile; the
  converter rewrites it away. (`\repeatbar` is ported — see the entry above.)

- **Beat markers `\beat` / `\beatdot`.** A brick-red dot marking a beat where the
  previous chord sustains, in two forms because the contexts need opposite spacing:
  `\beatdot` is the **bare** dot for OVER-A-WORD use (`^{\beatdot}` — "this word
  lands on a later beat, chord still ringing"), where the `^` already positions it;
  `\beat` is the **self-spacing** standalone marker for charts (`| A \beat \beat
  B |`), a `\beatdot` padded by the tunable `\beatspace` (default `0.8em`,
  `\setlength\beatspace{…}`, group-local). Use `\beatdot` inside `^{}`/`_{}`, `\beat`
  standalone — never `\beat` inside a chord (its `\hspace` would shove the dot off
  the word). The over-word dot is a real feature (used meaningfully in ~15 cover
  songs); whether a given file's dots are kept or were just cosmetic is a per-file
  (sometimes per-section) judgement the converter can't make — see the convert
  skill's flag for it.

- **Backing-vocal de-emphasis `\bg … \bgoff`.** A *switch* (not a `\bg{…}` wrapper)
  that greys + shrinks lyric text for subordinated backing vocals across `\measures`
  cells; auto-clears at each line end. **Position relative to a chord is meaningful,
  not a gotcha** — a `^{chord}` binds whatever *immediately* follows it as the word
  under the chord, so the three forms are three intents: `\bg ^{A}word` = chord
  **over** the grey word (`\bg` before the chord); `^{A} \bg word` = chord **floats
  free**, lyric starts after the hit (the space gives the chord an empty slot);
  `^{A}\bg word` = **ambiguous, avoid** — `\bg` is captured as the chord's word and
  greys only that one word. `\bgoff` likewise: end on a plain word (`word\bgoff`) or
  before a chord, never `^{A}\bgoff`. The reliable habit is **positional** (put the
  switch before a chord), *not* "a space on each side" — that space is exactly what
  detaches a chord when the switch follows it. Mechanism + the leadsheets
  chord-capture detail are in `NOTES.md`; full contract in the `.sty` `\bg` block.

# Files

## Key files 

- **This repository** — the custom `.sty` files (`MyLeadsheets.sty` et al.); the
  primary work area, edited **in place**. The repo may be checked out standalone
  or added to another project as a `.machinery` submodule; either way, *here* is
  where the `.sty` work happens — there is no separate "upstream" copy to edit
  instead.
- `/usr/local/texlive/2025/texmf-dist/tex/latex/leadsheets/` — system-installed leadsheets package
- `~/.config/vim/custom/vimtex.vim` — vimtex config; sets `aux_dir` so aux/log
  files land under `~/.cache/latexmk/…`, out of the working tree (where to look
  when checking that a compile is clean)
- `~/.config/vim/UltiSnips/song.snippets` (external — the author's vim snippets,
  bound to `<F3>`) — `.song` authoring snippets plus, at the **top**, a curated
  "KNOBS / adjustable parameters" cheat-sheet: a single-keypress recall list of
  the tunable lengths/colours/views in `MyLeadsheets.sty`. Keep that cheat-sheet
  in sync when a knob's name or default changes here.
- **Shadowing the system leadsheets files.** This repo carries its own
  `MyLeadsheets.sty` and the PR#46-fixed `leadsheets.library.songs.code.tex`,
  which must win over the texlive copies. Two ways that happens, depending on how
  the repo is deployed:
  - *Standalone* — shadowing works only because the repo lives **inside** a
    directory tree that the shell puts on `TEXINPUTS` recursively (Patrick's
    `~/.config/zsh/.zprofile` exports a leading `.` plus a recursive `~/repos`
    tree, then a trailing `:` to re-append the system default). This is tied to
    the checkout location.
  - *As a `.machinery` submodule* — the repo is *not* on that tree, so the parent
    project's `.latexmkrc` prepends `.machinery//` to `TEXINPUTS` at build time
    (a location-independent walk-up that finds the nearest `.machinery`). This is
    the portable mechanism; no shell setup needed.

- `SAMPLE-OUTDIR` (repo root, tracked) — template for `build-pdfs` output
  redirection: copy/rename it to `OUTDIR` (or `.OUTDIR`) inside a song directory
  to send that directory's built PDFs elsewhere. Its own header comments explain
  the format.

## Song files for testing

The `.sty` work is verified against real song files kept in their own repos, not
in this one (the old `samples/` directory is gone). I'll point you at the relevant
song directory per conversation; such a directory typically carries its own copy
of `build-pdfs`. `TEMP/` here (gitignored) holds local scratch copies for trying
changes without touching a canonical repo.

When working in a song repo, ignore compiled/binary assets: `*.pdf` and LaTeX aux
files (`*.aux`, `*.log`, `*.fls`, …), plus music assets `*.mp3`, `*.mscz`, `*.mid`,
`*.gsheet`, `*.gdoc`, `*.docx`. (I may occasionally ask you to read a particular
PDF.)
