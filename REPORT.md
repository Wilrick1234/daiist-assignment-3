# Assignment 3 Report

- **Name**: ✏️
- **Student ID**: ✏️
- **Group**: ✏️ [BBADBA 5A | BBADBA 5B | PPLEDBA 5A]

## Use case

✏️ *Your words: who asks (e.g. the product team or founder of a startup shipping an AI feature that handles personal data in the EU), what they do with the answer, and why a confident wrong answer costs more than "The corpus does not cover this question."*

## Corpus

The binding text (articles and annexes, without recitals) of three EU laws. Recitals were left out to keep two chunking designs within one day of the free embedding quota (about 1,000 texts a day); they explain the law but do not create obligations.

| Document | URL | Date | Licence | Words |
|---|---|---|---|---|
| GDPR, Regulation (EU) 2016/679 | https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32016R0679 | 27.4.2016 | © European Union; reuse authorised (Commission Decision 2011/833/EU) | 29,435 |
| AI Act, Regulation (EU) 2024/1689 | https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689 | 13.6.2024 | same | 51,697 |
| ePrivacy Directive 2002/58/EC, consolidated 19.12.2009 | https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:02002L0058-20091219 | 19.12.2009 | same | 4,763 |
| **Total** | | | | **85,895** |

`main.py index` downloads the three acts from the EU Publications Office (`publications.europa.eu/resource/celex/<CELEX>`, XHTML) into `corpus/`. The ePrivacy Directive is the consolidated 2009 version, because the cookie-consent rule in Article 5(3) only exists since the 2009 amendment.

## Ingestion

- **Extraction**: each act's XHTML marks every article (`id="art_6"`) and annex (`id="anx_III"`). `index.py` takes each one with its heading (e.g. "Lawfulness of processing") and chapter, and keeps paragraphs and lettered points as separate blocks.
- **Cleaning**: footnote references, the consolidated text's amendment markers (`▼M2`, `►M1 … ◄`) and non-breaking spaces are removed; whitespace and punctuation spacing are normalised.
- **Problem fixed**: the consolidated ePrivacy text is full of amendment markers such as `▼M2` and `►M1`, which ended up inside sentences and would have been embedded as noise. They are stripped before chunking (`MARKERS` in `index.py`). A second problem was in embedding: `gemini-embedding-2` returns one *aggregated* vector when given a list of strings, so each chunk is now sent as its own `Content`, and the code checks that it gets one vector per chunk.
- **Chunking, two designs**:
  - *article*: one chunk per article or annex; articles over 450 words are split at paragraph boundaries into parts of about 350 words ("Article 5, part 2 of 7"). Each chunk starts with `GDPR, Article 33 — Notification of a personal data breach…`.
  - *fixed*: 400-word windows with 100 words of overlap, ignoring structure. A window is tagged with every article it holds at least 40 words of.
- **Metadata per chunk**: `source`, `title`, `url`, `location` (Article/Annex), `locations`, `heading`, `chapter`, `part`/`parts`, `words`, `doc_hash`, `added`, `design`.
- **Chunks**: 378 (article design) and 286 (fixed design), embedded with `gemini-embedding-2` at 768 dimensions and stored in ChromaDB (`index/chroma`, 16 MB). Vectors are saved after every API call and only missing chunks are embedded, so a re-run embeds nothing.

## Test set

✏️ *Your words: how you wrote the questions.*

30 questions in `eval/questions.csv`: **18 single, 6 multi, 6 unanswerable**. Expected passages: 13 in the GDPR, 12 in the AI Act, 6 in the ePrivacy Directive. The column `original_location` keeps the precise pinpoints (e.g. `Article 5(1)(f)`); hit rates are measured at article level.

Examples:

- **q08 (single)**: "Our HR team wants an AI tool that infers employees' emotions from facial expressions to measure engagement. Is that allowed?" → AI Act, Article 5(1)(f): no, except for medical or safety reasons.
- **q22 (multi)**: "We're adding optional advertising cookies that track identifiable visitors. Can we install them before opt-in, and must withdrawing consent be as easy as accepting?" → ePrivacy Directive Article 5(3) and GDPR Article 7(3).

## Retrieval

Hit@k: at least one expected passage in the top k (`all@5`: every expected passage of a multi question in the top 5). Base corpus only, 24 answerable questions (`eval/retrieval.csv`, per question in `eval/retrieval_details.csv`).

