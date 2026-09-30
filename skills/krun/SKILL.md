---
name: krun
license: Apache-2.0
description: >
  Design and build software that makes fast, typed semantic decisions with Krun
  (the Krun One decision model, the /v1/decide API, the krun-ai Python SDK and
  the @krun-ai/sdk TypeScript SDK). Use when application code needs a judgment
  with probabilities over text, images, documents or audio: routing,
  classification, intent detection, tool or agent selection, verification,
  policy and moderation checks, scoring, ranking, triage, or deciding when to
  escalate to a human or a larger model. Also use when an LLM call only picks a
  label or answers yes/no, or when deciding where uncertainty should change
  application behavior. Krun answers choice, noul, score and multi questions
  that code composes; it does not generate text.
---

# Krun

Krun is a decision API. You send a `context` and up to 16 typed questions; Krun One scores the options you defined and
returns, per question, a structured answer with probabilities. It never writes free text, so every answer is one of
your ids, a number, or a list of your ids.

Hold on to this split when you design with it:

- **Code owns the workflow**: control flow, exact rules, lookups, arithmetic, thresholds, side effects.
- **Krun owns the semantic judgment** that code can't make on its own: what a message is about, whether a claim holds,
  how severe something is, which tool fits.

Treat Krun as a set of small semantic primitives you call from ordinary code, not as a classifier bolted on the side.

## Read the live docs before writing integration code

The docs at `https://docs.krun.ai` are the source of truth. Limits, SDK versions and field names change; this skill
teaches design, the docs pin the contract. Start with the index, then read only the pages the task needs. Every page
has a Markdown version (append `.md`).

| Task | Read |
|---|---|
| Find any page | https://docs.krun.ai/llms.txt |
| What Krun is for, and when to use an LLM instead | https://docs.krun.ai/concepts/decision-models.md |
| Pick a primitive | https://docs.krun.ai/concepts/decision-primitives.md, then `concepts/primitives/{choice,noul,score,multi}.md` |
| Interpret numbers | https://docs.krun.ai/concepts/confidence-and-probabilities.md and https://docs.krun.ai/concepts/abstention.md |
| Images, documents, audio | https://docs.krun.ai/concepts/multimodal-inputs.md and https://docs.krun.ai/api-reference/assets.md |
| Several questions in one call | https://docs.krun.ai/guides/multiple-questions.md |
| Intent or tool routing | https://docs.krun.ai/guides/intent-routing.md, https://docs.krun.ai/guides/tool-routing.md |
| Python / TypeScript | https://docs.krun.ai/sdks/python.md, https://docs.krun.ai/sdks/typescript.md |
| Raw HTTP schema | https://api.krun.ai/openapi.json |
| Errors, retries, timeouts | https://docs.krun.ai/api-reference/errors.md, https://docs.krun.ai/guides/production-best-practices.md |
| What doesn't work well yet | https://docs.krun.ai/limitations.md |

If this skill and the docs disagree, follow the docs.

## When Krun fits

Reach for Krun when the outcome space is known at request time and code needs to act on it: routing a ticket, picking
a tool or sub-agent, detecting intent, checking a policy condition, verifying that an answer is supported by evidence,
rating urgency or quality, tagging a document, triaging an image or voicemail, deciding whether a request needs
retrieval, a retry, a stronger model or a person. These are examples, not the boundary: any place where code would
otherwise ask "which of these?", "is this true?", "how much?" or "which of these apply?" about unstructured input.

## When it doesn't

Krun does not replace a generative model or ordinary code for:

- writing replies, articles, summaries or code; extracting free-form field values;
- open-ended research, browsing, or multi-step reasoning;
- exact rules, database lookups, arithmetic, date math, string matching.

In those workflows Krun can still be one bounded step: route the request before the LLM runs, decide whether expensive
reasoning is needed, or check a property of the generated output. Asked for a 2,000-word blog post, use a text
generator; at most use Krun to check the draft (for example a `score` for tone fit, or a `noul` for a policy
violation).

## Design backwards from behavior

1. **What should the application do?** List the actions: route to queue X, call tool Y, block, ask the user, escalate.
2. **What judgment does code need to pick an action?** Write it as one question about the input.
3. **Which primitive expresses it?** See below.
4. **What context does the judgment need?** Only that.
5. **How does code act on uncertainty?** Decide thresholds and the fallback path before shipping.

The option sets, policy text and level descriptions encode the product's rules. If the user hasn't given them (the real
queue names, the actual refund policy), ask, or propose a draft and mark it as one; don't invent them silently.

