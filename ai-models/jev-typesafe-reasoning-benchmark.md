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

⚠️ **Considerations:**
- Small sample (n=6); no formal accuracy benchmark (e.g. full SAT practice sets) run yet
- All items were "single-hop" — no multi-paragraph SAT reading-comprehension passages tested yet
- Not clear how performance holds up on truly obscure trivia (upper Millionaire ladder,
  $500K–$1M tier) vs. these more solvable examples

---

## Next Steps

- [ ] Test SAT reading-comprehension passages (long `state`, multiple `choice` questions per passage)
- [ ] Push into harder Millionaire-ladder trivia (obscure/high-difficulty questions)
- [ ] Try the `score` question type on a rubric-graded task
- [ ] Run a larger, more systematic accuracy benchmark against a public SAT practice set
- [ ] Compare `jev-latest` vs `jev-preview` on the same item set
