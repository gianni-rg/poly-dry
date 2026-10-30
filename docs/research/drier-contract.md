# `dryer` externally observable contract

Research note inventorying the behavior of the local `dryer` checkout at
`D:\Personal\GitHub\unclebob-dryer` (commit as checked out, no git commands run).
All claims are cited to exact paths/lines in that repo. "Source" = implementation
read directly; "Test" = behavior asserted by one of the three test files; "Docs-only"
= README claim not independently confirmed in source/tests.

## 1. CLI

- Entry point is the console script `dryer = "dryer.cli:main"`; `main()` calls
  `run(argv)` and `sys.exit`s its return code (`pyproject.toml:16-17`;
  `src/dryer/cli.py:258-259`). The repo-root `dryer` shell script bootstraps a
  `.venv` on first use and execs `python -m dryer "$@"` (`dryer:1-8`).
- Argument parsing is hand-rolled (no `argparse`/`click`); `parse_args` loops over
  `sys.argv[1:]` (or an injected `argv`) matching option strings directly
  (`src/dryer/cli.py:106-162`).
- Flags and defaults: `--root <path>` (default `.`), `-s/--source-root <path>`
  (repeatable), `--changed`, `--threshold N` (default `0.82`), `--min-lines N`
  (default `4`), `--min-nodes N` (default `20`), `--format F` (default `"text"`),
  `--edn` (sets format to `"edn"`), `--text` (sets format to `"text"`) — last of
  `--edn`/`--text`/`--format` wins since they mutate the same field in argument
  order (`src/dryer/cli.py:78-162`; Test: `tests/test_cli.py:255-258` shows
  `--edn --text` later wins and prints the text report).
- Non-flag args become `positionals`; a leading `-` with no matching flag name
  raises `Unknown option: <arg>` (`src/dryer/cli.py:154-158`).
- `-h`/`--help` short-circuits before any other parsing and prints `HELP` to
  stdout with exit 0 (`src/dryer/cli.py:109-110,238-241`; Test:
  `tests/test_cli.py:29-33`).