| Chunks | Search | hit@1 | hit@3 | hit@5 | all@5 | MRR@5 | words added at k=5 |
|---|---|---|---|---|---|---|---|
| **article** | **vector** | **0.96** | **1.00** | **1.00** | **1.00** | **0.97** | **1,374** |
| article | keyword (BM25) | 0.58 | 0.63 | 0.67 | 0.58 | 0.61 | 1,341 |
| article | hybrid (RRF) | 0.75 | 0.88 | 0.88 | 0.75 | 0.81 | 1,355 |
| fixed | vector | 0.79 | 0.96 | 0.96 | 0.88 | 0.86 | 2,000 |
| fixed | keyword | 0.46 | 0.67 | 0.67 | 0.58 | 0.56 | 1,997 |
| fixed | hybrid | 0.67 | 0.83 | 0.88 | 0.75 | 0.75 | 2,000 |

**Chosen design: article chunks, vector search, k = 5** (`index/config.json`), by the rule set before seeing the numbers: highest hit@3, ties broken by hit@5, then by fewer words added to the prompt. It also adds about 30% fewer words than the fixed design. Keyword search was weak because the questions use everyday wording ("make unsubscribing harder") while the law uses legal terms ("withdraw consent"); fusing it with vector search pulled the good vector ranking down.

## Tools and subagent

`server.py` (MCP, stdio, registered as `corpus` in `.vscode/mcp.json` and `.mcp.json`):

- **`search(query)`**: "Search the binding text of three EU laws: the GDPR (Regulation 2016/679), the AI Act (Regulation 2024/1689) and the ePrivacy Directive (2002/58/EC as amended in 2009, cookies and electronic marketing). Use it for any question about obligations, definitions, rights, deadlines, fines or exemptions under these laws, and search again with other words if the first results do not answer the question. Recitals and national laws are not included. Returns the 5 most relevant passages, each with a citation tag such as [GDPR, Article 33] to copy into the answer, the law's full title and URL, and a relevance score."
- **`add_document(path)`**: "Add one document to the searchable index: a .pdf, .html/.xhtml (EU acts from EUR-Lex are split by article), .txt or .md file given by its path on this machine… If the same file content is already indexed, nothing is embedded. If a document with the same name but different content is indexed, its old chunks are replaced. Returns the source name, how many chunks were added and an example citation tag."

One search result (top hit for q08):

```
Result 1  cite as [AI Act, Article 5] (part 2 of 7)  relevance 0.7331
Source: Regulation (EU) 2024/1689 (Artificial Intelligence Act) — https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689
AI Act, Article 5 — Prohibited AI practices (part 2 of 7)
(d) the placing on the market … (e) … facial recognition databases …
(f) the placing on the market, the putting into service for this specific purpose, or the use of AI systems to infer emotions of a natural person in the areas of workplace and education institutions, except where the use of the AI system is intended to be put in place or into the market for medical or safety reasons;
…
```

Subagent `.claude/agents/corpus-researcher.md` (tools: `mcp__corpus__search` only): it searches with the question, re-searches with the law's own terms (at most four searches), answers in at most six sentences with a citation tag after every claim, replies "The corpus does not cover this question." when the passages do not answer, and never fills gaps from memory, national law or case law.

**Transcript from the editor** ✏️ *(paste here: one question asked through the corpus-researcher subagent in Copilot or Claude Code, with its answer)*

## Adding a document

**Already indexed** (policy chosen: skip if identical, replace if changed): `add_document` hashes the file. If the same content is already in the index, it embeds nothing and says so. If a document with the same name but different content exists, its old chunks are deleted and the new version is embedded. A file can never replace one of the three base laws: it is renamed "… (added)" instead.

⏳ *Before/after demo: filled in after the demo run.*

## Harness and guardrails

`agent.py` uses LangChain `create_agent` on `gemini-3.5-flash-lite`, with the MCP server's tools and `AGENTS.md` as the system prompt.

**AGENTS.md rules** (each with its reason in the file): use only the corpus; apply the law to the user's situation; cite every material claim as `[GDPR, Article 33]` and never invent a citation; combine passages for multi-part questions; keep conditions and exceptions; refuse with exactly "The corpus does not cover this question."; keep answers clear and concise.

**Approval**: `HumanInTheLoopMiddleware` interrupts every `add_document` call. The terminal shows the tool and its arguments and asks `Approve? [y/N]`; anything but `y` rejects, and the model is told the user did not approve. Evaluation runs pass a function that always rejects.

**Limits**: at most 6 tool calls and 10 model calls per question (`ToolCallLimitMiddleware`, `ModelCallLimitMiddleware`).

**Extra guardrail, citation check**: after the agent answers, the harness parses every `[Source, Location]` tag and compares it with the tags returned by `search` in that run (`Article 6(1)(b)` counts as `Article 6`). If a tag was not retrieved, or a non-refusal answer has no tag, the agent gets one retry with the problem spelled out; if it fails again, the harness replaces the answer with the refusal.

