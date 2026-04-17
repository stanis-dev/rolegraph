# 05 — Evals

## Purpose

This document specifies the eval harness that grades each orchestration node and guardrail from [`02-orchestration.md`](02-orchestration.md) against labeled datasets, wires online feedback from reviewer decisions in [`04-ui.md`](04-ui.md), and defines the CI gate that blocks regressions. Every metric here has a tile in the `/evals` cockpit; every tile there has a metric here. The endpoint that serves the cockpit is `GET /evals/summary` from [`03-surfaces.md`](03-surfaces.md).

Upstream: [`01-entities.md`](01-entities.md), [`02-orchestration.md`](02-orchestration.md). Downstream: the `/evals` route in [`04-ui.md`](04-ui.md) renders these tiles.

## Principles

- **Every node gets its own eval.** System-level accuracy is an aggregate, not a substitute.
- **Thresholds are load-bearing.** Each metric has a v1 threshold; regressions against baseline fail CI.
- **Direction matters.** Some metrics are floors (higher is better); some are ceilings (lower is better). The harness respects both.
- **Humans in the loop feed the data.** Reviewer decisions from `POST /reviews` flow into LangSmith as feedback and into dataset curation.

## Dataset layout

All paths under `eval/datasets/`. Each dataset is a directory of JSONL files plus a `README.md` describing labeling conventions.

```
eval/
  datasets/
    requirement_extraction/       # 30 labeled JDs → expected RequirementGraph
      cases.jsonl
      docs/{case_id}/{filename}
      README.md
    evidence_extraction/          # 30 labeled CVs → expected EvidenceGraph
    grounding/                    # 50 synthetic claim/citation pairs (valid + invalid)
    redaction/                    # 20 packets with labeled PII + protected attributes
    qa_grounding/                 # 40 report + question → expected answer + required citations
    ambiguity/                    # 20 JDs with known-ambiguous requirements labeled
  evaluators/                     # TypeScript evaluator modules per metric (pure functions; unit-tested with Vitest)
  run.ts                          # CI entry point (invoked via `tsx`)
  sync-datasets.ts                # one-time LangSmith dataset sync (idempotent)
  baseline.json                   # last-accepted metric values
  baseline-history.md             # append-only log of deliberate baseline changes
```

### Case file shapes

Each `cases.jsonl` row is one labeled case. The labels are the **expected** outputs; the harness compares the system's outputs to them.

#### `requirement_extraction/cases.jsonl`

```json
{
  "caseId": "req_case_001",
  "packetPath": "docs/req_case_001/",
  "expected": {
    "requirements": [
      {
        "text": "5+ years of distributed systems experience",
        "kind": "must",
        "weightBand": [0.8, 1.0],
        "citationSpans": [
          { "documentFilename": "jd.pdf", "text": "5+ years of distributed systems experience" }
        ]
      }
    ],
    "ambiguityCountBand": [0, 3]
  }
}
```

Labeling keeps bands instead of exact numbers for weights and counts, so the eval does not punish defensible extractor variance.

#### `evidence_extraction/cases.jsonl`

Same shape, mirrored for `Evidence` (fields: `type`, `explicit`, `confidenceBand`, `citationSpans`).

#### `grounding/cases.jsonl`

```json
{
  "caseId": "grd_case_001",
  "rationale": "Requirement R1 is supported by E12 and E19.",
  "citedEvidenceIds": ["ev_12", "ev_19"],
  "evidenceTexts": {
    "ev_12": "Led a team migrating from monolith to microservices.",
    "ev_19": "Presented at SREcon on distributed systems reliability."
  },
  "expected": {
    "supported": true
  }
}
```

#### `redaction/cases.jsonl`

```json
{
  "caseId": "red_case_001",
  "documentPath": "docs/red_case_001/cv.md",
  "expected": {
    "spans": [
      { "start": 0, "end": 12, "category": "pii.name" },
      { "start": 45, "end": 58, "category": "protected.age" }
    ]
  }
}
```

#### `qa_grounding/cases.jsonl`

