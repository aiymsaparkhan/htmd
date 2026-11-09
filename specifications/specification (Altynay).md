1\. Introduction
----------------

This specification defines the testing and fuzzing architecture for the `htmd`
HTML-to-Markdown conversion library.

The primary objective is to verify structural, semantic, and textual
preservation across round-trip conversions (HTML $\rightarrow$ Markdown
$\rightarrow$ HTML) using property-based random generation alongside predefined
regression tests.

---

2\. Testing Objectives \& Failure Contracts
-------------------------------------------

The test suite enforces four distinct core properties:

1. <strong>Panic \& Crash Safety:</strong> The conversion pipeline must never
   panic or enter infinite loops.
2. <strong>Deterministic Output:</strong> Identical inputs and configuration
   options must yield identical outputs.
3. <strong>Exact Standard Behavior:</strong> Predefined test cases must match
   expected Markdown outputs.
4. <strong>Round-Trip Semantic Preservation:</strong> Converting valid HTML to
   Markdown and parsing it back into HTML must preserve document structure, text
   content, and essential semantic metadata.

### 2.1 Failure Categorization

Errors encountered during testing must be categorized explicitly:

* <strong>Conversion Error (`Result::Err`):</strong> Expected failure for
  intentionally unsupported operations or invalid states.
* <strong>Panic / Crash:</strong> Unhandled runtime panic or memory error during
  execution.
* <strong>Deterministic Mismatch:</strong> Processing identical inputs yields
  non-identical outputs.
* <strong>Round-Trip Semantic Mismatch:</strong> Significant structural or
  textual divergence between original HTML DOM and re-parsed round-trip HTML
  DOM.

---

3\. Round-Trip Pipeline \& Oracle Strategy
------------------------------------------

Comparing raw HTML string representations is invalid due to harmless syntactic
variations (e.g., attribute ordering, entity escaping, void tag closing
formats). Therefore, round-trip testing relies on tree-level DOM comparison
after normalization.

### 3.1 Pipeline Execution Flow

$$\text{HTML}_{\text{input}} \xrightarrow{\text{html5ever}} \text{DOM}_{\text{src}} \xrightarrow{\text{HTMD}} \text{Markdown} \xrightarrow{\text{pulldown-cmark}} \text{DOM}_{\text{rt}} \xrightarrow{\text{Compare}} \text{Oracle Verdict}$$

1. <strong>Parse Source HTML:</strong> Parse raw input string into a source DOM
   tree ($\text{DOM}_{\text{src}}$) using `html5ever`.
2. <strong>Convert to Markdown:</strong> Pass raw HTML to `htmd` configured in
   `Faithful` mode.
3. <strong>Re-parse to HTML:</strong> Pass the generated Markdown through
   `pulldown-cmark` (configured with standard extensions: GFM tables,
   strikethrough, task lists) to produce a round-trip DOM tree
   ($\text{DOM}_{\text{rt}}$).
4. <strong>DOM Normalization \& Comparison:</strong> Compare normalized
   representations of $\text{DOM}_{\text{src}}$ and $\text{DOM}_{\text{rt}}$.

### 3.2 DOM Normalization Rules

Prior to comparison, both $\text{DOM}_{\text{src}}$ and $\text{DOM}_{\text{rt}}$
must undergo normalization:

* <strong>Whitespace Collapsing:</strong> Normalize continuous whitespace
  sequences within text nodes into single spaces (except inside `<pre>` and
  `<code>` blocks). Strip leading and trailing whitespaces.
* <strong>Attribute Normalization:</strong> Sort attributes alphabetically by
  key. Ignore synthetic attributes injected strictly by parsers.
* <strong>Entity Unescaping:</strong> Resolve HTML character entities to native
  UTF-8 code points.
* <strong>Ignorable Element Exclusions:</strong> Ignore elements discarded by
  design under selected conversion options (e.g., `<script>`, `<style>`, inline
  comments).

---

4\. Configuration \& Mode Definitions
-------------------------------------

`htmd` supports execution modes that affect semantic lossiness:

* <strong>`Faithful` Mode (Primary Fuzzing Target):</strong> Designed to
  maximize structural and semantic preservation for lossless round-tripping.
  Must be used for all automated round-trip fuzzing.
* <strong>`Pure` Mode:</strong> Produces minimal, aesthetic Markdown,
  potentially discarding unsupported HTML tags or attributes. Tested against
  separate, deliberate single-pass assertions rather than strict round-trip
  equality.

### 4.1 Explicit Configuration Parameters

All test cases must state configuration parameters explicitly. Default fuzzer
options:

````rust
Options {
    heading_style: HeadingStyle::Atx,          // # Heading
    bullet_list_marker: BulletListMarker::Asterisk, // * Bullet
    code_block_style: CodeBlockStyle::Fenced,   // ```code```
    translation_mode: TranslationMode::Faithful,
}
````
