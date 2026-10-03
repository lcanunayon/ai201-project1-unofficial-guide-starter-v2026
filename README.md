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

<!-- Three or four sentences. Which corpus you picked, and the kinds of
     questions your system answers. Write it for someone who has never seen
     this repo.

     Milestone 5. -->

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

<!-- Two specific moments. For each: what you asked for, what came back, and
     what you changed about it.

     "I asked Claude to write the chunking function from my notes. It ignored
     the overlap, so I added that myself" is the level of detail we're after.
     "I used AI to help me code" is not.

     Milestone 5. -->

**1.**

**2.**

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

<!-- Your five criteria, three runs each. `python run_eval.py --label before`
     runs the questions, puts the OUT_OF_SCOPE ones through the gate, and
     writes it all into results/ for you. Targets come from criteria.md; the
     verdict column is your call.

     Criterion 3 is measured in one deterministic pass rather than three, so
     the same number goes in all three run columns. That's correct, not lazy.

     Milestone 1. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 |  |  |  |  |
| 2. Every answer names a source | 5 of 5 |  |  |  |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 |  |  |  |  |
| 4. | | | | | |
| 5. | | | | | |

<!-- Underneath, paste the REAL output for each criterion from one of your
     runs — the actual text your system produced, not a description of it.
     Name the file and function that produced it. -->

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
