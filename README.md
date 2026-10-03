# The Unofficial Guide

Angelo Canunayon — campus_life

> **This file is your submission.** Fill it in as you go — most sections get
> written during the milestone that produces them, not at the end.
>
> How the starter works, and every command you'll need, is in `RUNNING.md`.
> Leave that file alone.
>
> **Paste everything as text.** No screenshots, no video. A typed table gets
> full credit; a picture of the same table gets none.
>
> Delete these instruction blocks as you replace them. The `<!-- -->` comments
> are notes to you and don't show up when the page renders — you can leave them
> or remove them.

---

# Unit 1

## What This Does

This is a retrieval-augmented question answerer over `campus_life`, a corpus
of 88 short posts about student life at a university — dining halls, dorms,
courses, and the administrative rules nobody explains properly. It's built
for specific, factual questions where the right answer sits in one sentence
of one document: when campus job postings open, whether work-study income
counts against financial aid, which orientation sessions are worth
attending, how a specific course is graded. It retrieves the chunks closest
to the question, refuses to answer when nothing retrieved is actually close
(rather than guessing), and names the source document whenever it does
answer.

## Chunking Strategy

**Chunk size:** not fixed — split on paragraph breaks, not a character count.
**Overlap:** none.

The starter's 800-character window never cuts anything in this corpus — the
longest document is 563 characters — so the first finding was that "chunk
size" in the usual sense isn't the lever here at all. The real question,
reading the documents in Milestone 1, was whether a document holds one
thought or several. Counting paragraphs across all 88 documents: 56 are a
title plus exactly two body paragraphs, and the course and housing pages run
to three or four. Each body paragraph consistently carries one
self-contained idea — a course page separates format/assessment from
workload from advice; a noise page separates the actual noise fact from "go
to the library instead"; a dining page separates the experience (wait times,
what's good) from the logistics (hours, cost).

So `chunker.py::split_documents` splits each document on its paragraph
breaks (`\n\n`) and carries the title into every resulting piece, since a
chunk like "The bad: the heating is uneven" means nothing without knowing
which building it's about, and the chunk text itself never repeats the
building name — only the filename does. A document with only one paragraph
after its title (most of the short admin posts) stays a single chunk,
matching what the starter already did for those by accident.

There's no overlap because there's nothing to bridge — fixed-size windows
need overlap so a sentence cut in half at a boundary still shows up whole
somewhere; splitting on blank lines never cuts a sentence, so overlap would
only add noise.

One limitation I noticed and left alone: a few of the housing pages
(`housing_old_brewhouse.txt` and similar) cram two facts — laundry cost and
noise — into one paragraph, with no blank line between them, so that
paragraph doesn't fully split into separate thoughts. I left it as-is
because dedicated `_laundry.txt` and `_noise.txt` documents already cover
each topic in full elsewhere in the corpus, so the redundancy isn't costing
much; sentence-level splitting inside a paragraph felt like more complexity
than this corpus actually needs.

Re-indexing with this strategy: 183 chunks from 88 documents, averaging 167
characters (shortest 63, longest 397) — more, smaller chunks than the
starter's 88, but each one is a complete point rather than a whole post's
worth of mixed topics.

## Sample Chunks

**Chunk 1** — source: `admin_add_drop_deadline.txt#0` — produced by: `chunker.py::split_documents`

```
On the add/drop deadline

You can add a course through the end of the second week. Dropping is a longer window — through the end of week six — but a drop after week two shows as a W on your transcript. Nothing anywhere on the registrar's site says this plainly, and students find out from each other.
```

**Chunk 2** — source: `course_cs_340_exams.txt#1` — produced by: `chunker.py::split_documents`

```
CS 340 Databases — assessment

Start the term project in week three, not week eight; everyone learns this the hard way.
```

**Chunk 3** — source: `course_phys_130_workload.txt#0` — produced by: `chunker.py::split_documents`

```
Workload for PHYS 130 Mechanics

People keep asking so: 7 hours a week, plus 3 on lab weeks. That's real time, not optimistic time.
```

**Chunk 4** — source: `dining_verrill_street_grill_followup.txt#1` — produced by: `chunker.py::split_documents`

```
Re: Verrill Street Grill

Also worth saying: one register, so the queue is a single line no matter how busy. Nobody tells you this at orientation.
```

