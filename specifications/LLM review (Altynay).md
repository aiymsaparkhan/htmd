```
prompt: Review specification (html_to_markdown_testing.md) for the "round-trip" fuzzer to catch inconsistencies when converting from html to markdown and back to html via the htmd package. What is missing? What is unclear? Give us a guide how to fix such issues.
```


# Review: HTML-to-Markdown Round-Trip Fuzzer Specification

## Summary

[`html_to_markdown_testing.md`](./html_to_markdown_testing.md) is a useful starting point for one-way tests, but it does not yet specify a round-trip fuzzer. It sends HTML to HTMD and checks the Markdown; there is no Markdown-to-HTML step or rule for comparing the result with the input.

The repository already has a round-trip helper and focused round-trip assertions in [`tests/common/mod.rs`](../tests/common/mod.rs) and [`tests/basic_tests.rs`](../tests/basic_tests.rs). The specification should build on that precedent while defining a scalable, normalization-aware oracle for generated cases.

## What is missing

1. **A round-trip definition and oracle.** Specify the full pipeline—HTML → HTMD → Markdown parser → HTML—and define how to determine whether the output preserves the input. Raw HTML string comparison will report harmless differences such as attribute order, entity spelling, and parser-added elements.
2. **A defined translation mode and configuration.** HTMD defaults to `Pure`, which may discard information. The existing round-trip helper uses `Faithful` and `pulldown-cmark`. State which mode, options, and Markdown parser the fuzzer uses, and whether lossy `Pure` results are tested against separate expectations.
3. **A clear failure contract.** HTMD conversion returns a `Result`, so distinguish conversion errors from panics and round-trip mismatches. “Text should not be unexpectedly lost” needs a measurable definition.
4. **Reproducible generation.** Define PRNG and seed handling, per-run test count, size and depth limits, and saved metadata. A failure report should include round-tripped HTML, settings, and parser/generator versions—not only the original HTML and Markdown.
5. **A coverage plan.** The listed examples omit important cases such as ordered lists, tables, code, line breaks, thematic breaks, and nested block/inline contexts. Cover attributes and text edge cases as well as tag names, and distinguish supported elements from intentional fallbacks.
6. **A strategy for malformed input.** HTML parsers repair malformed markup. Decide whether to generate valid HTML separately from malformed cases, and compare the parsed input tree—not the raw malformed string—with the parsed round-trip output.

## What is unclear or inconsistent

- The unordered-list example expects `- One`, but HTMD’s default bullet marker is `*` ([`src/options.rs`](../src/options.rs)). State the options for each expected output, or adjust the example to match the defaults.
- “Same input and settings” does not say which settings are fixed, and “different seeds” does not explain how to replay a failure.
- “Valid UTF-8” is not a useful check on this API: it accepts and returns Rust strings, which are UTF-8 by construction.
- The specification notes that some HTML has multiple valid Markdown forms, but does not say how the fuzzer handles those differences. Markdown snapshots and round-trip preservation are separate properties and need separate assertions.

## Guide to fixing issues

1. **Write the contract first.** Choose `Faithful` for round-trip-preservation tests, document all options, and identify known intentional losses. Keep `Pure` tests separate with appropriate expectations.
2. **Use two kinds of oracle.** Retain exact Markdown expectations for selected examples. For generated HTML, parse both the original and returned HTML with the same HTML parser and compare normalized DOMs. Define which differences are ignorable and which matter—for example, whether attributes and text must be preserved.
3. **Constrain generation.** Generate structurally valid HTML with context-appropriate nesting, then run malformed HTML as a separately labeled input class. Bound document size and nesting depth.
4. **Make every failure replayable.** Record the seed, generated case, options, Markdown, returned HTML, and failure category. Minimize failing inputs and add them as deterministic regression tests.
5. **Triage before changing the converter.** Reproduce the case; check whether the mismatch is parser normalization, an intentional mode-specific loss, or a genuine conversion defect. For a defect, add a focused regression test, update the relevant conversion behavior and design documentation, then run the repository’s build, Clippy, and test commands.

## Specific example correction

Either set `bullet_list_marker` to `Dash` in the example’s stated options or change its expected Markdown to use the default `*` marker. Expected output must be tied to explicit options so a formatting preference is not mistaken for a conversion bug.

---
```
prompt: Review the updated specification in specification (Altynay).md, find issues to resolve. Write your feedback in LLM review (Altynay).md appending from the last text line (do not overwrite old review). You can add headings that shows each review's versions
```

## Review 2 — Updated specification (`specification (Altynay).md`)

### Overall assessment

The update addresses several gaps from the first review: it describes a round-trip pipeline, selects `Faithful` mode, and proposes comparing parsed DOMs rather than raw HTML strings. The remaining issues are mostly in the oracle and its implementation contract. In particular, the current normalization rules can erase differences the fuzzer should detect, and the proposed options example is not a complete `Options` value.

