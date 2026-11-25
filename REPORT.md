# Assignment 3 Report

- **Name**: Matias Viguera Barrietnos
- **Student ID**: 18542
- **Group**: BBADBA 5A

## Use case

*Who asks, what they do with the answer, and what a wrong answer costs compared with "I don't know".*

> **TO WRITE (your own words, 3–5 sentences).** Facts to use: the users are recreational divers and small salvage or survey companies working off the Spanish coast; they ask whether they may search for, lift, keep, sell or must report a wreck or an object found underwater; they act on the answer before contacting the authorities. A wrong "yes, you can keep it" can mean an administrative fine of up to €150,253.03 or €601,012.10 (Ley 16/1985, Art. 76), losing the finder's reward of half the appraised value (Art. 44.3–44.4), or seizure of the find. "I don't know" only costs them a call to the authorities or a lawyer. So a refusal is much cheaper than a confident wrong answer.

## Corpus

Four legal texts, all taken from the BOE (Spain's official gazette) as XML. Spanish legislation is not protected by copyright (Ley de Propiedad Intelectual, art. 13), and the BOE allows reuse of its content if the source is cited. The consolidated texts are the wording in force on 18 November 2026. Only the parts relevant to wrecks and finds were kept from the two longest texts.

| Document | Source (BOE) | Date | Kept | Words |
|---|---|---|---|---|
| UN Convention on the Law of the Sea (UNCLOS), Spanish ratification | [BOE-A-1997-3296](https://www.boe.es/diario_boe/txt.php?id=BOE-A-1997-3296) | BOE 14 Feb 1997 | Ratification instrument with Spain's declarations, preamble, Parts I–VII (sea zones), Art. 149, Part XVI (Art. 303) | 19,984 |
| UNESCO Convention on the Protection of the Underwater Cultural Heritage (2001) | [BOE-A-2009-3787](https://www.boe.es/diario_boe/txt.php?id=BOE-A-2009-3787) | BOE 5 Mar 2009 | All, including the annex rules (Normas 1–36) | 8,122 |
| Ley 16/1985 del Patrimonio Histórico Español | [BOE-A-1985-12534](https://www.boe.es/buscar/act.php?id=BOE-A-1985-12534) | BOE 29 Jun 1985, consolidated | All | 15,609 |
| Ley 14/2014 de Navegación Marítima | [BOE-A-2014-7877](https://www.boe.es/buscar/act.php?id=BOE-A-2014-7877) | BOE 25 Jul 2014, consolidated | Preliminary title, Title I (navigation in Spanish waters), Title VI (salvage, sunken goods) | 10,360 |
| **Total** | | | | **54,075** |

The texts are in Spanish; the test questions and answers are in English.

## Ingestion

`index.py` downloads each document from the BOE's XML services into `corpus/` and parses it into **units**: one per article, or per rule in the UNESCO annex. Each unit keeps its full location (document › part or title › chapter › section › article) and a link to the official text.

Cleaning:
- **Consolidated laws** keep every past wording of each article. The parser keeps only the version in force on the corpus date and drops the BOE's editorial notes ("Se modifica por…", "Téngase en cuenta…").
- **Article numbers** in Ley 16/1985 are written in words ("Artículo cuarenta y cuatro"); they are normalised to "Artículo 44" so all documents use the same labels.
- Signatures, footnotes and the list of states parties' tables are dropped.

**Extraction problem fixed:** the two treaties (UNCLOS and UNESCO) first came out empty. Their XML contains small `<texto>` blocks inside cross-references, and the parser took the first one instead of the document body. It now picks the `<texto>` element with the most paragraphs.

**Chunking**, two designs compared (see Retrieval):
- `article`: one chunk per article; articles over 300 words are split at paragraph boundaries. Each chunk starts with its document and location, e.g. `[Ley 14/2014 … › Capítulo IV › Sección 2.ª › Artículo 383]`.
- `fixed`: 150-word windows with 30 words of overlap, ignoring the structure.

**Metadata on every chunk:** source, title, location, article (and every article a fixed window spans), part, link, document hash, position, word count, design.

**Embedding:** `gemini-embedding-2`, 768 dimensions, with the document and query task prefixes; stored in ChromaDB in `index/` (14 MB). Each batch of 50 is saved as soon as it returns, and only chunks without a vector are embedded, so a rerun or the daily quota loses nothing.

**Chunks:** 459 (`article`) and 452 (`fixed`) for the corpus; the document added later (see below) added 32 and 66.

## Test set

32 questions in `eval/questions.csv`: **21 single**, **6 multi** (need two passages) and **5 unanswerable**. They were written from the documents, in plain English as a diver or salvage company would ask, before any tuning. Every expected source and article label was checked against the index, and every reference answer against the Spanish text of its article.

> **TO WRITE (1–2 sentences):** how the questions were written. They were written by a friend from a brief I prepared and that I checked one by one; say so here.

Two examples:

| id | kind | Question | Expected | Reference answer |
|---|---|---|---|---|
| q09 | single | We want to use sonar to find a fishing boat that sank last winter in Spain's territorial sea. We don't own it and won't lift anything. Who can approve the search, and can we get exclusive search rights? | Ley 14/2014, Art. 376 | You need the Spanish Navy's permission even for the search. Since you don't own the boat, the Navy can decide whether to grant permission, and any permission it grants you is non-exclusive. |
| q25 | multi | We're planning to study a historic wreck in a country's contiguous zone. How far from the baseline can that zone reach, and can a country that has joined the international heritage agreement require approval for our work there? | UNCLOS Art. 33; UNESCO 2001 Art. 8 | Up to 24 nautical miles. The country can regulate and authorise the work, and if it does, it must require the annex rules to be followed. |

## Retrieval

Hit = an expected article is among the top k results, over the 27 answerable questions (`eval/retrieval.csv`):

| Design | hit@1 | hit@3 | hit@5 | Multi: both passages in top 5 | Words added to the prompt (top 5) |
|---|---|---|---|---|---|
| `article` (one chunk per article) | **0.74** | **0.96** | **1.00** | 0.83 | 835 |
| `fixed` (150 words, 30 overlap) | 0.63 | 0.78 | 0.85 | 0.83 | 745 |

> **TO WRITE (2–3 sentences): the design you chose and why.** Facts: `article` is better at every k and always finds the expected article in the top 5; it costs about 90 more words per question (+12%); both find both passages for 5 of the 6 multi questions. The agent uses `article` with k = 5.

## Tools and subagent

`server.py` exposes two MCP tools. Their descriptions, which the model reads:

**search(query)**
> Search four legal texts, in Spanish, on shipwrecks and objects found underwater: Spain's Historical Heritage Law (Ley 16/1985: archaeological finds, permits, fines), Spain's Maritime Navigation Law (Ley 14/2014: salvage, sunken ships and goods, extraction), the UNESCO 2001 Convention on the Protection of the Underwater Cultural Heritage, and the UN Convention on the Law of the Sea (UNCLOS: sea zones, archaeological objects at sea). Use it for any question about who owns a wreck or a find, reporting duties, permits, salvage rewards, fines or sea zones. The query can be in English; Spanish legal terms (e.g. "hallazgo casual", "buque naufragado") often match better. Returns the 5 most similar passages, best first. Each has a "Cite as" label (document and article) to cite it by, its location, a link to the official text, and the passage itself in Spanish.

**add_document(path)**
> Add a document to the searchable index: a BOE XML, PDF, HTML, Markdown or text file at `path` on this computer. It is split into articles, pages or sections, embedded and stored, so later searches can find it. Use it only when the user asks to add a specific file. If the same content is already indexed, nothing is embedded again. If a document with the same name but different content is indexed, its old passages are replaced by the new ones. Returns what was added: the document's name, how many sections and words it has, and the label to cite its passages by.

`add_document` imports `index_file` from `index.py`, so it uses the same parsing, chunking and embedding as the corpus. The server loads `index/` at startup and never re-embeds the corpus.

One search result, for "Can I keep old coins I find in a shipwreck off the Spanish coast?" (first of five):

```
Result 1 (similarity 0.74)
Cite as: [Ley 16/1985, Artículo 44]
Location: Ley 16/1985, de 25 de junio, del Patrimonio Histórico Español › Título V. Del Patrimonio Arqueológico › Artículo 44
Link: https://www.boe.es/buscar/act.php?id=BOE-A-1985-12534#acuarentaycuatro
Artículo cuarenta y cuatro
1. Son bienes de dominio público todos los objetos y restos materiales que posean los valores que son propios del Patrimonio Histórico Español y sean descubiertos como consecuencia de excavaciones, remociones de tierra u obras de cualquier índole o por azar. El descubridor deberá comunicar a la Administración competente su descubrimiento en el plazo máximo de treinta días e inmediatamente cuando se trate de hallazgos casuales. […]
```

**Subagent** (`.claude/agents/corpus-researcher.md`) has only the `search` tool. Its instructions: search with the question, then again with Spanish legal terms if needed (at most three searches); answer in English in two to four sentences using only the results; cite every claim with its "Cite as" label; reply "The documents don't cover this." when they don't, or say which part they don't cover; never answer from its own knowledge.

Transcript from Claude Code in VS Code:

```
User: Use the corpus-researcher subagent to answer: Can I keep old coins I find in a shipwreck off the Spanish coast?

Agent: Keeping shipwreck coins question  (delegated to corpus-researcher)
  IN: A recreational diver asks: "Can I keep old coins I find in a shipwreck off the Spanish coast?"
      Search the corpus (use Spanish terms too, e.g. «hallazgo casual», «dominio público», «objetos arqueológicos»,
      «buque naufragado», «patrimonio cultural subacuático», «extracción», «premio») and cover: who owns such finds,
      whether the finder can keep them, any duty to report/hand over, any finder's reward, and any relevant UNESCO 2001
      rules on commercial exploitation. Return a short answer where every claim carries its exact "Cite as" label [...]
  Finished

Subagent report:
  Short answer: no. Under Spanish law the diver can't keep the coins, but reporting the find properly can earn a cash reward.
  - Who owns them: Objects with historical-heritage value found by chance belong to the public domain [...] and the Civil
    Code treasure rule (art. 351) does not apply to them [Ley 16/1985, Artículo 44]. A chance find («hallazgo casual») is
    any discovery of such objects made by chance [Ley 16/1985, Artículo 41]. Sunken Spanish state vessels and their cargo
    are state public-domain property [...] [Ley 14/2014, Artículo 382].
  - Reporting and handing over: For a chance find, the finder must tell the competent authority «inmediatamente»
    (immediately) [...] [Ley 16/1985, Artículo 44].
  - Permits: Lifting sunken ships or goods in Spanish internal waters or the territorial sea needs prior authorisation from
    the Navy [Ley 14/2014, Artículo 377]. Searching for them also needs Navy authorisation [Ley 14/2014, Artículo 376]. [...]
  - Finder's reward: The finder and the owner of the place where the object was found are entitled to a cash reward of half
    the legally appraised value, split equally between them [Ley 16/1985, Artículo 44]. [...]
  - UNESCO 2001: Underwater cultural heritage shall not be commercially exploited [UNESCO 2001, Artículo 2]. It must not be
    traded, sold, bought or bartered as commercial goods [UNESCO 2001, Norma 2]. [...]
  Key quote: «Son bienes de dominio público todos los objetos y restos materiales que posean los valores que son propios
  del Patrimonio Histórico Español» [Ley 16/1985, Artículo 44].
  Gaps: The documents don't cover whether ordinary coins with no historical or archaeological value fall under these
  rules, or the amount of any fine.

Final answer (main agent): No, you can't keep them. Under Spanish law, old coins with historical value found in a wreck
belong to the state, but reporting them can earn you a cash reward. [same points and citations as the report]
```

The subagent's "Gaps" line is partly wrong: fines are in the corpus (Ley 16/1985, Art. 76), but it never searched for them.

## Adding a document

**Already indexed:** `index_file` fingerprints the document's text. Identical content is skipped with nothing embedded; a document with the same name but different content has its old chunks replaced. The add-document call also shows the user's approval step (below).

**Unanswerable question:** q32, "What did the US court decide about ownership of the Atocha treasure found off Florida?"

**Document added:** *Treasure Salvors, Inc. v. Unidentified Wrecked and Abandoned Sailing Vessel*, 569 F.2d 330 (5th Cir. 1978), from [Public.Resource.Org](https://law.resource.org/pub/us/case/reporter/F2/569/569.F2d.330.76-2151.html) (US court opinions are public domain; the page is marked CC0). 7,882 words → 32 chunks (`article`) and 66 (`fixed`). Saved in `added/`; outputs in `eval/demo_*.txt`.

Before:
```
tool: search({'query': 'Atocha'})
The documents don't cover this.
```

Adding it:
```
The assistant wants to run add_document with {'path': 'added/atocha_5th_circuit_1978.html'}. Allow it? [y/N] Y
  corpus_article: embedded 32/32 new chunks
  corpus_fixed: embedded 66/66 new chunks
```

After:
```
tool: search({'query': 'Atocha Florida ownership court'})
The US courts confirmed Treasure Salvors, Inc. and Armada Research Corp. in their sole title and right to immediate
possession of the Nuestra Señora de Atocha, rejecting claims by the United States government
[atocha_5th_circuit_1978, Inicio]. The court applied the law of finds to the long-abandoned vessel, holding that
«title to abandoned property vests in the person who reduces that property to his or her possession»
[atocha_5th_circuit_1978, Inicio].
```

The evaluation in this report was run before this document was added.

## Harness and guardrails

`agent.py` uses LangChain's `create_agent` on `gemini-3.5-flash-lite`, with the server's tools over MCP and `AGENTS.md` as the system prompt. `AGENTS.md` rules, each with its reason in the file:

1. Search before every answer; search again with Spanish legal terms if needed.
2. Answer only from the search results, never from the model's own knowledge.
3. Cite every claim with its exact "Cite as" label; never cite an article that wasn't retrieved.
4. If the results don't answer the question, say "The documents don't cover this."
5. If they answer only part, answer that part and say which part they don't cover.
6. Answer in English, briefly, with one short Spanish quote in «» and its meaning.
7. When the user is about to act on a find, end with "This is information from the legal texts, not legal advice. Report any find to the authorities before acting."
8. Use `add_document` only when the user asks to add a specific file.

**Approval step:** LangChain's `HumanInTheLoopMiddleware` pauses before every `add_document` call and asks `Allow it? [y/N]` in the terminal; without a terminal (evaluation, GitHub Actions) the call is rejected. See the demo above.

**Call limits:** at most 8 model calls and 4 searches per question (`ModelCallLimitMiddleware`, `ToolCallLimitMiddleware`). Temporary Gemini errors (503, 429) are retried with backoff (`ModelRetryMiddleware`).

**Extra guardrail, citation check:** after the agent answers, code compares every `[Document, Article]` it cited with the "Cite as" labels its searches actually returned. If a citation wasn't retrieved, or a non-refusal answer has no citations, the agent gets one retry with feedback; if it fails again, the answer is replaced by a refusal.

**A case where it fired** (`eval/demo_add.txt`): when asked to add the Atocha ruling, the agent's first reply was a confirmation with no citations, so the check fired (`citation check, attempt 1: answer makes claims without any citation`). On the retry the agent searched again and returned a cited but off-topic answer about Ley 14/2014, Art. 373. This shows the check works, but also that it is too strict on replies that only report an action; it should skip the check when the reply makes no legal claim. It never fired in the 32 evaluation questions.

## Evaluation

`evaluate.py` (`main.py evaluate`) runs the agent on all 32 questions and saves answers, verdicts and a summary in `eval/`.

**Code checks:** an unanswerable question must get "The documents don't cover this" (correct refusal); a refusal on an answerable question is a wrong refusal; every citation must have been retrieved; whether the expected article was retrieved and cited.

**LLM judge:** `gemini-3.5-flash-lite` grades the remaining answers with the retrieved passages as evidence. Rubric:
- *correct = 1* only if the answer states every key fact of the reference answer and contradicts none; a missing or wrong key fact (a number, a deadline, an owner, a yes/no) makes it 0; every part of a multi-part question must be right.
- *supported = 1* only if every factual claim is in the evidence, even if a claim is true; faithful translations count as found; the disclaimer line and "the documents don't cover" statements are not claims.
- The judge writes its reason before the verdict.

The judge was first `gemini-3.5-flash`, a stronger model, but its free daily quota ran out after a few answers, so all answers were judged again with `gemini-3.5-flash-lite`, the same family as the agent.

**Results:**

| | Judge | Human grades (27 answerable) |
|---|---|---|
| Correct, answerable questions | 24/27 (89%) | 21/27 (78%) |
| Supported | 31/32 (97%) | 20/27 (74%) |
| Correct refusals (unanswerable) | 5/5 | — (graded by code) |
| Wrong refusals | 0 | 0 |
| Blocked by the citation check | 0 | |

**Agreement with the human grades (27 answers):** 89% on correct, 78% on supported. All 9 disagreements go the same way: the judge gave 1 where the human gave 0 (q03, q04, q05, q14, q22, q23 on supported; q20, q25, q27 on correct). The judge is more lenient than a person, mostly on overgeneralised claims and missing second parts.

> **TO WRITE (1–2 sentences):** who graded the 27 answers and how (a friend, using a grading sheet showing each answer next to the reference and the cited article text, before seeing the judge's verdicts).

## Failures

Failed questions by the human grades (11 of 27), with what the data shows:

| id | What went wrong (grader's note, short) | Expected article retrieved by the agent? | Judge caught it? | Suggested cause |
|---|---|---|---|---|
| q02 | Misses where the finds go and who sets the deadline | Yes (rank 1) | Yes | wrong answer from the right passage |
| q03 | "All archaeological items" too broad (Art. 44.5 exception) | Yes | No | unsupported claim |
| q04 | Details on distance and conservation are from Art. 42.2, not cited | Yes | No | unsupported claim |
| q05 | Omits that it's only an administrative offence if not a crime | Yes | No | unsupported claim |
| q14 | Misstates when a lift is justified (Norma 1) | Yes | Correct only | wrong answer from the right passage |
| q20 | Leaves out the country of origin | No: answered from UNESCO Art. 12 instead of UNCLOS Art. 149 | No | retrieval miss |
| q21 | Never answers whether the owner's rights survive | No: UNCLOS Art. 303 not retrieved | Yes | retrieval miss |
| q22 | Survey definition comes from Art. 41, which wasn't retrieved | Art. 42 yes, Art. 41 no | No | unsupported claim |
| q23 | Says "any shipwreck"; Art. 374 excludes state vessels | Yes | No | unsupported claim |
| q25 | Omits the duty to apply the annex rules | Yes | No | wrong answer from the right passage |
| q27 | Omits who designates the coordinator if Spain declines | Yes | No | wrong answer from the right passage |

Note on q20 and q21: when the retrieval evaluation searches with the question as written, the expected article ranks 1st (q20) and 2nd (q21). The agent rewrote the query into Spanish legal terms (rule 1) and missed it.

> **TO WRITE (your own analysis, 4–6 sentences):** confirm or change each suggested cause, give the counts by cause (suggested: 5 unsupported claim, 4 wrong answer from the right passage, 2 retrieval miss, 0 wrong refusal, 0 bad test question), and say what you would fix first and why. Ideas you may agree or disagree with: most failures are incomplete answers to multi-part questions and overgeneralisations, not retrieval; the judge misses exactly these; searching with both the original question and the Spanish rewrite would fix q20 and q21.

## Decision

**Cost per question** (32 questions, `eval/summary.md`): 2.3 model calls and about 5,400 tokens on average (median 4,420, max 10,492); median 4.7 s, mean 16.5 s (the mean includes waits while Gemini was overloaded). The judge adds about 2,750 tokens per judged answer. All within Gemini's free tier.

> **TO WRITE (your decision, 4–6 sentences): would you let users rely on it, for what, and with which safeguard?** Facts to weigh: it never answered an unanswerable question and never refused an answerable one; 78% of answers were fully correct and 74% fully supported by a strict human grader; the errors are omissions and overgeneralisations that a diver could act on; the AI judge overrates quality. Consider what safeguard you'd require (e.g. always showing the cited article text and link, the disclaimer, using it only as a first orientation before contacting the authorities).

## AI use

> **CHECK AND EDIT so it is accurate.** I used Claude (Anthropic) throughout. It proposed use-case options and the corpus (I chose the shipwreck use case and English questions); wrote `index.py`, `server.py`, `agent.py`, `evaluate.py` and the subagent file, which I ran, tested and committed; turned my decisions (partial answers, disclaimer, Spanish quotes, citation-check guardrail, strict judge rubric, judge model) into `AGENTS.md` and the judge rubric; prepared the brief and grading sheet my friend used; and drafted the descriptive sections of this report from the project's outputs. A friend wrote the 32 test questions from that brief and graded the 27 answers; Claude checked every question against its article and the format of the grades, but did not grade answers. The subagent transcript was produced with Claude Code in VS Code. The decisions and the analysis in the sections marked as mine are my own.