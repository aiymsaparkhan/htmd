# HTML to Markdown Testing Specification

## 1. Introduction

The purpose of this specification is to describe how to test the HTMD HTML to Markdown converter.

The main goal is to check that HTML elements are converted correctly and that the converter does not lose important information.

Testing should cover both simple HTML documents and more complex structures.

## 2. Testing Objectives

The tests should check the following:

- Correct conversion of common HTML elements.
- Preservation of text and document structure.
- Correct handling of nested elements.
- Handling of empty and malformed HTML.
- Stable output when the same input is processed multiple times.
- No unexpected crashes or panics.

## 3. Test Cases

The following HTML elements should be tested:

| Element | Example | Expected Markdown |
|---|---|---|
| Paragraph | `<p>Hello</p>` | `Hello` |
| Heading | `<h1>Title</h1>` | `# Title` |
| Bold text | `<strong>Text</strong>` | `**Text**` |
| Italic text | `<em>Text</em>` | `*Text*` |
| Link | `<a href="https://example.com">Link</a>` | `[Link](https://example.com)` |
| Unordered list | `<ul><li>One</li></ul>` | `- One` |
| Blockquote | `<blockquote>Quote</blockquote>` | `> Quote` |

These examples describe the expected behavior and should be checked against the converter's actual formatting rules.

## 4. Input Generation

This section defines the initial structured HTML generator.
It specifies planned behavior; it does not claim that the
generator has already been implemented or executed.

### 4.1 Initial Scope

The generator produces HTML fragments using this restricted grammar:

- A fragment contains one or more blocks.
- A block is a paragraph, a heading, or an unordered list.
- Paragraphs (`p`) and headings (`h1` to `h6`) contain inline nodes.
- An inline node is text, emphasis (`em`), or strong emphasis (`strong`).
- Emphasis and strong emphasis may contain text or other inline nodes.
- An unordered list (`ul`) contains one or more list items (`li`).
- Each list item contains inline nodes.

Paragraphs, headings, and lists must not appear inside inline elements.
Paragraphs must not contain other paragraphs.
Each non-void element must have a matching closing tag.

Attributes, links, images, tables, code blocks, and nested lists are
outside this initial generator's scope. They may be added as separate
generation profiles with their own rules.

### 4.2 Text Generation

Text values are selected from a fixed, versioned pool containing:

- ASCII letters and digits.
- Non-ASCII text, including Cyrillic letters.
- Markdown punctuation: asterisks, underscores, brackets, and backticks.
- Literal ampersands and angle brackets.
- Single, leading, trailing, and repeated spaces.

Text nodes must contain between 1 and 32 Unicode scalar values before
HTML escaping. Empty elements are covered by predefined cases.

When serializing text into HTML, escape `&`, `<`, and `>` as
`&amp;`, `&lt;`, and `&gt;`, respectively. Escape each text value once;
do not escape the generated markup itself.

Preserve generated spaces. The generator must not trim or normalize
text before passing it to the converter.

### 4.3 Generation Limits

The initial profile uses the following limits:

- At most 8 top-level blocks per fragment.
- Between 1 and 4 children per paragraph, heading, list, or list item.
- Between 1 and 4 children per emphasis or strong-emphasis element.
- Maximum element depth: 6, with top-level elements at depth 1.
- Maximum serialized input size: 4096 UTF-8 bytes.

Only choose elements that can be completed within the remaining depth.
When no further inline element can be added, generate text instead.

If a complete candidate exceeds the byte limit, discard it and
generate another candidate. Never truncate serialized HTML.

Allow at most 100 candidate attempts per requested case. If all attempts
exceed the limit, report a generation failure rather than looping
indefinitely or silently skipping the case.

### 4.4 Reproducibility

Use a seeded pseudorandom generator, such as `ChaCha8Rng` from
`rand_chacha`, initialized with a recorded unsigned 64-bit seed.

Before implementation is accepted, pin the generator dependency version
and record the generator revision. The same seed, generator revision,
dependency versions, and configuration must produce the same ordered
sequence of HTML inputs.

For each run, record:

- Seed.
- Requested and completed case counts.
- Generator revision and dependency versions.
- Generation limits and text-pool version.

For each failing case, additionally save its zero-based case index
and exact serialized HTML. Replaying saved HTML must be possible
without regenerating earlier cases.

### 4.5 Predefined Generator Cases

Run these fixed inputs in addition to random generation:

| Purpose | HTML input |
|---|---|
| Empty fragment | Empty string |
| Empty element | `<p></p>` |
| Basic text | `<p>Hello</p>` |
| Heading | `<h2>Title</h2>` |
| Nested formatting | `<p><strong>A<em>B</em></strong></p>` |
| Inline boundary spaces | `<p>A <em>B</em> C</p>` |
| Escaped text | `<p>A &amp; B &lt; C</p>` |
| Unicode text | `<p>Привет</p>` |
| Markdown punctuation | `<p>*literal* _text_ [label]</p>` |
| Multiple list items | `<ul><li>One</li><li>Two</li></ul>` |

These are input examples, not claims about exact Markdown output.
The verification section defines how conversion results are assessed.

### 4.6 Malformed Inputs

Malformed HTML belongs to a separate robustness profile.
It must not be mixed into the valid structured generator.

Label malformed cases explicitly and check them for crashes,
unexpected errors, and timeouts. Do not apply the valid-input
round-trip equality contract to raw malformed markup.

## 5. Verification Strategy

Each generated HTML document should be passed to the HTMD converter.

The output should be checked using several rules:

1. The converter should not crash or panic.
2. The output should be valid UTF-8.
3. The same input and settings should produce the same output.
4. Text content should not be unexpectedly lost.
5. Supported HTML elements should follow the expected conversion rules.

For selected test cases, the output should also be compared with an expected Markdown result.

## 6. Failure Handling

When a test fails, the following information should be saved:

- Original HTML input.
- Generated Markdown output.
- Random seed used for generation.
- Description of the failure.

Saving the random seed makes it possible to reproduce the same test later.

The smallest HTML input that still causes the failure should be saved whenever possible.

## 7. Test Execution

Testing should include both predefined examples and randomly generated HTML documents.

The predefined tests should run first because they cover basic functionality.

Random tests should run with different seeds to explore more possible HTML structures.

The number of generated tests can be increased over time.

## 8. Limitations

Random generation cannot guarantee that all possible HTML structures will be tested.

Some HTML structures may also have multiple valid Markdown representations.

Because of this, not every test should rely on exact string comparison. Some tests should check the preservation of content and basic document structure instead.

## 9. Conclusion

This specification describes an initial testing approach for the HTMD converter.

Combining predefined test cases with random HTML generation can help find conversion errors, unexpected behavior and edge cases.

The testing approach can be extended when new features and conversion rules are added.
