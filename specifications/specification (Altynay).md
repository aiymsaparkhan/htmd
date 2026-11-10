HTMD Round-Trip Testing and Fuzzing Specification
=================================================

## 1. Purpose
This document defines the test strategy for `htmd`.

The goal is to verify that supported HTML structure and semantics survive a
round trip:

HTML -> Markdown -> HTML

We use deterministic regression tests plus seeded fuzzing. The guarantee is
intentionally narrow: it applies only to HTML-namespace content, body-fragment
inputs, `Faithful` mode, and constructs not explicitly excluded.

## 2. Scope and Guarantee
The round-trip guarantee applies only to:

* HTML-namespace elements
* body-fragment inputs
* `Faithful` mode
* constructs not listed in the known-loss allowlist

Everything else is out of scope and must not be treated as a passing case.

## 3. Testing objectives and failure classes
The suite enforces four properties:

1. Panic and crash safety
2. Deterministic output
3. Exact snapshot behavior for predefined inputs
4. Round-trip preservation of supported semantics

Each result is assigned exactly one category:

| Category                | Meaning                                        | Result         |
| ----------------------- | ---------------------------------------------- | -------------- |
| Expected Error          | Error is documented and expected               | Pass           |
| Unexpected Error        | Error not in the expected-error list           | Fail           |
| Panic                   | Caught with `catch_unwind`                     | Fail           |
| Timeout                 | Worker exceeds wall-clock limit                | Fail           |
| Resource Limit          | Worker exceeds memory limit                    | Fail           |
| Non-determinism         | Same input, different output across runs       | Fail           |
| Out of Scope            | Non-HTML namespace appears in generated input  | Fail           |
| Round-Trip Mismatch     | Normalized DOMs differ without allowlist match | Fail           |
| Known Loss              | Difference matches allowlist entry             | Pass, recorded |
| Stale Allowlist Entry   | Allowlist no longer matches real cases         | Warning        |
| Isolation Setup Failure | Resource-isolation self-test fails             | Abort          |

Expected-error list: starts empty. A case is added only when there is a
documented reason why `htmd` is expected to return `Err`, plus a predefined test
asserting that behavior. Generated valid HTML is not expected to fail.

## 4. Round-trip oracle
Raw string comparison is not valid: attribute order, entity spelling, void-tag
formatting, and parser-added wrappers can differ even when semantics match.

The oracle compares parsed, normalized DOMs.

Pipeline:

```
HTML_in -> html5ever -> DOM_src
   | 
   +-> htmd (Faithful, configured options) -> Markdown
                                                  |
                                                  v
                                     pulldown-cmark -> HTML_rt
                                                  |
                                                  v
                           html5ever (same mode as source) -> DOM_rt

Normalize(DOM_src) -- compare --> Normalize(DOM_rt)
```

Steps:

1. Parse `HTML_in` as a body fragment with `html5ever`.
2. Convert the raw HTML string with `htmd`.
3. Parse the generated Markdown with `pulldown-cmark` using the selected
   profile.
4. Parse the generated HTML back with `html5ever` in the same mode.
5. Normalize both DOMs and compare them.

Only body-fragment children are compared; generated `html`, `head`, and `body`
wrappers are ignored.

## 5. Markdown parser profiles
`pulldown-cmark` is pinned to the version in `Cargo.lock`, and every failure
includes the exact version.

| Profile | Options                                     | Input type                                           |
| ------- | ------------------------------------------- | ---------------------------------------------------- |
| `core`  | `Options::empty()`                          | Generated cases with no GFM-only constructs          |
| `gfm`   | tables + strikethrough + task lists enabled | Generated cases with at least one GFM-only construct |

GFM-only constructs are: tables, captioned table structures, `del`/`s`, and
task-list checkboxes.

A generated case belongs to exactly one profile:

* `core` generator disables GFM-only constructs
* `gfm` generator forces at least one GFM-only construct
* no case is run under both profiles

A predefined snapshot test may cover GFM under `core` mode for single-pass
Markdown checks, but it does not assert round-trip equality when the parser
profile cannot reproduce those constructs.

## 6. DOM model and normalization
The comparison operates on normalized DOM trees.

Node kinds:

* Element: namespace, lowercase tag name, attributes, ordered children
* Text: decoded text content
* Comment: comment text
* Ignored: doctype and processing instructions

Namespace rule:

* Only HTML namespace is supported.
* Generated input must not introduce SVG, MathML, or other foreign-content
  elements.
* If a non-HTML namespace appears during generated runs, classify the case as
  `Out of Scope`.

Normalization passes run in order on both trees:

1. Collapse runs of ASCII whitespace inside non-opaque text nodes to a single
   space.
2. Trim spaces around `br` boundaries.
3. Collapse spaces across adjacent inline text leaves.
4. Trim leading and trailing whitespace at inline-run edges.
5. Remove empty text nodes and merge adjacent text siblings.

Opaque elements (`pre`, `textarea`, `code`) are compared exactly; their internal
whitespace is not normalized.

Block vs. inline:

* Block elements include `div`, `p`, `table`, `ul`, `ol`, `li`, `pre`, headings,
  etc.
