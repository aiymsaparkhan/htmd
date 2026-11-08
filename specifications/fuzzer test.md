# Bold text mutation smoke test

## Goal
Check that changing plain text inside a bold HTML element does not break
HTML-to-Markdown conversion.

## Seed
`<p><strong>bold</strong></p>`

## Mutation
Replace only the text inside `<strong>` with a non-empty word of 1–16
ASCII letters or digits. Keep the HTML tags unchanged.

## Expected result
In the default Pure translation mode, converting
`<p><strong>TEXT</strong></p>` produces `**TEXT**`.

The converter must not panic, and converting the same input again must
produce the same result.