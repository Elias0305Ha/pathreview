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
The main decision I had to make this week was scope. The issue as filed is about
heading-less documents, and the failing test only covers that case. But the same two
guards also drop the preamble before a document's first heading, which is real data loss
from the same root cause — I confirmed it while reproducing. I decided to fix both, because
fixing only the reported case would leave the identical bug live four lines away, which is
harder to defend in review than a slightly larger PR. I will flag the wider scope at the
top of the PR description and keep the two changes in separate commits so a maintainer can
ask me to split them cheaply.

Still open going into Week 9: what a chunk with no heading should carry in its metadata.
`heading_path` is currently always a non-empty string, so I need to grep its consumers in
`rag/` and `api/` before deciding between an empty string and omitting the key. Setext
headings I am deliberately leaving out of scope; that is a separate feature, not this bug.

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

_Written Thursday rather than Wednesday — I was a day late starting the build._

**Current progress:**

All five implementation sub-tasks from `PLAN.md` are done, in three commits.

First I settled the open question I carried out of Week 8. I grepped `heading_path` across
the repo: outside `structural_chunker.py` and its own test file, **nothing reads it** — not
`rag/`, not `api/`, not `agent/`. So the "stray ` > ` in a citation breadcrumb" risk I was
worried about does not exist, and I can write `heading_path: ""` with `heading_level: 0`
and keep the metadata shape uniform across every chunk. I also checked
`test_chunk_metadata_includes_heading_level`, which asserts `heading_level in [1, 2, 3]`:
its fixture document opens with `# Level 1` and has no preamble, so it never sees a level-0
chunk. That means I did not have to touch an existing assertion, which was the outcome I
wanted — quietly editing someone else's test to make my change pass is exactly what a
reviewer should catch.

Steps 1–4 (the fix) landed as two commits, split so the second is droppable. Tracing the
code before writing anything corrected an assumption in my plan: I had thought line 111 was
the preamble bug and line 115 the heading-less bug, one each. It is not that clean. For a
document with no headings, `heading_stack` and `current_section_lines` are *both* empty on
every line, so the line-111 guard drops the content before line 115 is ever reached — the
reported bug needs both guards relaxed. So the split is by symptom, not by line:

- `fix(ingestion): emit a chunk for documents with no headings` — relaxes both guards and
  moves section building into a new `_append_section` helper that skips whitespace-only
  content. This alone closes #149.
- `fix(ingestion): keep content that precedes the first heading` — routes the
  heading-boundary save through the same helper without the `heading_stack` gate, so a
  preamble becomes its own section.

I verified the second commit is genuinely separable: with only the first applied, preamble
lines are collected but still discarded at the heading boundary, so #149 stays fixed and
the preamble behaviour is unchanged. If a maintainer calls the wider scope creep, dropping
that commit costs nothing.

Fixing the empty-content case turned up a bug I had listed as an edge case but had not
realised was already live: a document of bare headings (`# A` / `## B` / `## C`) emits
chunks with **empty text** on `main` today, because the old inline save path appended
`"".strip()` without checking. `_append_section` guards it, so that is fixed as a side
effect of the refactor rather than as a separate change.

Step 5 (tests) is a third commit adding nine tests to `tests/unit/test_structural_chunker.py`,
matching the existing fixture and assertion style. I checked they are real regression tests
by restoring the pre-fix chunker and running them against it: eight of the nine fail. The
ninth, `test_content_after_last_heading_is_kept`, passes both before and after — it guards
existing behaviour rather than proving the fix, and I kept it for that reason.

**Pre-existing failures.** I recorded a baseline before changing anything, which turned out
to matter: `make test-unit` fails **53 tests across 16 files** on a clean checkout. Exactly
one of those, `test_document_with_no_headings`, is mine. `make check` is worse — ruff
reports 182 errors, black would reformat 52 files, and mypy stops early on missing stubs
for `jose`, `passlib` and `rank_bm25`. After my changes: 52 failures, 385 passing, up from
375. The only difference from baseline is my test flipping to pass. No new failures.

I deliberately did **not** run `make check`, because it invokes `black .`, which rewrites
all 52 files repo-wide. Burying a three-file fix in a repo-wide reformat is a good way to
get a PR ignored. I ran `black --check` on my two files instead, and formatted only the
code I added — the pre-existing `section_metadata.update({...})` blocks in the same file
are still non-compliant and I left them alone. Ruff on my two files reports the same 4
pre-existing errors as baseline (unused variables in tests I did not write); my additions
add none. I will document all of this in the PR description.

**Next steps:**
Open the draft PR, ask for peer review in Slack, and act on anything that comes back. Then
fill in Check-in 2 with the PR link, mark it ready for review, and submit the branch URL.

**Blockers:**
None on the code. The real risk is the peer review — it is the one item that depends on
someone else's schedule, and I am asking late in the week, so I am opening the PR before
polishing anything further rather than the other way round.

