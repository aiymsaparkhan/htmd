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

In addition to manually written test cases, random HTML documents can be generated.

The generator should use different HTML elements and combine them into documents.

Possible inputs include:

- Random text.
- Different heading levels.
- Nested lists.
- Links with different URLs.
- Images with different attributes.
- Nested paragraphs and formatting elements.
- Empty elements.
- HTML entities and special characters.

The generator should also create documents with different nesting depths.

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
