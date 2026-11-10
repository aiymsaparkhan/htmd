```
prompt: Review specification (html_to_markdown_testing.md) for the "round-trip" fuzzer to catch inconsistencies when converting from html to markdown and back to html via the htmd package. What is missing? What is unclear? Give us a guide how to fix such issues.
```

Review 1: HTML-to-Markdown Round-Trip Fuzzer Specification
----------------------------------------------------------

Summary
-------

[`html_to_markdown_testing.md`](./html_to_markdown_testing.md) is a useful
starting point for one-way tests, but it does not yet specify a round-trip
fuzzer. It sends HTML to HTMD and checks the Markdown; there is no
Markdown-to-HTML step or rule for comparing the result with the input.

The repository already has a round-trip helper and focused round-trip assertions
in [`tests/common/mod.rs`](../tests/common/mod.rs) and
[`tests/basic_tests.rs`](../tests/basic_tests.rs). The specification should
build on that precedent while defining a scalable, normalization-aware oracle
for generated cases.

What is missing
---------------

1. <strong>A round-trip definition and oracle.</strong> Specify the full
   pipeline—HTML → HTMD → Markdown parser → HTML—and define how to determine
   whether the output preserves the input. Raw HTML string comparison will
   report harmless differences such as attribute order, entity spelling, and
   parser-added elements.
2. <strong>A defined translation mode and configuration.</strong> HTMD defaults
   to `Pure`, which may discard information. The existing round-trip helper uses
   `Faithful` and `pulldown-cmark`. State which mode, options, and Markdown
   parser the fuzzer uses, and whether lossy `Pure` results are tested against
   separate expectations.
3. <strong>A clear failure contract.</strong> HTMD conversion returns a
   `Result`, so distinguish conversion errors from panics and round-trip
   mismatches. “Text should not be unexpectedly lost” needs a measurable
   definition.
4. <strong>Reproducible generation.</strong> Define PRNG and seed handling,
   per-run test count, size and depth limits, and saved metadata. A failure
   report should include round-tripped HTML, settings, and parser/generator
   versions—not only the original HTML and Markdown.
5. <strong>A coverage plan.</strong> The listed examples omit important cases
   such as ordered lists, tables, code, line breaks, thematic breaks, and nested
   block/inline contexts. Cover attributes and text edge cases as well as tag
   names, and distinguish supported elements from intentional fallbacks.
6. <strong>A strategy for malformed input.</strong> HTML parsers repair
   malformed markup. Decide whether to generate valid HTML separately from
   malformed cases, and compare the parsed input tree—not the raw malformed
   string—with the parsed round-trip output.

What is unclear or inconsistent
-------------------------------

* The unordered-list example expects `- One`, but HTMD’s default bullet marker
  is `*` ([`src/options.rs`](../src/options.rs)). State the options for each
  expected output, or adjust the example to match the defaults.
* “Same input and settings” does not say which settings are fixed, and
  “different seeds” does not explain how to replay a failure.
* “Valid UTF-8” is not a useful check on this API: it accepts and returns Rust
  strings, which are UTF-8 by construction.
* The specification notes that some HTML has multiple valid Markdown forms, but
  does not say how the fuzzer handles those differences. Markdown snapshots and
  round-trip preservation are separate properties and need separate assertions.

Guide to fixing issues
----------------------

1. <strong>Write the contract first.</strong> Choose `Faithful` for
   round-trip-preservation tests, document all options, and identify known
   intentional losses. Keep `Pure` tests separate with appropriate expectations.
2. <strong>Use two kinds of oracle.</strong> Retain exact Markdown expectations
   for selected examples. For generated HTML, parse both the original and
   returned HTML with the same HTML parser and compare normalized DOMs. Define
   which differences are ignorable and which matter—for example, whether
   attributes and text must be preserved.
3. <strong>Constrain generation.</strong> Generate structurally valid HTML with
   context-appropriate nesting, then run malformed HTML as a separately labeled
   input class. Bound document size and nesting depth.
4. <strong>Make every failure replayable.</strong> Record the seed, generated
   case, options, Markdown, returned HTML, and failure category. Minimize
   failing inputs and add them as deterministic regression tests.