Keep each question to one coherent judgment ("Is the customer asking for a refund?"), not a bundle ("angry and wants a
refund and is a VIP?"). Narrow is not the same as trivial: "Does the retrieved passage support the claim?" is one
judgment that code cannot make. Anything code can decide exactly, such as account tier, order age or amount, stays in
code.

## Primitives

| Primitive | Question it answers | Answer |
|---|---|---|
| `choice` | Which **one** of these unordered options? | `choice` (an option id, or `null` when it abstains), `probabilities` summing to ~1, `confidence`, `abstain`, `abstention_status` |
| `noul` | Does this yes/no proposition hold? | `noul`: the probability it holds |
| `score` | Where on this ordered scale? | `score`: the expected level index; `probabilities` per level; `legend`; `confidence` |
| `multi` | Which of these apply (any number)? | `values` (may be empty) and an independent probability per option |

How to choose:

- Exactly one of several alternatives with no order (department, intent, tool, document type): `choice`.
- A single condition that may or may not hold (needs human review, violates policy, claim supported): `noul`. Prefer it
  to a two-option `choice`: one probability is easier to threshold.
- A degree with a natural order (urgency, severity, quality, risk, relevance): `score`. Write levels as concrete
  descriptions, lowest first, not `"low"/"high"` or digits. Order carries the meaning.
- Zero or more labels from one set (issue types in a complaint, elements present in a document): `multi`. Its
  probabilities are independent and don't sum to 1.
- `multi` versus one `noul` per label: use `multi` when the labels share one selection instruction; use separate
  `noul` questions when each condition needs its own wording, `criteria`, or threshold (for example critical policy
  checks).
- `choice` versus `multi`: if two labels can both be true, it is not a `choice`.

Writing questions well:

- **Option ids are your code's ids.** They come back verbatim. Use readable ids like `lost_or_stolen_card`.
- **Label-only versus described options.** Intent `choice` questions whose options have empty descriptions (`""` or
  `null`) get `abstention_status: "calibrated"`; with descriptions, or with `task_type: "tool"`, abstention is
  `advisory`, a hint only. Start label-only with self-explanatory ids (`transaction_charged_twice`, not `q3`); add
  descriptions when ids can't carry the meaning or labeled samples show confusions, and then rely on your own
  thresholds rather than `abstain`.
- **Tools.** Set `task_type: "tool"` when options are tools or functions; use the real tool names as ids; offer only the
  tools available at that step (2 to 64 options per question). Krun picks the tool, it does not fill in arguments.
  Handle "no tool fits" in code: on `abstain` or a low `probabilities[choice]`, answer directly or ask the user.
- **Propositions.** Phrase `noul` instructions about the context ("Does the message mention a delivery date?"); add
  `criteria` when the true/false boundary is a policy choice.
- **Option sets are part of the model input.** Adding, removing or rewording options shifts every probability. Version
  them, and re-check accuracy after a change.

## Context

Send only the state the judgment depends on: the user message, the candidate answer and its evidence, the relevant
policy excerpt. Leave out IDs, metadata and history that don't bear on the question; they add tokens and noise.

`context` is either a string or a list of content parts. With several pieces, use `text` parts to label them in reading
order ("Customer message:", "Refund policy:") instead of inventing your own markup.

Images, documents and audio go through assets:

```text
upload the file (POST /v1/assets, SDK assets.create) → asset id → {"type": "image" | "document" | "audio", "asset_id": ...} part in context
```

Krun reads text-layer documents as text, runs OCR and vision on scans and screenshots, and transcribes audio. Don't add
your own OCR, speech-to-text or captioning step in front of it. Assets are project-scoped and expire after 24 hours;
there is no URL or inline-bytes field. There are per-request caps on media parts, file types, sizes, pages and audio
length, and video is not supported: check the multimodal page before designing a pipeline. Abstention on media inputs is
advisory.

## Probabilities and uncertainty

A typed answer is not a verified answer. Code decides what each level of uncertainty means for this product.

- `choice.confidence` is the margin between the top two probabilities, **not** the probability of being right. For the
  model's estimate of the chosen option, read `probabilities[choice]`.
- When `abstain` is true, `choice` is `null`. Never silently replace it with the top option; decide explicitly.
- `noul` is the probability itself. There is no universal cutoff: set one per question from the cost of a false yes
  versus a missed yes, and tune it on your own labeled traffic.
- `score` is an expected level, so you can sort, average or threshold it; `score.confidence` says how concentrated the
  distribution is.
- Calibration was measured on Krun's evaluation data. Validate thresholds on a sample of real inputs, and treat
  thresholds in docs and examples as starting points.
- Without labeled data yet, start conservatively: run Krun in shadow mode or send everything above a low bar to review,
  log answers with the request id, record the correct outcome (and send it as feedback), then set thresholds from that
  sample.

Escalation is application policy on top of a Krun judgment:

```text
Krun answer → code reads probability / abstain → policy
  ├─ clear and low-risk      → act automatically
  ├─ ambiguous to the user   → ask a clarifying question
  ├─ uncertain or high-stakes → human review (pass the top options along)
  └─ needs open-ended work   → hand to a larger reasoning model
```

Irreversible actions (payments, deletions, emails) deserve a stricter threshold or a confirmation step, especially for
tool routing, where abstention is only advisory.

## Latency

- **Batch independent judgments.** All questions about the same context go in one request (up to 16, any mix of types).
  That is one round trip and one model job, and each question is still scored independently.
- **Split only for real dependencies.** Use a second request when question B depends on A's answer (pick a category,
  then an intent inside it) or is about a different context.
