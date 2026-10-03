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

<!-- MET or MISSED for each of the five, against the target you wrote last
     unit — not a new one. Plus a sentence on how you decided. That sentence
     matters most where it was close.

     If your target said 4 of 5 and your runs came out 4, 3, 4, that's a MISS.
     The target has to hold, not show up occasionally.

     Milestone 2. -->

| # | Criterion | Verdict | How I decided |
|---|---|---|---|
| 1 |  |  |  |
| 2 |  |  |  |
| 3 |  |  |  |
| 4 |  |  |  |
| 5 |  |  |  |

## Diagnoses

<!-- For each miss: which stage caused it, and how. The stage alone isn't
     enough — you need the mechanism.

     Not a diagnosis: "Question 3 didn't work."
     A diagnosis:     "Question 3 asks about laundry costs. The answer is in
                       one sentence that got split across two chunks, so
                       neither chunk on its own contains it."

     The five stages: loading → chunking → embedding → retrieval → generation.

     Look for a pattern. If three misses all ask about numbers, that's one
     problem, not three.

     Missed nothing? Say so, then say honestly whether your targets were set
     low, and which one you'd tighten and to what.

     Milestone 3. -->

## The Improvement

**What I changed:**

**Why I picked it:**

<!-- Connect it to a specific diagnosis above in one sentence. If you can't,
     you picked a fix because it sounded impressive. -->

### Run Log — After

<!-- Same format, same five criteria, three runs each.
     `python run_eval.py --label after` -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 |  |  |  |  |
| 2. Every answer names a source | 5 of 5 |  |  |  |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 |  |  |  |  |
| 4. | | | | | |
| 5. | | | | | |

**Did it help?**

<!-- Say plainly whether it did, and how you know. If it made things worse,
     say that — a change that backfired, honestly reported, earns full credit
     and is more interesting than one that worked. What matters is that you can
     tell.

     Milestone 4. -->

## What's Still Broken

<!-- For each criterion still missed after your fix: what you'd do about it,
     and why you stopped where you did.

     "I ran out of time" is fine if it's true. Pretending nothing is left is
     not.

     Milestone 5. -->

## What I'd Do Differently

<!-- Knowing what you know now — which of your five criteria would you write
     differently, and why?

     Milestone 5. -->