**Chunk 5** — source: `housing_morrow_house.txt#1` — produced by: `chunker.py::split_documents`

```
Morrow House — what it's actually like

The good: cheapest housing tier by about $900 a year, and the singles are real singles.
```

Read individually, each one answers exactly one question without needing
anything before or after it: the add/drop penalty date, when to start the
CS 340 project, PHYS 130's weekly time cost, the Verrill Street Grill queue
layout, and Morrow House's main selling point.

## Sample Answer

**Question:** Which orientation week sessions are actually worth attending?

**Answer:**

```
Based on orientation_what_matters.txt, the two orientation week sessions worth going to are the one where you meet your academic adviser and the library walkthrough.

Sources retrieved: dining_halden_hall_followup.txt, dining_north_kitchen_followup.txt, housing_calder_annexe.txt, orientation_what_matters.txt
```

Top-k was 5, which pulled in three chunks that only share loose vocabulary
with the question (two "nobody tells you this at orientation" dining
asides, and a housing chunk about cluster lounges) — the model correctly
ignored all three and answered only from the one chunk that actually
covers orientation sessions, which is the behavior criterion 5 is checking
for.

**My relevance cutoff:** Kept at 0.6. I ran my five test questions and the
five in `OUT_OF_SCOPE`, and the two groups didn't just separate — they left
a wide, clean gap: every in-corpus question landed at 0.436 or below, and
every out-of-scope question landed at 0.787 or above. 0.6 sits almost
exactly in the middle of that gap (the midpoint is 0.61), so I didn't move
it. One thing I got wrong going in: criterion 3's rationale predicted the
ibuprofen-dosage question might land close to `health_center.txt` because
of shared "health" vocabulary. It didn't — it matched `money_textbooks.txt`
instead, and still landed comfortably out of scope at 0.849. The mechanism
I guessed was wrong, but the conclusion (the gate holds) was right anyway.

I also tested a "near miss" the gate lets through: "Is there a minimum GPA
required to stay enrolled full-time?" retrieves
`admin_graduation_requirements.txt` (which discusses credit hours and major
requirements, not GPA) at distance 0.494 — under the cutoff, so the gate
passes it. The second-layer grounding instruction in `generate.py` caught
it anyway: the model answered "I don't have enough information to answer
your question from the provided documents" rather than guessing a plausible
GPA number from its own training data. That's the exact failure mode the
milestone describes, and the existing `GROUNDING_INSTRUCTION` already
handles it, so I left it unchanged rather than tightening something that
wasn't broken.

| Question | In corpus? | Best distance |
|---|---|---|
| When do on-campus job postings open each semester? | Yes | 0.436 |
| What's the maximum number of hours a week I'm allowed to work at a campus job during the semester? | Yes | 0.188 |
| Does income from a work-study job count against my financial aid the same way a non-work-study campus job does? | Yes | 0.117 |
| How quickly do popular courses fill up during registration? | Yes | 0.319 |
| Which orientation week sessions are actually worth attending? | Yes | 0.220 |
| What is the capital of Mongolia? | No | 0.787 |
| How do I change the oil in a diesel engine? | No | 0.923 |
| Who won the 1994 World Cup? | No | 0.847 |
| What is the recommended dosage of ibuprofen for a headache? | No | 0.849 |
| How do I write a for loop in Rust? | No | 0.860 |

## How I Used AI

**1.** In Milestone 2 I gave Claude my five draft test questions before
writing anything into `questions.py`. It checked each one against the actual
documents in `corpora/campus_life/documents/` rather than taking my wording
at face value, and reported that four of five had no single right answer
anywhere in the corpus — "what's the most popular classes to take," for
instance, has no document that ranks courses by popularity, only one that
says popular courses fill within two days of registration opening. It
proposed specific rewrites anchored to real documents (job-posting timing,
the 20-hour work cap, work-study vs. financial aid, orientation sessions
worth attending). I didn't just accept the wording — I had it run each
rewrite through `retrieve` first so I could see the actual distance and
confirm the right document came back before putting it in `questions.py`.