### Issues to resolve

1. **Normalization can hide real text regressions.** “Strip leading and trailing whitespaces” is ambiguous about whether this means per text node or per document. Trimming each text node can remove meaningful spaces between adjacent inline nodes (for example, `a <em>b</em>`), while collapsing whitespace everywhere can alter preformatted or raw-text content. Define context-aware normalization: preserve inline boundary spaces, compare block text according to an explicit whitespace policy, and preserve exact text in contexts such as `<pre>`. Add small cases that prove the normalizer catches lost/added spaces.
2. **Attribute comparison does not yet test the stated metadata property.** Sorting attributes handles order, but “ignore synthetic attributes injected strictly by parsers” gives no list or rule for identifying them. Specify the exact ignored attributes (ideally none unless demonstrated necessary), and compare meaningful attributes such as `href`, `src`, `alt`, `title`, and relevant classes. Where an attribute is intentionally not representable, document a narrow, element-specific exception instead of globally ignoring it.
3. **Ignored elements conflict with `Faithful` mode and can mask failures.** The rules exclude `<script>`, `<style>`, and comments, but do not establish that the chosen configuration intentionally drops them. Faithful mode aims for near-preservation, not guaranteed losslessness; the library documentation describes the result as “almost” identical, and the translation design records known round-trip losses. Do not ignore these nodes by default. Either compare them or explicitly list verified, intentional exclusions and test those exclusions separately.
4. **The mode guarantee is too strong.** Calling `Faithful` “lossless” conflicts with the library’s documented “almost identical” behavior and known limitations. Replace that claim with a goal such as “preserve supported semantics,” then define an explicit allowlist of known losses. Otherwise, known defects may be dismissed as expected behavior.
5. **The Markdown parser configuration is underspecified and may not match the test harness.** “Standard extensions: GFM tables, strikethrough, task lists” should identify the exact `pulldown-cmark` options and version. The current shared round-trip helper uses `CommonMarkOptions::empty()` ([`tests/common/mod.rs`](../tests/common/mod.rs)); enabling extensions changes how Markdown is interpreted and can create results that existing tests do not reproduce. Choose and document one parser configuration, or test extension-enabled cases as a separate profile.
6. **The pipeline omits an explicit HTML parse of the rendered output.** `pulldown-cmark` renders Markdown to an HTML string; it does not produce the `DOM_rt` in the diagram. Add the HTML parse step for that string, specify the same parser and document/fragment mode for both sides, and define whether comparisons use the full document or only body children. This matters because HTML parsers can synthesize `html`, `head`, and `body` nodes.
7. **The `Options` example is incomplete Rust.** The struct has additional required fields, including `hr_style`, `br_style`, `link_style`, `link_reference_style`, `ul_bullet_spacing`, `ol_number_spacing`, and `preformatted_code` ([`src/options.rs`](../src/options.rs)). As written, the snippet cannot construct `Options`. Show a complete configuration or a compilable struct update (`..Options::default()`), and state which defaults remain in effect. This also resolves the claim that all parameters are explicit while only four are listed.
8. **Failure categories need expected/unexpected criteria.** `Result::Err` is called an “Expected failure” for unsupported operations or invalid states, but the spec does not define which inputs should return errors or how such errors are distinguished from bugs. Define the expected error cases; treat any other `Err` as a failure. Also, a Rust panic can sometimes be caught, but memory exhaustion/abort and infinite loops are not ordinary panics: specify a timeout and process-isolation strategy if those are in scope.
9. **The fuzzer’s generation and replay procedure is still absent.** The document does not yet say how inputs are generated, which tags/attributes/text are in scope, how depth and input size are bounded, how seeds are stored/replayed, or how failures are minimized and promoted to regression tests. Add a generation section and an execution/failure-artifact section. Keep valid structured generation separate from malformed-HTML fuzzing, since `html5ever` repairs malformed source before conversion.
10. **“Significant divergence” and exclusions need a deterministic verdict.** Define the DOM representation being compared (node types, tag names, text, comments, namespaces, attributes, and child order) and precisely which differences are allowed. Without this, different implementations can classify the same case differently, and broad exclusions can turn the oracle into a false pass.

### Suggested resolution order

1. Correct the round-trip pipeline and choose the exact Markdown and HTML parser configurations.
2. Define a normalization/comparison contract with tests for inline spaces, preformatted text, attributes, comments, and parser-added document wrappers.
3. Replace “lossless” with a supported-preservation contract and enumerate intentional losses narrowly.
4. Make the options example compilable and fully state the effective settings.
5. Add bounded generation, seed replay, timeouts, failure artifacts, shrinking, and regression-test promotion rules.

The existing design record and round-trip tests are useful sources for known behavior and exceptions; align the new oracle with them rather than allowing broad normalization to silently weaken the checks.