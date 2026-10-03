# The Unofficial Guide

name: Stella Ji corpus: `city_guides`

---

# Unit 1

## What This Does

This system answers practical trip-planning questions about a fictional
region, using the `city_guides` corpus: fourteen travel guides, nine on
individual towns and villages (Kestrelford, Corry Vale, Givens Mill and
others) and five that cover the whole region on eating, walking, transport,
seasons and accessibility. You can ask things like when a town's bakery sells
out, how to get between villages by bus, or which month to visit, and it
answers only from the guides, naming the file each answer came from. If
nothing in the guides is close enough to the question, it says it doesn't
have enough information instead of guessing.

## Chunking Strategy

**Chunk size:** one `##` section per chunk, with no fixed character count.

```
chunked  94 chunks, 322 characters on average (shortest 174, longest 762), produced by chunker.py::split_documents
```

**Overlap:** 0

The `city_guides` documents are long guides that the author has already
divided into labelled sections (`## Getting there`, `## Eat and drink`,
`## When to go`...), and each section covers one subject. The starter's
800-character windows ignored those boundaries: it made 51 chunks and cut
straight through sections, often mid-sentence. So I cut where the author
did. `split_documents` follows four rules:

1. **Cut before every `## ` heading.** One section, one chunk.
2. **Keep each document's opening part** (the `# H1` title plus any text
   before the first `##`) **as its own chunk.** `guide_corry_vale.md` gives
   its population there and nowhere else, so dropping it would lose that fact.
   *(Changed in unit 2: the opening part is now merged into the first
   section's chunk. See The Improvement.)*
3. **Drop any part with a heading but no body.** `guide_walking.md`,
   `guide_eating.md` and `guide_seasons.md` go straight from the title to
   the first `##`, which left 23–26 character chunks that could answer
   nothing.
4. **Add the document's `# H1` title to the top of every section chunk.**
   Nine of the fourteen guides use the same section headings, so a chunk
   like "## Eat and drink / A tearoom attached to the mill..." never says
   which place it is about. The title is what tells the search which town
   a section belongs to.

There is no overlap because overlap repairs cuts made in arbitrary places.
These cuts fall where the author put a boundary, so nothing is cut in half.

**Side effect I noticed in Milestone 4:** because every chunk carries its
town name, a question that names a town pulls in that town's unrelated
sections too. "What time does the bakery in Kestrelford sell out?" returned
Kestrelford's "When to go" and "Getting around" in its top 5.

**Known issue:** Rule 2 (keep each document's opening part as its own chunk)
can produce chunks with no facts, e.g. `guide_accessibility.md#0`. I haven't
yet checked how many of the 14 opening chunks are like this. I'm leaving the
rule in place for now and will revisit it in Unit 2.

## Sample Chunks

**Chunk 1** — source: `guide_accessibility.md#0` — produced by: `chunker.py::split_documents`

```
# Getting around the region with limited mobility

An honest assessment rather than a promotional one. Some of these places are
difficult and it is better to know in advance.
```

> **Note — known weakness.** This chunk carries no facts. It is the opening
> part of `guide_accessibility.md` (the `# H1` line plus its preamble), which
> `split_documents` keeps as a chunk of its own under rule 2. That rule exists
> because `guide_corry_vale.md` states its population in the preamble and
> nowhere else. Here the preamble is only an introduction, so the chunk can't
> answer a question on its own. At retrieval time it can only act as noise: it
> may match broad questions about accessibility and take a top-k slot from a
> section that actually holds the answer.
>
> **Why I kept it for now:** a fix has to keep the Corry Vale population while
> removing empty preambles like this one. I'd rather make that change after
> Unit 2 shows whether this chunk actually shows up in retrieval results.
>
> **Possible fixes to test later:**
> 1. Merge the opening part into the first `##` section chunk instead of
>    keeping it separate.
> 2. Keep the opening part only if it is longer than a minimum length.
>    The threshold still needs to be checked against the Corry Vale preamble.

**Chunk 2** — source: `guide_corry_vale.md#5` — produced by: `chunker.py::split_documents`

```
# Corry Vale

## Where to stay

Perhaps thirty beds in the entire valley, spread across two pubs and a handful of farmhouse rooms. In summer these are booked months ahead. Camping is permitted on two marked fields and nowhere else.
```

**Chunk 3** — source: `guide_givens_mill.md#2` — produced by: `chunker.py::split_documents`

```
# Givens Mill

## Getting around

Everything is on one street along the river. The mill is at one end and the church at the other, eight minutes apart. The riverside path continues in both directions for as far as you want to walk.
```

**Chunk 4** — source: `guide_kestrelford.md#4` — produced by: `chunker.py::split_documents`

```
# Kestrelford

## What to see

The market square on a Saturday morning is the main event and has run continuously since the 1400s. The parish church has a 13th-century tower you can climb for £2. The old trackbed walk runs six miles to the next village along an easy gradient and is the best half-day here.
```

**Chunk 5** — source: `guide_pellew_sands.md#6` — produced by: `chunker.py::split_documents`

```
# Pellew Sands

## When to go

June and September for the beach without the crowds. July and August are busy and the town is at its most itself, for better and worse. Winter is bleak, largely closed, and has a following among people who like that sort of thing.
```

## Sample Answer

**Question:** What time does the bakery in Kestrelford sell out?

**Answer:**

```
The bakery in Kestrelford sells out by 11am.

(Sources: `guide_kestrelford.md` and `guide_eating.md`)

Sources retrieved: guide_eating.md, guide_kestrelford.md
```

**My relevance cutoff:** 0.7

| Question | In corpus? | Best distance |
| -------- | ---------- | ------------- |
| How many people live in the largest village in Corry Vale? | Yes | 0.1637 |
| What time does the bakery in Kestrelford sell out? | Yes | 0.3221 |
| In what year did Marchwood's covered market begin operating? | Yes | 0.3827 |
| Which evening meal is hardest to find across the region...? | Yes | 0.5106 |
| Why can't visitors use one bus ticket across the whole region? | Yes | 0.6374 |
| What is the capital of Mongolia? | No | 0.8026 |
| What is the recommended dosage of ibuprofen for a headache? | No | 0.8350 |
| How do I write a for loop in Rust? | No | 0.8365 |
| How do I change the oil in a diesel engine? | No | 0.8881 |
| Who won the 1994 World Cup? | No | 0.9753 |

In-corpus questions landed between 0.16 and 0.64; out-of-scope questions
between 0.80 and 0.98, leaving a gap from 0.64 to 0.80. The starter's
default of 0.6 wrongly refused the bus-ticket question (0.637), even though
the top chunk (guide_regional_transport.md, "Buses") holds the answer. Its
distance is high because the question says "one bus ticket" while the
document says "three operators". I put the cutoff at 0.7, which leaves
about 0.06 of margin below it and 0.10 above.

The risk is near-miss questions. "Is there a cinema in Kestrelford?" isn't
covered by the guides, but it scored 0.401, closer than two of my in-corpus
questions, because the place name matches the "# Kestrelford" title on every
chunk. No cutoff could refuse it without also refusing real questions. It
passed the gate, and the grounding instruction caught it: the model said it
didn't have enough information. It still cited guide_kestrelford.md in that
refusal, which I may tighten later.

## How I Used AI

### Unit 1

**1.** After printing five chunks, I pasted them into Claude and asked
whether each could answer a question on its own. It said chunks 2–5 could,
and pointed out that chunk 1 (`guide_accessibility.md#0`) held only an
introduction and no facts. That chunk comes from my rule 2, which keeps each
document's opening part so the Corry Vale population isn't lost. Claude
suggested either merging the opening part into the first section or keeping
it and documenting the problem. I chose not to change the chunker yet,
because any fix still has to keep the Corry Vale fact, and I'd rather see in
Unit 2 whether this chunk actually shows up in retrieval. I added a note
under the chunk instead.

**2.** When I set the relevance cutoff, Claude helped me read my ten
distances and suggested 0.7, predicting that near-miss questions would land
between 0.6 and 0.8. I tested that with "Is there a cinema in Kestrelford?",
which the guides don't cover. It scored 0.401, closer than two of my real
questions, so the prediction was wrong: no cutoff could stop it. I replaced
that sentence in my README with the measured result, and wrote down that the
grounding instruction, not the gate, is what caught this question.

### Unit 2

**1. Arguing the opposite verdict.** For Milestone 2 I asked Claude to argue
against each MET verdict. One argument was that criterion 2 checks only that
a source is named, and that run 3 of the evening-meal question might contain
a claim its source doesn't support. I checked `guide_corry_vale.md` myself:
it says one pub "serves food seven days a week" and gives no hours, so
"Sunday evening" was the model's inference. I kept the verdict as MET,
because the criterion was met as written, and recorded the gap in the verdict.

**2. A number with no basis.** Claude suggested tightening criterion 1 to
"the answer chunk leads by at least 0.02". I asked what the industry standard
was. It said there isn't one, because distances depend on the embedding
model, and suggested measuring how much rephrasing moves them instead. I ran
three phrasings each of my two narrowest questions. Distances moved by about
0.05, but the answer chunk ranked first in all 6. That overturned Claude's
guess that a narrow lead meant the ranking would flip, and I rewrote my
diagnosis around what the rephrasing showed.

**3. Pushing back on the fix.** Claude recommended merging opening chunks
into the first section. I objected that the merged chunk would still carry
the title "Getting around the region" and might keep matching on it. Claude
agreed it could, and that became risk 1 in my improvement write-up. The
after test showed it didn't happen: the chunk dropped out of every top 5.

Claude drafted much of the English in this README. I checked every number
against my own output files, and where the drafts made a claim I hadn't
verified (for example, that the bus-ticket chunk contained the answer), I
checked it before keeping it.

---

# Unit 2

## Run Log — Before

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
| --- | --- | --- | --- | --- | --- |
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Chunks are whole sections (heading start, sentence end) | 0 exceptions | 94/94 | 94/94 | 94/94 | MET |
| 5. No-town questions reach the cross-cutting guide | 2 of 2 | 2/2 | 2/2 | 2/2 | MET |

Source: `results/run_2026-09-29_1810_before.md`, produced by `run_eval.py::main`
(top-k 5, cutoff 0.7, 3 runs, caching off). Criteria 1, 3, 4 and 5 depend only
on chunking and retrieval, which are deterministic, so the same number appears
in all three columns. Criterion 2 depends on the generated answer and is the
only one that could vary. The wording changed on every run, which confirms
the cache was off.

### Output behind each criterion (Run 1)

**Criteria 1 and 2** — `run_eval.py::main`, from `results/run_2026-09-29_1810_before.md`

```
Q: How many people live in the largest village in Corry Vale?
Best distance: 0.1637 · Sources retrieved: guide_corry_vale.md, guide_walking.md
The largest village in Corry Vale has 900 people (guide_corry_vale.md).
```

**Criterion 3** — `run_eval.py::check_out_of_scope`

```
refused  (best distance 0.803)  What is the capital of Mongolia?
refused  (best distance 0.888)  How do I change the oil in a diesel engine?
refused  (best distance 0.975)  Who won the 1994 World Cup?
refused  (best distance 0.835)  What is the recommended dosage of ibuprofen for a headache?
refused  (best distance 0.836)  How do I write a for loop in Rust?
-> gate refused 5 of 5

$ python app.py ask "What is the capital of Mongolia?"
I don't have enough information about that.
```

**Criterion 4** — chunks from `chunker.py::split_documents`, checked by `check_chunks.py`

```
94 chunks checked, 0 fail
```

**Criterion 5** — `run_eval.py::main`

```
Q: Why can't visitors use one bus ticket across the whole region?
Best distance: 0.6374 · Sources retrieved: guide_accessibility.md, guide_kestrelford.md, guide_marchwood.md, guide_regional_transport.md
Visitors cannot use one bus ticket across the whole region because there are three different operators running in the region, and they do not accept each other's tickets (guide_regional_transport.md).
```

## Verdicts

| # | Criterion | Verdict | How I decided |
| --- | --- | --- | --- |
| 1 | Retrieved chunk contains the answer (4 of 5) | MET | 5/5 in all three runs. For each question I checked that the expected fact (11am, 1863, Sunday, 900, three operators) appears in a chunk that was retrieved, not just in the answer. The one I flagged as the main risk, the Corry Vale population in the preamble, came back at 0.164, the closest of all five. |
| 2 | Every answer names a source (5 of 5) | MET | All 15 answers name at least one file. The format varied from run to run (`Source:`, `Sources:`, a filename in brackets), but every answer had one. The criterion checks only that a source is named, and one answer shows the gap: in run 3 of the evening-meal question the model added that Corry Vale is a place to get this meal and cited `guide_corry_vale.md`. That file says one pub "serves food seven days a week" and gives no hours, so "Sunday evening" was the model's own inference. The source was named, but the claim goes beyond it. This criterion can't catch that. |
| 3 | Gate stops out-of-corpus questions (4 of 5) | MET | 5/5 refused, closest at 0.803 against a 0.7 cutoff. Each refused question returns exactly "I don't have enough information about that", as the criterion says (checked with `python app.py ask "What is the capital of Mongolia?"`). The two questions I expected to sit closest to the boundary (ibuprofen, diesel) came back at 0.835 and 0.888; the closest was actually Mongolia at 0.803, so the result met the target but my reasoning about which questions were at risk didn't hold. |
| 4 | Chunks are whole sections (no exceptions) | MET | `check_chunks.py` reports 94 chunks checked, 0 fail. This measures format only: `guide_accessibility.md#0` passes but holds no facts. |
| 5 | No-town questions reach the cross-cutting guide (2 of 2) | MET | Both questions retrieved the right document in all three runs: `guide_eating.md` at 0.511 and `guide_regional_transport.md` at 0.637. The second is only 0.063 under my 0.7 cutoff. Under the starter's 0.6, the gate would have refused it: retrieval would still have found the right chunk, but it would never have reached the model. The criterion is met, but by a narrow margin. |

## Diagnoses

**No criterion was missed.** All five met their targets in all three runs.
The assignment asks what that means, and I think it means some of my targets
were set too safe: they checked *whether* something happened, not *how
reliably*. I tested one of them further before deciding what to tighten.

### Were my targets too low?

| # | What the test showed | Tightened version |
| --- | --- | --- |
| 1 and 5 | The answer chunk was in the top 5 every time. But for both no-town questions it ranked first by only 0.004 and 0.005, so I tested whether that ranking survives rephrasing (below). It did. | For each question, the chunk holding the answer ranks **first under three different phrasings**. The two narrowest questions pass this (6 of 6). I haven't tested the other three yet. |
| 2 | Every answer named a file, but run 3 of the evening-meal question cited `guide_corry_vale.md` for a claim the file doesn't make. | Every factual claim in an answer appears in the file it cites, 15 of 15. My current system scores 14 of 15. |
| 3 | All five out-of-scope questions sat at 0.80 or higher, far from the cutoff. "Is there a cinema in Kestrelford?", which the guides don't cover, scored 0.401 and passed the gate; the model then declined. | Add five near-miss questions (about the region but not answered by the guides); the system refuses at least 4 of 5, whether the gate or the model does the refusing. Not yet tested beyond the one question. |

**Why rephrasing, not a fixed margin:** I first considered requiring the
answer chunk to lead by a set distance (0.02). There's no standard value for
this, because distances depend on the embedding model. So I measured how much
rephrasing moves them: the same question in three wordings moved the answer
chunk's distance by about 0.05 (bus: 0.595 to 0.647; evening meal: 0.468 to
0.511). A margin smaller than that says little either way. Checking whether
the rank survives rephrasing tests the actual risk directly.

| Question and phrasing | Answer chunk | Rank | Runner-up | Lead |
| --- | --- | --- | --- | --- |
| Bus, original | 0.6374 | 1 | Marchwood opening chunk 0.6416 | 0.004 |
| Bus, "single bus ticket work everywhere" | 0.5946 | 1 | Marchwood opening chunk 0.6599 | 0.065 |
| Bus, "buy one ticket for all the buses" | 0.6472 | 1 | Marchwood "Getting around" 0.6754 | 0.028 |
| Evening meal, original | 0.5106 | 1 | Pellew Sands "Eat and drink" 0.5151 | 0.005 |
| Evening meal, "hardest to get… still served" | 0.5022 | 1 | Pellew Sands "Eat and drink" 0.5180 | 0.016 |
| Evening meal, "difficult to find… still offer it" | 0.4684 | 1 | Corry Vale "Eat and drink" 0.4855 | 0.017 |

### Pattern: chunks match on place and topic, which puts noise in front of the model

Stage: **chunking**, which shapes what **embedding** captures and so what
**retrieval** returns. It then reaches **generation**.

Every chunk starts with `# Place` and `## Section`, and most sections are only
a few sentences long, so a chunk's embedding is carried largely by its place
and topic words. Rephrasing showed the answer chunk still ranks first. The
cost is what comes with it:

- **Same-topic sections crowd the top 5.** For the evening-meal question,
  single-town "## Eat and drink" sections took 2–3 of the top 5 slots under
  every phrasing, always within 0.02 of the answer.
- **Opening chunks get in on title words alone.** `guide_accessibility.md#0`,
  the chunk with no facts I flagged in unit 1, reached the bus question's
  top 5 under two of three phrasings. `guide_marchwood.md`'s opening chunk
  came second under two.
- **Naming a town pulls in that town's chunks.** The bakery question returned
  Kestrelford's "When to go" and "Getting around"; "Is there a cinema in
  Kestrelford?" scored 0.401 though the guides never mention a cinema.

**Where this leads:** Corry Vale's "## Eat and drink" was in the evening-meal
top 5 under all three phrasings, second under one. It says one pub "serves
food seven days a week", with no hours. In run 3 the model turned that into
"you can still get a Sunday evening meal in Corry Vale" and cited the file.
Retrieval kept handing the model a near-miss chunk, and generation filled
the gap: `GROUNDING_INSTRUCTION` says to use only the documents, but it
doesn't forbid inferring beyond them. This is the only wrong claim in 15
answers, and it's the end of the chain above, not a separate problem.

## The Improvement

**What I changed:** rule 2 of `chunker.py::split_documents`. Each document's
opening part (the `# H1` line plus any text before the first `##`) used to be
a chunk of its own; now it is merged into the chunk for the first section.
Nothing else changed.

**Why I picked it:** I'm merging each document's opening part into its first
section so that opening chunks with no facts, like `guide_accessibility.md#0`,
stop taking a top-5 slot. In unit 1 I kept that chunk and said I'd wait for
unit 2 to see whether it showed up in retrieval. It did: in 2 of 9 test
queries.

**Why I thought it might not work, before running it:**

1. The merged chunk still carries the title "Getting around the region", so
   it could keep matching on title words. (I raised this one myself.)
2. The Corry Vale population would share a chunk with a section, which could
   make it harder to retrieve. Criterion 1 tests exactly this.
3. A merged chunk covers two subjects, so it could be too broad.

A hint against risk 1: accessibility's four section chunks carry the same
title, yet none of them reached the top 5 in any query. Only the opening
chunk, which is nearly all title, did.

### Before and after

**The five criteria** (same test, same settings):

| Criterion | Target | Before (runs 1/2/3) | After (runs 1/2/3) |
| --- | --- | --- | --- |
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5, 5/5, 5/5 | 5/5, 5/5, 5/5 |
| 2. Every answer names a source | 5 of 5 | 5/5, 5/5, 5/5 | 5/5, 5/5, 5/5 |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 |
| 4. Chunks are whole sections | 0 exceptions | 94/94 | 84/84 |
| 5. No-town questions reach the cross-cutting guide | 2 of 2 | 2/2, 2/2, 2/2 | 2/2, 2/2, 2/2 |

All five were met before and are still met, so these show the change broke
nothing. They can't show whether it helped. That's what the next table is
for.

**Targeted measure:** the same 9 queries (my 5 test questions plus 4
rephrasings), top 5 each, from `results/retrieve_before.txt` and
`results/retrieve_after.txt`.

| Measure | Before | After |
| --- | --- | --- |
| `guide_accessibility.md` opening chunk in a top 5 | 2 of 9 queries | 0 of 9 |
| Opening-part chunks in a top 5 that aren't the answer | 5 appearances | 3 (all Marchwood's merged chunk, each further away than before) |
| Bus question: answer chunk's lead over #2 | 0.004 | 0.047 |
| Corry Vale population: answer chunk distance | 0.164 (rank 1) | 0.286 (rank 1) |
| Answer chunk ranked first | 9 of 9 | 9 of 9 |
| Closest out-of-scope question | 0.803 | 0.835 |