5. <strong>Triage before changing the converter.</strong> Reproduce the case;
   check whether the mismatch is parser normalization, an intentional
   mode-specific loss, or a genuine conversion defect. For a defect, add a
   focused regression test, update the relevant conversion behavior and design
   documentation, then run the repository’s build, Clippy, and test commands.

Specific example correction
---------------------------

Either set `bullet_list_marker` to `Dash` in the example’s stated options or
change its expected Markdown to use the default `*` marker. Expected output must
be tied to explicit options so a formatting preference is not mistaken for a
conversion bug.

---

```
prompt: Review the updated specification in specification (Altynay).md, find issues to resolve. Write your feedback in LLM review (Altynay).md appending from the last text line (do not overwrite old review). You can add headings that shows each review's versions
```

Review 2 — Updated specification (`specification (Altynay).md`)
---------------------------------------------------------------

### Overall assessment

The update addresses several gaps from the first review: it describes a
round-trip pipeline, selects `Faithful` mode, and proposes comparing parsed DOMs
rather than raw HTML strings. The remaining issues are mostly in the oracle and
its implementation contract. In particular, the current normalization rules can
erase differences the fuzzer should detect, and the proposed options example is
not a complete `Options` value.

### Issues to resolve

1. <strong>Normalization can hide real text regressions.</strong> “Strip leading
   and trailing whitespaces” is ambiguous about whether this means per text node
   or per document. Trimming each text node can remove meaningful spaces between
   adjacent inline nodes (for example, `a <em>b</em>`), while collapsing
   whitespace everywhere can alter preformatted or raw-text content. Define
   context-aware normalization: preserve inline boundary spaces, compare block
   text according to an explicit whitespace policy, and preserve exact text in
   contexts such as `<pre>`. Add small cases that prove the normalizer catches
   lost/added spaces.
2. <strong>Attribute comparison does not yet test the stated metadata
   property.</strong> Sorting attributes handles order, but “ignore synthetic
   attributes injected strictly by parsers” gives no list or rule for
   identifying them. Specify the exact ignored attributes (ideally none unless
   demonstrated necessary), and compare meaningful attributes such as `href`,
   `src`, `alt`, `title`, and relevant classes. Where an attribute is
   intentionally not representable, document a narrow, element-specific
   exception instead of globally ignoring it.
3. <strong>Ignored elements conflict with `Faithful` mode and can mask
   failures.</strong> The rules exclude `<script>`, `<style>`, and comments, but
   do not establish that the chosen configuration intentionally drops them.
   Faithful mode aims for near-preservation, not guaranteed losslessness; the
   library documentation describes the result as “almost” identical, and the
   translation design records known round-trip losses. Do not ignore these nodes
   by default. Either compare them or explicitly list verified, intentional
   exclusions and test those exclusions separately.
4. <strong>The mode guarantee is too strong.</strong> Calling `Faithful`
   “lossless” conflicts with the library’s documented “almost identical”
   behavior and known limitations. Replace that claim with a goal such as
   “preserve supported semantics,” then define an explicit allowlist of known
   losses. Otherwise, known defects may be dismissed as expected behavior.
5. <strong>The Markdown parser configuration is underspecified and may not match
   the test harness.</strong> “Standard extensions: GFM tables, strikethrough,
   task lists” should identify the exact `pulldown-cmark` options and version.
   The current shared round-trip helper uses `CommonMarkOptions::empty()`
   ([`tests/common/mod.rs`](../tests/common/mod.rs)); enabling extensions
   changes how Markdown is interpreted and can create results that existing
   tests do not reproduce. Choose and document one parser configuration, or test
   extension-enabled cases as a separate profile.
6. <strong>The pipeline omits an explicit HTML parse of the rendered
   output.</strong> `pulldown-cmark` renders Markdown to an HTML string; it does
   not produce the `DOM_rt` in the diagram. Add the HTML parse step for that
   string, specify the same parser and document/fragment mode for both sides,
   and define whether comparisons use the full document or only body children.
   This matters because HTML parsers can synthesize `html`, `head`, and `body`
   nodes.
7. <strong>The `Options` example is incomplete Rust.</strong> The struct has
   additional required fields, including `hr_style`, `br_style`, `link_style`,
   `link_reference_style`, `ul_bullet_spacing`, `ol_number_spacing`, and
   `preformatted_code` ([`src/options.rs`](../src/options.rs)). As written, the
   snippet cannot construct `Options`. Show a complete configuration or a
   compilable struct update (`..Options::default()`), and state which defaults
   remain in effect. This also resolves the claim that all parameters are
   explicit while only four are listed.
