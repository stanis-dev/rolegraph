# 03 — Surfaces

## Purpose

This document specifies the surfaces that expose the orchestration from [`02-orchestration.md`](02-orchestration.md) to consumers: a **Hono** REST API (consumed by the UI in [`04-ui.md`](04-ui.md)), an **MCP server** built on `@modelcontextprotocol/sdk` that re-exposes the same graphs as tools for external agent clients, and **LangGraph Studio** as the visible orchestration / debugging surface. All payloads use the entity types in [`01-entities.md`](01-entities.md) — JSON field names are camelCase to match the Zod schemas.

Upstream: [`01-entities.md`](01-entities.md), [`02-orchestration.md`](02-orchestration.md). Downstream: [`04-ui.md`](04-ui.md).

## Hono REST

### Conventions

- **Base URL (v1 local):** `http://localhost:8000`.
- **Content type:** JSON (`application/json`) unless stated otherwise.
- **IDs:** all path parameters are ULIDs with the type prefix from [`01-entities.md`](01-entities.md).
- **Auth:** none in v1 (see [`00-overview.md`](00-overview.md)).
- **CORS:** Hono's `cors` middleware pinned to `http://localhost:3000` (the Next.js dev server).
- **Validation:** every route uses `@hono/zod-validator` against a Zod schema from `@rolegraph/shared`.
- **Errors:** RFC 7807 `application/problem+json` with the shape:

  ```json
  {
    "type": "https://rolegraph.local/errors/validation",
    "title": "Validation failed",
    "status": 422,
    "detail": "requirementGraph.requirements[3].citations is empty",
    "code": "VALIDATION_ERROR",
    "runId": "run_01H..."
  }
  ```

  `code` is a stable machine-readable key. `runId` is present when the error originated inside an orchestration run.

### Hono handler shape

Every handler follows the same pattern:

```ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { cors } from "hono/cors";
import { FitStartRequestSchema } from "@rolegraph/shared";
import { fitPipeline } from "./orchestration/fitPipeline";

const app = new Hono();
app.use("/*", cors({ origin: "http://localhost:3000" }));

app.post(
  "/fit",
  zValidator("json", FitStartRequestSchema),
  async (c) => {
    const body = c.req.valid("json");
    const result = await fitPipeline.start(body);
    return c.json(result, 202);
  },
);
```

### Endpoints

Every endpoint that triggers a graph returns a `runId`. Terminal endpoints return the final artifact directly. Paused runs are resumed via `POST /reviews`.

#### `POST /packets`

Upload a job or candidate packet. One call, one packet; may contain multiple documents.

- **Request:** `multipart/form-data`
  - `kind`: `"job"` | `"candidate"`
  - `roleTitle`: optional string (only used when `kind === "job"`)
  - `files[]`: one or more files; supported MIME: `application/pdf`, `text/plain`, `text/markdown`
- **Response:** `201 Created` → `Packet`
- **Failure:** `415` if any file has an unsupported MIME; `422` on empty upload.

#### `POST /roles/:packetId/extract`

Runs `redactPii` → `extractRequirements` only. Idempotent: if a requirement graph already exists at the current packet version, returns it. Passing `?force=true` bumps the graph `version` and re-extracts.

- **Path:** `packetId` = `pkt_*`
- **Query:** `force?: boolean = false`
- **Response:** `200` → `{ "runId": "run_...", "requirementGraph": RequirementGraph, "redactionLog": RedactionLog }`
- **Failure:** `404` if packet missing; `409` if packet is not a `JobPacket`; `422` on extraction validation failure.

#### `POST /candidates/:packetId/extract`

Mirror of the role extractor. Runs `redactPii` → `extractEvidence`.

- **Path:** `packetId` = `pkt_*`
- **Query:** `force?: boolean = false`
- **Response:** `200` → `{ "runId": "run_...", "evidenceGraph": EvidenceGraph, "redactionLog": RedactionLog }`
- **Failure:** `404`; `409` if packet is not a `CandidatePacket`; `422`.

#### `POST /fit`

Entry point for the full `fitPipeline`. Reuses any existing requirement / evidence graphs at their latest versions unless `force: true` is set.

- **Request:**
  ```json
  {
    "rolePacketId": "pkt_...",
    "candidatePacketId": "pkt_...",
    "force": false
  }
  ```
- **Response:** `202 Accepted` → `{ "runId": "run_...", "status": "running" | "paused" | "completed", "reportId": "fit_..." | null }`
  - If the run completes without interrupting at the HITL gate, `status === "completed"` and `reportId` is populated.
  - If the gate fires, `status === "paused"` and `reportId` is `null` until a `ReviewDecision` resumes the run.
- **Failure:** `404` if either packet is missing; `409` if their `kind` values don't match job/candidate respectively; `422` on validation failures.