### Run Log — After

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
| --- | --- | --- | --- | --- | --- |
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Chunks are whole sections (heading start, sentence end) | 0 exceptions | 84/84 | 84/84 | 84/84 | MET |
| 5. No-town questions reach the cross-cutting guide | 2 of 2 | 2/2 | 2/2 | 2/2 | MET |

Source: `results/run_2026-09-29_2311_after.md`, produced by `run_eval.py::main`
(same settings as before: top-k 5, cutoff 0.7, 3 runs, caching off).
Criterion 4 from `check_chunks.py`: "84 chunks checked, 0 fail".
Targeted retrieval check: `results/retrieve_after.txt` (compare with
`results/retrieve_before.txt`).

**Output from the after run** — `run_eval.py::main`:

```
Q: Why can't visitors use one bus ticket across the whole region?
Best distance: 0.6374 · Sources retrieved: guide_kestrelford.md, guide_marchwood.md, guide_regional_transport.md
Visitors cannot use one bus ticket across the whole region because three different operators run services, and they do not accept each other's tickets (*guide_regional_transport.md*).
```

```
Q: How many people live in the largest village in Corry Vale?
Best distance: 0.2861 · Sources retrieved: guide_corry_vale.md, guide_walking.md
The largest village in Corry Vale has 900 people (guide_corry_vale.md).
```

