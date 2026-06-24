# Song sheets — project reference & log

Reference material kept out of CLAUDE.md for lookup as needed: package issues,
resolved problems, implemented-feature mechanism internals, and tooling/debugging
notes. CLAUDE.md holds the live authoring contracts and design goals; this file
is the "when you actually touch that code" detail. Scope is the whole song-sheet
project, not just the `leadsheets` package.

The project is in **maintenance/tweak mode** — the intended features and formats
are implemented and the planned feature work is **complete**; there is no active
backlog. A few sections below document reference-only escape hatches (e.g. `\mw`)
or accepted behaviours, not open TODOs. Some "verified on …" mentions name old
dev/reference files (`*-REFERENCE` and the deleted `samples/` test songs) by bare
filename; these are a record of how things were checked, not live paths.

## Reminders / how to investigate

- **System package source:** `/usr/local/texlive/2025/texmf-dist/tex/latex/leadsheets/`
  - chord placement: `leadsheets.library.chords.code.tex`
  - song/verse environments: `leadsheets.library.songs.code.tex`
  - music symbols (bars etc.): `leadsheets.library.musicsymbols.code.tex`
- **PR#46 fix** lives in `~/1sys/tex/songs-machinery/` and shadows the system copy via
  `TEXINPUTS="$HOME/1sys/tex//:"` (set in `~/.config/zsh/.zprofile`). When
  compiling test files by hand outside that shell, export it explicitly:
  `export TEXINPUTS="$HOME/1sys/tex//:"`.
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

### `\newcommand` with `#1` parameters can't be defined inside `song`

**Symptom:** A *parameterised* macro defined after `\begin{song}` silently
misbehaves — the parameter is never substituted. E.g.
`\newcommand{\ChorusLyrics}[1][]{\measures[#1]{…}}` then `\ChorusLyrics[repeat]`
produces **normal** bars, not repeat bars (the elegant "one command, optional
style" idea — `\Foo` vs `\Foo[repeat]`). No error is raised. Probing shows `#1`
arrives as the literal characters `#1`, so `\measures[#1]` passes the *string*
"#1" as the bar style, which isn't `repeat`/`final`, so it falls through to normal
bars. (No-argument `\newcommand`s — how the reuse/factoring pattern works — are
**fine**; this only bites macros that use `#`.)

**Cause:** Inside the `song` environment leadsheets makes `#` an ordinary
character (so chord sharps like `F#`, `C#` can be typed literally in `^{F#}`).
With `#` no longer catcode 6, a `\newcommand` body defined there treats `#1` as
literal text, not a parameter slot — the declared `[1]` argument is grabbed and
then silently dropped.

**Workarounds tried, all worse than the problem:**
- Restore `\catcode`\#=6` in a group around the def — the song body is processed
  **twice** (a measuring pass + the real pass), so the def re-runs → `Command …
  already defined`, and group scoping fights global persistence.
- Share the *cells* as a no-`#` macro and wrap (`\measures{\ChorusCells}`) — the
  repeat bar does fire, but leadsheets' chord splitter chokes
  (`Argument of \__leadsheets_set_chord:nwn has an extra }`); it needs the literal
  `{^{C}word}` cells, not a macro that expands to them.

**Workaround / takeaway:** Don't parameterise macros inside `song`. For a section
that recurs with a *different bar style* (a `[repeat]`/`[final]` last chorus), just
write that one section **inline** with the style baked in (`\measures[repeat]{…}`)
— the DRY cost is a single section, and the chord splitter prefers literal cells
anyway. The `pr-factor-song-repeats` tool deliberately never emits parameterised
commands for this reason; its near-miss report only *flags* "identical except
`[repeat]`" so you handle that spot by hand.

## Resolved

- PR#46 fix for `leadsheets.library.songs.code.tex` placed in `~/1sys/tex/songs-machinery/`,
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
`Uberlin{,-CAPO}.song` and `TransposeDemo{,-OriginalKey}.song`.

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
`songs.code.tex`). That is the **default** path: concert-pitch input in, capo shapes out. Set
`transpose-capo=false` instead and `capo=N` transposes nothing — it is just the
"capo N" label — so you write the **capo-relative shapes directly** (capo-centric).
The choice is per wrapper; put `transpose-capo` in the wrapper **body**, not its
preamble, so a songbook's preamble-gobble doesn't drop it (like `\resize`). Chords
transpose — and a `key` property is then **required**, else leadsheets warns per
chord — only when something actually transposes (`transpose=`, or `transpose-capo`
with a `capo=`); a capo-centric wrapper (no transpose) leaves `key=` optional,
though you still set it for the displayed label. A
wrapper with no `capo` is unaffected: capo absent / `capo=0` is a full-octave
transpose = identity (verified — no enharmonic drift). The title template adds the
"capo N" note via `\ifsongproperty{capo}{… \notebox{capo \songproperty{capo}} …}{}`,
so the concert wrapper (no `capo`) simply has no note — no flag plumbing. The
displayed key is whatever `key=` says — set it to match the chords the sheet shows
(the shape key on a capo-centric capo sheet; the concert key on its `transpose=`
concert sibling, which it will **not** follow automatically) — because the template
prints `\songproperty{key}` *raw*, never through the chord transposer (the stock
template *would* transpose it). `\fixedchord`, for a ♯/♭ glyph in a key like `F#m`, is now ported
and live (`MyLeadsheets.sty`).

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

