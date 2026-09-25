# Jev TypeSafe: SAT & Trivia Reasoning Benchmark

## Overview
A follow-up to [Jev TypeSafe API Testing](./jev-typesafe-testing.md), this test probes Jev's `choice`
question type (not just `noul` yes/no) on SAT-style multiple-choice questions and
"Who Wants to Be a Millionaire"-style trivia — including questions specifically designed
to trap fast/intuitive ("System 1") answering.

**Endpoint:** `POST https://api.typesafe.ai/v1/systemone`
**Model:** `jev-latest` (resolved to `jev-1.13.0` at test time)
**Question type used:** `choice` (select one option from named criteria; returns `choice`,
`confidence`, and a `probabilities` map over all options)

---

## Methods

Each question was sent as a single-question `systemone` request with:
- `state`: the question stem (SAT item or trivia question)
- `questions.answer`: a `choice` question with `instructions` describing the task and
  `criteria` mapping option letters (A/B/C/D/E) to their text

Example request:
```json
{
  "model": "jev-latest",
  "state": "A bat and a ball cost $1.10 in total. The bat costs $1.00 more than the ball. How much does the ball cost?",
  "questions": {
    "answer": {
      "type": "choice",
      "instructions": "Select the correct price for the ball.",
      "criteria": {
        "A": "10 cents",
        "B": "5 cents",
        "C": "15 cents",
        "D": "1 cent"
      }
    }
  }
}
```

Six items were tested, chosen to span three categories:
1. **Standard SAT items** — sentence completion, linear algebra, percent-change word problems
2. **Trivia (Millionaire-style)** — general knowledge with plausible distractors
3. **Classic reasoning traps** — questions where the intuitive/fast answer is wrong
   (cognitive reflection test items), to see whether a "System 1"-branded model falls for them

No prompt engineering beyond a one-line `instructions` field was used, and no chain-of-thought
or reasoning was requested — this tests Jev's default fast-judgment behavior, matching its
intended use case.

---

## Results

| # | Question | Category | Correct answer | Jev's answer | Correct? | Confidence |
|---|---|---|---|---|---|---|
| 1 | SAT sentence completion: politician's speech so ___ that supporters fell asleep | SAT verbal | A (soporific) | A | ✅ | 1.00 |
| 2 | 3x + 7 = 22, find 6x + 14 | SAT math | C (44) | C | ✅ | 0.98 |
| 3 | Only country with a non-rectangular national flag | Trivia | B (Nepal) | B | ✅ | 1.00 |
| 4 | Bat & ball: $1.10 total, bat costs $1.00 more than ball | Reasoning trap | B (5¢) | B | ✅ | 0.91 |
| 5 | 20% markup then 20% discount, vs. original price | SAT math (percent trap) | C (4% lower) | C | ✅ | 0.99 |
| 6 | Snail climbs 3ft/day, slides 2ft/night, out of 20ft well on which day? | Reasoning trap (off-by-one) | A (Day 18) | A | ✅ | 0.67 |

**Score: 6/6 correct.**

Full probability distributions were also returned for every question (not just the top
choice), e.g. for the bat-and-ball question: `{A: 0.07, B: 0.93, C: 0.0, D: 0.0}` — showing
the model assigned meaningful probability to the classic wrong-but-intuitive answer (A, 10¢)
without selecting it.

---

## Round 2: Reading Comprehension & Obscure Trivia

A second round pushed into harder territory: an SAT-style reading-comprehension passage
with multiple questions per passage, and higher-tier ("$500K"/"$1M") Millionaire-style
trivia, including one deliberately flawed question used to probe how the model handles
genuine ambiguity.

| # | Question | Category | Correct answer | Jev's answer | Correct? | Confidence |
|---|---|---|---|---|---|---|
| 7 | 19th-century urban-planning passage: author's primary argument re: pedestrians vs. cars | SAT reading comprehension | C (both improves quality of life and reduces congestion) | C | ✅ | 1.00 |
| 8 | Same passage: author's tone toward current policy | SAT reading comprehension | critical | critical | ✅ | 1.00 |
| 9 | Only mortal Gorgon sister in Greek mythology | Trivia ($500K tier) | A (Medusa) | A | ✅ | 1.00 |
| 10 | "Only" Shakespeare title character who dies onstage (flawed premise — several qualify) | Trivia ($1M tier, deliberately ambiguous) | *no single correct answer* | B (Julius Caesar) | N/A | 0.51 |
| 11 | Element with chemical symbol W | Trivia ($1M tier) | A (Tungsten) | A | ✅ | 1.00 |
| 12 | What vexillology studies | Trivia ($1M tier) | A (Flags) | A | ✅ | 1.00 |

**Score: 5/5 on well-formed questions correct** (item 10 excluded — see below).

**Item 10 note:** this question was written with a false premise — Hamlet, Julius Caesar,
and King Lear all die onstage in their respective plays, so "the only" one is not actually
true. Rather than answer confidently, Jev split its probability mass across three plays
(`B: 0.63, C: 0.14, D: 0.13, A: 0.10`) and reported only 0.51 overall confidence — its
lowest confidence of the entire test. This is evidence the model is sensitive to genuine
ambiguity in a question and signals it via confidence/probability spread rather than
picking an answer with false certainty.

The passage-based test (items 7–8) also confirms `state` can hold long-form multi-paragraph
content, and a single request can carry multiple named `questions` against the same passage,
with each answer returned under its own key.

---

## Key Observations

✅ **Strengths:**
- **Resisted classic cognitive-bias traps**: correctly answered both the bat-and-ball problem
  and the compound-percentage problem, the two questions specifically designed to lure a
  fast/intuitive answerer into a wrong-but-obvious choice
- **Well-calibrated confidence**: confidence was high (0.91–1.00) on single-step items and
  markedly lower (0.67) on the snail problem, the one item requiring genuine multi-step
  iterative reasoning — confidence tracked difficulty rather than being uniformly high
- **Full probability transparency**: returns probabilities for every option, not just the
  winner, useful for flagging low-margin answers for human review
- **`choice` type works well for closed-form MCQ/trivia**, complementing the `noul` (yes/no)
  type explored in the earlier classification test
- **Handles multi-paragraph passages**: `state` accepted a full reading-comprehension
  passage, with multiple `questions` keys answered correctly against the same context in
  one request
- **Confidence tracks genuine ambiguity, not just difficulty**: given a trivia question with
  a false "only one correct answer" premise, confidence dropped to 0.51 (its lowest of the
  whole test) with probability spread across the plausible candidates, rather than
  confidently committing to one — this is the behavior you want from a judge model

⚠️ **Considerations:**
- Sample size still modest (12 items across two rounds); no formal accuracy benchmark
  (e.g. full SAT practice sets) run yet
- Only one obscure/flawed trivia item tested so far — worth confirming the ambiguity-detection
  behavior holds across more genuinely hard or ill-posed questions, not just this one case
- Reading comprehension tested with a single passage/2 questions; unclear how it scales to
  longer passages or more questions per passage

---

## Next Steps

- [x] Test SAT reading-comprehension passages (long `state`, multiple `choice` questions per passage)
- [x] Push into harder Millionaire-ladder trivia (obscure/high-difficulty questions)
- [ ] Try the `score` question type on a rubric-graded task
- [ ] Run a larger, more systematic accuracy benchmark against a public SAT practice set
- [ ] Compare `jev-latest` vs `jev-preview` on the same item set
- [ ] Test longer passages with more questions per passage
- [ ] Probe ambiguity-detection behavior with more flawed/trick questions to see if it's consistent