`make` is not installed in my Git Bash environment, so I ran the underlying commands from
`.venv/Scripts/` directly (`pytest tests/unit -m unit`, `ruff check`, `black --check`,
`mypy`) — same commands the Makefile targets wrap.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/684

**Branch:** `fix/149-structural-chunker-no-headings`

**What you built:**
`StructuralChunker` dropped every document that contained no ATX headings, returning zero
chunks so the document was embedded as nothing and never reached the RAG index — silently,
with no exception and no log line. I relaxed the two guards in `_extract_sections` that
assumed a heading had always been seen, so content lines are collected unconditionally and
the trailing section is flushed regardless of whether `heading_stack` is empty. Section
building moved into a new `_append_section` helper that skips whitespace-only content,
which keeps empty and whitespace-only input returning `[]` and also stops a document of
bare headings from emitting chunks with empty text — a bug that was already live on `main`.

**Tests added or updated:**
`tests/unit/test_structural_chunker.py` — nine new tests, no existing test modified.

Heading-less documents: `test_headingless_document_preserves_full_text`,
`test_headingless_document_metadata` (asserts `heading_path == ""` and `heading_level == 0`),
`test_headingless_document_preserves_source_metadata`, and
`test_large_headingless_document_is_sub_chunked`, which asserts the document really does
exceed `SECTION_TOKEN_LIMIT` before checking it splits into multiple non-empty chunks.

Preamble: `test_preamble_before_first_heading_is_kept` and `test_preamble_is_its_own_chunk`,
the latter checking the preamble is not silently merged into the first heading's section.

Edge cases from `PLAN.md`: `test_headings_only_document_emits_no_empty_chunks`,
`test_content_after_last_heading_is_kept`, and
`test_hash_without_space_is_treated_as_content` for `#NotAHeading`, which CommonMark does
not treat as a heading.

I checked these are genuine regression tests rather than tests that merely describe the new
code: I restored the pre-fix `_extract_sections` and ran them against it, and eight of the
nine fail. The ninth, `test_content_after_last_heading_is_kept`, passes both before and
after — it guards existing behaviour against regression, and I kept it knowingly.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes

Both boxes are checked in the sense the assignment defines for a codebase with documented
pre-existing failures: **my changes introduce no new failures.** Neither command passes
outright on this repository, and did not before I touched it. Measured against a baseline I
recorded on a clean checkout before making any changes:

| Check | Baseline | After this PR |
| --- | --- | --- |
| `pytest tests/unit -m unit` | 53 failed, 375 passed | 52 failed, 385 passed |
| `ruff check .` | 182 errors | 182 errors |
| `black --check .` | 52 files would reformat | 52 files would reformat |
| `mypy` (6 packages) | 5 errors, checking halted | 5 errors, checking halted |

Diffing the sorted failure lists from before and after gives exactly one line of
difference: `test_document_with_no_headings` flipping from fail to pass. The 52 remaining
failures are spread across sixteen files — `test_review_service.py` (13),
`test_bias_detector.py` (9), `test_pii_scrubber.py` (5) and others — none of them related
to chunking. The mypy errors are missing library stubs for `jose`, `passlib` and
`rank_bm25`, plus a numpy stub requiring Python 3.12. All of this is documented in the PR
description so a reviewer does not have to take my word for it.

I deliberately did not run `make check` itself, because it invokes `black .`, which rewrites
52 files repo-wide. Burying a three-file bugfix inside a repo-wide reformat would make the
PR much harder to review and much easier to ignore. I ran `black --check` on my two files
and formatted only the code I added; the pre-existing non-compliant blocks in the same file
are untouched. I said so in the PR and offered to reformat if the maintainer prefers it.

**Draft PR feedback received from:** none — requested in `#ai201-community-su26`, no
response before submission.

This is the part of the week I handled worst, and I would rather record that accurately than
dress it up. I started building on Thursday instead of Monday, which left no real window for
someone to read the PR before the deadline. Posting it in Slack on the last day and waiting
would have meant missing the submission, so I opened it as a ready PR rather than a draft
and shared the link in `#ai201-community-su26` anyway. Review can still arrive on an open PR
and I will respond to anything that comes back within the 48 hours `CONTRIBUTING.md` asks
for.

The lesson is specific rather than general: of everything due this week, peer review was the
only item that depended on another person's schedule, and it was therefore the only one I
could not compress by working harder on the last day. That is the item that should have gone
first. The code took a few hours; the review window needed days, and I spent them on
planning I had largely finished in Week 8.

**What I would still change.** Three things I chose not to do, recorded so they are
decisions rather than omissions. `chunk_index` is already wrong — it is set to `len(chunks)`
only on the non-sub-chunked branch, so indices collide once `SemanticChunker` contributes
chunks. My fix produces more chunks and makes it more visible, but it is a separate bug and
I offered to file it rather than quietly widening the PR. Setext headings are still
unsupported; after this fix those documents at least stop vanishing. And `# Just A Heading`
with no body still yields no chunk, which is arguably wrong but is a behaviour decision I
did not think was mine to make unilaterally.
