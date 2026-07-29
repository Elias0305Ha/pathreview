## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/149

**Issue title:** Structural chunker silently drops documents that contain no headings

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The structural chunker splits documents by markdown headings. When a document has no
headings, it returns an empty list. That means the document never gets processed into
chunks and never enters the RAG index. The user sees feedback that appears valid, even
though the file itself was never actually read or analyzed. A fix would make it fall back
to processing the entire document as a single chunk when no headings are found. This
affects `_extract_sections` in `ingestion/chunking/structural_chunker.py`.

**Branch name:** fix/149-structural-chunker-no-headings

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

**Selection notes:**
I chose this issue because it is a Tier 1 task and a good opportunity for me to get
familiar with a codebase of this size for the first time. The fix only requires changes in
one file, so the scope is clear and manageable. I can verify the fix by running the
existing failing test, `test_document_with_no_headings`, and confirming that it passes
after the change. The issue was also easier to understand because it already included
reproduction steps, which helped me identify the problem and how to test the solution.
Once I get more comfortable with the codebase and understand the workflow better, I plan
to take on higher-tier issues with more complex changes.

---

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/Elias0305Ha/pathreview/commit/fb7fd060db79c6686936df750e992d0407edcd40

**Reproduction summary:**
I set up the local environment and ran the existing unit test suite for the structural
chunker, where `test_document_with_no_headings` fails with `assert 0 >= 1` — a
heading-less document produces zero chunks instead of one. I then probed the chunker
directly and found the same root cause silently discards any text that isn't preceded by
an ATX heading, including the preamble paragraph before a document's first heading.

### How I reproduced it

Environment (Windows, from the repo root):

```
py -3.12 -m venv .venv
.venv\Scripts\python.exe -m pip install -e ".[dev]"
```

Note: I had to build the venv against Python 3.12 rather than my default 3.14, because
several pinned dependencies have no prebuilt wheels for 3.14 on Windows.

Run the failing test:

```
.venv\Scripts\python.exe -m pytest tests/unit/test_structural_chunker.py -v
```

Observed result — 14 passed, 1 failed:

```
tests/unit/test_structural_chunker.py::TestStructuralChunker::test_document_with_no_headings FAILED

    def test_document_with_no_headings(self, chunker):
        """Test document with no headings returns single chunk."""
        text = "This is plain text without any markdown headings. " * 20
        result = chunker.chunk(text, {"source": "test"})

>       assert len(result) >= 1
E       assert 0 >= 1
E        +  where 0 = len([])
```

### What I observed directly

Calling `StructuralChunker().chunk(text, metadata)` on a range of inputs:

| Input | Chunks returned | Expected |
| --- | --- | --- |
| Plain text, no headings | **0** | 1 or more |
| Preamble paragraph, then `# Title`, then body | **1** (preamble silently dropped) | 2 |
| `#NotAHeading` (no space after `#`) | **0** | 1 |
| Setext heading (`Title` underlined with `=====`) | **0** | 1 or more |
| `# Just A Heading` with no body | **0** | 1 |

### Where the bug lives

`_extract_sections` in `ingestion/chunking/structural_chunker.py`:

- Line 111 — `if heading_stack or current_section_lines:` gates content collection, so
  lines seen before the first heading are never appended to `current_section_lines`.
- Line 115 — `if current_section_lines and heading_stack:` gates the final section save,
  so even collected content is discarded when no heading was ever matched.

With no headings anywhere, both guards fail, `_extract_sections` returns `[]`, the loop in
`chunk()` never executes, and `chunk()` returns `[]`. Nothing raises and nothing is logged,
which is what makes the data loss silent.

**PLAN.md link:** https://github.com/Elias0305Ha/pathreview/blob/fix/149-structural-chunker-no-headings/PLAN.md

**Walkthrough video (recommended):** _(not recorded)_

**Blockers or open questions:**
Going into Week 9 my main open question is scope. The issue as filed is about
heading-less documents, and the failing test only covers that case. But the same two
guards also drop the preamble before a document's first heading, which is real data loss
from the same root cause. My current plan fixes both, since fixing only the reported case
would leave the identical bug in place one line away — but I want to confirm with a
maintainer on the PR that the broader fix is welcome rather than scope creep. Setext
headings I am deliberately leaving out of scope; that is a separate feature, not this bug.