8. <strong>Failure categories need expected/unexpected criteria.</strong>
   `Result::Err` is called an “Expected failure” for unsupported operations or
   invalid states, but the spec does not define which inputs should return
   errors or how such errors are distinguished from bugs. Define the expected
   error cases; treat any other `Err` as a failure. Also, a Rust panic can
   sometimes be caught, but memory exhaustion/abort and infinite loops are not
   ordinary panics: specify a timeout and process-isolation strategy if those
   are in scope.
9. <strong>The fuzzer’s generation and replay procedure is still
   absent.</strong> The document does not yet say how inputs are generated,
   which tags/attributes/text are in scope, how depth and input size are
   bounded, how seeds are stored/replayed, or how failures are minimized and
   promoted to regression tests. Add a generation section and an
   execution/failure-artifact section. Keep valid structured generation separate
   from malformed-HTML fuzzing, since `html5ever` repairs malformed source
   before conversion.
10. <strong>“Significant divergence” and exclusions need a deterministic
    verdict.</strong> Define the DOM representation being compared (node types,
    tag names, text, comments, namespaces, attributes, and child order) and
    precisely which differences are allowed. Without this, different
    implementations can classify the same case differently, and broad exclusions
    can turn the oracle into a false pass.

### Suggested resolution order

1. Correct the round-trip pipeline and choose the exact Markdown and HTML parser
   configurations.
2. Define a normalization/comparison contract with tests for inline spaces,
   preformatted text, attributes, comments, and parser-added document wrappers.
3. Replace “lossless” with a supported-preservation contract and enumerate
   intentional losses narrowly.
4. Make the options example compilable and fully state the effective settings.
5. Add bounded generation, seed replay, timeouts, failure artifacts, shrinking,
   and regression-test promotion rules.

The existing design record and round-trip tests are useful sources for known
behavior and exceptions; align the new oracle with them rather than allowing
broad normalization to silently weaken the checks.

---

```
prompt: Review the updated specification in specification (Altynay).md, find issues to resolve. Write your feedback in LLM review (Altynay).md appending from the last text line (do not overwrite old review).
```

Review 3 — v3 (`specification (Altynay).md`)
--------------------------------------------

### Issues still to resolve

1. <strong>The parser-profile rules contradict the table and strikethrough
   cases.</strong> Section 3.2 says `core` is used for all fuzzing, then says
   `gfm` is run separately for cases using tables or `<del>`/`<s>`. Those
   constructs will therefore also be checked under `Options::empty()`, which
   does not enable the corresponding Markdown extensions; the resulting DOM
   mismatch can be a parser-profile artifact rather than an `htmd` defect.
   Specify whether such inputs are excluded from `core` or run in both profiles
   with different verdicts. Also, checkbox inputs are mentioned as a GFM trigger
   but are not in the element tiers or attribute generator, so either add their
   generation and expected handling or remove that condition.
2. <strong>The compared DOM's namespace and supported-input scope are
   inconsistent.</strong> Section 3.3 says the element namespace is “HTML only,”
   while Tier 3 includes “other elements” and the HTML parser can produce
   foreign-content namespaces (for example SVG). Define whether those nodes are
   generated and supported; if they are, compare namespace URIs, and if not,
   exclude them explicitly from the generator and state the boundary of the
   preservation guarantee.
3. <strong>Whitespace normalization still lacks an executable block
   model.</strong> “Block-level siblings,” “block container,” and “first/last
   text of a block” are not defined, nor is the traversal/order of the
   normalization steps. Implementations could consequently discard different
   text nodes while passing the listed self-tests. Specify the exact
   block-element set and whether “first/last text” means direct children or
   descendant text (including text nested in inline elements); add self-tests
   for whitespace-only text inside inline wrappers at block edges and around
   mixed inline/block children.
4. <strong>The seed plan is not yet fixed enough for reproducible CI
   runs.</strong> The specification requires a fixed seed list and case count
   but provides neither values nor defaults, and “for example ChaCha8” leaves
   the generator algorithm and implementation version open. Pin the PRNG
   implementation and version, publish the CI seed list and cases per seed, and
   define how a seed and case index map to a generated case. Keep the scheduled
   random seed, but ensure its artifact records enough information to replay it
   independently of later generator changes.