**Corollary — a mid-song `\setleadsheets{transpose=M}` in a shared body fights a
capo wrapper (the same root cause).** Capo and `transpose=` share *one* variable,
`\l__leadsheets_transpose_steps_int` — the single semitone shift applied to every
chord. `transpose=M` (`transposing.code.tex:222–224`) does an **absolute set** of
that int to `M` (no addition); the capo's contribution is folded into the *same*
int **once**, at song start, by `\__leadsheets_check_capo:` (`songs.code.tex:490–505`:
`steps = (transpose_bool ? steps : 12) − capo`). So a mid-song `transpose=M` simply
**overwrites** whatever `check_capo` computed, and `check_capo` never re-runs. In a
no-capo wrapper that overwrites `12` (≡ identity) with `M` — correct, an honest
mid-song key change. In a `capo=N` wrapper it overwrites `12 − N` (the capo offset)
with `M`, discarding the capo **entirely**: post-change net shift `= M`, not the
`M − N` you want, i.e. a full **capo's-worth (N semitones) too high**. (Note this is
*worse* than the both-in-one-wrapper case above, where the subtraction still happens
because both are set before `check_capo` runs.)

*Worked example — `AtTheEndOfTheDay{,-CAPO}.song`* share `…--input.song`, which does
`\setleadsheets{transpose=1, enharmonic=sharp}` mid-song for the late key change.
Concert (`capo` absent): pre-change Db, post-change D ✓. CAPO (`capo=1`): pre-change
C ✓ (capo working, `12−1`), but post-change the `transpose=1` overwrites the `11` to
`1`, so Db→**D** when the right capo shape is Db (net should be `1−1=0`). Confirmed by
source trace; matches the rendered PDF. **Nothing is broken** — it is the inherent
shape of one once-computed transpose int plus `transpose=` being an absolute set.

*If a shared mid-song key change must coexist with a capo wrapper*, the post-change
shift has to be `M − capo` per wrapper. Two routes, neither wired up here: (a) **public
API** — parameterise the directive, `\setleadsheets{transpose=\MidKeyChange,
enharmonic=sharp}` in the shared body, each wrapper `\newcommand`ing it (concert `1`,
CAPO `0`); l3keys expands the macro value (`transpose=\macro` works, per *Transpose*
above). (b) **re-run `\__leadsheets_check_capo:`** immediately after the transpose in
the shared body — it recomputes `M − capo` for *each* wrapper automatically (concert
`1−0`, CAPO `1−1`), fixing both from one line, but it reaches a `\__` private, so it
is a last resort against the "avoid leadsheets internals" goal. The cleaner habit, if
the key change is the point, is to treat post-change as its own arrangement axis.

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
is reachable.** This bit an old raw-pipe `Uberlin--input.song` copy (under
the `~/1sys/tex//` `TEXINPUTS` tree) silently shadowing a real song's
live `Uberlin--input.song`; the sample copy was renamed to `UberlinDemo--input.song`
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

