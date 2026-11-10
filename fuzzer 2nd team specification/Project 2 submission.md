Project 2 submission
====================

Team members are Altynay Kishkenebayeva, Aiym Saparkha,
<span id="author-1794145613557" class="fui-StyledText ___s8pl3v0 fz5stix f1a3p1vp f1cmbuwj" dir="auto">Daulet
Bakytkhanov</span>,
<span id="author-1794145678874" class="fui-StyledText ___s8pl3v0 fz5stix f1a3p1vp f1cmbuwj" dir="auto">Ali
Zhumatayev</span>. The specification consists of the file
[fuzzer-spec.md](fuzzer-spec.md).

Draft submission
----------------

### Contributions

* Altynay: Run the first 4 rounds of LLM reviews with constant updates
* Aiym: 
* Daulet: Generated initial specification
* Ali: 

### Compiler test

The following statements summarize LLM output on each team member's computer,
based on the prompt "Verify that you can compile and run the project in the
current git repo."

#### Altynay

The project compiled and passed `cargo clippy -- --deny warnings` and `cargo
test` (36 unit tests passed, 1 ignored; integration/doc tests passed). The
specification is not yet fully executable: generator rules and CI case counts
are underspecified, normalization and replay need precise contracts, and
out-of-scope inputs, comments/raw content, and soft memory limits need clearer
verdicts. These issues can be resolved by specifying the generator,
deterministic normalization/replay behavior, and platform-specific test
outcomes; implementing the fuzzer itself will require adding a harness and
worker isolation.

#### Aiym

(Summarized LLM output)

#### Daulet

(Summarized LLM output)

#### Ali

(Summarized LLM output)

### Spec review rounds

| Round | Findings (issues) | Summary and round-4 status                                                                                                                                                                                            |
| ----- | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | 10                | Identified missing round-trip checks, configuration, failure rules, generation/replay, and malformed-input coverage. Round 4 adds the oracle, modes, failure classes, and seeds; generator details remain incomplete. |
| 2     | 10                | Requested a precise oracle, normalization, parser profiles, error/crash handling, and replay. Round 4 addresses most structure; normalization and generator/replay details remain unclear.                            |
| 3     | 6                 | Flagged profile/scope conflicts, seed/resource rules, normalization, and a broken section reference. Round 4 fixes most; normalization and replay/resource guarantees still need clarification.                       |
| 4     | 7                 | Remaining issues: generator coverage, case-count breakdown, reproducible replay, normalization, comments/raw content, soft memory limits, and section numbering. Not yet resolved.                                    |

### Last LLM review

```
prompt: Review the updated specification in specification, point out what has improved and what are issues. Write your feedback in LLM review.md appending from the last text line (do not overwrite old review).
```

### What has improved

* The guarantee is now explicitly limited to HTML-namespace body fragments in
  `Faithful` mode, and out-of-scope generated content is no longer counted as a
  successful preservation case.
* `core` and `gfm` inputs are separated, with GFM-only constructs defined and no
  ambiguous cross-profile round-trip verdict.
* The seed list, cases-per-run target, failure artifact fields, and platform
  resource-limit approaches are more concrete than in the previous revision.
* The missing revision-mapping claim has been removed. The options example
  remains compilable through `Options::default()`, and the non-lossless
  `Faithful` guarantee is appropriately qualified.

### Issues to resolve

1. <strong>The generator is no longer specified sufficiently to implement or
   assess coverage.</strong> The latest version omits the prior grammar, element
   tiers, attribute/text domains, size/depth limits, and malformed-input
   construction rules. “Seeded fuzzing” alone does not determine what inputs are
   generated. Restore a bounded generation contract, including which supported
   constructs and combinations are covered and how malformed inputs are
   produced.
2. <strong>The stated 15,000-case CI total is not fully auditable.</strong> Five
   master seeds times 1,000 cases yields 5,000 cases per class/profile
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
   whitespace at inline-run edges” can erase meaningful whitespace depending on
   traversal and node boundaries. Define precisely what is trimmed and in what
   order, how inline descendants participate, and how opaque subtrees skip
   normalization. Add required self-tests for inline boundary spaces, spaces
   adjacent to `<br>`, and whitespace around inline wrappers at block edges. The
   previous revision's useful normalizer self-test table should be retained or
   replaced by an equivalent explicit test contract.
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

---

Final submission
----------------

### Contributions

* Altynay:
* Aiym:
* Daulet:
* Ali:

### Spec review rounds

(Table showing spec review rounds)

### Implementation review rounds

(Table showing implementation review rounds)

### Last LLM spec review

### Last LLM implementation review
