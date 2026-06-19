# Song sheet creation using LaTeX

This repo contains custom LaTeX `.sty` files, based on the `leadsheets` package
(https://github.com/cgnieder/leadsheets) which uses KOMA-Script under the hood
(`leadsheet.cls` loads `scrartcl`).  The package doesn't seem to be maintained
any longer. I've cloned the repo into the directory
`~/Documents/5create/latex/leadsheets-clone`, but the actual functional package is found
here: `/usr/local/texlive/2025/texmf-dist/tex/latex/leadsheets/` Note also the
file `leadsheets.library.songs.code.tex` which fixes an issue that was caused by
a more general LaTeX update; see https://github.com/cgnieder/leadsheets/pull/46. 

- `MyLeadsheets-TEMP-REFERENCE.sty` is an older document that I hacked together
  over years, without any knowledge of KOMA-Script, and using a LaTeX
  compilation method that hid all warnings. Nonetheless, it did what I wanted it
  to do and produced functional song sheet pdfs. This file can be used as a
  reference-only (no edits necessary) for desired formats, features, and
  functionality. 
- **`MyLeadsheets.sty`** — This is the main file, to be re-constructed cleanly,
  carefully, from the ground up, incorporating formats, features, and
  functionality as we go.

See **`NOTES.md`** for reference detail kept out of these instructions: package
issues, resolved problems, implemented-feature mechanism internals, and
tooling/debugging notes. This file holds the live authoring contracts and design
goals; NOTES.md is the "when you actually touch that code" detail.

## Design goals for the overhaul

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
    glyph renders, e.g. `key={\fixedchord{Gm}}`. `capo=N` adds a `\notebox{capo N}`
    label after the key (fingering shapes, *not* a key change — see "Output
    views and arrangements").
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
  appends a block to the **bottom** of the song's last page: a header `Remarks`,
  then the environment's own free-text body. A bare `\begin{remarks}\end{remarks}`
  prints just the header. The block reads **nothing** from the song (pure free
  text — no title, tempo, or property pulled in), so it has no song-id plumbing and
  no coupling to the preceding song. It auto-positions with `\vfill` (flows to a new
  page if it doesn't fit) and travels into a songbook like any post-`\end{song}`
  body. Restyle via the `\MyLSremarks*` knobs (`…label`, `…headfont`, `…rule`).
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
  lengths and spacing without flipping between PDFs. (`phone` view still planned.)
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

## Known issues / work in progress

- **`\measures` min-width is not a column grid.** Wide (lyric-heavy) measures
  expand past `\minmeasure`, so barlines do *not* align vertically across the
  lines of a section — they're only guaranteed *no closer* than the minimum. If
  an aligned grid is ever wanted (bar N at the same x on every line), that's a
  separate, harder feature (fixed-width / `tabular`-column measures).

- **`\repeatbar` — ported and live.** The TikZ repeat glyph (two brick-red dots +
  a slash) is now in `MyLeadsheets.sty` (the "Inline markers" block), so legacy
  raw-pipe `.song` files that use `$\repeatbar$` compile. It is a **standalone
  macro**, not wired into the `\measures`/chart builder; whether repeat marks
  should instead be a `\measures` style remains an open design question, but the
  port doesn't pre-empt it (the builder already has the `repeat` *style*, giving
  `|: … :|` barlines, which is a different thing from the inline glyph).
  `samples/Uberlin.song` is still a transitional raw-pipe file (`| ^{Gm}word |`,
  hand `\hspace`), now also exercising the new header properties (`arrangement`,
  `tuning`, `genre`); its bridge uses bare `_{…}` chords so it compiles, but it is
  not yet converted to `\measures`.

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

# Files

## Key files 

- `/Users/preynol1/repos/1sys/tex/songs/` — the custom `.sty` files (primary work area)
- `/usr/local/texlive/2025/texmf-dist/tex/latex/leadsheets/` — system-installed leadsheets package
- `~/.config/vim/custom/vimtex.vim` — vimtex plugin configuration; sets `aux_dir`
- `~/.config/latexmk/latexmkrc` — global latexmk config; puts aux/log files in
  `~/.cache/latexmk/${project_name}_$hash/` (used by `Generate*` commands; separate from
  vimtex's cache dir, no conflict)
- `~/.config/zsh/.zprofile` — sets `TEXINPUTS="$HOME/repos/1sys/tex//:"` so that files in
  the songs repo (including the PR#46-fixed `leadsheets.library.songs.code.tex`) shadow the
  system copies

## Sample files (initial)

- `/Users/preynol1/repos/4music/rem-songs` — **current testing focus.** A
  separate git repo of R.E.M. `.song` files (flat at the top level) plus its own
  copy of the `build-pdfs` script. This is where new format/feature work is
  exercised in bulk now.
- `samples/` (in this directory) — test song files used to verify clean
  compilation; `TheOneILove.song` is the primary test file used so far. 
- `~/Documents/4music/_song-sheets-WIP/samples/TheOneILove-REFERENCE.song` (and associated pdf) is
  the reference. This file produced several pdfs with formats and features that
  I'd like to either re-incorporate, or re-think. The single `.song` file was
  used to also produce `TheOneILove-Chart-REFERENCE.pdf`
  `TheOneILove-Chords-REFERENCE.pdf` `TheOneILove-Lyrics-REFERENCE.pdf`)
  

## More Sample files in ~/Documents/4music/_song-sheets-WIP (eventual)

The directories `covers-copy` `rem-copy` `stevie-copy` `written-copy` `xmas-copy`
(found in `samples`) are copies of song sheets that I've created in the past.
(The R.E.M. ones in `rem-copy` are legacy; active R.E.M. work now lives in the
`/Users/preynol1/repos/4music/rem-songs` repo noted above.)

- IGNORE .PDFs (although I may ask to read particular pdfs if necessary)
- IGNORE .mp3, .mscz, .mid, .gsheet, .gdoc, .docx
