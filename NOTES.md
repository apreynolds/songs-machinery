# leadsheets package — issues log

Running log of bugs / surprising behaviour in the (unmaintained) `leadsheets`
package, as found while rebuilding `MyLeadsheets.sty`.

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