**2.** In Milestone 3 I asked Claude to design the chunking strategy, not
just write code to a spec I'd already decided. It counted paragraphs across
all 88 documents first and reported that 56 of them bundle two distinct
thoughts under one title, before writing `split_documents`. The function it
wrote splits on blank lines and prepends the document's title to every
resulting chunk. When I reviewed the actual output, I caught something it
had flagged but not fixed: a handful of housing pages (`housing_old_
brewhouse.txt` and similar) cram their laundry cost and noise level into one
paragraph with no blank line between them, so those two facts stay fused in
a single chunk instead of splitting. I decided to leave it rather than add
sentence-level splitting, since dedicated `_laundry.txt` and `_noise.txt`
documents already cover each topic in full elsewhere in the corpus — that's
a judgment call I made, not one the tool made for me.

<!-- ── Stretch features ─────────────────────────────────────────────────────
     Doing one? Say so here BEFORE you start. A feature this README never
     claims earns nothing.
     ───────────────────────────────────────────────────────────────────────── -->

---

# Unit 2

<!-- These sections get ADDED to what's already above. Don't delete or rewrite
     unit 1 — the point is that someone can see what you said before you knew
     how it went. -->

## Run Log — Before

Full data: [`results/run_2026-10-03_1630_before.md`](results/run_2026-10-03_1630_before.md),
produced by `run_eval.py::main` (question/answer runs) and
`run_eval.py::check_out_of_scope` (the gate table). No `scorer.py` exists
yet, so the Verdict cells below are my own read of the pasted output, not
an automated judgment. Rate-limit note: the first attempt at this run
crashed mid-way with a 429 from Gemini's free tier (its real cap is 15
requests/minute, below the starter's default `REQUESTS_PER_MINUTE = 30`). I
lowered that to 12 in `config.py` so the built-in pacer waits before hitting
Google's hard limit instead of after. That's a pacing fix to let the test
run at all, not the one improvement this unit asks for — nothing about
retrieval, chunking, or generation changed.

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5 | 5/5 | 5/5 |  |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 |  |
| 4. Chunks hold one complete document, not a fragment of one | 9 of 10 | 10/10 | 10/10 | 10/10 |  |
| 5. The generated answer states the fact, not just retrieves it | 4 of 5 | 4/5 | 4/5 | 4/5 |  |

Criteria 1, 3, and 4 are deterministic (retrieval and chunking don't change
between runs), so the same number appears in all three columns — same
reasoning as the starter's own criterion-3 example, extended to the other
two structural checks. Criteria 2 and 5 depend on the generated answer, and
the three runs really are three separate model calls: the wording changes
run to run even though the pass/fail count doesn't (see the job-postings
question below, where all three runs say something different).

One thing the aggregate number hides: criterion 5's target of 4/5 was hit
in every run, but it's the *same* question missing every time, not a
different one at random. Question 3's `expects` field is `"doesn't count"`;
the model consistently writes `"do not count"` — correct, ungrounded in
nothing, just a different contraction. A literal substring check marks that
a miss in all three runs. Diagnosed properly in Milestone 3 below.

### Criterion 1 — retrieved chunk contains the answer

From `python app.py retrieve "What's the maximum number of hours a week I'm
allowed to work at a campus job during the semester?" --top-k 3`, produced
by `store.py::search`:

```
#   distance   source                           preview
----------------------------------------------------------------------------------------------------
1   0.1880     money_jobs.txt                   On-campus work  Maximum is 20 hours a week during te...
```

### Criterion 2 — every answer names a source

From run 1 of `results/run_2026-10-03_1630_before.md`, produced by
`generate.py::answer_from_chunks`:

```
No, work-study earnings do not count against your financial aid the way ordinary income (from non-work-study campus jobs) does.

Source: admin_campus_jobs_and_financial_aid.txt
```

### Criterion 3 — gate stops out-of-corpus questions

From `results/run_2026-10-03_1630_before.md`, produced by
`run_eval.py::check_out_of_scope` (cutoff 0.6):

```
| Out-of-scope question | Best distance | Gate |
|---|---|---|
| What is the capital of Mongolia? | 0.787 | refused |
| How do I change the oil in a diesel engine? | 0.923 | refused |
| Who won the 1994 World Cup? | 0.847 | refused |
| What is the recommended dosage of ibuprofen for a headache? | 0.849 | refused |
| How do I write a for loop in Rust? | 0.860 | refused |
```