- **Parallelize across contexts.** For many items, send concurrent requests (for example `AsyncKrun` with
  `asyncio.gather`, or `Promise.all`), within your rate limit.
- **Plan for cold starts.** The model scales to zero; the first request after idle can take much longer than a warm
  one. Keep client timeouts at the value the docs recommend (the 0.3.0 SDK defaults follow it), retry
  `UPSTREAM_TIMEOUT`, and give interactive flows a fallback path instead of blocking the user.
- **Measure, don't assume.** Track end-to-end p50/p95 from your side, input tokens (`usage.input_tokens`; there are no
  output tokens), accuracy through feedback, and abstention rate per question. A rising abstention rate often means
  traffic drifted or an option is missing.

## Composing with generative models

Use the cheapest component that can do each step:

- Krun routes a request to the right model, prompt, tool or agent; the LLM does the open-ended work there.
- Krun decides whether a request needs retrieval or a stronger model at all.
- An LLM drafts candidates; Krun scores or checks them (`score` for quality, `noul` for "answer is supported by the
  sources"); code keeps the best or retries.
- An LLM extracts a value; Krun checks a bounded property of it; code validates the exact format.

## Patterns

- **Support triage**: `choice` queue (label-only ids) + `noul` wants a human + `score` urgency, in one request. Abstain
  or high `noul` → human queue with the top suggestion.
- **Agent tool routing**: `choice` with `task_type: "tool"` over the tools valid at this step → code validates
  arguments and confirms side effects; keep a "no tool" path.
- **Moderation / policy**: one `noul` per critical rule with explicit `criteria` and its own threshold; `multi` for
  descriptive tags where one shared threshold is fine.
- **Ranking**: `score` each candidate on one concrete dimension, one request per candidate (each candidate is its own
  context; don't pack several into one), sent concurrently → sort in code. Scores are comparable across requests only
  when the instructions and levels are identical. Filter on exact attributes in code first.
- **Answer verification**: context = question + evidence + answer; `noul` "Is every claim in the answer supported by
  the evidence?" → accept, regenerate or escalate.
- **Document intake**: upload → `choice` document type + `multi` elements present + `noul` for the property that gates
  the workflow → uncertain documents to review.

## Minimal code

Keys stay server-side: read `KRUN_API_KEY` from the environment or a secret manager, never ship it to a browser or app.
Check the SDK pages for the current signatures before extending these.

Python (`pip install krun-ai`, 0.3.0+ for multi, content parts and assets):

```python
from krun import Krun

client = Krun()  # reads KRUN_API_KEY

result = client.decide(
    context="I was charged twice for my order and I want to talk to someone now.",
    questions={
        "queue": {"type": "choice", "options": {"billing": "", "shipping": "", "returns": ""}},
        "wants_human": {"type": "noul", "instructions": "Is the customer asking to talk to a person?"},
    },
)

queue = result.choice("queue")
if queue.abstain or result.noul("wants_human").noul >= 0.8:  # example threshold: tune on your data
    print("human review; best guess:", max(queue.probabilities, key=lambda k: queue.probabilities[k]))
else:
    print("route to", queue.choice)
print(result.request_id)  # log it; needed for feedback
```

TypeScript (`npm install @krun-ai/sdk`, Node.js 20+, server-side; 0.3.0+ for multi and assets; fields are camelCase):

```ts
import { Krun } from "@krun-ai/sdk";

const krun = new Krun(); // reads KRUN_API_KEY

const result = await krun.decide({
  context: "The blender arrived with a cracked jar.",
  questions: {
    issues: {
      type: "multi",
      instructions: "Select every issue the customer reports.",
      options: { damaged_item: null, late_delivery: null, wrong_item: null },
    },
  },
});

console.log(result.answers.issues.values, result.requestId);
```

HTTP:

```bash
curl https://api.krun.ai/v1/decide \
  -H "Authorization: Bearer $KRUN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"context": "Find my meetings tomorrow.",
       "questions": {"tool": {"type": "choice", "task_type": "tool",
         "options": {"calendar_search": "Search calendar events", "send_email": "Send an email"}}}}'
```

The request id is in the `X-Request-ID` response header (`result.request_id` in Python, `result.requestId` in
TypeScript). Send corrections with `POST /v1/feedback` (`client.feedback(...)`) so you can measure accuracy on your
traffic.

## Avoid

- One request per question about the same context.
- Treating `confidence` as accuracy, or `abstain: false` on advisory questions as a guarantee.
- Hard-coding one threshold for every question or domain.
- Asking Krun what code can compute exactly, or asking it to generate text.
- Compound questions, `choice` for labels that can co-occur, `score` levels without concrete descriptions.
- Pre-processing media with your own OCR or transcription before sending it to Krun.
- API keys in client-side code, or blocking a user-facing request on a cold start without a fallback.
- Relying on remembered limits or SDK signatures instead of the live docs.