**`\bookincludedir{dir}` — auto-include a whole folder (mechanism).** Goal: a book
(e.g. `ORIGINALS.song`) that `\bookinclude`s every `dir/*.song` without a hand list.
TeX can't enumerate a directory, so this needs shell-escape (already on). Two-stage,
two scratch files, because of one hard constraint and one leadsheets quirk:
- **Constraint — `\write18` can't cleanly emit `{ } \`.** Those are catcode-special
  in the TeX source of the `\write18` argument, so the shell stage emits only *bare*
  filenames: `ls dir/*.song | grep -v -- --input.song | sort -t. -k1,1 > .\jobname.songlist`
  (the leading dot keeps that scratch file hidden in the song directory).
  That pipeline has **no** TeX-special chars (no `{ } $ # ~ % &` or literal `\` beyond
  `\jobname`), so it survives `\write18` untouched. `grep -v -- --input.song` drops
  the shared-body includes. `sort -t. -k1,1` keys on the path up to the `.song`
  extension, so a base name sorts before its own arrangement wrappers: with the
  extension out of the key, `Song` is a prefix of `Song-CAPO`, and a prefix always
  sorts first, so `Song.song` precedes `Song-CAPO.song` (plain `sort` would put
  `-CAPO` first, `-` < `.`).
- **Quirk — leadsheets mis-parses a macro-delivered filename.** The obvious next step
  (read each line and call `\bookinclude{\theline}`) **fails**: handing
  `\includeleadsheet` a filename via an in-memory macro corrupts its `\filename@parse`
  (observed: a `\filename@path …song/` runaway + `Paragraph ended before
  \XKV@d@fine@k@y`, i.e. xkeyval choking, on the *first* song). A byte-identical
  *literal* `\bookinclude{dir/Foo.song}` line read from a file via `\input` works
  fine. So stage two reads `.\jobname.songlist` (under `\endlinechar=-1`, so no
  end-of-line space is glued onto the name) and **re-emits** it as
  `\jobname.bookdir.tex`, a file of literal `\bookinclude{…}` lines, then `\input`s
  that — reproducing a hand-written booklist exactly. `\MyLScharlb`/`\MyLScharrb`
  write the braces as catcode-12 characters (built with `[ ]` as temp group
  delimiters); `\input` re-tokenises them as real `{ }` groups.
Because the final route is `\input` of literal `\bookinclude`s, `\MyLSaddsongdir`'s
body-resolution fix (above) still applies per song. Both scratch files regenerate
each compile (the book tracks the folder) and are git-ignored (`*.songlist`,
`*.bookdir.tex`). Shell-escape off / non-pdftex engine just warns. A directory whose
songs reference **relative graphics** (e.g. Peregrine's `\includegraphics{y-embedded-images/…}`)
hits the *same* cwd-mismatch as the body gotcha, but for images — `\input@path`
doesn't cover `\includegraphics`, so that would need a `\graphicspath`/`import`-style
fix, not done here.

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

### Remarks block

A `remarks` environment, authored **after** `\end{song}`, appends a bottom-of-page
block (`MyLeadsheets.sty`, "Remarks" section). Authoring contract is CLAUDE.md's
*Remarks block* bullet; this is the how.

**Pure free text — no song coupling.** The block prints a styled `Remarks` header
followed by the environment's own body, and reads **nothing** from the song. An
earlier revision surfaced the song's title, `tempo`, and a `remarks` *property*
here, which forced a wrinkle: `\songproperty` resolves against
`\l_leadsheets_current_song_id_tl`, set only *locally* inside the song's group
(`songs.code.tex:291`, within `\group_begin:`/`\group_end:` at 203/318), so it has
reverted by the time a post-`\end{song}` env runs. That was worked around by
capturing the id in the title template into a global and restoring it locally in
the env. **All of that is gone**: title dropped (redundant — it's already at the
page top / on the song's last page), the `remarks` property deleted (nothing set
it), and `tempo` dropped from the block. With no property reads, there is no song-id
capture and no `\g__myls_song_id_tl` — the block is decoupled from the song.

**View filtering.** An optional comma-list restricts the block to named views:
`\begin{remarks}[chords]` (chords view only), `[chords,full]` (those two); a bare
`\begin{remarks}` (empty list) prints in every view; the `debug` view is exempt and
always shows every block. Implemented with the xparse `O{} +b` body-swallow (same
mechanism as the chords/lyrics verse twins — body captured as an argument, then
typeset or discarded). The decision (`\__myls_remarks_show:nTF`) is expl3 — a
`\clist_if_in:NeTF` exact-item membership test of `\View` against the list, plus a
`\str_if_eq:eeTF` debug-exemption — so the environment is *not* plain LaTeX, but the
author's prose body keeps document catcodes (a `+b` argument is tokenised where
`\begin{remarks}` appears, not where the environment is defined). The `\View` token
the test reads is the renamed default-view name (was `default`, now **`full`**;
`\providecommand\View{full}`).

**Why outside the song env.** Inside `\begin{song}…\end{song}` leadsheets makes
`^ _ | :` active (chord/barline shorthands — `songs.code.tex:143–146`, activated
in `\leadsheets_specials:`) and applies the bold "song" body format, both hostile
to prose. Outside, catcodes/formatting are normal: `|` and `:` print literally
(the real win — inside they'd become barlines/repeats); `^`/`_` still need escaping
but that's plain-LaTeX text mode, not the song env. (The block no longer *needs*
song properties, so "outside" now costs nothing — there's no id to recover.) The
header uses `\hrule` in vertical mode (after `\par`) and sets `\parindent=0pt`
locally.

**Placement / songbook.** `\par\addvspace{\MyLSremarksskip}` at the env top sets
the block off from the song body by a fixed (slightly rubber) gap — it is **not**
pushed to the page bottom. The original design used `\par\vfill` to drop it to the
foot of the last page, but the position then depended on how full the page was: a
page-filling song looked fine, but a short song blew the `\vfill` open and left the
block marooned in mid-page whitespace (confirmed by rendering a short song —
BangAndBlame — against a full one — BeginTheBegin — in `rem-songbook`). The fixed
gap keeps it just below the last section in both cases; if it doesn't fit it flows
to the **top** of the next page (a leading `\addvspace` is discarded at a page top),
not stranded at the foot. The body is set in `\MyLSremarksbodyfont` (`\small`).
Dropping `\vfill` also removes the old (then-untested) worry about it fighting
`\bookinclude`'s own `\MyLStochint` `\vfill` on the same last page — verified clean
in a songbook. In a songbook the block is included for free: `\includeleadsheet`
rewrites the file's `\end{document}`→`\endinput` (`external.code.tex:43–49`), so all
body content between `\begin{document}` and `\end{document}` — including a
post-`\end{song}` remarks block — is pulled in.

**Shared remarks across wrappers** (arrangement-by-wrapper): put
`\begin{remarks}…\end{remarks}` in a `Song--remarks--input.song` and `\input` it
**outside** each wrapper's `\begin{song}`. Because that `\input` is outside the song
env, the `_`-in-filename gotcha (active `_` inside `song`) does **not** apply there,
though `--` stays consistent with convention.

**Header is not a `\section`.** Deliberately a styled paragraph
(`\MyLSremarksheadfont`), not `\section*`, so it never reaches the TOC, a PDF
bookmark, or a running head. Knobs: `\MyLSremarkslabel` / `…headfont` / `…rule` /
`…bodyfont`, plus the length `\MyLSremarksskip` (gap above; `\setlength`).

### Backing-vocal de-emphasis (`\bg` / `\bgoff`)

A switch that greys + shrinks lyric text across `\measures` cells (`MyLeadsheets.sty`,
"Backing-vocals de-emphasis"). `\bg` sets a **global** bool (`\l__myls_bg_bool`) *and*
applies `\MyLSbgfont`; `\__myls_measurebox` re-applies the font at the start of every
cell while the bool is set (each cell is its own box/group, so a plain font switch
would not carry across cells), and the bool is force-cleared at each `\measures`
line's end. `\bgoff` clears the bool and applies `\MyLSbgreset`.

**The chord-position rule (why three forms exist).** `^{chord}word` is
`\getorprintchord` → `\chord` → `\__leadsheets_set_chord:nwn #1#2~#3`
(`leadsheets.library.chords.code.tex:65`): the chord grabs the next **space-delimited
word** (`#2`) and typesets it under the chord inside `\leadsheets_place_above`'s
`\group_begin:…\group_end:` + `tabular` (`chords.code.tex:159`). So a `\bg` placed
*immediately after* a chord is captured into `#2` and runs **inside that group** — its
colour/size revert at `\group_end:`, grey-ing only that one word (the global bool still
flips, which is why later whole cells still grey). Hence:

- `\bg ^{A}word` — `\bg` runs in cell scope first; the word under the chord inherits
  grey (no reset in `place_above`, only `\linespread{1}\selectfont`). **Chord over the
  grey word.**
- `^{A} \bg word` — the space after `^{A}` makes `#2` *blank*, so the chord gets the
  1 em empty-chord slot (`\l__leadsheets_empty_chord_dim`) and detaches; `\bg` then runs
  in cell scope. **Chord floats free, grey lyric after.** (A space after *any* chord
  detaches it this way — independent of `\bg`.)
- `^{A}\bg word` — the captured-in-group case above. **Ambiguous; avoid.**

No clean `.sty`-only fix exists: the capture is leadsheets' space-delimited scan over
*input* tokens, before `\bg` expands, so redefining `\bg` can't escape it; only patching
leadsheets' chord internals would — against the "avoid leadsheets internals" goal.
Verified visually on `CantGetThere.song`'s chorus (the file that surfaced it).

### Barline appearance — width and length

Every bar in a chart (`\normalbar`, `\thickbar`, and the composed `\leftrepeat` /
`\rightrepeat` / `\stopbar`) is a `\rule` drawn by leadsheets'
`\genericbar{<width>}` (`musicsymbols.code.tex`), which hardwires the rule's
*length* to the current font size: `\leadsheets@barheight = \leadsheets@size`
(`\f@size pt`), rule from `-0.2x` (depth) to `+0.8x` (above baseline). Two
appearance levers, both set at the top of `MyLeadsheets.sty` next to each other:

- **Width (thickness)** — public knob. `\def\normalbarwidth{.09em}` overrides
  leadsheets' thin `.02em` default. Read at bar-use time, so a plain `\def` wins.
- **Length (height)** — no public API, so we `\renewrobustcmd*\genericbar` to
  multiply the rule length by `\MyLSbarheightscale` (default **1.25** — a slight
  increase from the stock 1.0; chosen by eye against `BangAndBlame` at 1.0/1.25/1.5).
  The `-0.2` depth fraction scales with it, so the bar grows symmetrically and stays
  centred on the line. Tune globally in the `.sty` or per-song with
  `\renewcommand\MyLSbarheightscale{…}`.

Notes: redefining the single `\genericbar` chokepoint scales **all** bar types
together (they all compose it) and leaves `\measures`' width-alignment untouched
(that measures bar *width*, not height). It reaches into a leadsheets `@`-internal
(`\genericbar` / `\leadsheets@size` / `\leadsheets@barheight`) — justified only by
the absence of a documented bar-length API. The repeat `:` colon
(`\leadsheets@repeatcolon`) is a separate text glyph and does **not** scale, so on
`|:` / `:|` bars the rules grow but the dots don't (negligible at ~1.25).

### Transposition spelling & `\simplifyaccidentals`

leadsheets chooses a transposed chord's sharp/flat **lean** from the **transpose step
count**, not the resulting key. `\leadsheets_transpose:nnN` (`transposing.code.tex`) sets
the target preference to `\l__leadsheets_prefer_key_prop[steps mod 12]` (a fixed
least-accidentals-by-key table) unless `enharmonic=sharp/flat` is set, in which case it
uses that. Step count equals the resulting key **only when transposing from C/Am**, so
from any other key — and especially under a capo, whose shift is the large complement
`12 − N` (`\__leadsheets_check_capo:`) — the lean can be the wrong one for the key you
actually land in. (Worked example: capo 1 = steps 11, `prefer_key_prop[11] = sharp`, so a
section landing in the Db-area spells `C#/D#m/F#…` rather than the fewer-accidental
`Db/Ebm/Gb`. The pre-change part is all-natural C major, so the mis-lean is invisible
there — which is why only the post-key-change section of `AtTheEndOfTheDay-CAPO` looked
off.)

`enharmonic=sharp/flat` overrides the lean, but it forces the **theoretical** scale of
that step count, which at the extremes carries double-accidentals and `Cb/Fb/B#/E#` — e.g.
capo 1 + `enharmonic=flat` spells a C-natural as `Dbb`. That is *correct* notation for a
key nobody would choose, hence:

**`\simplifyaccidentals` (`MyLeadsheets.sty`).** An opt-in switch (`\newif\ifMyLSsimplify`,
plus `\simplifyaccidentalsoff`) that rewrites the 14 theoretical spellings to conventional
names — `Dbb→C, Ebb→D, Gbb→F, Abb→G, Bbb→A, Cb→B, Fb→E` and the sharp mirror
`C##→D, D##→E, F##→G, G##→A, A##→B, B#→C, E#→F`. The 14 patterns are pairwise disjoint, so
order is immaterial; the single-accidental rules even fix rarer extras by composition
(`Cbb→Bb`, `B##→C#`). It is **orthogonal to `enharmonic=`** (which picks the lean); the
usual capo pairing is `enharmonic=flat` + `\simplifyaccidentals`. Read live at each chord
and set as a local assignment, so it is `\setleadsheets`-scoped: whole sheet (body, before
`\begin{song}`), rest of song, or one section (reverts at `\end`); in a songbook put it in
the **body**, not a song preamble (gobbled), like `\resize`. Do **not** switch it on when
you deliberately want a theoretical key (C# major's `E#/B#` are correct there) — that is
what `enharmonic=` alone is for.

**Implementation — WRAP, do not replace.** `\leadsheets_chord_print:n` looks like an
identity (`chords.code.tex:52`) but the `chords/format` key re-sets it to the brick-red
`\chordname` formatter (accidental glyphs + quality superscripts), which MyLeadsheets does
in its `chords/format` block. So the hook **snapshots** that formatter
(`\cs_set_eq:NN \__myls_orig_chord_print:n …`) and, when on, cleans the raw name first
(`\__myls_simplify:N`; the chord's `#` is catcode-other, matched with `\c_hash_str` as
`\MyLSformattuning` does) then hands the result to the saved formatter; off, it calls the
saved formatter unchanged (verified pixel-identical to the un-hooked `.sty`). A first
version `\cs_set:Npn`-**replaced** chord_print and silently dropped all chord formatting
(black, no glyphs) — caught only by eyeballing the render, not the chord *string* in the
log, so test this code path visually. The snapshot is taken after MyLeadsheets'
`chords/format`; a later `chords/format` re-set in a document would bypass the simplifier
(no song does this).

### Key label under transpose — handled via `\fixedchord` (legacy note)

**Not deferred — resolved by hand.** In practice the displayed key is set
directly: a `transpose=` wrapper writes the key it wants in `key={\fixedchord{…}}`
(accidentals as `\sharp`/`\flat`), and `\fixedchord` renders that glyph *without*
transposing it — so the printed label always reads correctly, independent of
`transpose=`. The auto-follow conditional sketched below was therefore never
needed; it is kept only as a record of how it *would* be wired if ever wanted.

The title template prints `\songproperty{key}` raw, so a `transpose=` wrapper's
PDF shows the *authored* key (e.g. "C") even though its chords are in D;
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

### `\resize` — single-knob vertical tightener

Authoring contract is CLAUDE.md's *Fitting — `\resize`*; this is the how. Lives in
`MyLeadsheets.sty`'s "`\resize`" block (just above the coloured-box templates).

**Shape.** `\newcommand\MyLSrf{1}` (the factor) + `\newcommand\resize[1][0.95]
{\renewcommand\MyLSrf{#1}}`. `\MyLSrf` is read **live** at each box and each
section-format expansion (it is just a digit-string), so the command works in the
preamble *or* the body — nothing is frozen at call time. `\resize` only
`\renewcommand`s the factor; never call it inside a `song`/group if you want a
document-wide effect (it would revert at the group end like any local def).

**The three levers, and why factor 1 is an exact no-op.** The section boxes today
leave `top`/`bottom`/`boxsep`/`beforeafter skip` **unset**, inheriting tcolorbox's
`reset@core` defaults (`tcolorbox.sty`:2832 ff.): `top=2mm, bottom=2mm, boxsep=1mm,
beforeafter skip balanced=0.5\baselineskip plus 2pt`. `\resize` makes the box
**restate those exact values, each prefixed by `\MyLSrf`**:
`top=\MyLSrf\dimexpr2mm\relax`, … , `beforeafter skip balanced={\MyLSrf\dimexpr
0.5\baselineskip\relax plus 2pt}`. The third lever is the interline gap:
`verses-format`/`chorus/format` change `\openup\mysonglinesep` →
`\openup\MyLSrf\mysonglinesep`. At `\MyLSrf=1` every product is the stock value, so
the output is **pixel-identical** to the pre-`\resize` `.sty` — verified by
rendering `AtTheEndOfTheDay` (full view) at 100dpi under both and `md5`-matching all
three pages.

**Arithmetic.** `\MyLSrf\dimexpr2mm\relax` is the decimal-coefficient × internal-
dimen form (e-TeX): a macro expanding to `0.95` times a `\dimexpr` dimen is a valid
`<dimen>`; the glue `…\relax plus 2pt` likewise parses as a `<glue>` in the
`beforeafter skip` key. `\openup\MyLSrf\mysonglinesep` is `0.95\mysonglinesep`
(coefficient × length register) — no `\dimexpr` needed there. The
`beforeafter skip` keeps `\baselineskip`-relative (evaluated per box, in the body
font), so it tracks the real body leading rather than whatever leading happened to
be in force when `\resize` was called.

**Scope deliberately vertical-only.** Font size, `geometry`, and horizontal
measures (`\minmeasure`, `\measurepad`, chord size) are **not** scaled — songs feed
a shared songbook where uniform type matters (CLAUDE.md design goals). So `\resize`
reclaims height only; on a song whose overflow page holds a large chunk (not a hair)
it cannot help without an unflatteringly small factor (`AtTheEndOfTheDay` full view
needs ≈0.75 to drop page 3 — its whole last third spills, so it is not really the
marginal case the knob is for).

**Songbook scoping is automatic — no reset code in `\bookinclude`.** Two facts
combine. (1) `\includeleadsheet` runs with `gobble-preamble` (it discards everything
up to the first `\begin`, `leadsheets.library.external.code.tex`:151,
`#1 \begin #2`), so a `\resize` in a song's **preamble** is gobbled in a book —
like `\ForceEvenPages` — and must be placed in the **body** (after
`\begin{document}`) to take effect. (2) `\bookinclude` wraps the include in
`\begingroup…\endgroup` (for `\input@path`), and `\resize` is a plain local
`\renewcommand`, so a body `\resize` is local to that song's group and **reverts at
its `\endgroup`** — the next song starts at the inherited factor. Verified with a
two-song book: song A's body `\resize[0.6]` changed A's page but left B
**pixel-identical** to a control book where A had no `\resize` (no leak); and A's
page *did* change vs that control (the body resize genuinely applies inside a book).
Consequence, deliberately not "fixed": a `\resize` in the **songbook preamble** is a
book-wide default that each song can locally override and revert from. An explicit
`\renewcommand\MyLSrf{1}` in `\bookinclude` was therefore *rejected* — it would buy
nothing (the group already resets) and would clobber that book-wide default.

## Tooling / debugging

### Vimtex quickfix behaviour

Vimtex's log parser only surfaces `Package X Warning:` patterns into the quickfix
window — `Class X Warning:` messages (e.g. KOMA's own warnings about incompatible
packages) appear in the raw log but are invisible in the quickfix. When diagnosing
warnings, check the raw log at `~/.cache/latexmk/{dirname}-{jobname}/` (vimtex
compilations) or `~/.cache/latexmk/{parent-dir}-{jobname}/` (`build-pdfs`).

## build-pdfs — status

`build-pdfs` (repo root, `~/1sys/tex/songs-machinery/build-pdfs`) compiles each
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
  (`V1:`/`C1:`/`In:`) automatically. Verified on `ChordsVariantDemo.song`.
- **lyrics view** — same `compile_view()` path (the `case` arm is just
  `chords | lyrics`), injecting `\def\View{lyrics}` for `Song--lyrics.pdf`. The
  `.sty` flips `print-chords=false` for this view only; the family selection is
  the same as the default build (lyric bodies print, chart twins swallowed). See
  *Views from one `.song`* item 2 for the `^`/`_` detail.

- **phone view** — `compile_view()` injects `\def\View{phone}`, producing
  `Song--phone.pdf`. Content is **identical to `full`** (same family path in the
  `.sty`: phone is neither chords/lyrics/debug, so the chart twins are swallowed
  and the lyric bodies render, with `print-chords` left true). The *only* `.sty`
  differences, all keyed on the early `\ifMyLSphone`: (a) geometry loads
  `paperheight=\MyLSphoneheight` (`22in`) at letter width / `\MyMargin` instead of
  `letterpaper`; (b) `\clearoddpage` degrades to a plain `\clearpage` (no odd-page
  padding — `\bookinclude` calls it, so the whole songbook is covered); (c) the
  back-to-contents link size `\MyLStoclinksize` is `\large` not `\footnotesize`.
  No font-size or margin change (an earlier draft bumped to 20pt + long paper à la
  the reference's `\Longpaper`, but the simpler "same layout, taller page" matches
  the two-letter-pages soft target and keeps the log clean — the 20pt bump dragged
  in `T1/cmr size not available` and `Very small head height` warnings). Verified
  full vs phone on a standalone song (1 tall page vs 2 letter pages) and a
  two-song `\bookinclude` book (phone 3 pp / no blanks vs full 6 pp / padded).

- **grayscale modifier (`-gray`)** — an *orthogonal* "colour off" axis that
  composes with the content views, exposed as the tokens `full-gray`,
  `chords-gray`, `lyrics-gray` (plus bare `gray` = `full-gray`). It is **not** a
  fourth content view: the `.sty`'s view block (just before the content-axis
  `\ifx` detection) strips a recognised gray token, sets `\ifMyLSgray`, and
  reduces `\View` to its bare content name (`full`/`chords`/`lyrics`). So every
  downstream branch — family selection, **remarks `[chords]` filtering**,
  `print-chords` — sees only the base name and is untouched; e.g. `lyrics-gray`
  strips over-word chords exactly like `lyrics`. (Keeping the modifier off `\View`
  is what preserves remarks filtering, which is an exact-string `\clist_if_in` of
  `\View` against the `[chords,…]` list — `grayscale-chords` would have broken it.)
  Colour is killed in two keyed-on-`\ifMyLSgray` spots: (a) `xcolor` loads with the
  **`monochrome`** package option (it sets `\colors@false`, so all `\color`/
  `\textcolor`/coloured TikZ glyphs fall back to black — verified: chords,
  noteboxes, `\bg`, difficulty/genre/arrangement all print black); (b)
  `\myleadsheetsboxbegin` routes `colback` through `\MyLSboxcolback`, which expands
  to `white` under gray instead of the part-type tint, so section boxes are plain
  white (boxrule stays 0mm — no border). The white override is independent of
  `monochrome` (belt-and-suspenders): a 6%-gray tint would survive monochrome as a
  faint gray, which the explicit `white` avoids.
  - *build-pdfs side.* The loop accepts `gray | chords-gray | lyrics-gray` (and
    `full-gray`), injecting `\def\View{<token>}` and naming `Song--<token>.pdf` —
    **except** `full-gray`, which it normalises to `gray` first (`[[ $view ==
    full-gray ]] && view=gray`) so the file is `Song--gray.pdf`, not
    `Song--full-gray.pdf`. Verified end-to-end (`%! views: full-gray, chords-gray,
    lyrics-gray` → `Song--gray.pdf` + two `--*-gray.pdf`) and by rendering each to
    PNG (black text, white boxes, correct base content per view). Deliberately
    knob-free (rare, print-only).

The presentation axis still folds together with capo/transpose via the wrapper
model: put
`%! views: chords` in a capo/transpose wrapper and that wrapper gets its own
`--chords`/`--lyrics` siblings, e.g. `Song-CAPO.song` → `Song-CAPO--chords.pdf`.
The `-CAPO`/`-OriginalKey` (single-hyphen, *different musical content*) vs
`--chords`/`--lyrics` (double-hyphen, *different presentation*) suffix split is the
convention to preserve.

## Output views — design history

The output-views work here is **complete** (views 1–4 all DONE, grayscale DONE)
and is kept as design history. The ideas once tracked as future work are all
settled: the transpose-time key label is set by hand via `\fixedchord` in `key=`
(see *Key label under transpose* above, kept as a legacy note); `\measures`/chart
behaviour is satisfactory as-is, with no aligned-grid rework planned;
`[both]`/`[shared]` was decided against in favour of the macro-reference pattern;
`\mw{<dim>}` is retained only as a documented, low-priority escape hatch; and the
few unconverted legacy raw-pipe files compile fine and are no concern.

The reference produced several PDFs from one source via a tangle of `\newtoggle`
flags that cross-set each other (`LyricsWithChords`, `LyricsNoChords`,
`ChordProgression`, `ChordsAndLyrics`, `SmallLyrics`, `Structure`) and four
content-gating macros (`\chords`, `\lyrics`, `\lyricswithchords`, `\structure`),
each `\iftoggle{…}{#1}{\@gobble{#1}}`. The cost landed in the `.song`: the same
material was typed 3–4× per section, once per view. The overhaul should reach
the same outputs with far less duplication. Planned views:

1. **Full — lyrics + chords.** The `^{C}word` lyric body (chords over words). The
   default build / suffixless `Song.pdf`; `\View` defaults to `full`. Baseline. *Done.*
2. **Lyrics-only — DONE, the `lyrics` view.** The **same** `^{C}word` body with
   leadsheets' `print-chords=false`. No duplication — one body serves both 1 and 2.
   Selected by `\def\View{lyrics}`; renders the same family as the `full` view
   (lyric types print, chart twins swallowed) and only flips `print-chords`. Key
   detail: `print-chords=false` suppresses **only** `^{C}word` chords (`^` =
   `\getorprintchord`, gated by `print-chords` in `\leadsheets_place_above`); the
   `_{C}` chord-only tokens in `\measures` go through `\writechord` and print
   regardless (`songs.code.tex:143-144`). So lyric sections become words-only while
   instrumental sections (a `\measures` line of `_{C}` chords, no `^{}word`) keep
   their progression — the sensible outcome, and it falls out of the documented
   `^`/`_` split with no extra code. Verified on `ChordsVariantDemo.song`.
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
4. **Phone-format — DONE, the `phone` view.** The **same** `full` lyrics + chords
   layout on a taller page (8.5×22in vs letterpaper; same width, margins, font).
   Selected by `\def\View{phone}`; renders the same family as `full`. Deliberately
   simpler than the reference's `\Longpaper` (which also bumped to 20pt) — see the
   *phone view* bullet under *build-pdfs — status* for the three `\ifMyLSphone`
   differences and why the font bump was dropped.

View selection is a single `\def\View{…}` command-line flag (injected by
`build-pdfs`) reduced to **one** clean view state in the `.sty`, not the
reference's six cross-set toggles — realised for `chords`, `lyrics` and `phone`
(each a small `\newif`/`\ifx` line in the view-detection block near the top of the
`.sty`, with the family swallow in "select which family renders", the
`print-chords` switch at the file end, and — for `phone` — the geometry, the
`\clearoddpage` guard, and the link-size knob).

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
`ChordsVariantDemo.song` (`versechords[minmeasure=3cm]` widens V1's bars;
C1/Out revert).

**`[both]`/`[shared]` was decided against** — the macro-reference pattern (define
the shared body once, reference it in both twins; see CLAUDE.md's *Shared
instrumental sections*) is the chosen answer. The proposal below is kept only as a
record.

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

**Status: not planned.** Current charts are satisfactory; the list below is
reference only. `\mw` is kept as a low-priority escape hatch (see its bullet), and
barline-grid alignment is not on the roadmap.

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