### Criterion 4 — chunks hold one complete document, not a fragment

From `python app.py chunks -n 10`, produced by `chunker.py::split_documents`
— all 10 sampled chunks end on a period, none mix text from more than one
source document:

```
Chunk 3  |  source: course_cs_340_exams.txt#1  |  produced by: chunker.py::split_documents
CS 340 Databases — assessment

Start the term project in week three, not week eight; everyone learns this the hard way.

Chunk 8  |  source: housing_calder_annexe_noise.txt#1  |  produced by: chunker.py::split_documents
Noise levels in Calder Annexe

If you're someone who needs quiet to work, the library is open until 2am during term and that's what most people in this building end up doing.
```

### Criterion 5 — the generated answer states the fact, not just retrieves it

Passing example (run 1 of `results/run_2026-10-03_1630_before.md`, produced
by `generate.py::answer_from_chunks`) — `expects: "first week"`:

```
On-campus job postings open in the first week of each semester (money_jobs.txt).
```

The consistent miss — same run, `expects: "doesn't count"`:

```
No, work-study earnings do not count against your financial aid the way ordinary income (from non-work-study campus jobs) does.

Source: admin_campus_jobs_and_financial_aid.txt
```

## Verdicts

No criterion needed revising — all five turned out to be measurable exactly
as written in `criteria.md`, and none of the targets were loosened to get
here. The one place I considered a revision (criterion 5) turned out not to
qualify, explained below.

| # | Criterion | Verdict | How I decided |
|---|---|---|---|
| 1 | Retrieved chunk contains the answer (4 of 5) | MET | 5/5 in all three runs, and since retrieval doesn't change between runs this is really one measurement, not three. I read the actual chunk text for all five questions (not just the source filename) and each one states the fact being asked about directly — e.g. the chunk for the hours-cap question literally reads "Maximum is 20 hours a week during term." Not close. |
| 2 | Every answer names a source (5 of 5) | MET | Read all 15 answers (3 runs × 5 questions) individually. Every one names a filename, in one of three equivalent formats (parenthetical, "Source:", "according to"). Not close. |
| 3 | Gate stops out-of-corpus questions (4 of 5) | MET | 5/5, refused in one deterministic pass. Caveat worth stating plainly: all five `OUT_OF_SCOPE` questions are wildly off-topic (capital of Mongolia, a Rust for-loop), so this mostly proves the gate catches obvious misses, not that it's well-calibrated at the boundary. The boundary evidence is the GPA question from Milestone 4 of unit 1, which isn't part of this criterion's measurement — it's evidence for criterion 5 instead, since it tests the second grounding layer, not the gate. |
| 4 | Chunks hold one complete document, not a fragment (9 of 10) | MET | 10/10 in the sampled set. I didn't stop at the sample — I checked sentence-boundary integrity across all 183 chunks programmatically, and 0 fail. Not close, and more solid than the milestone asked for. |
| 5 | Generated answer states the fact, not just retrieves it (4 of 5) | MET | 4/5 in all three runs — but it's the *same* question failing every time (work-study vs. financial aid), not a different one each run. Looked hard at whether this deserved a revision instead of a plain MET: the criterion itself is fine and was clearly measurable — "contains the expects phrase" is a literal, checkable string match, and it correctly reported that `"doesn't count"` isn't a substring of `"do not count"`. Nothing about the criterion's wording in `criteria.md` was ambiguous or unmeasurable, so this doesn't qualify as the kind of revision the brief describes. What's actually fragile is `questions.py`'s `expects` field for that one question — a different file, and a result-level problem, not a measurement-level one. That's where I'm taking the one improvement in Milestone 4, not into `criteria.md`. |

## Diagnoses

**I missed nothing — all five criteria came out MET.** Per the brief, that's
a reason to check whether the targets were safe rather than proof the
system is excellent. Going criterion by criterion:

- **Criterion 2** (every answer names a source, 5/5) **was too easy.** I
  already said as much in `criteria.md` when I wrote it: naming a source
  isn't model behavior, it's a fixed part of the prompt template in
  `generate.py` — the model is handed `[from {filename}]` for every chunk
  and told to cite it. The only way to miss this is a formatting bug, and
  there isn't one. This criterion tests `generate.py::build_prompt`, not the
  system's judgment.