### Did it help?

**In one sentence:** yes. On the same 9 retrieval queries run before and
after, the fact-less opening chunk dropped out of every top 5 (2 of 9 → 0 of
9), while all five criteria stayed met.

**What it fixed:** the chunk with no facts is gone from every top 5,
including from what the model is given (the bus question's "Sources
retrieved" lost `guide_accessibility.md`). Risk 1 didn't happen: merged with
"## Straightforward", the chunk no longer resembled a bus question. The bus
answer went from winning by 0.004 to winning by 0.047, and the slot it freed
went to the railway section, which is at least about transport.

**A side effect I didn't predict:** opening chunks, being mostly title and
overview, were also what out-of-scope questions matched best (Mongolia
matched Corry Vale's opening chunk, the 1994 World Cup matched Givens
Mill's). With them merged, three out-of-scope questions moved further away
and the gap between in-corpus and out-of-scope widened from 0.637–0.803 to
0.637–0.835.

**What it cost:** risk 2 happened in a mild form. The Corry Vale population
chunk moved from 0.164 to 0.286, and its lead over the next chunk shrank from
0.18 to 0.06. It still ranks first and all three answers were correct.

**What it didn't fix:** Marchwood's opening part, now merged with its first
section, still reaches the bus questions' top 5 in 3 of 9 queries. The
evening-meal question's retrieval was identical before and after, so the
Corry Vale chunk still reaches the model. None of the three after-run answers
repeated the "Sunday evening in Corry Vale" inference, but since the model
saw the same chunks, that is chance, not this change.

## What's Still Broken

No criterion is missed after the fix. These problems are still real, and
none of my five criteria would catch them.

**1. The model can infer past the text** (generation).
In run 3 of the before test, the model turned Corry Vale's "serves food
seven days a week" into "you can still get a Sunday evening meal in Corry
Vale". That chunk still reaches the model for the evening-meal question, so
the chain that produced the claim is unchanged.
*What I'd do:* add a line to `GROUNDING_INSTRUCTION` telling the model not to
draw conclusions a document doesn't state, then run the evening-meal question
many more times before and after (it appeared once in three runs, so three
runs can't show whether a fix worked).
*Why I stopped:* the unit allows one change, and I spent it on chunking.

**2. Chunks still match on place and topic, not on the fact** (chunking →
retrieval).
For the evening-meal question, single-town "## Eat and drink" sections take
2–3 of the top 5 under every phrasing, within 0.02 of the answer. Questions
that name a town pull in that town's unrelated sections. Marchwood's merged
opening chunk still reaches the bus questions' top 5 in 3 of 9 queries.
*What I'd do:* try hybrid search, since BM25 should favour chunks that share
a rare word with the question ("bakery") over chunks that only share a place
name.
*Why I stopped:* one change per unit. I'd also need to work out first which
score the relevance gate would use, because if it moved to a combined score
the 0.7 cutoff would need re-measuring, and that would be a second change.

**3. Near-miss questions get past the gate** (retrieval / gate).
"Is there a cinema in Kestrelford?" isn't covered by the guides but scored
0.401, closer than two of my real questions. No cutoff can refuse it without
refusing real questions. The model declined to answer, but it cited
`guide_kestrelford.md` while declining.
*What I'd do:* add five near-miss questions to the test and measure how many
the system refuses end to end, whether the gate or the model refuses.
*Why I stopped:* I've tested only one near-miss question, and a proper test
needs a new criterion, which belongs in the next unit's planning rather than
this one.

**4. What the fix cost.**
The Corry Vale population chunk's lead over the next chunk shrank from 0.18
to 0.06. It still ranks first, but it's the one place my change made
retrieval less certain.
*What I'd do:* keep an eye on it as the question set grows. If a rephrased
population question ever ranks another chunk first, the opening part may
need to stay separate when it holds facts.
*Why I stopped:* it still ranks first and every answer was correct.

I also listed "a merged chunk could be too broad" as risk 3, and checked it
before submitting. The longest chunk after the change (887 characters) is the
merged `guide_accessibility.md#0` itself: the two-sentence introduction plus
"## Straightforward". It's long because that section covers three towns, but
it stays on one subject, the places that are easiest with limited mobility,
and the introduction adds length without adding a subject. So risk 3 didn't
really happen.

## What I'd Do Differently

My criteria checked whether something happened, not how reliably or how
well. All five passed first time, and the problems the test did find sat in
the gaps between them. Next time:

**Criteria 1 and 5** asked only whether the answer chunk reached the top 5.
I'd ask whether it ranks first, and I wouldn't use a fixed distance margin
to measure how safely: the right value depends on the embedding model and
would have to be re-measured whenever the model or corpus changes. Three
phrasings per question worked at this scale, but it doesn't grow well. I'd
build a larger, more varied question set with the expected chunk labelled for
each, measure the share of questions where that chunk ranks first, and use a
small lead only to flag questions for a closer look.

**Criterion 2** checked that an answer names a file, not that the answer
matches it. The one wrong claim in 30 answers named a source. I'd write it
as: every factual claim in an answer appears in the file it cites.

**Criterion 3** used out-of-scope questions so far from the corpus that the
closest was 0.80 against a 0.7 cutoff. The hard case is a question about the
region that the guides don't answer. I'd add near-miss questions and count a
refusal from either the gate or the model.

**Criterion 4** measured format only, so a chunk with no facts passed. I
wouldn't try to measure "has facts" directly. I'd measure its effect instead,
as I did in Milestone 4: how often a chunk that can't answer the question
takes a top-5 slot.
