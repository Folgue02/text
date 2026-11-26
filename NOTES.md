# Notes: problems found while building folgue02/text

Tools: `ferry 0.12.0` / `lks` built from the lakescript repo (`target/debug`).

## 1. `ferry` on PATH is stale (0.11.2)
`which ferry` → `~/.cargo/bin/ferry`, `ferry --version` → `ferry 0.11.2`, while the repo is at 0.12.0.
Not a bug in ferry, but nothing warns about it (no "a newer lks/ferry is in this project" or version mismatch between `ferry` and the `lks` it runs). Everything below uses `target/debug/ferry`.

## 2. `ferry new author/name` creates `./name`, not `./author/name`
`ferry new folgue02/text --lib` → `Created the library folgue02/text in .../text`. Fine, but the help/notes don't say which folder you'll get; I expected it might mirror the ID. It also creates no `tests/` folder and no `module.md`, even for `--lib`, although `ferry doc` treats `src/<author>/<name>/module.md` as the front page of the library. A stub `module.md` (and maybe `tests/`) for `--lib` would help.

## 3. Reserved word as module name: accepted at `use`, fails at the call with an unhelpful error
The task suggested a module called `case`. modules.md says it needs an alias, but the compiler doesn't say so:
```
// src/a/b/case.lks
pub string x() = "1";
// tests/t.lks
use a/b/case;
void testX() { assert case::x() == "1"; }
```
`ferry check` →
```
error: expected an expression, found `case`
 --> tests/t.lks:2:23
```
The `use a/b/case;` line itself is accepted silently. Suggest: error (or warning) on the `use` line, "`case` is a reserved word: import it with an alias (i.e. `use a/b/case as c;`)". I named the module `casing` instead.

## 4. `ferry check` only reports the errors of the first file that has any
```
// src/a/b/w.lks
number bad2() = "s";
// src/a/b/z.lks
number bad() = "s";
```
`ferry check` shows only the `w.lks` error and says `a/b has 1 error`. Fix it, rerun, and only then `z.lks` shows up. Same with an error in `src/` hiding the errors in `tests/` (an error in `tests/t.lks` was not reported while `src/a/b/case.lks` had one). The total "has 1 error" is therefore wrong, not just incomplete.

## 5. Diagnostic messages contain raw Markdown from the notes
```
pub enum Align { Left, Right, Center }
```
→ `error: the values of an enum are written in SCREAMING_CASE *(i.e. `LEFT`)*, and end with `;`` 
and `pub E f() = E.A;` → `error: `E` is an enum: its values are created where it's declared *(i.e. `E::...`)*`.
The `*(i.e. ...)*` is the notes' Markdown style leaking into terminal output. Also the second message is confusing for the actual mistake (`.` instead of `::`); "use `E::A` to name a value of an enum" would say what's wrong.

## 6. The enum error also says "end with `;`", but the one-line form is the surprise
`enum Align { LEFT, RIGHT, CENTER; }` works; coming from other languages the trailing `;` inside braces is unexpected even with no methods. Not a bug, but worth a mention in the error only when the `;` is actually missing.

## 7. Paths in diagnostics are inconsistent
Errors in `src/` print the absolute path (`--> /private/tmp/.../src/a/b/w.lks:1:17`), errors in `tests/` and warnings print a relative one (`--> tests/t.lks:2:28`, `--> src/a/b/case.lks:1:5`). Relative to the project root everywhere would be shorter and consistent.

## 8. "`x` returns a string, not a number" reads backwards
`pub string x() = 1;` → `` `x` returns a string, not a number ``. It means "is declared to return a string, but this is a number", but reads like a statement about what it actually returns. Compare the clearer variable form: `` `$n` is a number, but its value is a string ``. Suggest the same shape: "`x` returns a string, but this value is a number".

## 9. Multi-line values in failed asserts break the layout
```
assert wrap::justify('aa b cc dd eee', 8) == "aa  b cc\ndd   eee";
```
```
  left:  'aa  b cc
dd eee'
  right: 'aa  b cc
dd   eee'
```
(continuation lines are only indented by the test-runner's prefix, not aligned with `left:`), so the two sides are hard to compare. Showing them escaped (`'aa  b cc\ndd eee'`) or as an aligned block/diff would help. (The test was mine and wrong; the display made it slower to see.)

## 10. `ferry doc` refuses to build when a *test* has a type error
`error: a/b has 1 error, so its docs weren't made` — caused by an error in `tests/t.lks`, which isn't part of the docs. Arguably it should only need `src/` to check.

## 11. No warning for an undocumented `pub` item
`pub string y() = "1";` without `///` gives no warning from `ferry check` or `ferry doc`. The comment-mismatch warning exists (`the comment describes `$y`, but `g` has no parameter `$y``, good), so an opt-in "missing docs" lint would fit.

## 12. Library overlaps a lot with the stdlib (design note, not a bug)
`string` already has `camelCase`, `snakeCase`, `kebabCase`, `padStart/padEnd/center`, `truncate`, `wrap`, `collapseWhitespace`. So this package only adds what's missing: headline title case, identifier splitting (`parseHTTPHeader` → words, which `snakeCase` alone may not do the same way), fit/columns, wrap with indents and justification, slugify with accent folding. Missing from the stdlib that would have helped: a Unicode normalize/strip-accents (`fold` here is a hand-written table), and a `string.words()` variant that splits identifiers.

## What worked well
`ferry new --lib`, `ferry check`, `ferry test` output, `test::throws`, `ferry doc` HTML and `ferry doc casing::titleCase` terminal lookup (short `module::name` form works), `module.md` as the front page, the `$param:` mismatch warning.
