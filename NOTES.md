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

### `\input` filenames inside `song` must not contain `_`

When extracting a song body to a shared file that thin wrapper `.song`s `\input`
(the arrangement-by-wrapper strategy — one `Song--input.song` body, several
`Song.song` / `Song-CAPO.song` wrappers that each `\input` it inside their own
`\begin{song}[]{…}`), the included filename **must not contain an underscore**.
Inside the `song` environment leadsheets makes `_` an *active character* (the
`_{C}` chord-only token syntax), so `\input{Song__input.song}` tokenizes the
`__` as chord markup and the filename is mangled → compilation breaks. Use `--`
instead: `Song--input.song`. (The body file itself is never compiled directly; it
has no preamble — just the `\begin{verse}…` sections — and is read only via
`\input` from a wrapper.)

## Resolved

- PR#46 fix for `leadsheets.library.songs.code.tex` placed in `~/repos/1sys/tex/songs/`,
  shadowing the unpatched system copy via `TEXINPUTS`
- `\prop_new:c` hack removed from `MyLeadsheets.sty` (no longer needed with PR#46 fix)
- vimtex log parsing: resolved — `aux_dir` set in `vimtex.vim`
  dynamically (`~/.cache/latexmk/{dirname}-{jobname}/`) so vimtex knows where its log is

## Mechanism reference

Internals of the IMPLEMENTED output features. The *condensed* authoring
contracts (what to type in a `.song`) live in CLAUDE.md's "Output views and arrangements" and
"Songbooks" sections; this is the full how-it-works, output mapping, and rationale
for when the mechanism itself needs touching.

### Arrangement-by-wrapper (capo / transpose / multiple keys)

Any arrangement whose *musical content* differs — capo shapes vs concert, a real key
change, or several keys of one song — is a **separate wrapper `.song`**, not a
magic comment. The song body (the `\begin{verse}…` sections, **no preamble**)
lives in `Song--input.song`; each wrapper sets its own `\begin{song}[…]{…}`
options and `\input`s that body. `build-pdfs` compiles each wrapper to one PDF and
**skips `*--input.song`** (no preamble → can't compile). Verified on
`samples/Uberlin{,-CAPO}.song` and `samples/TransposeDemo{,-OriginalKey}.song`.

This **replaced** the earlier magic-comment machinery — all removed: the
`%! capo-pair` / `%! transpose-pair` comments, the injected `\def\ConcertArrangement`
/ `\def\SongTranspose` overrides, the `.sty`'s `\ConcertArrangement`/`\ifMyLSconcert`
block, and `build-pdfs`'s `wants_capo_pair` / `wants_transpose_pair` /
`compile_concert` / `compile_original_key`.

**Why the rewrite.** The injection approach produced *one* `.song`'s siblings via
forced `latexmk -g` rebuilds that spliced `\def`s ahead of `\input`, and needed
the `\providecommand`/macro indirection (`transpose=\SongTranspose`) purely so the
injected `\def` could win. Wrappers make each arrangement an ordinary standalone
document: no injection, no `-g`, a literal `transpose=2`, and — the real payoff —
different versions drop straight into a songbook (`\bookinclude` each wrapper),
which the single-source approach couldn't do.

**Capo — leadsheets does the work.** `transpose-capo = true` is set once,
globally, in `MyLeadsheets.sty`. With a `capo=N` property present, leadsheets
transposes every chord by `12 − N` semitones = **down N** (mod octave) — exactly
the shapes fretted with the capo on (`\__leadsheets_check_capo:`,
`songs.code.tex`). Chords transpose only when a `key` property also exists, so
**every wrapper must set `key=`** (concert pitch) or leadsheets warns per chord. A
wrapper with no `capo` is unaffected: capo absent / `capo=0` is a full-octave
transpose = identity (verified — no enharmonic drift). The title template adds the
"capo N" note via `\ifsongproperty{capo}{… \notebox{capo \songproperty{capo}} …}{}`,
so the concert wrapper (no `capo`) simply has no note — no flag plumbing. The
displayed `key=` stays concert pitch because the template prints
`\songproperty{key}` *raw*, never through the chord transposer (the stock template
*would* transpose it). `\fixedchord`, for a ♯/♭ glyph in a key like `F#m`, is still
unported.

**Transpose — also pure leadsheets.** A wrapper carrying `transpose=N` (a plain
literal now) shifts the sounding pitch by N semitones; the original-key sibling is
a wrapper that omits `transpose=`. l3keys expands a macro value via `\int_set`, so
`transpose=\macro` *would* still work — but there's no longer any reason to use
one, since nothing has to undo it.

**Constraint — never `capo=N` and `transpose=M` in the same wrapper.** The capo
axis owns transposition (`capo=N` + `transpose-capo`); a literal `transpose=M`
combines with it (`net = M − N`) and corrupts that sheet. If a baseline transpose
is ever needed alongside a capo, bake it into the authored chords. Wrappers make
this easy to keep straight: each carries one or the other.

**Filename gotcha.** The shared body must use `--`, not `__`: inside `song`,
leadsheets makes `_` an active character (the `_{C}` chord token), so an
`\input`-ed filename containing `_` is mangled. See the "`\input` filenames inside
`song`…" note under *Issues*.

**Songbook body-resolution gotcha (fixed in `\bookinclude`).** A wrapper's bare
`\input{Song--input.song}` resolves relative to the *main job's* directory, not
the wrapper file's. Standalone that is the song's own folder (body found); but
`\bookinclude{song-files/Song.song}` runs from the songbook's parent dir, so the
bare name no longer resolves — either "File not found", or (worse, silent) a
same-named body elsewhere on the recursive `TEXINPUTS` is picked up instead. Fix:
`\bookinclude` now runs `\MyLSaddsongdir{#1}` inside a `\begingroup`, which
`\filename@parse`s the song's own directory out of the include argument and
prepends `{dir/}` to `\input@path` so the nested `\input` searches it too; the
group scopes it per song. **Caveat — `\input@path` is consulted only after the
bare name fails via cwd/`TEXINPUTS`, so a stale same-named body still wins if one
is reachable.** This bit `samples/Uberlin--input.song` (an old raw-pipe copy under
the `~/repos/1sys/tex//` `TEXINPUTS` tree) silently shadowing the live
`rem-songs/song-files/Uberlin--input.song`; renamed to `UberlinDemo--input.song`
so its bare name can't collide. Moral: don't give a sample body the same
`<RealSong>--input.song` name as a real song's body anywhere under `TEXINPUTS`.

**Escape hatch: the `import` package (not used; here if needed).** `import` is the
canonical answer to "my `\input` breaks once files live in subdirectories":
`\import{dir/}{file}` / `\subimport{rel/}{file}` keep a *current-directory stack*
so an imported file's own `\input`/`\includegraphics` resolve relative to where
*that file* lives, to arbitrary depth. We deliberately did **not** reach for it:
the songbook crosses a directory boundary in exactly *one* place, and the dir is
handed to us as the `\bookinclude` argument, so the kernel's own `\input@path`
(what `import` is itself a wrapper around) does the job in ~8 lines with no
dependency. `import` would earn its keep only if the include graph gets genuinely
deep — bodies that `\input` *their own* subdir files, per-song asset/graphics
folders, etc. One thing it *would* buy even now: because it builds an **explicit**
path rather than relying on search *order*, it makes the local file win over a
same-named `TEXINPUTS` copy — i.e. it would have dodged the Uberlin shadow without
the rename (see the caveat above). We chose kernel + no-colliding-names instead.
  - **KOMA compatibility:** expected fine, though not empirically tested in this
    project. `import` works purely at the file-inclusion layer (a dir stack plus
    redefined `\input`/`\include`/`\includegraphics`); it touches nothing KOMA
    cares about — page layout, fonts, sectioning, `\setkomafont` — so the concerns
    are orthogonal. It redefines `\include`, but leadsheets pulls songs via
    `\includeleadsheet` (not `\include`), so even that doesn't collide. If adopted,
    smoke-test a one-song book first, but no conflict is anticipated.

**Composes with the presentation axis.** A wrapper may itself carry
`%! views: chords` to get the chords/lyrics siblings of *that* key/capo —
e.g. `Song-CAPO.song` + `%! views: chords` → `Song-CAPO.pdf` and
`Song-CAPO--chords.pdf`. The old "folding the two axes" future-work item is thus
resolved by construction.

### Song header / title template

The header is one `\section{\texorpdfstring{<rich>}{<bookmark>}}` built by
`\definesongtitletemplate{mytitle}` and set live with `\setleadsheets{title-template
= mytitle}`. Using a real `\section` (not `\section*`) is deliberate: in a songbook
the heading text is what `\tableofcontents` reuses as the TOC entry, so the rich
one-line header *is* the TOC line. `secnumdepth=0` drops the section number;
`\DeclareTOCStyleEntry[...]{section}{section}` gobbles the page number
(`\MyLSgobblepagenumber`), zeroes `pagenumberwidth`, and weakens the linefill to
`\hfil` (1fil) so it loses the tug-of-war with the template's own `\hfill` (1fill)
that pushes genre/difficulty to the right margin — see that block's comment for the
two-infinities trick. The `\texorpdfstring` first branch is typeset + written to the
TOC; the second is a plain `title (band)` string for the hyperref bookmark.

**Expandability is load-bearing.** Everything inside the `\section{}` must be
*expandable* so it folds to literal tokens when KOMA `\protected@write`s the heading
to the `.toc` (evaluated then, while the current-song context is valid).
`\ifsongproperty` qualifies — it is `\DeclareExpandableDocumentCommand` built on
`\prop_if_in:` (`leadsheets.library.properties.code.tex`). `\ifanysongproperty` does
**not** (it is `\NewDocumentCommand`), so it would survive into the `.toc` and run at
`\tableofcontents` time, when the song id is gone — wrong answer. That is why the
genre/difficulty right-margin group guards its lone `\hfill` with *nested*
`\ifsongproperty` rather than `\ifanysongproperty{genre,difficulty}`.

**Arrangement slot (`\MyLSarrangementslot`).** A robust (`\NewDocumentCommand`) macro so it
survives the `.toc` write intact, then re-measures at typeset time so the box adapts
to the TOC font as well as the page font. It sets the bracketed tag into a box,
measures a 6-monospace-char reference box (any 6 mono glyphs — all equal width), and
emits `\hbox_to_wd` left-aligned (content + `\hfill`) when narrower than the
reference, else natural width; the result is `\raisebox`-lifted. Two subtleties:
(1) the empty-vs-absent distinction relies on `\ifsongproperty` testing *key
presence*, not value emptiness, so `arrangement={}` is true (→ blank box, an alignment
indent) while an unset `arrangement` is false (→ nothing); the macro itself still guards
the brackets with `\tl_if_empty:N` so an empty value yields a truly blank box, not
`[]`. (2) The colour wraps the whole `[word]` *inside* the macro (brackets included),
so the call site passes a **plain** `\songproperty{arrangement}` — wrapping the value in
`\textcolor` at the call site would make the emptiness test always-false (it sees
`\textcolor{...}{...}`, never empty) and break the blank-box case. Knobs:
`\MyLSarrangementfont` / `\MyLSarrangementcolor` / `\MyLSarrangementraise`.

**Tuning line.** Emitted *after* the `\section{}` (still inside the template), so it
is ordinary body text below the heading — structurally invisible to the TOC entry,
the bookmark, and the running head (all three derive from the `\section` title).
`\nopagebreak` ties it to the heading; a negative `\vspace{-\MyLStuningraise}` pulls
it up snug under the title; the value goes through `\MyLSformattuning` (accidentals
rendered, never transposed — its own mechanism). Knobs: `\MyLStuningfont` /
`\MyLStuningraise`. `tempo` was dropped from the header entirely in this redesign.

### Remarks (NOTES block)

A `remarks` environment, authored **after** `\end{song}`, appends a bottom-of-page
notes block (`MyLeadsheets.sty`, "Remarks" section). Authoring contract is
CLAUDE.md's *Remarks / NOTES block* bullet; this is the how.

**Why outside the song env.** Inside `\begin{song}…\end{song}` leadsheets makes
`^ _ | :` active (chord/barline shorthands — `songs.code.tex:143–146`, activated
in `\leadsheets_specials:`) and applies the bold "song" body format, both hostile
to prose. Outside, catcodes/formatting are normal: `|` and `:` print literally
(the real win — inside they'd become barlines/repeats); `^`/`_` still need escaping
but that's plain-LaTeX text mode, not the song env.

**Reading the song's properties from outside (the one wrinkle).** `\songproperty`
expands `\leadsheets_get_property:Vn \l_leadsheets_current_song_id_tl {…}`. The
property *store* is global (`\prop_gput:cnV`, keyed by song id —
`properties.code.tex:42`), so it survives `\end{song}`; but
`\l_leadsheets_current_song_id_tl` is set only *locally* inside the song's group
(`songs.code.tex:291`, inside the `\group_begin:`/`\group_end:` at 203/318), so it
has reverted by the time the `remarks` env runs. Fix: the title template (which
runs inside the song) calls `\MyLScapturesongid`, copying that id into the global
`\g__myls_song_id_tl`; the `remarks` env restores it **locally** (reverts at
`\end{remarks}`) and `\songproperty`/`\ifsongproperty` then resolve against this
song. `\l_leadsheets_current_song_id_tl` is leadsheets' **public** (single-`_`)
variable, so this uses no private internals — only the song-redesign-discouraged
double-`_` names are avoided. The capture fires on both the measuring and the real
pass (the body runs twice); harmless, the global is reassigned the same value.

**Structure / catcode care.** Helper `\__myls_remarks_head:` *and* the
`\NewDocumentEnvironment{remarks}` live in **one** `\ExplSyntaxOn` block so the
`:`-bearing helper name tokenises validly; the stored env code runs later under
normal catcodes (the author's body is unaffected). All literal strings ("NOTES",
"Tempo:") live in `\newcommand` knobs *outside* expl3 — keeping them inside would
hit expl3's `:`-is-a-letter / `~`-is-a-space rules. The block uses `\hrule` in
vertical mode (after `\par`), and sets `\parindent=0pt` locally.

**Placement / songbook.** `\par\vfill` at the env top drops the block to the page
bottom; the document's final `\clearpage` (or the next `\bookinclude`'s) lets the
glue stretch. It flows onto a new page if it doesn't fit. In a songbook the block
is included for free: `\includeleadsheet` rewrites the file's
`\end{document}`→`\endinput` (`external.code.tex:43–49`), so all body content
between `\begin{document}` and `\end{document}` — including a post-`\end{song}`
remarks block — is pulled in. (Untested: a `remarks` `\vfill` combined with
`\bookinclude`'s own `\MyLStochint` `\vfill` on the same last page — both stretch,
could interact; standalone is the target use.)

**Shared remarks across wrappers** (arrangement-by-wrapper): put
`\begin{remarks}…\end{remarks}` in a `Song--remarks--input.song` and `\input` it
**outside** each wrapper's `\begin{song}`. Because that `\input` is outside the song
env, the `_`-in-filename gotcha (active `_` inside `song`) does **not** apply there,
though `--` stays consistent with convention.

**Header is not a `\section`.** Deliberately a styled paragraph
(`\MyLSremarksheadfont`), not `\section*`, so it never reaches the TOC, a PDF
bookmark, or a running head. Knobs: `\MyLSremarkslabel` / `…dash` / `…tempolabel` /
`…headfont` / `…metafont` / `…rule`.

### Deferred — the key label following transposition

The title template prints `\songproperty{key}` raw, so a `transpose=` wrapper's
PDF currently shows the *authored* key (e.g. "C") even though its chords are in D;
the original-key wrapper happens to read correctly. Making the key follow the
transposition is the same conditional flagged for capo — the stock template uses
`\writechord{\songproperty{key}}` (`songs.code.tex:481`), which transposes — so:
branch the key print on `\ifsongproperty{capo}` (capo → raw concert key; else →
`\writechord`, which transposes for `transpose=` and is identity otherwise).
`\fixedchord` is now ported and live (it suppresses transposition for one chord, so
`key={\fixedchord{Gm}}` renders the accidental without moving it); it is the
opposite of what this conditional needs but shares the same transpose-boolean
plumbing, so it is the worked example to crib from.

### Even-page padding & songbook back-to-contents link

Three tools for two-page (spread) digital viewing, all in `MyLeadsheets.sty`'s
"Even-page / odd-start padding" + "Songbook back to contents link" sections.
Authoring contract is CLAUDE.md's *Songbooks & even-page output*; this is the how.

**Shared parity arithmetic.** After a `\clearpage` the `page` counter holds the
number the *next* shipped page will get, i.e. `(pages-shipped + 1)`. So an **odd
counter ⇒ an even number of pages has shipped** (and vice versa) — the one fact
both padders test. Verified empirically; my first guess at the direction was
backwards, hence the `\ifodd\value{page}\else …\fi` shape (pad on *even*). The
parity is always measured on the natural content **before** any blank is added,
so the decision is identical on every run — no oscillation under latexmk. (A
`\pageref{LastPage}`-style test would instead count the blank it just added and
flip parity each rebuild.) The blank itself is `\MyLSblankpage` =
`\null\thispagestyle{empty}\clearpage` (`\null` forces the empty page to ship).

**`\ForceEvenPages`** — `\newif\ifMyLSforceeven` + an `\AtEndDocument` hook: if
opted in, pad the *total* to even. Standalone only; in a songbook `\AtEndDocument`
fires once for the whole book, and a song's own `\ForceEvenPages` never runs
anyway (`gobble-preamble` is true in the `external` library —
`leadsheets.library.external.code.tex`, `\__leadsheets_gobble_preamble:wn`, which
also disables `\usepackage`/`\RequirePackage` and rewrites `\end{document}`→
`\endinput`).

**`\clearoddpage`** — `\clearpage` then pad iff the next page would be even, so the
following block starts odd. This is what KOMA's `\cleardoubleoddpage` does, but
that gates its blank behind `\if@twoside` (`scrartcl.cls`:1239–1245,
`\cleardoubleoddpageusingstyle`), and this class is single-sided; going `twoside`
would swap margins and restore headers, so we pad explicitly.

**`\bookinclude{file}`** = `\clearoddpage` then
`\begingroup\MyLSaddsongdir{file}\includeleadsheet{file}\endgroup` then
`\MyLStochint`. The `\MyLSaddsongdir`/group handles shared-body path resolution
(see the *Songbook body-resolution gotcha* note above); the include's lasting
effects — TOC/bookmark writes, counters — are global, so the group does not lose
them. The hint runs *after* the include returns (at the song's
`\end{document}`→`\endinput`), so it sits at the end of that song's content,
before the next `\bookinclude`'s `\clearoddpage`. Folding it into the wrapper (not
per-song) means an individual `.song` carries nothing and a standalone compile
cannot be affected.

**`\MyLStochint` — the fit test (the hard part).** Place a back-to-TOC `\hyperlink`
at bottom-right *iff* it fits, else nothing — must **never** force a new page.
Mechanism: `\par\penalty0` (the `\penalty0` makes the page builder account for the
last line so `\pagetotal` is current), then remaining `= \pagegoal − \pagetotal`
(`\pagegoal` reads `\maxdimen` on a still-empty page → treated as a full
`\textheight`); if remaining `> \ht+\dp` of the link box `+ \MyLStocgap`
(`2\baselineskip` safety), emit `\vfill\nointerlineskip\hbox to\hsize{\hfill\box0}`.
`\vfill` drops it to the bottom; the next `\clearpage` flushes it. Because slack is
measured first and we append strictly less than it, the page can't overfill.
Verified by a line-count sweep across the 1- and 2-page boundaries (textheight
777.9pt, baselineskip 17pt): page count with-link always equalled the natural
count, and the link was *skipped* exactly on the near-full last pages (N=44,45 and
89,90) and *placed* where there was room. On the real songbook the GoTo annotation
resolved to the TOC page (`mutool show … N` → `/S /GoTo /D [6 0 R /Fit]`).

**Catcode gotcha (cost a build).** `@` is **not** a catcode letter in this
`.sty` — the whole package uses the `\MyLS…` prefix, never `\MyLS@…`. A first
attempt naming the helper `\MyLS@blankpage` failed at load ("Missing
`\begin{document}`", with `@blankpage` typeset as text); likewise the fit test
must use scratch registers `\dimen0`/`\box0`, **not** `\dimen@`/`\z@`. An
`\AtBeginDocument` `\providecommand\hyperlink`/`\hypertarget` fallback keeps the
songbook macros harmless if `hyperref` is somehow absent (won't clobber the real
ones, which load earlier).

## Tooling / debugging

### Vimtex quickfix behaviour

Vimtex's log parser only surfaces `Package X Warning:` patterns into the quickfix
window — `Class X Warning:` messages (e.g. KOMA's own warnings about incompatible
packages) appear in the raw log but are invisible in the quickfix. When diagnosing
warnings, check the raw log at `~/.cache/latexmk/{dirname}-{jobname}/` (vimtex
compilations) or `~/.cache/latexmk/${project_name}_$hash/` (Generate* commands).

## build-pdfs — status

`build-pdfs` (repo root, `~/repos/1sys/tex/songs/build-pdfs`) compiles each
`.song` to its default PDF, then reads the `%! views:` magic comment in the
**leading** comment block (first non-blank, non-`%` line ends the block, mirroring
the TeX-engine magic comment) to build presentation siblings. It **skips
`*--input.song`** (shared bodies — no preamble). Capo / transpose / multiple-key
sheets are no longer the script's concern: each is its own wrapper `.song` (see
*Arrangement-by-wrapper* above) that arrives as a separate file in the build loop. The
script carries full inline comments; this is just the orientation and the live
status. See *Output views and arrangements* in CLAUDE.md for the authoring contracts.

The only sibling-producing magic comment:

    %! views: chords, lyrics   presentation axis -> Song--<view>.pdf

**Built:**
- Default PDF for each `.song` (always produced); `*--input.song` skipped.
- `-h`/`--help`; no args = every `*.song` in cwd, else the named files. A per-song
  failure is reported and skipped; non-zero exit if any failed.
- Aux files to `~/.cache/latexmk/{parent-dir}-{jobname}/` via `-auxdir`, sharing
  vimtex's incremental-build database; the PDF lands next to the source. No
  `latexmk -c` (the `.fdb_latexmk` database is worth keeping).
- **chords view** — `compile_view()` injects `\def\View{chords}` (forced
  `-g`), producing `Song--chords.pdf`. The `.sty` honours `\View` via two
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
- **lyrics view** — same `compile_view()` path (the `case` arm is just
  `chords | lyrics`), injecting `\def\View{lyrics}` for `Song--lyrics.pdf`. The
  `.sty` flips `print-chords=false` for this view only; the family selection is
  the same as the default build (lyric bodies print, chart twins swallowed). See
  *Views from one `.song`* item 2 for the `^`/`_` detail.

**Pending:** the `phone` view. (Folding the presentation axis together with
capo/transpose is no longer pending — it falls out of the wrapper model: put
`%! views: chords` in a capo/transpose wrapper and that wrapper gets its own
`--chords`/`--lyrics` siblings, e.g. `Song-CAPO.song` → `Song-CAPO--chords.pdf`.)
The `-CAPO`/`-OriginalKey` (single-hyphen, *different musical content*) vs
`--chords`/`--lyrics` (double-hyphen, *different presentation*) suffix split is the
convention to preserve.

## Future work — output views from one `.song`

The reference produced several PDFs from one source via a tangle of `\newtoggle`
flags that cross-set each other (`LyricsWithChords`, `LyricsNoChords`,
`ChordProgression`, `ChordsAndLyrics`, `SmallLyrics`, `Structure`) and four
content-gating macros (`\chords`, `\lyrics`, `\lyricswithchords`, `\structure`),
each `\iftoggle{…}{#1}{\@gobble{#1}}`. The cost landed in the `.song`: the same
material was typed 3–4× per section, once per view. The overhaul should reach
the same outputs with far less duplication. Planned views:

1. **Default — lyrics + chords.** The `^{C}word` lyric body. Baseline. *Done.*
2. **Lyrics-only — DONE, the `lyrics` view.** The **same** `^{C}word` body with
   leadsheets' `print-chords=false`. No duplication — one body serves both 1 and 2.
   Selected by `\def\View{lyrics}`; renders the same family as the default build
   (lyric types print, chart twins swallowed) and only flips `print-chords`. Key
   detail: `print-chords=false` suppresses **only** `^{C}word` chords (`^` =
   `\getorprintchord`, gated by `print-chords` in `\leadsheets_place_above`); the
   `_{C}` chord-only tokens in `\measures` go through `\writechord` and print
   regardless (`songs.code.tex:143-144`). So lyric sections become words-only while
   instrumental sections (a `\measures` line of `_{C}` chords, no `^{}word`) keep
   their progression — the sensible outcome, and it falls out of the documented
   `^`/`_` split with no extra code. Verified on `samples/ChordsVariantDemo.song`.
3. **Chart-only (no lyrics) — DONE, the `chords` view.** A **separate** body
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

View selection is a single `\def\View{…}` command-line flag (injected by
`build-pdfs`) reduced to **one** clean view state in the `.sty`, not the
reference's six cross-set toggles — realised for `chords` and `lyrics` (each a
small `\newif`/`\ifx` block in the "select which family renders" section, plus the
`print-chords` switch at the file end); `phone` extends the same lever.