- **Criterion 3** (gate stops out-of-corpus questions, 4/5, got 5/5) **was
  also too easy, for a sharper reason.** All five `OUT_OF_SCOPE` questions
  are from a different world entirely — capital of Mongolia, a Rust for-loop
  — and distances for all five landed at 0.787 or higher, far above the 0.6
  cutoff. That's not the gate being well-tuned; it's the test not going near
  the edge where tuning would matter. The real edge case is the GPA question
  I tried in unit 1's Milestone 4 ("Is there a minimum GPA required to stay
  enrolled full-time?", distance 0.494 — *passes* the gate on shared
  vocabulary with `admin_graduation_requirements.txt`, which never actually
  states a GPA). The gate let it through exactly as designed; what caught it
  was the second-layer grounding instruction, a different stage entirely.
  Criterion 3 as written can be met at 5/5 forever without ever testing that
  boundary.
- **Criteria 1 and 4** had real margin but aren't free passes the way 2 and
  3 are — criterion 1 requires reading the actual chunk content and judging
  whether it answers the question (not guaranteed by any code path), and
  criterion 4 I verified against all 183 chunks, not just the sampled 10,
  and still found zero violations. I'd leave both as calibrated correctly
  for this corpus rather than tighten them.
- **Criterion 5 was the one that actually did its job.** It cleared its
  target (4/5) in all three runs, but at exactly the minimum, not with
  margin, and it's the same question failing every time rather than a
  different one at random — that's a real, reproducible finding a looser
  criterion would have hidden.

**Diagnosing the one reproducible sub-failure anyway**, even though it
didn't flip a verdict: "Does income from a work-study job count against my
financial aid the same way a non-work-study campus job does?" has
`expects: "doesn't count"`. All three runs retrieve the correct chunk
(`admin_campus_jobs_and_financial_aid.txt`, distance 0.117 — not close to a
retrieval problem) and the model states the fact correctly every time, but
always as "do **not** count" rather than "does**n't** count" — e.g. run 1:
*"No, work-study earnings do not count against your financial aid the way
ordinary income (from non-work-study campus jobs) does."* Working
backwards through the stages: loading and chunking aren't implicated (the
source sentence itself reads "Work-study earnings don't count against your
financial aid the way ordinary income does" — the chunk already has the
contraction). Embedding and retrieval aren't implicated (0.117 is the best
distance across all ten of my test questions, in or out of scope). The
mechanism is specifically in **generation**: the model is paraphrasing a
true fact into a grammatically-equivalent but lexically-different form, and
the failure only shows up because `questions.py`'s `expects` field does a
literal substring check against one specific contraction. This is a
single-question issue, not a pattern across questions — the other four
`expects` phrases (`"first week"`, `"20"`, `"two days"`, `"academic
adviser"`) are numbers or short fixed noun phrases with effectively one way
to say them, which is exactly why they never had this problem. **Would I
tighten anything?** Yes — criterion 3's out-of-corpus test set is the one
I'd change first if I were revising targets, by swapping in boundary
questions like the GPA one instead of wildly off-topic ones. I'm not making
that change this unit, since the one allowed improvement (Milestone 4) is
better spent on the reproducible sub-failure above, which has a specific,
checkable fix rather than a test-design change.

## The Improvement

**What I changed:** Added one rule to `GROUNDING_INSTRUCTION` in
`generate.py`: *"When a document states a specific fact in exact wording —
a number, a yes/no policy, a named requirement — reuse that wording closely
instead of paraphrasing it into a different phrase with the same meaning."*
Nothing else — not chunking, not retrieval, not top-k, not the gate.

**Why I picked it:** Milestone 3's diagnosis traced criterion 5's one
reproducible failure to generation specifically: loading, chunking,
embedding, and retrieval were all clean for that question (0.117 distance,
correct chunk, every run), but the model paraphrased "don't count" into "do
not count." Hybrid search and a second chunking strategy — the two options
the brief calls most likely to help — are both retrieval-stage fixes, and
my diagnosis found no retrieval-stage problem, so neither would connect to
what I actually found. Tightening the grounding prompt was the option that
matched the stage my diagnosis named.

### Run Log — After

