# Jev TypeSafe API Testing

## Overview
Jev is TypeSafe AI's "System One" model API for fast semantic evaluation of text against yes/no questions. It's designed to classify, evaluate, and understand intent from unstructured text.

**Endpoint:** `POST https://api.typesafe.ai/v1/systemone`  
**Model:** `jev-latest` (currently `jev-1.13.0`)  
**Authentication:** Bearer token in `Authorization` header

---

## How It Works

### Basic Request Structure
```json
{
  "state": "The text you want to evaluate",
  "model": "jev-latest",
  "questions": {
    "question_key": {
      "type": "noul",
      "instructions": "Your question here"
    }
  }
}
```

### Response Format
```json
{
  "model": "jev-1.13.0",
  "answers": {
    "question_key": {
      "type": "noul",
      "noul": 0.95
    }
  },
  "usage": {
    "input_tokens": 303,
    "output_tokens": 43
  }
}
```

**Key points:**
- `noul` value: 0 = no, 1 = yes (continuous 0–1 scale)
- Scores above 0.5 indicate "yes", below 0.5 indicate "no"
- Fast evaluation with reasonable token usage

---

## Test Results

### Test 1: High Urgency (Crisis)
**Input:** "The system is down and customers cannot access their accounts. This happened 30 minutes ago."

| Question | Score | Interpretation |
|----------|-------|-----------------|
| is_urgent | 0.96 | Correctly identified as urgent |
| is_customer_impacting | 0.98 | Correctly identified as customer-facing |

### Test 2: Routine Task (Low Urgency)
**Input:** "Reminder: team standup is at 3pm today"

| Question | Score | Interpretation |
|----------|-------|-----------------|
| is_urgent | 0.23 | Correctly identified as non-urgent |

### Test 3: Mixed Sentiment
**Input:** "The new dashboard looks great but it's missing the export feature"

| Question | Score | Interpretation |
|----------|-------|-----------------|
| is_positive | 0.55 | Neutral-to-slightly-positive (captured the mixed tone) |
| is_feature_request | 0.88 | Correctly identified as a feature request |

### Test 4: Ambiguous / Non-Actionable
**Input:** "We might need to consider looking into potential improvements"

| Question | Score | Interpretation |
|----------|-------|-----------------|
| is_actionable | 0.15 | Correctly flagged as vague and not actionable |

---

## Key Observations

✅ **Strengths:**
- **Nuanced understanding**: Handles mixed sentiment and context well
- **Fast & efficient**: Low token usage (303 tokens for a 2-question request)
- **Precise calibration**: 0.23 vs 0.96 shows good discrimination between urgent and routine
- **Handles ambiguity**: Correctly identifies vague statements as non-actionable

⚠️ **Considerations:**
- Binary classification only (noul type) — no multi-choice or open-ended responses
- No explanation for scores, just numerical values
- Requires explicit instructions in each question for consistent evaluation

---

## Use Cases

1. **Ticket/Issue Triage**: Classify incoming support tickets by urgency, customer impact, spam
2. **Sentiment Analysis**: Evaluate customer feedback, product reviews
3. **Intent Detection**: Identify feature requests, bug reports, questions in text
4. **Content Moderation**: Flag off-topic or policy-violating content
5. **Priority Routing**: Automatically route high-urgency items to senior teams

---

## Next Steps

- [ ] Test on more domain-specific content (e.g., code reviews, legal docs)
- [ ] Measure accuracy against human judgments
- [ ] Compare latency with other classification APIs
- [ ] Explore batch evaluation if available
- [ ] Test error handling and edge cases (very long texts, special characters)