```json
{
  "caseId": "qa_case_001",
  "reportPath": "docs/qa_case_001/report.json",
  "question": "Which must-haves are genuinely missing vs ambiguous?",
  "expected": {
    "shouldRefuse": false,
    "requiredCitationRefs": ["req_3", "req_7", "ev_12"],
    "forbiddenClaims": ["ranked above other candidates"]
  }
}
```

#### `ambiguity/cases.jsonl`

```json
{
  "caseId": "amb_case_001",
  "packetPath": "docs/amb_case_001/",
  "expected": {
    "ambiguousRequirementTexts": [
      "experience with enterprise systems"
    ]
  }
}
```

## Metrics

Every metric has: a stable `key` (used by `/evals/summary` and the UI tiles), a direction (`floor` or `ceiling`), a v1 threshold, a dataset, and a computation rule.

### Extraction metrics

| Key | Direction | v1 threshold | Dataset | Computation |
| --- | --- | --- | --- | --- |
| `requirement_precision` | floor | `≥ 0.80` | `requirement_extraction` | TP / (TP + FP) where a predicted requirement is TP if its `text` matches an expected `text` under fuzzy equality (token Jaccard ≥ 0.8) and `kind` matches. |
| `requirement_recall` | floor | `≥ 0.75` | `requirement_extraction` | TP / (TP + FN) using the same matching rule. |
| `requirement_kind_accuracy` | floor | `≥ 0.85` | `requirement_extraction` | Of matched requirements, the fraction whose `kind` equals the expected `kind`. |
| `evidence_precision` | floor | `≥ 0.75` | `evidence_extraction` | Analogous to requirements, using `text` + `type`. |
| `evidence_recall` | floor | `≥ 0.70` | `evidence_extraction` | Analogous. |
| `evidence_type_accuracy` | floor | `≥ 0.80` | `evidence_extraction` | Analogous. |

### Provenance metrics

| Key | Direction | v1 threshold | Dataset | Computation |
| --- | --- | --- | --- | --- |
| `citation_validity` | floor | `≥ 0.95` | `requirement_extraction` + `evidence_extraction` | Fraction of produced `TextSpan`s that satisfy `document.text[start:end] == span.text`. |
| `unsupported_claim_rate` | **ceiling** | `≤ 0.05` | `grounding` | Fraction of rationales judged unsupported by the grounding validator on the labeled `supported == true` cases (false-negative rate of the validator); reported complementary to support. |

### Redaction metrics

| Key | Direction | v1 threshold | Dataset | Computation |
| --- | --- | --- | --- | --- |
| `pii_recall` | floor | `≥ 0.95` | `redaction` | Fraction of expected PII/protected spans that the redactor detected (exact span overlap ≥ 0.5). |
| `pii_precision` | floor | `≥ 0.85` | `redaction` | Fraction of detected spans that match an expected span. |

### Q&A metrics

| Key | Direction | v1 threshold | Dataset | Computation |
| --- | --- | --- | --- | --- |
| `qa_refusal_when_insufficient` | floor | `≥ 0.90` | `qa_grounding` (cases where `shouldRefuse === true`) | Fraction of insufficient-grounding cases where the system correctly refused. |
| `qa_citation_validity` | floor | `≥ 0.95` | `qa_grounding` (cases where `shouldRefuse === false`) | Of non-refusals, fraction where every claim is covered by a `TextSpan` in `answerCitations` that overlaps an expected `requiredCitationRefs` item. |

### Ambiguity metric

| Key | Direction | v1 threshold | Dataset | Computation |
| --- | --- | --- | --- | --- |
| `ambiguity_recall` | floor | `≥ 0.70` | `ambiguity` | Fraction of expected ambiguous requirement texts that produce at least one `ambiguityFlags` entry on the matched `Requirement`. |

### Cross-cutting metrics

