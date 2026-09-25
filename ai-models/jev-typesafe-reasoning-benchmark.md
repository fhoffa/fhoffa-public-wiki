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

## Round 3: "Beat Jev" — a browser game

The 14 well-formed items from rounds 1–2 (minus the flawed Shakespeare question, plus a few
new ones for variety — vocab, probability, a syllogism trap, a riddle) were consolidated into
a fixed dataset and built into a playable browser game:

**[Beat Jev](https://claude.ai/artifact/UXkfKUS4tVW7HLzFTXtmp2)** — race Jev on the same 14
questions, item by item. Your reaction time is measured from when the question appears to
when you click an answer; Jev's time and confidence are real numbers captured from the API
calls below, not simulated. At the end you get a head-to-head: accuracy and total time, you
vs. Jev, plus a full question-by-question breakdown.

**Methods for the game data:** each of the 14 questions was sent once to `POST
/v1/systemone` (model `jev-latest`) as a `choice` question, with client-side wall-clock
timing (Python `time.perf_counter()` around the HTTP call) recorded alongside the response.
The baked dataset stores, per question: the correct answer, Jev's choice, its confidence, and
its latency in milliseconds. The game replays this fixed, pre-measured data against a live
human timer rather than calling the API per play — Jev's API key isn't something a public
static page can hold safely, and pre-baking keeps every player's "opponent" identical.

**Result on this batch: Jev went 14/14, average latency 302ms** (fastest 249ms, slowest
652ms on the one sentence-completion item). Full per-item data:

| Question | Category | Difficulty | Jev correct? | Confidence | Latency |
|---|---|---|---|---|---|
| Critic's review sentence completion | SAT Verbal | easy | ✅ | 1.00 | 652ms |
| Vexillology studies... | Trivia | easy | ✅ | 1.00 | 266ms |
| Element symbol W | Trivia | easy | ✅ | 1.00 | 285ms |
| Wrote *Pride and Prejudice* | Trivia | easy | ✅ | 1.00 | 318ms |
| 3x+7=22, find 6x+14 | SAT Math | medium | ✅ | 0.99 | 330ms |
| Only mortal Gorgon sister | Trivia | medium | ✅ | 1.00 | 269ms |
| Only non-rectangular flag | Trivia | medium | ✅ | 1.00 | 269ms |
| Planet with day longer than its year | Trivia | medium | ✅ | 1.00 | 249ms |
| 20% markup then 20% discount | SAT Math | hard | ✅ | 0.98 | 254ms |
| Marble probability (no replacement) | SAT Math | hard | ✅ | 0.95 | 279ms |
| Bat & ball classic trap | Reasoning Trap | hard | ✅ | 0.93 | 269ms |
| Syllogism trap (Bloops/Razzies/Mots) | Logic | hard | ✅ | 1.00 | 255ms |
| Snail-in-the-well | Reasoning Trap | very hard | ✅ | 0.60 | 280ms |
| "What has keys but no locks" riddle | Riddle | very hard | ✅ | 0.98 | 257ms |

**On the Haiku column:** the game ships a "TBD" placeholder for Claude Haiku rather than a
number. A quick attempt to source Haiku's answers via a subagent call showed ~11 seconds of
overhead per question — almost entirely agent-framework spin-up, not model inference time —
which would make for a misleading speed comparison against Jev's clean ~300ms API responses.
Filling in Haiku (and possibly other models) properly needs a real head-to-head setup: direct
API calls to each model with consistent, isolated timing, plus token cost per answer. That's
tracked as a to-do below rather than guessed at now.

---

## Round 4: Position Bias & a Real Miss

Two follow-up questions prompted this round: **had Jev actually gotten anything wrong yet**,
and **were the 14 game questions written with a bias toward certain answer letters?**

The second question turned out to have an uncomfortable answer. Tallying the correct-answer
letter across all 14 game questions:

| Letter | Count |
|---|---|
| A | 4 |
| B | 8 |
| C | 2 |
| D | 0 |
| E | 0 |

The correct answer was **B or A in 12 of 14 questions**, and **never D or E**. That's a bias
in how the questions were authored, not evidence about Jev — but it meant the earlier 14/14
score couldn't rule out Jev simply leaning toward early letters rather than reasoning about
content.

**Method:** each of the 14 questions was resent to `/v1/systemone` with its answer choices
shuffled and the correct one deliberately relocated to **D** (or E for the 5-option vocab
question), keeping the question text and all four/five answer texts identical. If Jev were
pattern-matching letter position, moving the right answer to a chronically under-tested slot
should hurt it.

**Result: 13/14 still correct.** Most questions were unaffected — same answer, similar or
even higher confidence, regardless of which letter the correct text was assigned to. That's
evidence the model is generally keying off content, not letter position.

**But one question flipped: the algebra item ("If 3x + 7 = 22, what is 6x + 14?").**

| Ordering | Criteria | Jev's answer | Correct? | Confidence |
|---|---|---|---|---|
| Original | A: 22, B: 30, C: **44**, D: 50 | C | ✅ | 0.99 |
| Rotated | A: 30, B: 22, C: 50, D: **44** | A | ❌ | 0.59 |

Rerunning the rotated version 5 more times gave: **A, D, D, A, A** (probabilities each time
sitting close to 50/50, e.g. `{A: 0.55, D: 0.45}`, `{A: 0.47, D: 0.53}`, `{A: 0.59, D: 0.41}`).
Across 6 total calls on the rotated ordering, Jev answered correctly only twice. The likely
mechanism: "30" is what you get from solving 3x+7=22 → x=5 → 6x=30, then forgetting to add
the +14 — a genuine, plausible arithmetic slip, not a random error. What changed between the
two tables isn't the math, only which answer text sits behind which letter and where the
"30" decoy sits relative to it — yet it flipped the model between confidently right (0.99)
and a coin flip (~0.5) on the exact same underlying problem.

**Answering the original questions directly:**
- **Yes, Jev has now gotten something wrong** — reliably, under a specific answer ordering,
  it's close to a coin flip on the 3x+7=22 problem, and that's a real miss, not a fluke of
  one unlucky call.
- **The apparent "always A or B" pattern was a test-construction artifact**, not a property
  of Jev — rotating the correct answer to D held for 13 of 14 questions with high confidence.
  The one question that broke wasn't the one Jev was "biased toward getting right by
  position" — it was a question already sitting near its own margin of confidence, where
  surface-level answer framing was enough to tip it either way.

The **Beat Jev** game has since been updated (v2) to render each question's answer choices in
a random order every time it's played, so no fixed letter position is ever favored for human
players either.

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
- **Mostly robust to answer-order shuffling**: relocating the correct answer to the
  least-tested letter position (D/E) held correct on 13 of 14 questions, evidence the model
  is generally reasoning about content rather than pattern-matching letter position

⚠️ **Considerations:**
- Sample size still modest; no formal accuracy benchmark (e.g. full SAT practice sets) run yet
- Only one obscure/flawed trivia item tested so far — worth confirming the ambiguity-detection
  behavior holds across more genuinely hard or ill-posed questions, not just this one case
- Reading comprehension tested with a single passage/2 questions; unclear how it scales to
  longer passages or more questions per passage
- **Not order-invariant on at least one item**: the 3x+7=22 algebra question swung from
  confidently correct (0.99) to a near coin-flip (~0.5, wrong more often than not across 6
  runs) purely from reordering which letter the correct answer sat behind. Confidence dropping
  alongside the flip is the right behavior — it wasn't confidently wrong — but it shows the
  model's answer to an *objectively fixed* math problem isn't fully stable to surface
  presentation. Worth checking whether this is isolated to borderline-confidence items or a
  broader pattern before trusting `choice` answers on arithmetic without a confidence-based
  review threshold

---

## Next Steps

- [x] Test SAT reading-comprehension passages (long `state`, multiple `choice` questions per passage)
- [x] Push into harder Millionaire-ladder trivia (obscure/high-difficulty questions)
- [x] Build a playable "Jev vs. human" game ([Beat Jev](https://claude.ai/artifact/UXkfKUS4tVW7HLzFTXtmp2))
- [x] Check for answer-letter position bias in the test set, and whether Jev is order-invariant
- [ ] **Proper Haiku (and other model) benchmark: speed, cost, and accuracy on the same 14-question set**,
      measured via direct API calls with isolated timing (not agent-framework overhead) —
      needed before the game's Haiku column can show real numbers
- [ ] Investigate the algebra-question instability further: is it specific to that item, to
      near-50/50-confidence items generally, or to arithmetic questions with a plausible
      "forgot the last step" distractor? Try a few more order-rotated arithmetic questions
      to see if the pattern generalizes
- [ ] Try the `score` question type on a rubric-graded task
- [ ] Run a larger, more systematic accuracy benchmark against a public SAT practice set
- [ ] Compare `jev-latest` vs `jev-preview` on the same item set
- [ ] Test longer passages with more questions per passage
- [ ] Probe ambiguity-detection behavior with more flawed/trick questions to see if it's consistent