**One case where it fired** (`eval/guardrail_case.json`): asked "What does GDPR Recital 47 say about direct marketing as a legitimate interest?", which is not covered because recitals are not indexed, the first draft had no citation tag. The guardrail sent it back, and the retry answered "The corpus does not cover this question." In the 30-question evaluation it never fired: every answer passed on the first try.

## Evaluation

**Code checks** (`evaluate.py`, all 30 questions):

| Check | Result |
|---|---|
| Citation tags all retrieved in that run (non-refusals) | 24 / 24 |
| Answer cites an expected article (answerable) | 24 / 24 |
| Expected passage among the retrieved results (answerable) | 24 / 24 |
| Refused exactly when unanswerable | 30 / 30 |
| Refusal text exactly "The corpus does not cover this question." | 6 / 6 |
| Citation guardrail fired | 0 |

**Judge**: `gemini-3.1-flash-lite`, a different model from the agent, using the rubric in `eval/rubric.md` (correctness and support graded separately, 0 or 1, no partial credit for multi questions, exact-refusal rule for unanswerable questions). It gets the question, its type, the expected answer and locations, every retrieved passage and the answer.

| Questions | n | Mean correctness | Mean support | Strict pass |
|---|---|---|---|---|
| All | 30 | 1.00 | 1.00 | 1.00 |
| Single | 18 | 1.00 | 1.00 | 1.00 |
| Multi | 6 | 1.00 | 1.00 | 1.00 |
| Answerable (single + multi) | 24 | 1.00 | 1.00 | 1.00 |
| Unanswerable | 6 | 1.00 | 1.00 | 1.00 |

Correct: 24 / 24 answerable. Unsupported: 0 (judge), 1 (my grades: q21). Correct refusals: 6 / 6. Wrong refusals: 0. Missed refusals: 0.

**Agreement with my grades** (29 questions graded in `eval/human_labels.csv` before reading the judge's verdicts):

| Grade | Agreement | Cohen's κ | My zeros | Judge zeros |
|---|---|---|---|---|
| Correct | 29 / 29 (100%) | undefined (both gave 1 everywhere) | 0 | 0 |
| Supported | 28 / 29 (96.6%) | 0.00 | 1 (q21) | 0 |

κ is 0 for support even with 96.6% agreement because the judge never gave a 0, so it agrees no better than chance on the one case that matters. Our only disagreement is q21. ✏️ *Your words: why you graded q21 unsupported. (Facts: the agent cited GDPR Article 22, GDPR Article 35 and AI Act Article 14, and all three were retrieved. The Article 35 passage, paragraphs 1–5, includes the DPIA and data-protection-officer text.)*

## Failures

✏️ *Your analysis. Facts to work from:*

- *The judge found 0 failures; you found 1 (q21, support).*
- *q26, q27 and q30 were refused without any search: the model judged them off-topic by itself. That's the right outcome, but it didn't come from checking the corpus.*
- *All 24 answerable questions had an expected passage in the top 5, and all 30 passed the code checks. The six unanswerable questions ask about salaries, vendor prices, an auditor's fee, a banner approval code and course ratings.*
- *Causes to use: retrieval miss, wrong answer from the right passage, unsupported claim, wrong refusal, bad test question. Say what to fix first.*

## Decision

✏️ *Your words: would you let users rely on it, for what, and with which safeguard?*

Cost and latency per question (30 questions, `gemini-3.5-flash-lite` free tier): on average **3,401 input + 193 output tokens**, **3.0 s** (median 2.4 s), 0.9 searches. Indexing cost 664 embeddings, one time; queries cost one embedding each, cached in `index/query_cache.json`.

## AI use

✏️ *Check and complete.* I used Claude (Anthropic) throughout:
- **Code**: it wrote `index.py`, `retrieval.py`, `server.py`, `agent.py`, `evaluate.py` and the subagent file, and ran the indexing and evaluation.
- **Corpus**: it found and parsed the corpus.
- **Tools**: it built the corpus reader and grading pages I used.
- **Report**: it filled in the factual sections, numbers and tables of this report.
- **CSV**: it converted my questions to the CSV columns the assignment requires.

I made the decisions, choosing among options Claude laid out:
- use case and corpus scope;
- the two designs to compare and the rule for picking one;
- the duplicate-document policy;
- the citation-check guardrail.

I wrote the test questions, the `AGENTS.md` rules, the judge rubric, my grades, the failure analysis and the decision.