5. <strong>The stated resource limits have no enforcement contract.</strong>
   Process isolation is the right boundary for hangs and aborts, but the spec
   does not say how the 512 MiB limit is enforced, whether it is per worker or
   includes descendants, or what happens on platforms without an equivalent
   process memory limit. Identify the supported enforcement mechanism and define
   the fallback (or fail the resource-isolation test setup) rather than treating
   a worker timeout as proof that memory use was bounded.
6. <strong>The revision note points to a missing section.</strong> The
   introduction says Section 9 maps the ten Review 2 issues, but the document
   ends at Section 8. Add the mapping section or correct the revision note so
   readers can verify the claimed resolutions.

---

Review 4 — latest specification
-------------------------------

### What has improved

* The guarantee is now explicitly limited to HTML-namespace body fragments in
  `Faithful` mode, and out-of-scope generated content is no longer counted as a
  successful preservation case.
* `core` and `gfm` inputs are separated, with GFM-only constructs defined and
  no ambiguous cross-profile round-trip verdict.
* The seed list, cases-per-run target, failure artifact fields, and platform
  resource-limit approaches are more concrete than in the previous revision.
* The missing revision-mapping claim has been removed. The options example
  remains compilable through `Options::default()`, and the non-lossless
  `Faithful` guarantee is appropriately qualified.

### Issues to resolve

1. <strong>The generator is no longer specified sufficiently to implement or
   assess coverage.</strong> The latest version omits the prior grammar,
   element tiers, attribute/text domains, size/depth limits, and malformed-input
   construction rules. “Seeded fuzzing” alone does not determine what inputs
   are generated. Restore a bounded generation contract, including which
   supported constructs and combinations are covered and how malformed inputs
   are produced.
2. <strong>The stated 15,000-case CI total is not fully auditable.</strong>
   Five master seeds times 1,000 cases yields 5,000 cases per class/profile
   combination, so the total assumes exactly three combinations. Name those
   combinations explicitly. In particular, the malformed-HTML run is not
   assigned a Markdown parser profile or a case count, although the execution
   order includes it.
3. <strong>Seed replay is still underspecified.</strong> Fixed master seeds and
   `rand_chacha` version logging are progress, but the specification does not
   pin the PRNG algorithm/version as a requirement or define how a seed and case
   index deterministically produce a case. Failure artifacts should record the
   generator version (and any derived seed/state needed), so a failure remains
   reproducible after generator changes; storing the original input should
   remain the definitive replay path.
4. <strong>The normalization rules do not yet define a deterministic
   algorithm.</strong> “Trim spaces around `br` boundaries,” “collapse spaces
   across adjacent inline text leaves,” and “trim leading and trailing
   whitespace at inline-run edges” can erase meaningful whitespace depending
   on traversal and node boundaries. Define precisely what is trimmed and in
   what order, how inline descendants participate, and how opaque subtrees skip
   normalization. Add required self-tests for inline boundary spaces, spaces
   adjacent to `<br>`, and whitespace around inline wrappers at block edges.
   The previous revision's useful normalizer self-test table should be retained
   or replaced by an equivalent explicit test contract.
5. <strong>Comments and raw-content elements are treated as preserved without
   establishing that they are representable.</strong> The comparison requires
   comments and `<script>`/`<style>` nodes to match unless excluded, but the
   current specification does not say how the Markdown pipeline can preserve
   them or classify their known losses. Clarify whether these constructs are
   outside the supported guarantee, are expected to fail until implemented, or
   may enter a narrow, evidenced allowlist. Do not have generated cases silently
   turn such mismatches into unexpected baseline failures.
6. <strong>The resource-limit result needs to distinguish hard from soft
   enforcement.</strong> On macOS/other, resident-size polling is explicitly a
   soft limit, yet the result categories call exceeding the memory limit a
   `Resource Limit` failure and the parent is said to kill the worker when it
   exceeds that limit. State how delayed polling/overshoot is reported and
   whether runs on those platforms can claim the same resource-safety property
   as hard-limited platforms.
7. <strong>Section numbering is inconsistent.</strong> The document labels
   “Limitations” as Section 6 after Section 12, then jumps to Section 14 for
   “Conclusion.” Renumber these headings sequentially and update any references
   to them.