Full data: [`results/run_2026-10-03_1643_after.md`](results/run_2026-10-03_1643_after.md).

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET (unchanged) |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET (unchanged) |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET (unchanged) |
| 4. Chunks hold one complete document, not a fragment of one | 9 of 10 | 10/10 | 10/10 | 10/10 | MET (unchanged) |
| 5. The generated answer states the fact, not just retrieves it | 4 of 5 | 4/5 | 4/5 | 4/5 | MET (unchanged) |

Side-by-side on the one question the change targeted, all three runs each
side (produced by `generate.py::answer_from_chunks`):

| Run | Before | After |
|---|---|---|
| 1 | "No, work-study earnings **do not** count against your financial aid..." | "No. Work-study earnings **don't** count against your financial aid..." |
| 2 | "No, work-study earnings **do not** count against your financial aid..." | "No. Work-study earnings **don't** count against your financial aid..." |
| 3 | "No, work-study earnings **do not** count against your financial aid..." | "No. Work-study earnings **don't** count against your financial aid..." |

Source document (`admin_campus_jobs_and_financial_aid.txt`): *"Work-study
earnings **don't count** against your financial aid the way ordinary income
does."*

**Did it help? Yes and no — and the "no" is the more interesting finding.**
It helped at the mechanism I actually targeted: before the change, the
model expanded the source's contraction into "do not count" in all three
runs, drifting from the source's exact wording; after the change, it
reproduces "don't count" verbatim in all three runs. That's a clean,
repeatable effect, not noise.

It did **not** move criterion 5's measured count — 4/5 before, 4/5 after,
the same question failing both times. Checking why turned up something I
didn't expect: `questions.py`'s `expects` field for this question is
`"doesn't count"`, but the source document has never said that — it says
**"don't count"** (correctly agreeing with the plural subject "earnings"),
and it always has. My own `expects` string was wrong from the moment I
wrote it in Milestone 2, independent of anything the model did. Before the
fix, the model's "do not count" matched neither my `expects` string nor the
source's actual wording. After the fix, the model's "don't count" matches
the source exactly — a real improvement — but still not my `expects`
string, which asks for a contraction the corpus never uses. I'm not editing
`questions.py` to fix that typo this unit; the brief's one rule says the
only change this unit is the improvement, and fixing a test-authoring typo
is a second change, not a continuation of this one. It's recorded here
instead, honestly, as something the measurement found about itself rather
than about the system.

## What's Still Broken

No criterion is in MISSED state, before or after the fix. That's not the
same as nothing being left — the MET verdicts hide four real things I
chose not to touch this unit.

- **Criterion 5, question 3, still doesn't literally match.** The
  improvement made the model's wording *more* correct (it now quotes the
  source's "don't count" exactly instead of drifting to "do not count"),
  but `questions.py`'s `expects` field for that question is `"doesn't
  count"` — a string the source document has never contained. The
  criterion-level count (4/5) hasn't moved and won't, no matter what I do
  to generation, because the thing that's wrong is the test fixture, not
  the system. **What I'd do:** fix the `expects` field to `"don't count"`,
  matching the source's actual wording, and re-run to confirm it flips to
  5/5. **Why I stopped:** the brief's one rule for this unit is one
  change, already spent on the grounding prompt. Fixing a second thing —
  even a one-line typo in a different file — would make this two changes,
  and I'd rather report the typo honestly than quietly patch it on the way
  out.
- **Criterion 3's test set never probes the boundary.** All five
  `OUT_OF_SCOPE` questions are wildly off-topic, so 5/5 proves the gate
  catches obvious misses, not that 0.6 is the right cutoff. The one
  boundary case I have — the GPA question from unit 1 — isn't part of this
  criterion's measurement at all; it passes the gate and gets caught by
  generation instead, which is evidence for criterion 5's territory, not
  criterion 3's. **What I'd do:** replace a couple of the `OUT_OF_SCOPE`
  questions with plausible-but-uncovered campus questions (GPA
  requirements, meal-plan refunds, housing deposit timelines — things that
  share vocabulary with real documents but aren't actually answered
  anywhere) and see whether the gate alone, without the generation
  backstop, still holds. **Why I stopped:** this is a test-design change,
  not a system change, and it wasn't what my Milestone 3 diagnosis pointed
  at — the diagnosis pointed at generation, and I followed it rather than
  chasing a second interesting thread.