- Exit codes: `0` after a help print or after a report is written
  (`src/dryer/cli.py:239-257`); `1` on a usage/parse `ValueError` (missing value,
  non-numeric threshold, non-integer/negative count, unknown option)
  (`src/dryer/cli.py:142-146`; Test: `tests/test_cli.py:46-48,266-291`); `2` when
  the resolved format is not `"text"`/`"edn"` (`src/dryer/cli.py:147-152`; Test:
  `tests/test_cli.py:41-44`). `--changed` also propagates git's own exit code via
  `GitStatusError.code` when `git status` fails (`src/dryer/cli.py:188-194,
  243-246`; Test: `tests/test_cli.py:217-228` expects `128` for "not a git
  repository").
- `--min-lines`/`--min-nodes` accept `0` but reject negative integers and
  non-integers (`src/dryer/cli.py:97-104`; Test: `tests/test_cli.py:271-284`).
- Path-fragment matching: any positional that is not an existing file/dir under
  `--root` is treated as a substring filter applied to the POSIX-style resolved
  path of every candidate file (`src/dryer/cli.py:168-179, 228-232`; Test:
  `tests/test_cli.py:109-119` — `"board"` matches `src/board/*.py` only).

## 2. Input discovery

- Directory walk uses `os.walk`, pruning `dirnames` in place so skipped
  directories are never descended into (`src/dryer/discover.py:86-97`).
- `SKIP_DIRS` (always skipped): `.git .hg .svn .idea .venv venv node_modules
  vendor target dist build out coverage .metrics testdata __pycache__
  .clj-kondo`. `TEST_DIRS` (skipped during an implicit walk, but see below):
  `test tests spec specs __tests__` (`src/dryer/discover.py:26-45`).
- Per-file test detection (`is_test_file`) additionally flags `*_test.go`,
  Clojure `*_test.clj[c|s|d]`/`*_test.bb`, TS/TSX `*.test.*`/`*.spec.*` for
  `.ts .tsx .mts .cts`, `conftest.py`, `*_test.py`, and `test_*.py`
  (`src/dryer/discover.py:57-78`; Test: `tests/test_cli.py:150-172` enumerates
  every pattern).
- `iter_source_files` dedupes by resolved path and sorts by POSIX path string
  (`src/dryer/discover.py:99-105`).
- **Explicit paths are not filtered by test status**: a file or directory named
  directly on the command line is included even if it would otherwise be
  skipped as a test file — `_explicit_files` only applies `language_of`, not
  `is_test_file` (`src/dryer/cli.py:177-180`; Test:
  `tests/test_cli.py:122-140` explicitly includes `tests/test_a.py`). Skip-dir
  pruning for *directories* still applies when an explicit directory argument is
  walked via `iter_source_files`.
- `--changed`: resolves the repo top-level via `git rev-parse --show-toplevel`,
  then `git status --porcelain -z --no-renames --untracked-files=all -- .`
  scoped to `--root`; deletions are dropped (`path.is_file()` check), renames
  report only the new path (`--no-renames`), and results are restricted to
  paths under the resolved root (`src/dryer/cli.py:160-186`; Test:
  `tests/test_cli.py:195-209`). Changed files are further filtered by
  `_tracked_source`: not in `SKIP_DIRS`, has a known language extension, and not
  `is_test_file` — so `--changed` *does* exclude test files, unlike explicit
  path args (`src/dryer/cli.py:149-158,203-205`; Test:
  `tests/test_cli.py:240-251,328-331`).
- `--source-root` bypasses explicit-path/changed logic entirely and always walks
  via `iter_source_files` (applying the default skip rules)
  (`src/dryer/cli.py:208-210`; Test: `tests/test_cli.py:303-308`).
- With no positionals, no `--source-root`, and no `--changed`, the whole
  `--root` tree is walked (`src/dryer/cli.py:211-212`).

## 3. Candidate extraction per language

- Language is chosen purely by file extension via `EXTENSIONS`
  (`.clj .cljc .cljs .cljd .bb`→clojure, `.java`→java, `.go`→go,
  `.ts .tsx .mts .cts`→typescript, `.rs`→rust, `.py`→python), with an explicit
  exception that any `*.d.ts` file has no language (`None`)
  (`src/dryer/discover.py:18-54`; Test: `tests/test_cli.py:26-34`).
- **Clojure**: every top-level form is read by a hand-written reader
  (`src/dryer/clojure_read.py`); a form is a candidate iff it is a non-empty
  list whose head symbol is not `ns` (`src/dryer/clojure_read.py:513-519`; Test:
  `tests/test_clojure.py:289-295`). There is no function/method restriction —
  any top-level list (`def`, `defn`, arbitrary calls) qualifies.
- **Java**: `method_declaration` nodes with a `block` body, excluding any nested
  inside another `method_declaration`, `constructor_declaration`, or
  `lambda_expression` (so constructors are never visited as a type, and local
  classes' methods are excluded only because they sit inside the enclosing
  method) (`src/dryer/extract.py:66-77`); abstract methods have no `block` and
  are excluded by the `_has` check (Test:
  `tests/test_languages.py:405-421` — abstract `declared` and the local class
  `Inner.run` are both absent; only the outer `alpha` method, lines 3-10, is
  kept).
- **Go**: `function_declaration`/`method_declaration` nodes with a `block`
  (`src/dryer/extract.py:80-83`); a function with no body (forward declaration)
  is skipped (Test: `tests/test_languages.py:423-434`).
- **Python**: `function_definition` nodes that are not nested inside another
  `function_definition` — a `class_definition` in between does *not* count as
  nesting, so methods are still top-level candidates, but closures/inner
  functions are excluded and compared as part of their enclosing function
  (`src/dryer/extract.py:58-64,86-89`; Test:
  `tests/test_languages.py:184-206` for nested-function exclusion;
  `tests/test_languages.py:400-404` for the class-contributes-its-method-only
  case).
- **TypeScript/TSX**: `function_declaration`/`method_definition` kept when they
  have a `statement_block` and are not nested inside another TS function type;
  `arrow_function`/`function_expression` are kept only when assigned to a
  `variable_declarator`/`public_field_definition`/`property_definition` and not
  nested inside another function — i.e. top-level/exported arrow functions
  count, callbacks passed as arguments do not (`src/dryer/extract.py:92-120`).
  Grammar selection is per-file: `.tsx` files parse with the `tsx` grammar,
  everything else with `typescript` (`src/dryer/extract.py:29-32`; Test:
  `tests/test_languages.py:122-139` shows both `.ts` and `.tsx` arrow functions
  matched).
- **Rust**: `function_item` nodes with a `block`, excluding ones whose immediate
  parent is itself a `block` (so a closure assigned via `let f = || {...}` is
  not separately extracted — the closure stays folded into the enclosing
  function's normalized tree) and excluding any function nested (at any depth)
  under a `mod tests { ... }` module (`src/dryer/extract.py:50-56,123-131`;
  Test: `tests/test_languages.py:214-236` for `mod tests` exclusion;
  `tests/test_languages.py:451-459` "closure in a let stays inside the
  function").
- Language detection for tree-sitter languages happens once per file in
  `_tree_entries`/`_PICKERS`, dispatching to a per-language node picker and
  (for TypeScript) a per-file grammar name function (`src/dryer/extract.py:
  134-152`).

## 4. Normalization and similarity

- **Clojure** (`src/dryer/clojure_read.py:488-511`): a list becomes
  `[:list <head-normalized-with-name> <args-normalized-without-name>...]`;
  vectors/sets/maps become `[:vector ...]`/`[:set ...]`/`[:map k v ...]`.
  The head symbol of a list keeps its literal name (`[K("symbol"), name]`);
  every other symbol becomes a bare `:symbol` marker, every keyword becomes
  `:keyword`, every other atom becomes `:literal`
  (`src/dryer/clojure_read.py:435-440`). Reader conditionals keep only the
  `:clj`/`:default` branch and splice list/vector branches into the parent
  (`src/dryer/clojure_read.py:369-383`; Test:
  `tests/test_clojure.py:88-99,228-245`). Comments and `#_` discards are
  dropped by the reader itself, never reaching normalization (Test:
  `tests/test_clojure.py:116-129`).
- **Tree-sitter languages** (`src/dryer/astnorm.py`): comments are skipped
  (`_SKIP`, lines 16-22); identifiers/type-identifiers/field-identifiers keep
  only a generic `:symbol` marker *unless* they are the callee of a
  call/constructor expression, member/attribute access target, or operator
  token, in which case the literal name/operator text is kept via
  `[K("symbol"), text]` (`src/dryer/astnorm.py:143-230,255-285`). Literal nodes
  (`string, number, true/false/nil/null, regex`, etc.) become a bare
  `:literal`, except interpolated/template strings with an `interpolation`
  child, which are *not* treated as plain literals (`src/dryer/astnorm.py:
  44-70,157-162`; Test: `tests/test_languages.py:362-367`
  "an interpolated string is not a plain literal"). Type arguments
  (`type_arguments`/`type_parameters`) are normalized but kept as ordinary
  children, not stripped (`src/dryer/astnorm.py:84-85,204-213`; Test:
  `tests/test_languages.py:330-346` confirms constructed generic names like
  `Vec`, `HashMap`, `ArrayList` remain in the fingerprint set).
- Fingerprinting (`src/dryer/shape.py`): `fingerprints(node)` returns the
  **union** of (a) the Clojure-`pr-str`-style printed form of every subtree
  (`_printed`, post-order, iterative to avoid recursion limits on deep trees)
  and (b) the printed form of every individual atom in the tree
  (`_atom_prints`) (`src/dryer/shape.py:90-99`). `node_count` is 1 + count of
  every nested list element, atoms count as 1 (`src/dryer/shape.py:72-81`).
- **Jaccard formula**: `jaccard(left, right) = len(left & right) / len(left |
  right)`, returning `0.0` when the union is empty
  (`src/dryer/shape.py:102-106`). This is applied directly to the two entries'
  `fingerprints` frozensets in `find_duplicates`
  (`src/dryer/scan.py:70-78`).

## 5. Filtering

- `_keep` in `scan.py` applies `--min-lines`/`--min-nodes` immediately after a
  file is scanned, *before* any pairwise comparison: `lines = end_line -
  start_line + 1 >= min_lines` and `entry.nodes >= min_nodes`
  (`src/dryer/scan.py:19-21,34-35`).
- Only entries sharing the same `language` are ever compared; grouping happens
  in `find_duplicates` via a `by_language` dict (`src/dryer/scan.py:66-70`).
- Identical-span pairs (same file, same start/end line) are skipped via
  `_same_span` before scoring — relevant when the same file is scanned twice
  (e.g. named both explicitly and via a directory walk)
  (`src/dryer/scan.py:44-51,76-77`).
- `--threshold` is applied last, per pair, after scoring: `if score <
  threshold: continue` (`src/dryer/scan.py:79-80`). Path-fragment filters
  (positional non-path args) are applied even earlier, to the file list itself,
  before `scan_files` ever reads a file (`src/dryer/cli.py:228-236`).

## 6. Output/report formats

- Text report: `"No duplicate candidates found.\n"` when empty; otherwise
  blocks of `DUPLICATE score=%.2f\n  <left file>:<start>-<end>\n  <right
  file>:<start>-<end>` joined by a blank line, trailing newline
  (`src/dryer/report.py:10-20`; Test: `tests/test_clojure.py:49-59`).
- EDN report: `{:candidates []}\n` when empty; otherwise
  `{:candidates [\n <row>\n <row>\n]}\n` where each row is a map with
  `:score` (printed via `%.12f` then trimmed of trailing zeros/dot, forced to
  keep one decimal, e.g. `0.875`), `:language`, `:left`/`:right` span maps
  (`:file :start-line :end-line`), and `:left-nodes`/`:right-nodes`
  (`src/dryer/report.py:23-62`; Test: `tests/test_clojure.py:28-45` pins the
  exact formatting).
- `.metrics/dry.edn` is written via `write_metrics` on **every** run that
  reaches the scan step, including when `candidates` is empty — it is created
  with `mkdir(parents=True, exist_ok=True)` and always overwritten
  (`src/dryer/report.py:65-71`; Test: `tests/test_cli.py:81-90` checks file
  contents after a non-empty run; no test exercises the empty-but-source-files
  case for file writing specifically, but the code path is unconditional
  whenever `files` is non-empty). It is **not** written when there are no
  source files at all — `run()` returns `0` with `"No source files to
  analyze.\n"` before calling `scan_files`/`write_metrics`
  (`src/dryer/cli.py:244-247`; Test: `tests/test_cli.py:105-107`).
- Sort/order: `find_duplicates` sorts candidates by `_duplicate_key` —
  `(-score, language, left.file, left.start_line, right.file,
  right.start_line)` — so **highest score first** (closest match first, as the
  README claims), with language then file/line as tie-breakers
  (`src/dryer/scan.py:53-61,89`; Test: `tests/test_clojure.py:184-190` "three
  copies make three pairs" all score 1.0, consistent but does not alone prove
  descending order; the `-score` key is a direct source read, not merely
  inferred).

## 7. Error handling

- Clojure unreadable tail: `read_source` returns `(forms_read_so_far,
  "<line>: <message>")` the moment a `ReadError` is raised; everything read
  before the error is kept and used for comparison (`src/dryer/clojure_read.py:
  397-413`). `scan_files` surfaces this as `warnings.append(f"{file}:
  {warning}")` and `run()` prints each as `dryer: <file>:<warning>` to stderr,
  but still proceeds to score and report the forms that *were* read
  (`src/dryer/scan.py:30-33`; `src/dryer/cli.py:249-250`; Test:
  `tests/test_clojure.py:191-204` "unreadable tail does not drop an earlier
  form" — confirms the exact README claim).
- Non-Clojure languages: `entries_in_source` never produces a warning string
  for tree-sitter languages (`src/dryer/extract.py:154-158`); tree-sitter
  itself does not raise on malformed input — it produces `ERROR`/`MISSING`
  nodes, which are simply not one of the recognized candidate node types and
  so silently contribute no entries (no explicit test for malformed
  Java/Go/TS/Rust/Python files was found in the three test files).
- Unknown `--format` produces `action="format"`, prints `"Unknown format:
  <value>\n"` to stderr, and `run()` returns `exit_code=2` without ever
  selecting files or scanning (`src/dryer/cli.py:147-152,242-243`; Test:
  `tests/test_cli.py:40-44`).
- A `GitStatusError` during `--changed` is caught in `run()`, prints
  `exc.message` to stderr, and returns `exc.code` (git's own exit status, e.g.
  `128`) without printing "No source files..." or writing metrics
  (`src/dryer/cli.py:243-246`; Test: `tests/test_cli.py:217-228`).

## Gaps / unverified claims

- **README says** Java excludes "methods inside anonymous or local classes" as
  if it were a dedicated rule. **Source/tests show** this is only an emergent
  effect of the "exclude nested-inside-another-method_declaration" check
  (`src/dryer/extract.py:66-77`); there is no anonymous-class-specific logic,
  and no test names an *anonymous* class specifically (only a local class,
  `tests/test_languages.py:405-421`). Likely accurate in practice but the
  mechanism is more general than the README implies.
- **README says** Rust excludes "closures" outright. **Source** shows only
  closures whose *immediate parent is a `block`* are folded away (i.e. closures
  bound via `let`); a closure passed directly as a call argument (not a
  `block`-parent) is not covered by the exclusion check and its behavior is
  unverified by any test — `tests/test_languages.py:451-459` tests only the
  `let f = || {...}` case.
- **README's run instructions** (`./dryer`, `.venv` bootstrap) were read from
  `dryer:1-8` but not exercised; no test invokes the shell script.
- **No test exercises malformed/unparsable source for Java, Go, TypeScript,
  Rust, or Python.** The source-level behavior (tree-sitter `ERROR` nodes
  silently yielding zero entries, no warning) is inferred from the absence of
  any non-Clojure warning path in `extract.py`/`scan.py`, not from a test
  assertion.
- **`.metrics/dry.edn` on an empty-candidates-but-nonzero-files run**: the code
  path is unconditional (`write_metrics` is always called once `files` is
  non-empty, `src/dryer/cli.py:249-251`), but no test directly asserts the file
  is written/overwritten when candidates is `[]` and files is non-empty (only
  `test_edn_stdout`, `tests/test_cli.py:93-98`, checks stdout for that case,
  not the file).
- **Descending-score ordering** is confirmed directly in source
  (`_duplicate_key` uses `-score`, `src/dryer/scan.py:54`) but no test asserts
  ordering across *differently scored* pairs in one run; existing tests either
  produce a single candidate or several candidates that all score `1.0`.
- No README claim was found to be contradicted by source or tests; the
  "missing or unreadable form … forms read before it still compared" claim is
  fully confirmed (see §7).