The **Reference's `Structure` option is abandoned** — not being carried forward.

Possible **class option for grayscale** for any view (phone excepted, but no
strong reason either way) — for B&W printing, mirroring the reference's
`\BlackAndWhite` branch that swapped tinted section boxes for ruled ones.

### `.song` API — parallel section environments (DONE)

**Implemented.** The `chords` view uses **per-type parallel environments**: the
lyric body stays in `verse`/`chorus`/…, and the chart goes in a parallel twin
`<type>chords` (`versechords`, `choruschords`, `introchords`, …). The active
view prints exactly one family; the other's bodies are swallowed (xparse `+b`).
Usage:

    \begin{verse}
      ... ^{C}word lyric body — prints in lyrics+chords and lyrics-only ...
    \end{verse}
    \begin{versechords}
      ... \measures chart body — prints in chart-only ...
    \end{versechords}

Both families box by default now: `verse`/`chorus`/… carry their coloured box as
their **default** `template` (set in the `.sty`'s "make each box the part type's
DEFAULT template" block), so a bare `\begin{verse}` is already boxed — the
`[template=…]` option is needed only to override (borrow another box, or
`[template=itemize]` for a plain section). Each twin is likewise a real
`\newversetype` whose **default** `template=` is its counterpart's coloured box
(`versechords` → `versebox`), so it needs **no `[template=]`** in the `.song` and
reproduces the lyric section's colour *and*
label (`V1:`/`C1:`/`In:`) automatically — the labels line up section-for-section
because the sections are authored in parallel. `choruschords` deliberately drops
chorus's italic (it takes the global upright/bold `verses-format`). Defined in the
`.sty`'s "Output view: chords" block: the `\newversetype` table, a
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

- **Behaviour in the lyrics-only view.** A chord-only solo has nothing left
  once chords are suppressed. Options: omit the section, leave it blank, or show
  a short note (the reference used `\notebox{8 bar solo}` here).
- **The token name.** With three views, `[both]` presumes the
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