- **Criterion 2 tests formatting, not attribution accuracy.** Every answer
  names *a* source, but nothing checks that it's naming the *right* one
  when multiple documents get retrieved. I never caught a case of this in
  three runs, but I also never specifically tried to provoke one.
  **What I'd do:** write a test question where the correct chunk and a
  plausible-but-wrong chunk come from similarly-worded documents, and check
  whether the model cites the one it actually used. **Why I stopped:**
  no diagnosis pointed here — I noticed it while judging criterion 2 in
  Milestone 2, logged it, and left it for a unit where it's the thing
  being tested rather than a tangent.
- **The housing-page laundry/noise merge from unit 1's Milestone 3 is
  still there.** A few housing pages (`housing_old_brewhouse.txt` and
  similar) cram laundry cost and noise level into one unsplit paragraph,
  so that chunk still mixes two facts. **What I'd do:** split on an
  in-paragraph marker like "On noise:" for the handful of documents that
  have it. **Why I stopped:** it's never caused a measured failure —
  dedicated `_laundry.txt`/`_noise.txt` documents retrieve ahead of it for
  the questions that would care — so it stayed a known cosmetic issue
  rather than something worth spending this unit's one change on.

## What I'd Do Differently

**Criterion 5** would change the most. "Contains the expects phrase" bakes
in a single hand-typed string as ground truth, and this unit found that the
string itself can simply be wrong — I wrote `"doesn't count"` without
checking it against the source document's actual contraction. Next time
I'd write the criterion as "the answer states the fact in terms a reader
could verify against the named source," and pull the `expects` phrase by
quoting the source document directly instead of writing it from memory of
the question I'd just asked.

**Criterion 3** I'd redesign, not just reword. Five questions from a
different world entirely was the right call for *finding* a cutoff in
Milestone 4 of unit 1 — the gap had to be obvious before I could place a
number in it. But reusing the same five questions in unit 2 to certify the
gate's behavior tests something much easier than what the gate is actually
for. I'd keep the original five for calibration and add a second, harder
set of boundary questions for the unit-2 test — the GPA-style questions
that share vocabulary with real documents but aren't answered anywhere.

**Criterion 2** I'd leave the target alone but add a second clause about
attribution being *correct*, not just present, since this unit made me
notice the gap between the two without ever actually catching a wrong
citation.

**Criteria 1 and 4** I wouldn't touch. Both took real, specific evidence to
clear (criterion 1 required reading actual chunk content per question;
criterion 4 held up when I checked all 183 chunks instead of the sampled
10), and neither came out MET by construction the way 2 and 3 did.

## How I Used AI — Unit 2

Added to, not replacing, the two moments logged under Unit 1 above.

**3.** In unit 2's Milestone 3, I asked Claude to trace criterion 5's one
reproducible sub-failure through the five pipeline stages. It correctly
diagnosed generation-stage paraphrasing ("don't count" → "do not count")
as the mechanism, backed by the retrieval distance (0.117, the best of any
test question) ruling out the earlier stages. I didn't stop at that
diagnosis — in Milestone 4, after applying the fix, I had it check the
*exact* source document wording character-for-character against both the
before and after answers rather than just checking the `expects` string.
That's what surfaced the more interesting finding underneath: the source
document has always said "don't count," never "doesn't count," so my own
`expects` field (written in unit 1's Milestone 2) was wrong from the start,
independent of anything the model did. The first diagnosis wasn't wrong,
it was incomplete — rereading the primary source instead of trusting my
own prior work caught something the stage-by-stage trace alone didn't.

**4.** When picking this unit's one improvement, I asked Claude to connect
the fix to the diagnosis rather than default to the brief's two
suggestions. It pointed out that hybrid search and a second chunking
strategy both target retrieval, and my diagnosis had found retrieval clean
(same exact chunk, same exact distance, every run), so neither would
actually test the thing that broke. It proposed tightening the grounding
prompt, and specifically wrote the new rule in general terms — "reuse
exact wording for load-bearing facts" — rather than one that named
contractions or this specific question. I checked that choice before
accepting it: a rule narrow enough to only fix "don't count" vs. "do not
count" would have made this one test pass without making the system better
at anything else, which is the kind of change that looks like progress on
a chart and isn't. The general version is what actually went into
`generate.py`.
