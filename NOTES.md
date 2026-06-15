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

Internals of the IMPLEMENTED compile-time-transpose features. The authoring
contracts (what to type in a `.song`, output naming, constraints) stay in
CLAUDE.md; this is the how-it-works detail for when the mechanism itself needs
touching.

### Capo variants

Working end-to-end for `samples/Uberlin.song` (concert source, `key={Gm}`,
`capo=3`): `compile-songs` produces `Uberlin.pdf` (concert) and `Uberlin-CAPO.pdf`
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
`compile-songs` when the song carries `%! capo-pair`. With it the script builds
**two** PDFs: `Song.pdf` = concert and `Song-CAPO.pdf` = capo shapes. The concert
one injects `\def\ConcertVariant{true}` before `\input` (a `-pdflatex` override
**forced with `latexmk -g`**, since the injected `\def` doesn't change the source
and latexmk would otherwise keep a stale PDF); the capo-shapes one is just a
native build under `-jobname=Song-CAPO`.

**Division of labour:**

- **`capo=N` property** — single source of truth for the fret (and, alone, gives
  the capo-shapes sheet via pure leadsheets — no script needed).
- **`%! capo-pair` magic comment** — boolean; asks `compile-songs` for the
  concert sibling too.
- **`.sty`** (the "Capo variants" block) — `\providecommand\ConcertVariant{false}`
  + `\newif\ifMyLSconcert`; on the concert build (`\ConcertVariant` = `true`) it
  emits `\setleadsheets{transpose-capo=false}` (chords stay concert) and the
  title template's `\ifMyLSconcert\else … \notebox{capo \songproperty{capo}} …`
  suppresses the note. *Gotcha:* the flag is compared with `\ifx`, so the helper
  `\MyLSconcertflag` must be a plain `\def` (not `\newcommand`, which is `\long`
  and won't `\ifx`-match the injected non-long `\def`).
- **`compile-songs`** — `wants_capo_pair()` reads the comment; `compile_concert()`
  does the injected `-g` build; `build_song()` routes plain `Song.pdf`→concert and
  `Song-CAPO.pdf`→native shapes.

### Transpose variants

Original-key sibling, parallel to capo but for a *real* key change (the chords'
sounding pitch moves, unlike capo's fingering shapes). Working for
`samples/TransposeDemo.song` (transcribed in C, `transpose=2`): `compile-songs`
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
- **`compile-songs`** — `wants_transpose_pair()`; default PDF = native transposed
  build; `compile_original_key()` injects `\def\SongTranspose{0}` (forced `-g`) for
  `SONG-OriginalKey.pdf`. **No `.sty` change at all** — leadsheets does the
  transposition.

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