#### `GET /fit/:reportId`

Returns the finalized `FitReport`.

- **Response:** `200` → `FitReport`
- **Failure:** `404` if the report doesn't exist; `409` if the underlying run is still paused — include `runId` in the problem+json.

#### `GET /runs/:runId`

Run status. Used by the UI to poll paused runs. Backed by `graph.getState(threadConfig)` from LangGraph.js.

- **Response:** `200` →
  ```json
  {
    "runId": "run_...",
    "graph": "fitPipeline" | "qaPipeline",
    "status": "running" | "paused" | "completed" | "rejected" | "failed",
    "currentNode": "redactPii" | "extractRequirements" | ... | null,
    "riskFlags": [RiskFlag],
    "reportId": "fit_..." | null,
    "error": ProblemDetails | null,
    "createdAt": "2026-...",
    "updatedAt": "2026-..."
  }
  ```
  `riskFlags` is populated only when `status === "paused"`.

#### `POST /reviews`

Submit a `ReviewDecision`, resuming the paused run. Internally the backend calls `graph.invoke(new Command({ resume: decision }), threadConfig)`.

- **Request:** `ReviewDecision` with `id` and `timestamp` omitted (server assigns):
  ```json
  {
    "reportId": "fit_...",
    "reviewerId": "stan",
    "action": "approve" | "edit" | "reject",
    "changes": [ { "op": "replace", "path": "/matches/2/status", "value": "weak" } ],
    "rationale": "Downgraded match to weak because E7 is scope-only."
  }
  ```
  `changes` is required when `action === "edit"`.
- **Response:** `200` → `{ "decisionId": "rev_...", "runId": "run_...", "resumedStatus": "running" | "completed" | "rejected" }`
- **Failure:** `404` if the report is not paused; `422` on invalid patch; `409` if the run has already resumed.

#### `POST /query`

Runs the `qaPipeline` against a finalized report.

- **Request:**
  ```json
  {
    "reportId": "fit_...",
    "question": "Which must-haves are genuinely missing vs ambiguous?"
  }
  ```
- **Response:** `200` →
  ```json
  {
    "runId": "run_...",
    "answer": "string | null",
    "citations": [TextSpan],
    "refusalReason": "insufficient_grounding" | null
  }
  ```
  If `refusalReason` is set, `answer` is `null` and `citations` is empty.
- **Failure:** `404` if the report doesn't exist; `409` if the underlying `FitReport` is not finalized.

#### `GET /traces/:runId`

Returns a summarized trace plus a LangSmith deep link. The UI's trace inspector calls this.

- **Response:** `200` →
  ```json
  {
    "runId": "run_...",
    "langsmithRunUrl": "https://smith.langchain.com/...",
    "auditEvents": [AuditEvent],
    "redactionLog": RedactionLog | null,
    "groundingReport": GroundingReport | null
  }
  ```
- **Failure:** `404`.

#### `GET /evals/summary`

Latest eval metrics snapshot for the cockpit.

- **Response:** `200` →
  ```json
  {
    "generatedAt": "2026-...",
    "metrics": [
      {
        "name": "requirement_precision",
        "value": 0.83,
        "threshold": 0.80,
        "status": "pass" | "warn" | "fail",
        "dataset": "requirement_extraction",
        "baseline": 0.81
      }
    ]
  }
  ```
  Metric keys mirror [`05-evals.md`](05-evals.md). Keys use snake_case because they are written by the eval harness and are strings, not object fields.

### Endpoint → graph map

| Endpoint | Graph | Notes |
| --- | --- | --- |
| `POST /packets` | none | persistence only |
| `POST /roles/:id/extract` | subgraph of `fitPipeline` (`redactPii` + `extractRequirements`) | |
| `POST /candidates/:id/extract` | subgraph of `fitPipeline` (`redactPii` + `extractEvidence`) | |
| `POST /fit` | full `fitPipeline` | may pause at `gateRiskyOutput` |
| `POST /reviews` | resumes paused `fitPipeline` via `Command({ resume })` | |
| `POST /query` | `qaPipeline` | |
| `GET /fit/:id`, `GET /runs/:id`, `GET /traces/:id`, `GET /evals/summary` | read-only | no graph executed |

## MCP server

Built with `@modelcontextprotocol/sdk` (TypeScript). Runs as a standalone process that imports the same graph instances as the Hono backend — no duplication. The MCP server speaks the MCP stdio transport (for Claude Desktop / Cursor) and/or SSE.

### Server wiring (sketch)

```ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { fitPipeline, qaPipeline } from "./orchestration";
import {
  ExtractRequirementsInputSchema, ExtractEvidenceInputSchema,
  ComputeFitInputSchema, AnswerQuestionInputSchema,
} from "@rolegraph/shared";

const server = new Server(
  { name: "rolegraph", version: "0.1.0" },
  { capabilities: { tools: {} } },
);

server.setRequestHandler(/* ListTools */ ..., () => ({
  tools: [
    { name: "extract_requirements", inputSchema: zodToJsonSchema(ExtractRequirementsInputSchema) },
    { name: "extract_evidence",     inputSchema: zodToJsonSchema(ExtractEvidenceInputSchema) },
    { name: "compute_fit",          inputSchema: zodToJsonSchema(ComputeFitInputSchema) },
    { name: "answer_question",      inputSchema: zodToJsonSchema(AnswerQuestionInputSchema) },
  ],
}));

server.setRequestHandler(/* CallTool */ ..., async (req) => {
  switch (req.params.name) {
    case "compute_fit": { /* call fitPipeline.start(...) */ }
    /* ... */
  }
});

await server.connect(new StdioServerTransport());
```

### MCP tools

Each tool is a thin adapter over the corresponding REST endpoint. JSON shapes mirror the REST payloads.

| MCP tool | Input | Output | Mirrors REST |
| --- | --- | --- | --- |
| `extract_requirements` | `{ packetId }` | `{ requirementGraph, redactionLog }` | `POST /roles/:id/extract` |
| `extract_evidence` | `{ packetId }` | `{ evidenceGraph, redactionLog }` | `POST /candidates/:id/extract` |
| `compute_fit` | `{ rolePacketId, candidatePacketId }` | `{ runId, status, reportId? }` | `POST /fit` |
| `answer_question` | `{ reportId, question }` | `{ answer, citations, refusalReason? }` | `POST /query` |

### HITL over MCP

`compute_fit` returns immediately with `status` — MCP clients poll via a separate `get_run_status` tool (same shape as `GET /runs/:runId`) and submit reviews via a `submit_review` tool (same shape as `POST /reviews`). Exposed for completeness; the primary review experience is the Next.js UI.

### Why TS MCP SDK vs Langflow

Earlier drafts relied on Langflow to register MCP tools. Keeping the stack TS end-to-end means the MCP SDK is the direct path: both the Hono backend and the MCP server import the same `@langchain/langgraph` instances from `apps/backend/src/orchestration/*`, so the two surfaces cannot drift. See [`00-overview.md`](00-overview.md) for the stack decision.

## LangGraph Studio

LangGraph Studio is the visible orchestration surface — the TS equivalent of the Langflow canvases in earlier drafts. It is a local UI that reads the compiled graph and renders it as a step-through debugger with live state inspection.

### Wiring

- `langgraph.json` at the repo root declares entry points:
  ```json
  {
    "graphs": {
      "fit_pipeline": "./apps/backend/src/orchestration/fitPipeline.ts:fitPipeline",
      "qa_pipeline": "./apps/backend/src/orchestration/qaPipeline.ts:qaPipeline"
    },
    "env": "./.env"
  }
  ```
- `npx langgraph dev` starts the Studio locally. It imports the same graph instances as the Hono backend and the MCP server — there is no separate canvas file to maintain.
- Studio shows every node, state diffs at each step, and interrupt state when `humanReview` pauses. It also forwards traces to LangSmith when `LANGSMITH_API_KEY` is set.

### Demo role

For the portfolio demo flow, Studio replaces the Langflow canvases: it visually walks through the graph, pauses on HITL, and hands off to the Next.js UI when the reviewer takes over. No bespoke canvas JSON is checked in.

## Error model

Every surface returns `application/problem+json` on error. Stable `code` values:

| `code` | HTTP | Meaning |
| --- | --- | --- |
| `VALIDATION_ERROR` | 422 | Request or node output failed validation. |
| `NOT_FOUND` | 404 | Resource does not exist. |
| `CONFLICT` | 409 | Resource exists but is in an incompatible state (wrong `kind`, run still paused, patch against resumed run, etc.). |
| `UNSUPPORTED_MEDIA_TYPE` | 415 | Upload MIME not supported. |
| `GRAPH_RUN_FAILED` | 500 | Orchestration error; `runId` populated. |
| `LLM_PROVIDER_ERROR` | 502 | Upstream model provider error after retries. |

## Server-only endpoints

None in v1 — every endpoint above has a UI consumer (see [`04-ui.md`](04-ui.md)). If a future endpoint is backend-internal (admin, migration), it must be explicitly marked **"server-only"** in this document.

## References

- Upstream: [`01-entities.md`](01-entities.md), [`02-orchestration.md`](02-orchestration.md).
- Downstream: [`04-ui.md`](04-ui.md) — every endpoint here has a named UI consumer there; [`05-evals.md`](05-evals.md) — `/evals/summary` is backed by the eval harness.