* Everything else is inline, including `a`, `em`, `strong`, `span`, `img`, `br`,
  and `input`
* Inline runs are maximal sequences of non-block children within a block
  container

Whitespace-only runs at block boundaries are removed, but inline boundary spaces
are preserved when they are semantically meaningful.

## 7. Comparison rules
* Attribute maps are compared as sets; order is irrelevant.
* All attributes are compared by default.
* Markdown-inexpressible attributes are handled only through an attribute
  exception registry.
* Core metadata must always match unless explicitly exempted: `href`, `src`,
  `alt`, `title`, `class` on `code`/`pre`, and `start` on `ol`.
* Comments and script/style nodes are compared like any other node unless
  explicitly excluded.
* Exclusions require explicit evidence and a dedicated predefined assertion.

## 8. Configuration and modes
`Faithful` mode is the fuzzing target. It aims to preserve supported semantics
across the round trip. It is not a claim of full lossless conversion.

`Pure` mode is for readable minimal Markdown and may discard data; it is checked
via direct Markdown assertions, not round-trip equality.

Default effective options used in tests:

```rust
let options = Options {
    heading_style: HeadingStyle::Atx,
    bullet_list_marker: BulletListMarker::Asterisk,
    code_block_style: CodeBlockStyle::Fenced,
    translation_mode: TranslationMode::Faithful,
    ..Options::default()
};
```

All other option fields inherit from `Options::default()` at the pinned `htmd`
version. The harness logs the full effective option list with every failure.

Known-loss allowlist:

* Starts empty
* Each entry records the HTML pattern, observed diff, source, and status
* Matches must be narrow and programmatic
* A mismatch is a failure unless it matches an allowlist rule

## 9. Predefined tests
Predefined tests run first and compare Markdown exactly.

Examples:

| Element        | HTML                                     | Expected Markdown             |
| -------------- | ---------------------------------------- | ----------------------------- |
| Heading        | `<h1>Title</h1>`                         | `# Title`                     |
| Bold           | `<strong>Text</strong>`                  | `**Text**`                    |
| Italic         | `<em>Text</em>`                          | `*Text*`                      |
| Link           | `<a href="https://example.com">Link</a>` | `[Link](https://example.com)` |
| Unordered list | `<ul><li>One</li></ul>`                  | `* One`                       |

These snapshots must be verified against the actual output before commit.

## 10. Execution order and seeds
Execution order:

1. Predefined tests
2. Seeded valid-HTML fuzzing (`core`, then `gfm`)
3. Seeded malformed-HTML fuzzing

Default CI seed plan:

* Master seeds: `1, 2, 3, 42, 20260301`
* 1,000 cases per class/profile per seed
* 15,000 total CI cases for the default plan
* Scheduled runs may use a fresh entropy-derived seed; the seed is logged and
  stored with every artifact

Each generated case is run with:

* panic guard via `catch_unwind`
* wall-clock timeout (default 5s)
* memory limit (default 512 MiB)
* determinism check by converting the same case twice

## 11. Worker model and resource limits
Fuzzing runs in worker subprocesses because infinite loops and OOM aborts cannot
be caught as ordinary panics.

The parent process:

* sends a case to the worker
* waits for the result
* kills the worker if it exceeds the timeout or memory limit
* records the failure as `Timeout` or `Resource Limit`

Memory enforcement depends on platform:

* Linux with cgroups: hard limit via `memory.max`
* Linux without cgroups: `RLIMIT_AS` plus polling `/proc/<pid>/status`
* Windows: Job Object memory limit
* macOS/other: resident-size polling only (soft limit)

Before fuzzing starts, a self-test verifies that both kinds of worker kill
happen correctly. If the isolation test fails, the run aborts with `Isolation
Setup Failure`.

## 12. Failure artifacts and minimization
Each failure writes a JSON artifact containing:

* master seed and case index
* input class and parser profile
* original HTML source
* effective options
* versions of `htmd`, `html5ever`, `pulldown-cmark`, and `rand_chacha`
* generated Markdown and round-trip HTML
* normalized DOMs and the first differing path
* failure category and allowlist match, if any
* active memory-limit mode and limits

Failure reduction flow:

1. Reproduce the failure from the stored input.

2. Reduce the tree by removing nodes, hoisting children, and shortening
   text/attributes.

3. Keep only reductions that preserve the same failure category.

4. Save the minimal reproducer as a regression test.

5. Triage:

   * oracle bug -> fix normalization and add a self-test
   * intentional loss -> add allowlist entry
   * conversion defect -> fix `htmd` and update design docs
## 6. Limitations
* Random generation cannot cover every structure.
* The preservation check is only for supported semantics, HTML-namespace
  content, and the active parser profile.
* Registry entries must stay narrow and evidence-backed.
* Memory-limit claims are valid only on platforms with real enforcement.

## 14. Conclusion
This spec combines exact snapshot tests with a seeded, replayable round-trip
fuzzer and an explicit oracle. That separation makes it possible to distinguish:

* real conversion bugs
* parser/normalization effects
* intentional known losses

The result is a focused but strict validation strategy for `htmd`.
