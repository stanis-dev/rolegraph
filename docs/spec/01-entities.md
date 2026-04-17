# 01 — Entities

## Purpose

This document defines the typed vocabulary the rest of the spec is written in. Every artifact the system produces or persists is one of these types. A later session should be able to generate Zod schemas and TypeScript types directly from this document without making additional design choices.

Upstream: [`00-overview.md`](00-overview.md). Downstream: [`02-orchestration.md`](02-orchestration.md), [`03-surfaces.md`](03-surfaces.md), [`04-ui.md`](04-ui.md), [`05-evals.md`](05-evals.md).

## Cross-cutting conventions

- **Language.** TypeScript (`strict`) on Node.js 22 LTS. Schemas are Zod v3 definitions; runtime types are derived via `z.infer<typeof Schema>`.
- **Location.** All schemas live in `packages/shared/src/entities/` and are re-exported from `packages/shared/src/index.ts`. Both `apps/backend` and `apps/web` import from `@rolegraph/shared`.
- **IDs.** [ULID](https://github.com/ulid/spec) strings generated via `ulidx`, prefixed by type. All ID fields are `z.string()` with a regex constraint; the regex is centralized as `idSchema(prefix)`.
  - Prefixes: `pkt_`, `doc_`, `req_`, `ev_`, `fit_`, `match_`, `rev_`, `aud_`, `run_`.
  - Helper:
    ```ts
    export const idSchema = (prefix: string) =>
      z.string().regex(new RegExp(`^${prefix}[0-9A-HJKMNP-TV-Z]{26}$`), {
        message: `Expected ID with prefix "${prefix}"`,
      });
    ```
- **Provenance.** Every *extracted* object (`Requirement`, `Evidence`) has a non-empty `citations: TextSpan[]`. Enforced by `z.array(TextSpanSchema).min(1)`; violations fail the producing node (see [`02-orchestration.md`](02-orchestration.md)).
- **Timestamps.** ISO-8601 UTC strings. Zod shape: `z.string().datetime({ offset: true })`.
- **Versioning.** Graphs are versioned per packet via a monotonically increasing integer starting at `1`. Re-extraction bumps `version` and writes a new file; prior versions are retained. Reports pin the exact graph versions they were computed from.
- **Immutability.** Once a `FitReport` is finalized, it is not mutated. Review edits produce a new report with a pointer back to its predecessor.
- **Discriminated unions.** `Packet` uses `z.discriminatedUnion("kind", [...])` with `kind: z.literal("job" | "candidate")`.
- **Closed-vocabulary enums.** All enum fields below are closed; Zod rejects unknown values.
- **Internal / reserved types.** `PacketBase` (the union's shared fields) and `Relationship` (RequirementGraph's reserved-for-later relation type) are defined here for completeness but are not referenced directly by [`02`](02-orchestration.md)/[`03`](03-surfaces.md)/[`04`](04-ui.md)/[`05`](05-evals.md). Consumers see them only through `JobPacket`/`CandidatePacket` and `RequirementGraph.relationships` respectively.

## TextSpan

The atomic unit of provenance. Every claim the system makes resolves to one or more `TextSpan`s.

```ts
export const TextSpanSchema = z.object({
  documentId: idSchema("doc_"),
  start: z.number().int().nonnegative(),   // inclusive char offset
  end: z.number().int().positive(),        // exclusive char offset
  text: z.string(),                        // verbatim substring
}).refine((s) => s.start < s.end, {
  message: "TextSpan.start must be < end",
});
export type TextSpan = z.infer<typeof TextSpanSchema>;
```

Notes:
- `text` is denormalized on purpose so a report can be rendered without re-reading the source document.
- `end <= document.text.length` is checked at the boundary where the span is produced against a concrete `Document`.
- The substring invariant — `document.text.slice(span.start, span.end) === span.text` — is checked by the **citation validator** guardrail in [`02-orchestration.md`](02-orchestration.md).

## Document

```ts
export const PageSpanSchema = z.object({
  page: z.number().int().positive(),       // 1-based
  start: z.number().int().nonnegative(),   // char offset in Document.text
  end: z.number().int().positive(),
});
export type PageSpan = z.infer<typeof PageSpanSchema>;

export const DocumentSchema = z.object({
  id: idSchema("doc_"),
  filename: z.string(),
  mime: z.enum(["application/pdf", "text/plain", "text/markdown"]),
  text: z.string(),                        // normalized plain-text extraction
  pages: z.array(PageSpanSchema).nullable().optional(),
});
export type Document = z.infer<typeof DocumentSchema>;
```

`Document.text` is the canonical coordinate system for `TextSpan`. Extractors (PDF → text, etc.) happen before `Document` is constructed; the pipeline never re-reads raw bytes.

## Packet

Discriminated union. A run takes exactly one of each kind as input.

```ts
const PacketBaseShape = {
  id: idSchema("pkt_"),
  documents: z.array(DocumentSchema).min(1),
  createdAt: z.string().datetime({ offset: true }),
};

export const JobPacketSchema = z.object({
  kind: z.literal("job"),
  roleTitle: z.string().nullable().optional(),
  ...PacketBaseShape,
});
export type JobPacket = z.infer<typeof JobPacketSchema>;

export const CandidatePacketSchema = z.object({
  kind: z.literal("candidate"),
  ...PacketBaseShape,
});
export type CandidatePacket = z.infer<typeof CandidatePacketSchema>;

export const PacketSchema = z.discriminatedUnion("kind", [
  JobPacketSchema,
  CandidatePacketSchema,
]);
export type Packet = z.infer<typeof PacketSchema>;

// Exposed as a type only; not a Zod schema consumers use directly.
export type PacketBase = Omit<Packet, "kind" | "roleTitle">;
```

A `JobPacket` may contain a JD, FAQ, leveling rubric, recruiter note, and hiring-manager brief (one `Document` each). A `CandidatePacket` may contain a CV plus a portfolio doc, case-study PDF, or project README bundle.

## Requirement

```ts
export const RequirementKindSchema = z.enum([
  "must", "preferred", "seniority", "domain", "location",
]);
export type RequirementKind = z.infer<typeof RequirementKindSchema>;

export const RequirementSchema = z.object({
  id: idSchema("req_"),
  text: z.string().min(1),                  // canonical statement produced by the extractor
  kind: RequirementKindSchema,
  weight: z.number().min(0).max(1),         // extractor-assigned salience
  ambiguityFlags: z.array(z.string()),      // free-form tags, e.g. "underspecified"
  citations: z.array(TextSpanSchema).min(1),
});
export type Requirement = z.infer<typeof RequirementSchema>;
```

`weight` is a salience hint from the extractor (how load-bearing the requirement appears in the packet), not a reviewer-assigned priority. Reviewer overrides live in `ReviewDecision.changes`.

## RequirementGraph

```ts
export const RelationshipTypeSchema = z.enum([
  "implies", "conflicts_with", "subsumes",
]);

export const RelationshipSchema = z.object({
  fromId: idSchema("req_"),
  toId: idSchema("req_"),
  type: RelationshipTypeSchema,
});
export type Relationship = z.infer<typeof RelationshipSchema>;

export const RequirementGraphSchema = z.object({
  packetId: idSchema("pkt_"),
  version: z.number().int().positive(),
  requirements: z.array(RequirementSchema),
  relationships: z.array(RelationshipSchema).default([]),  // reserved; v1 leaves empty
});
export type RequirementGraph = z.infer<typeof RequirementGraphSchema>;
```

`relationships` is reserved for v1 and intentionally empty by default. It exists in the schema so downstream code does not have to migrate when we start extracting it.

## Evidence

```ts
export const EvidenceTypeSchema = z.enum([
  "role", "project", "technology", "metric", "scope", "domain",
]);
export type EvidenceType = z.infer<typeof EvidenceTypeSchema>;

export const EvidenceSchema = z.object({
  id: idSchema("ev_"),
  text: z.string().min(1),                  // canonical statement
  type: EvidenceTypeSchema,
  explicit: z.boolean(),                    // true = stated; false = inferred
  confidence: z.number().min(0).max(1),
  freshnessDate: z.string().date().nullable().optional(),   // ISO date if inferable
  citations: z.array(TextSpanSchema).min(1),
  linkedRequirementIds: z.array(idSchema("req_")).default([]),
});
export type Evidence = z.infer<typeof EvidenceSchema>;
```

`linkedRequirementIds` is a hint, not a final mapping. The alignment engine produces the authoritative mapping via `CoverageMatch`.

## EvidenceGraph

```ts
export const EvidenceGraphSchema = z.object({
  packetId: idSchema("pkt_"),
  version: z.number().int().positive(),
  evidences: z.array(EvidenceSchema),
});
export type EvidenceGraph = z.infer<typeof EvidenceGraphSchema>;
```

## CoverageMatch

The output unit of the alignment engine. One per requirement.

```ts
export const MatchStatusSchema = z.enum([
  "strong", "weak", "none", "contradictory",
]);
export type MatchStatus = z.infer<typeof MatchStatusSchema>;

export const CoverageMatchSchema = z.object({
  id: idSchema("match_"),
  requirementId: idSchema("req_"),
  evidenceIds: z.array(idSchema("ev_")),    // empty iff status == "none"
  status: MatchStatusSchema,
  confidence: z.number().min(0).max(1),
  rationale: z.string().min(1),             // short LLM-written explanation citing listed evidences
}).refine(
  (m) => (m.status === "none") === (m.evidenceIds.length === 0),
  { message: "evidenceIds must be empty iff status == 'none'" },
);
export type CoverageMatch = z.infer<typeof CoverageMatchSchema>;
```

Invariant enforced at the schema level: `status === "none"` iff `evidenceIds` is empty. Violations fail the alignment node.

## FitReport

The top-level artifact a reviewer and the UI consume.

```ts
export const FitReportSchema = z.object({
  id: idSchema("fit_"),
  runId: idSchema("run_"),
  rolePacketId: idSchema("pkt_"),
  roleGraphVersion: z.number().int().positive(),
  candidatePacketId: idSchema("pkt_"),
  candidateGraphVersion: z.number().int().positive(),
  matches: z.array(CoverageMatchSchema),
  ambiguities: z.array(z.string()),
  clarificationQuestions: z.array(z.string()).max(3),
  coverageScore: z.number().min(0).max(1),
  confidence: z.number().min(0).max(1),
  predecessorId: idSchema("fit_").nullable().optional(),   // set if this report supersedes one after review edits
  createdAt: z.string().datetime({ offset: true }),
});
export type FitReport = z.infer<typeof FitReportSchema>;
```

Critical: `coverageScore` and `confidence` score *the documented alignment*. They are **not** a hire/no-hire signal. This invariant is enforced by prompt design in [`02-orchestration.md`](02-orchestration.md) and surfaced as a label in the UI — see [`04-ui.md`](04-ui.md).

## ReviewDecision

```ts
export const ReviewActionSchema = z.enum(["approve", "edit", "reject"]);
export type ReviewAction = z.infer<typeof ReviewActionSchema>;

// RFC 6902 JSON Patch op
export const JsonPatchOpSchema = z.object({
  op: z.enum(["add", "remove", "replace", "move", "copy", "test"]),
  path: z.string(),
  value: z.unknown().optional(),
  from: z.string().optional(),
});
export const JsonPatchSchema = z.array(JsonPatchOpSchema);
export type JsonPatch = z.infer<typeof JsonPatchSchema>;

export const ReviewDecisionSchema = z.object({
  id: idSchema("rev_"),
  reportId: idSchema("fit_"),
  reviewerId: z.string().min(1),                           // free-form string in v1 (no auth)
  action: ReviewActionSchema,
  changes: JsonPatchSchema.nullable().optional(),          // required when action === "edit"
  rationale: z.string().min(1),
  timestamp: z.string().datetime({ offset: true }),
}).refine(
  (d) => d.action !== "edit" || (d.changes && d.changes.length > 0),
  { message: "changes is required when action === 'edit'" },
);
export type ReviewDecision = z.infer<typeof ReviewDecisionSchema>;
```

Semantics:
- `approve` → finalize report unchanged, resume run.
- `edit` → apply `changes` to the gated draft report, produce a new `FitReport` with `predecessorId` set, resume run.
- `reject` → mark the run as rejected; no finalized report is written; a `FitReport` with `matches=[]` is **not** produced.

## AuditEvent

One row per node execution per run. Written to `audit.sqlite` (`audit_events` table).

```ts
export const AuditEventSchema = z.object({
  id: idSchema("aud_"),
  runId: idSchema("run_"),
  stage: z.string().min(1),                 // node name, e.g. "extract_requirements"
  inputsHash: z.string().length(64),        // sha256 hex
  outputsHash: z.string().length(64),
  modelId: z.string().nullable(),           // e.g. "claude-opus-4-7"; null for deterministic nodes
  promptVersion: z.string().nullable(),     // pinned prompt version tag
  seed: z.number().int().nullable(),
  costUsd: z.number().nonnegative().nullable(),
  latencyMs: z.number().int().nonnegative(),
  timestamp: z.string().datetime({ offset: true }),
});
export type AuditEvent = z.infer<typeof AuditEventSchema>;
```

`inputsHash` / `outputsHash` let the eval harness detect stage-level regressions without storing full payloads for every run. The canonicalization rule is: `JSON.stringify` with sorted keys, UTF-8, then sha256.

## ID scheme

| Prefix | Type |
| --- | --- |
| `pkt_` | `Packet` |
| `doc_` | `Document` |
| `req_` | `Requirement` |
| `ev_`  | `Evidence` |
| `match_` | `CoverageMatch` |
| `fit_` | `FitReport` |
| `rev_` | `ReviewDecision` |
| `aud_` | `AuditEvent` |
| `run_` | orchestration run |

`idSchema(prefix)` rejects IDs whose prefix does not match. Prefixes are part of the contract, not a presentational hint.

## Storage layout

All paths are relative to `apps/backend`'s working directory.

```
data/
  packets/
    {pkt_id}.json                 # serialized Packet
  graphs/
    {pkt_id}/
      requirements/v{n}.json      # RequirementGraph (only if kind === "job")
      evidence/v{n}.json          # EvidenceGraph (only if kind === "candidate")
  reports/
    {fit_id}.json                 # FitReport
audit.sqlite                      # tables: audit_events, review_decisions
```

SQLite tables (`better-sqlite3` executes the DDL on startup):

```sql
CREATE TABLE IF NOT EXISTS audit_events (
  id TEXT PRIMARY KEY,
  run_id TEXT NOT NULL,
  stage TEXT NOT NULL,
  inputs_hash TEXT NOT NULL,
  outputs_hash TEXT NOT NULL,
  model_id TEXT,
  prompt_version TEXT,
  seed INTEGER,
  cost_usd REAL,
  latency_ms INTEGER NOT NULL,
  timestamp TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS audit_events_run_id ON audit_events (run_id);

CREATE TABLE IF NOT EXISTS review_decisions (
  id TEXT PRIMARY KEY,
  report_id TEXT NOT NULL,
  reviewer_id TEXT NOT NULL,
  action TEXT NOT NULL CHECK (action IN ('approve','edit','reject')),
  changes_json TEXT,                     -- JSON-serialized JsonPatch, nullable
  rationale TEXT NOT NULL,
  timestamp TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS review_decisions_report_id ON review_decisions (report_id);
```

JSON files are written atomically (write-temp-then-rename via `fs.promises.rename`). File names are the entity ID; the filesystem is the "table."

## Upgrade path

The file-on-disk layout is deliberately shallow and readable. Migrating to Postgres + object storage requires:
- packets / graphs / reports tables mirroring the Zod shapes (trivial with e.g. Drizzle),
- object-store URLs replacing filesystem paths for documents,
- no changes to the Zod schemas above.

That migration is out of scope for v1 (see [`00-overview.md`](00-overview.md)).

## References

- Upstream: [`00-overview.md`](00-overview.md).
- Downstream: [`02-orchestration.md`](02-orchestration.md) (nodes consume and produce these types), [`03-surfaces.md`](03-surfaces.md) (REST payloads are these types), [`04-ui.md`](04-ui.md) (UI components render these types), [`05-evals.md`](05-evals.md) (eval datasets label expected values of these types).
