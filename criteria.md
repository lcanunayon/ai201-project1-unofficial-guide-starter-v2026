# Acceptance criteria — The Unofficial Guide

Five criteria that say what "working" means for this system, written in unit 1
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"Retrieval works"* is an opinion. *"For at
least 4 of my 5 test questions, the top results include a chunk containing the
answer"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter or looser one. A reason that says something about your corpus or your
pipeline earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## 1. Retrieved chunks contain the answer

For at least 4 of my 5 test questions, the retrieved chunks include one that
contains the answer.

**Why this target:** Four of my five questions each have exactly one document
that states the fact being asked about (job posting timing, the 20-hour work
cap, the work-study vs. financial-aid distinction, registration fill speed),
so I expect those to be easy for even a basic embedding search. The fifth
(orientation sessions worth attending) sits in a document that shares
vocabulary — weeks, schedules, "worth it" framing — with a few unrelated docs
(health center hours, the club fair), so I'm leaving room for exactly one
near-miss rather than claiming 5 of 5 before I've seen it work.

---

## 2. Every answer names a source

Every answer the system produces names at least one source document.

**Why this target:** This is the one criterion I'm setting at 5 of 5, not 4.
Naming a source isn't something the model has to get right content-wise — it's
a fixed part of the output format (`app.py` prints "Sources retrieved: ..."
from the retrieved chunks' metadata whenever the gate lets a question
through), so it should either happen every time the gate passes a question or
never. If even one answer comes back without a source, that's a formatting
bug in my code, not statistical noise, and I want the target to say so.

---

## 3. The relevance gate stops out-of-corpus questions

When I ask a question my documents clearly don't cover, the relevance gate
stops it and the system returns "I don't have enough information about that" —
in at least 4 of 5 tries.

<!-- The five questions are the ones in `OUT_OF_SCOPE` at the bottom of
     `questions.py`, and `run_eval.py` puts them through the gate and writes
     what happened into your run log. Swap them for your own if you'd rather —
     just keep five of them, or the "4 of 5" above has nothing to be 4 of. -->

**Why this target:** I haven't measured the actual gap between in-corpus and
out-of-corpus distances yet — that's Milestone 4 — so I don't know whether
all five `OUT_OF_SCOPE` questions will land cleanly above the cutoff. A
question like the ibuprofen-dosage one touches a topic (health) that
`health_center.txt` also discusses, just from a completely different angle,
so it could plausibly land closer to in-scope territory than the other four.
4 of 5 leaves room for exactly one such accident without hiding a gate that's
actually broken.

---

## 4. Chunks hold one complete document, not a fragment of one

At least 9 of the 10 chunks printed by `python app.py chunks -n 10` end on a
real sentence boundary (a period or question mark), not mid-word or
mid-clause, and no chunk contains text from more than one source document.

**Why this target:** campus_life documents average about 317 characters and
usually pack their answer into a single sentence (Milestone 1's reading —
the housing lottery post, the grade appeals post, and the dining dollars post
were each one sentence doing all the work). At the default 800-character
chunk size, most documents should fit in one chunk whole, so a cut-off
sentence here wouldn't be a borderline judgment call — it would mean the
chunker split something that didn't need splitting. I'm not asking for 10 of
10 because a handful of documents (the course pages) run longer and could
legitimately need a real split somewhere.

---

## 5. The generated answer states the fact, not just retrieves it

For at least 4 of my 5 test questions, the final answer text — not just the
retrieved chunk — contains the `expects` phrase from `questions.py`.

**Why this target:** Criterion 1 only checks that the right chunk gets
retrieved. That's necessary but not enough: the model could retrieve
`money_jobs.txt` and still answer the hours-cap question by talking about
job-posting timing instead of the "20" I actually need. This criterion
checks the thing I actually care about — that the answer a student reads
contains the specific fact, not just that the right document was nearby. I'm
keeping it at 4 of 5 for the same reason as criterion 1: the orientation
question is the one I'd bet against first.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 2 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 1. Retrieved chunks contain the answer

         For at least 4 of my 5 test questions, the retrieved chunks include
         one that contains the answer.

         **Why this target:** ...

         > **Revised in unit 2:** For at least 4 of 5 questions, the top three
         > results contain the answer.
         >
         > **Why revised:** I couldn't judge "the chunks include one that
         > contains the answer" the same way twice — I scored two questions
         > differently on Monday than on Wednesday. The new version is
         > something I can actually check.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said 4 of 5 but got 2 of 5, so 2 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.

     The whole reason the originals stay visible is so someone can see what you
     said before you knew the answer.
     ───────────────────────────────────────────────────────────────────────── -->
