# Parser Feasibility for Target Languages

Status: research note (Wayfinder ticket 03). Does **not** select a final parser architecture — it collects evidence for that decision.

Scope: Clojure, Java, Go, TypeScript, Rust, Python, C, C++, C#, Dart, JavaScript — candidate parsers accessible from a Rust CLI, with emphasis on maintained Tree-sitter grammars. Claims below were verified against GitHub API (`pushed_at`, stars, license, archived flag), crates.io API (`max_version`, published date, `repository` field), and raw README/grammar.js source, fetched 2026-10-30.

## Summary table

| Language | Grammar repo | Last push | License | Rust crate (version / published) | Verdict |
|---|---|---|---|---|---|
| Java | [tree-sitter/tree-sitter-java](https://github.com/tree-sitter/tree-sitter-java) | 2026-09-13, 278★ | MIT | `tree-sitter-java` 0.23.5 (2024-12-21) | Viable |
| Go | [tree-sitter/tree-sitter-go](https://github.com/tree-sitter/tree-sitter-go) | 2026-09-13, 422★ | MIT | `tree-sitter-go` 0.25.0 (2025-08-29) | Viable |
| TypeScript | [tree-sitter/tree-sitter-typescript](https://github.com/tree-sitter/tree-sitter-typescript) | 2026-09-17, 537★ | MIT | `tree-sitter-typescript` 0.23.2 (2024-11-11) | Viable |
| Rust | [tree-sitter/tree-sitter-rust](https://github.com/tree-sitter/tree-sitter-rust) | 2026-09-16, 533★ | MIT | `tree-sitter-rust` 0.24.2 (2026-03-27) | Viable; `syn` is a native alternative |
| Python | [tree-sitter/tree-sitter-python](https://github.com/tree-sitter/tree-sitter-python) | 2026-09-13, 572★ | MIT | `tree-sitter-python` 0.25.0 (2025-09-11) | Viable |
| C | [tree-sitter/tree-sitter-c](https://github.com/tree-sitter/tree-sitter-c) | 2026-09-16, 393★ | MIT | `tree-sitter-c` 0.24.2 (2026-04-22) | Viable, scoped to C99-derived grammar |
| C++ | [tree-sitter/tree-sitter-cpp](https://github.com/tree-sitter/tree-sitter-cpp) | 2026-09-17, 463★ | MIT | `tree-sitter-cpp` 0.23.4 (2024-11-11) | Viable |
| C# | [tree-sitter/tree-sitter-c-sharp](https://github.com/tree-sitter/tree-sitter-c-sharp) | 2026-09-29, 322★ | MIT | `tree-sitter-c-sharp` 0.23.5 (2026-04-14) | Viable |
| Dart | Two diverging forks — see below | UserNobody14: 2026-10-18, 107★; nielsenko: 2026-10-25, 10★ | Both MIT | `tree-sitter-dart` 0.3.0 (2026-10-25), wraps **nielsenko**, not UserNobody14 | Viable, but fork choice needs an explicit decision |
| JavaScript | [tree-sitter/tree-sitter-javascript](https://github.com/tree-sitter/tree-sitter-javascript) | 2026-09-17, 497★ | MIT | `tree-sitter-javascript` 0.25.0 (2025-09-01) | Viable |
| Clojure | [sogaiu/tree-sitter-clojure](https://github.com/sogaiu/tree-sitter-clojure) (canonical grammar) | 2025-08-26, 190★ | CC0-1.0 | Only published crate is `tree-sitter-clojure` 0.1.0 (2025-07-07), a third-party wrapper, **not** sogaiu's own release | Grammar solid; **Rust crate supply chain is weak — needs a different approach** |

## Per-language notes

**Java** — Official `tree-sitter` org repo, pushed 2026-09-13, 278 stars, MIT (`api.github.com/repos/tree-sitter/tree-sitter-java`). Crate `tree-sitter-java` 0.23.5 published 2024-12-21 (`crates.io/api/v1/crates/tree-sitter-java`). `grammar.js` defines `record_declaration`, `sealed`/`non-sealed`/`permits`, and `record_pattern`, so modern Java (records, sealed classes, pattern matching) is covered.

**Go** — Official org repo, pushed 2026-09-13, 422 stars. Crate `tree-sitter-go` 0.25.0, published 2025-08-29. README documents no caveats (`raw.githubusercontent.com/tree-sitter/tree-sitter-go/master/README.md`).

**TypeScript** — Official org repo, pushed 2026-09-17, 537 stars. One crate ships both `typescript` and `tsx` grammars; README notes the `tsx` parser also handles Flow-annotated JS (`raw.githubusercontent.com/tree-sitter/tree-sitter-typescript/master/README.md`). Crate `tree-sitter-typescript` 0.23.2, published 2024-11-11.

**Rust** — Official org repo, pushed 2026-09-16, 533 stars. Crate `tree-sitter-rust` 0.24.2, published 2026-03-27. It is a CST over source text: macro invocations are parsed as token trees, not macro-expanded, which matters if structural comparison needs expanded code. Native alternative `syn` (`github.com/dtolnay/syn`) is far more heavily adopted (pushed 2026-10-30, 3,437 stars, Apache-2.0, crate `syn` 3.0.9, ~2.85B cumulative crates.io downloads) but only parses syntactically valid, non-error-tolerant Rust.

**Python** — Official org repo, pushed 2026-09-13, 572 stars. Crate `tree-sitter-python` 0.25.0, published 2025-09-11, no documented limitations. Native alternative `rustpython-parser` (`github.com/RustPython/Parser`, MIT, 117 stars, pushed 2026-09-01) has its crate last published 2024-08-06 — over a year stale relative to tree-sitter-python, so it is the weaker option today.

**C** — Official org repo, pushed 2026-09-16, 393 stars. README states the grammar was "adapted from a C99 grammar" — a documented scope limitation if C11/C17/C23 constructs matter structurally; grammar.js was not independently inspected for specific modern-C coverage. Crate `tree-sitter-c` 0.24.2, published 2026-04-22.

**C++** — Official org repo, pushed 2026-09-17, 463 stars, no documented caveats in README. Crate `tree-sitter-cpp` 0.23.4, published 2024-11-11 — the stalest of the C-family crates; C++20/23 feature-level coverage (modules, concepts) not independently verified.

**C#** — Still maintained directly under the `tree-sitter` org (no ownership transfer needed), pushed 2026-09-29, 322 stars. README gives the most explicit version-coverage statement of all 11 grammars: "Comprehensive support[ for] C# 1 through 14.0," with two named exceptions — contextual keywords (`async`/`var`/`await`) not usable as identifiers in every valid position, and file-based-app preprocessor directives (`#:property`, `#:package`, `#:sdk`, `#:project`) not yet recognized (`raw.githubusercontent.com/tree-sitter/tree-sitter-c-sharp/master/README.md`). Crate `tree-sitter-c-sharp` 0.23.5, published 2026-04-14.

**Dart — flag.** Two actively-diverged community grammars exist, neither under the `tree-sitter` org: `UserNobody14/tree-sitter-dart` (107★, pushed 2026-10-18, MIT) is the historically best-known one but its README has no language-version coverage statement. The published crates.io crate `tree-sitter-dart` 0.3.0 (2026-10-25) instead wraps **`nielsenko/tree-sitter-dart`** (10★, pushed 2026-10-25) — confirmed via the crate's `repository` field (`crates.io/api/v1/crates/tree-sitter-dart`). The nielsenko README is explicit about coverage: it "covers Dart through 3.13 ... records, patterns, class modifiers, extension types, null-aware elements, dot shorthands, digit separators, private named parameters (3.12) and primary constructors (3.13)," using the Dart analyzer as its reference implementation. The two forks are different codebases with different coverage guarantees — which one is intended needs an explicit decision before depending on the published crate.

**JavaScript** — Official org repo, pushed 2026-09-17, 497 stars. README targets "the latest version of the [ECMAScript] spec" (ECMA-262 2025) plus JSX, and candidly notes "some very new features may not be supported yet" (`raw.githubusercontent.com/tree-sitter/tree-sitter-javascript/master/README.md`). Crate `tree-sitter-javascript` 0.25.0, published 2025-09-01. Native alternatives `oxc_parser` (crate 0.156.0, published 2026-10-27) and `swc_ecma_parser` (crate 47.0.0, published 2026-10-30) publish far more frequently and produce typed ASTs rather than a generic CST — worth weighing against tree-sitter's cross-language grammar uniformity.

**Clojure — flag, strongest concern.** The canonical grammar `sogaiu/tree-sitter-clojure` is well-maintained: pushed 2025-08-26, 190 stars, CC0-1.0, with a documented `doc/limits.md` listing known non-support (certain template/DSL files, reader-macro-modifying code such as clojure-scribble/piplin/chiara, legacy `^`-as-metadata shorthand from Clojure 1.0.x, ClojureCLR pipe-symbol syntax, and NUL-byte files) (`raw.githubusercontent.com/sogaiu/tree-sitter-clojure/master/doc/limits.md`). However, **sogaiu does not publish a Rust crate**. The only crate named `tree-sitter-clojure` on crates.io is version 0.1.0, published once on 2025-07-07, whose own README states it "exists to maintain Clojure support for [Harper]" (an unrelated spell-check tool) — a side-effect packaging from a 0-star repo (`jerielverissimo/tree-sitter-clojure`), not the grammar maintainer's own release process (`api.github.com/repos/jerielverissimo/tree-sitter-clojure`; `raw.githubusercontent.com/jerielverissimo/tree-sitter-clojure/master/README.md`). This is a materially weaker Rust-ecosystem story than every other language evaluated.

## Languages likely needing a different approach

- **Clojure**: the grammar is solid but its only Rust crate is an unvetted, single-publish third-party wrapper. Treat the crate as unreviewed, or vendor-build `sogaiu/tree-sitter-clojure` directly via `tree-sitter-cli`/a build script rather than depending on the published crate.
- **Dart**: two non-canonical, diverging grammars; the published crate silently tracks the less-starred fork (nielsenko) rather than the better-known one (UserNobody14). Requires an explicit fork decision, not an assumption from crate name alone.
- **Rust**: tree-sitter-rust is viable for CST extraction, but if macro-expanded semantics matter, `syn` (optionally plus a macro-expansion step) is the far more heavily adopted native alternative, at the cost of requiring syntactically valid, error-intolerant input.
- **Python**: `rustpython-parser` is a native alternative but its crate has not been published since 2024-08-06, a longer gap than any tree-sitter grammar crate checked here.
- **JavaScript/TypeScript**: `oxc_parser` and `swc_ecma_parser` publish far more frequently than `tree-sitter-javascript`/`tree-sitter-typescript` and emit typed ASTs — worth weighing against tree-sitter's cross-language uniformity, though neither was checked for explicit spec-coverage documentation to the depth of the C# README.

## Supporting infrastructure

The core `tree-sitter` Rust crate is at 0.27.1 (published 2026-10-08, MIT, `rust_version` 1.90, per `crates.io/api/v1/crates/tree-sitter`); upstream repo tagged `v0.27.1` on 2026-10-30. ABI/version-pinning compatibility across grammar-crate major/minor versions was not independently verified against the `tree-sitter` crate's CHANGELOG in this pass — recommended as a follow-up before locking grammar crate versions.

## Gaps and uncertainties

- C/C++ `grammar.js` files were not inspected for specific C11/C17/C23 or C++20/23 feature coverage (e.g., `_BitInt`, concepts, modules) — only README self-descriptions were checked.
- `UserNobody14/tree-sitter-dart` was not checked for a separate, unpublished Rust binding crate of its own.
- `oxc_parser`/`swc_ecma_parser` were checked only for crate publish freshness, not for README-documented spec-coverage claims.
- The missing `license` field on `jerielverissimo/tree-sitter-clojure` via the GitHub API was not independently re-confirmed by checking for a LICENSE file in the repo tree (a missing API field can reflect a non-SPDX license rather than true absence).
- `tree-sitter` ABI-compatibility history across grammar crate versions was not traced through the changelog; flagged above as a follow-up.
