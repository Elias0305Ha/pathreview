\## Week 7 — Issue selection



\*\*Issue link:\*\* https://github.com/ascherj/pathreview/issues/149



\*\*Issue title:\*\* Structural chunker silently drops documents that contain no headings



\*\*Tier:\*\* \[x] Tier 1  \[ ] Tier 2  \[ ] Tier 3



\*\*Problem summary:\*\*

The structural chunker splits documents by markdown headings. When a document has no headings, it returns an empty list. That means the document never gets processed into chunks and never enters the RAG index. The user sees feedback that appears valid, even though the file itself was never actually read or analyzed. A fix would make it fall back to processing the entire document as a single chunk when no headings are found. This affects `\_extract\_sections` in `ingestion/chunking/structural\_chunker.py`.



\*\*Branch name:\*\* fix/149-structural-chunker-no-headings



\*\*Setup confirmation:\*\* \[ ] App runs locally at localhost:5173



\*\*Cohort ledger:\*\* \[ ] Issue added to cohort ledger



\*\*Selection notes:\*\*

I chose this issue because it is a Tier 1 task and a good opportunity for me to get familiar with a codebase of this size for the first time. The fix only requires changes in one file, so the scope is clear and manageable. I can verify the fix by running the existing failing test, `test\_document\_with\_no\_headings`, and confirming that it passes after the change. The issue was also easier to understand because it already included reproduction steps, which helped me identify the problem and how to test the solution. Once I get more comfortable with the codebase and understand the workflow better, I plan to take on higher-tier issues with more complex changes.