| Key | Direction | v1 threshold | Source | Computation |
| --- | --- | --- | --- | --- |
| `reviewer_agreement_kappa` | floor | `≥ 0.6` (substantial, per Cohen) | manual reviewer-agreement batch | See [reviewer-agreement procedure](#reviewer-agreement-procedure). |
| `stage_latency_p95_ms` | ceiling | `≤ 45000` | LangSmith traces | p95 over all `AuditEvent.latency_ms` for non-HITL nodes across the last 100 runs. |
| `cost_per_run_usd` | ceiling | `≤ 0.60` | LangSmith traces | Mean of Σ `AuditEvent.cost_usd` per run across the last 100 runs. |

## Reviewer-agreement procedure

- Curate a **20-packet manual batch** (10 role packets + 10 candidate packets paired, so 10 fit analyses).
- Two independent human reviewers label each `CoverageMatch` with a `status` (`strong`/`weak`/`none`/`contradictory`), without seeing the system's output.
- Compute Cohen's κ between each human reviewer and the system's `status` labels.
- Report `reviewer_agreement_kappa` as the mean of the two pairwise κ values.
- Repeat quarterly; drift below threshold is a high-priority signal regardless of whether offline accuracy metrics are stable.

## LangSmith wiring

### Offline (datasets + evaluators)

- `eval/run.ts` is the single entry point (invoked via `tsx eval/run.ts ...`).
- It iterates datasets, runs the relevant graph or node, computes each metric via the matching evaluator in `eval/evaluators/` (each evaluator is a plain TS function; unit-tested with **Vitest**), and writes results to both:
  - a local `eval/results/{timestamp}.json`,
  - a LangSmith experiment using `evaluate` from the LangSmith JS SDK (`import { evaluate } from "langsmith/evaluation"`). Dataset IDs mirror the folder names.
- Datasets are synced to LangSmith once via `eval/sync-datasets.ts`; re-running it is idempotent.

### Online (traces + feedback)

- Every graph run produces a LangSmith trace (the `trace_refs.langsmith_run_url` in `FitReport`).
- `POST /reviews` pushes an online feedback record to the trace:
  - key: `reviewer_decision`,
  - value: `action` (approve / edit / reject),
  - correction: `changes` when `action == "edit"`,
  - comment: `rationale`.
- LangSmith online monitors fire when:
  - a production `unsupported_claim_rate` over the last 50 runs exceeds `0.05`,
  - `reject` rate over the last 20 runs exceeds `0.2`,
  - median `stage_latency_p95_ms` over the last 20 runs exceeds threshold.

## CI gate

- CI invokes `pnpm -F backend eval:ci` which runs `tsx eval/run.ts --baseline eval/baseline.json --tolerance 0.02`.
- For each metric:
  - If the metric is a **floor** and `value < baseline - tolerance` → **fail**.
  - If the metric is a **ceiling** and `value > baseline + tolerance` → **fail**.
  - Otherwise → **pass**.
- The first pass after a deliberate threshold change requires an explicit `--accept-baseline` flag that updates `eval/baseline.json` and records the PR / commit + rationale in `eval/baseline_history.md`.
- Build output includes the full metric table, so a PR reviewer sees each metric's delta before merging.
- CI artifact: `eval/results/{timestamp}.json` is uploaded for offline inspection.

## Coverage claim

Each node / guardrail in [`02-orchestration.md`](02-orchestration.md) is covered by at least one metric:

| Node / guardrail | Metrics |
| --- | --- |
| `redact_pii` | `pii_recall`, `pii_precision` |
| `extract_requirements` | `requirement_precision`, `requirement_recall`, `requirement_kind_accuracy`, `citation_validity`, `ambiguity_recall` |
| `extract_evidence` | `evidence_precision`, `evidence_recall`, `evidence_type_accuracy`, `citation_validity` |
| `align` | (indirectly graded by `reviewer_agreement_kappa`) |
| `validate_grounding` | `unsupported_claim_rate` |
| `gate_risky_output` | `reviewer_agreement_kappa` (gate fires ↔ reviewer would flag) |
| `human_review` | online `reviewer_decision` feedback (no offline dataset) |
| `finalize_report` | `stage_latency_p95_ms`, `cost_per_run_usd` |
| `answer_query` | `qa_refusal_when_insufficient`, `qa_citation_validity` |

## References

- Upstream: [`01-entities.md`](01-entities.md), [`02-orchestration.md`](02-orchestration.md).
- Downstream (consumer): [`03-surfaces.md`](03-surfaces.md) (`GET /evals/summary`), [`04-ui.md`](04-ui.md) (`/evals` cockpit; every metric key above is rendered there).
