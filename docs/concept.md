# RoleGraph — Concept Brief

Yes — your hiring/openings idea **does** align, but the strongest version is not "AI recruiter." It is a **governed, evidence-grounded role-fit workbench**.

That change matters for both signaling and scope. In the EU, AI systems used to analyze/filter job applications or evaluate candidates are explicitly listed as **high-risk** under Annex III of the AI Act, so a portfolio that centers on automated ranking is the wrong flagship. The better signal is: *I can build a high-stakes AI workflow that is structured, auditable, guarded, observable, and eval-driven.* That maps much better to a staff-level AI systems profile anyway. Langflow/LangChain are a good fit here: Langflow supports agents and MCP and can expose flows as tools; LangChain gives you structured output, guardrails, and human-in-the-loop middleware; LangSmith supports both offline and online evaluation; and current OpenAI guidance is explicit that evals and governance should sit at the core of production agent design. ([AI Act Service Desk][1])

## The project I would build

**RoleGraph — an evidence-grounded hiring intelligence workbench**

The point is to let someone **upload a job packet and a candidate packet, interrogate the fit, inspect the evidence, review risky outputs, and see the agent traces/evals**. It should feel like a serious internal tool, not a toy chatbot.

### What it is

A focused system that does four things well:

1. turns a job description into a **structured requirement graph**
2. turns a CV/portfolio packet into an **evidence graph**
3. maps evidence to requirements and exposes the result through an interactive **fit workspace**
4. wraps the whole thing in **review gates, guardrails, observability, and evals**

### What it is not

Not an ATS.
Not a sourcing tool.
Not interview scheduling.
Not outbound email automation.
Not candidate-to-candidate ranking.
Not automated hire/no-hire decisions.

That boundary is important. It keeps scope tight, improves demo quality, and avoids the project collapsing into "mini Greenhouse clone."

## Why this is the right flagship for you

This concept shows the exact things that are scarce and staff-signaling in AI systems work:

* document ingestion and normalization
* typed extraction instead of free-form prompt soup
* evidence-grounded reasoning
* human-review controls for high-impact outputs
* eval-driven iteration
* traceability and operational visibility
* reusable platform surface via API/MCP

Most portfolio projects stop at "chat with your docs." This one shows that you understand **workflow design under real constraints**. That is a much stronger signal than breadth.

## The feature set

### 1) Job packet → Requirement Graph

This is the first core feature and one of the main sources of "wow."

The system ingests a job packet: JD, FAQ, leveling rubric, recruiter note, and optionally a hiring-manager brief. It converts that into a **typed requirement graph**, not just a summary. LangChain's structured-output path is useful here because it lets the agent return validated machine-readable objects rather than plain text. ([LangChain Docs][2])

The graph should include:

* must-have requirements
* preferred requirements
* seniority/scope signals
* domain-context requirements
* location/timezone constraints
* ambiguity flags
* source spans for every extracted requirement

The depth requirement here is the important part: **every node must carry provenance**. When a reviewer clicks "distributed systems performance," they should see the exact sentence(s) in the job packet that created that node.

Do not make this a generic knowledge graph project. Keep it narrow and useful: a reviewer-editable JSON graph with weighted nodes is enough.

### 2) Candidate packet → Evidence Graph

The second core feature is the mirror image.

The system ingests a candidate packet: CV plus optional portfolio doc, case-study PDF, or project README bundle. It extracts **evidence objects**, not a global impression. Langflow can already handle file inputs and runtime file uploads; that makes the interaction tactile and demo-friendly. ([Langflow Documentation][3])

Each evidence object should include:

* evidence type: role, project, technology, metric, scope, domain
* explicit vs inferred
* confidence
* freshness if inferable
* source span
* linked requirement candidates

This is where the project starts to look advanced. A normal demo says, "the candidate looks strong." Your system should say, "requirement R7 is supported by evidence E12 and E19; R11 is weakly supported; R14 is not evidenced."

That is a much better signal than another polished chat answer.

### 3) Evidence-backed Alignment Engine

This is the centerpiece.

It takes the requirement graph and the evidence graph and builds a **fit workspace** with:

* requirement-by-requirement coverage
* "strong evidence / weak evidence / not evidenced / contradictory"
* confidence per match
* unresolved ambiguity list
* up to 3 targeted clarification questions

The output should be **two-mode**:

* **Reviewer mode**: evidence map, ambiguity flags, weak spots
* **Candidate mode**: fit explanation, likely gaps, questions to clarify the match

The critical design choice: do **not** output a final hiring recommendation. Output an **evidence coverage view** plus uncertainty. That makes the system more mature, more defensible, and more aligned with the high-risk nature of employment AI in the EU. ([AI Act Service Desk][1])

If you want one score, make it a **coverage/confidence score**, not a candidate score. Scoring the *documented alignment* is safer and cleaner than pretending the system should judge the person.

### 4) Openings Q&A

You specifically asked for something touchable. This is the interaction layer that makes the project easy to demo.

Add a chat surface that can answer questions like:

* "What are the strongest must-haves for this role?"
* "Which gaps are genuinely missing vs just ambiguous?"
* "What evidence supports the claim that this candidate has platform depth?"
* "Which roles in this packet best match a backend/platform-heavy profile?"

Every answer should cite the underlying job or candidate packet. The point is not a broad RAG chatbot. The point is a **constrained interrogable workspace** over a structured fit analysis.

This is the user-facing "I can touch it" layer.

### 5) Governance and review controls

This is one of the features that will most clearly separate you from standard portfolio demos.

LangChain now has first-class guardrail patterns, built-in PII middleware, and human-in-the-loop middleware that can pause execution, persist graph state, and resume after approve/edit/reject decisions. ([LangChain Docs][4])

Use that to build:

* **PII / protected-attribute scrubbing** before analysis
* **high-impact output gate** for disqualifying or low-confidence claims
* **approve / edit / reject** review flow
* audit log of what changed and why

For the portfolio version, I would require review when:

* the system produces a negative fit conclusion
* the grounding score is below threshold
* the answer includes unsupported inferences
* the output references protected or personal attributes

This feature does two things at once: it makes the demo more impressive, and it shows that you understand how to build AI systems for consequential workflows rather than just generating fluent text.

### 6) Trace + Eval Cockpit

This is mandatory if your goal is leverage.

LangSmith supports offline evaluation on curated datasets and online evaluation on live traffic, and LangSmith tracing captures the full execution trace of an LLM request. OpenAI's eval-driven design guidance is very explicit that evals should be the core engineering process rather than an afterthought, and its governed-agents guidance pushes policies-as-code, observability, and precision/recall testing for defenses. ([LangChain Docs][5])

So the project should ship with an **eval cockpit** that tracks at least:

* requirement extraction accuracy
* evidence-grounding correctness
* unsupported-claim rate
* citation/span validity
* redaction recall/precision
* reviewer agreement
* latency per stage
* cost per run

This is the most staff-signaling layer in the whole project. A lot of engineers can build the app. Far fewer can show the evals, the traces, the thresholds, and the regression story.

## The right amount of "wow"

The "wow" should come from **depth**, not extra product surface.

If I had to pick the three features that create the strongest demo impact, I would pick these:

1. **clickable evidence lineage** for every claim
2. **human-review interrupt** on risky outputs
3. **live eval/trace panel** showing how the system was validated

That trio immediately makes the project feel like a real production system rather than a portfolio toy.

## What I would explicitly leave out

To keep it focused, I would **not** build these into v1:

* candidate-to-candidate ranking
* comparison to a real existing candidate pool
* interview scheduling
* outreach/email automation
* sourcing/search across the open web
* a full ATS database
* a many-role, many-tenant workflow
* a multi-agent swarm just because it sounds fancier

On architecture, LangChain's own guidance says not every complex task needs multi-agent; often a single agent with the right tools is better. For this project, that is the right call. Use one orchestrator with specialized tools/nodes rather than a theatrical multi-agent mesh. ([LangChain Docs][6])

## The architecture I would use with Langflow / LangChain

Use **Langflow as the visible orchestration layer** and **LangChain/LangGraph as the typed control plane**.

Concretely:

* **Langflow**

  * visual top-level flow
  * file ingestion
  * interactive playground/demo surface
  * API exposure
  * MCP server exposure so the same flow can be called from Claude/Cursor/Windsurf

* **LangChain / LangGraph**

  * structured extraction schemas
  * stateful orchestration
  * human-in-the-loop interrupts
  * guardrails
  * routing logic
  * evaluation hooks

* **LangSmith**

  * traces
  * offline eval datasets
  * online monitoring

Langflow's MCP support is especially useful here because it lets you expose the workbench as a reusable tool, which is a stronger staff-level signal than a standalone web demo. Langflow can register project flows as tools for MCP clients, and its agent components can use tools and MCP servers directly. ([Langflow Documentation][7])

## The one stretch feature I would add

If you want one stretch feature, make it this:

**Role benchmark mode**, not candidate-pool ranking.

It should compare the **role** to a curated snapshot corpus of similar openings and answer things like:

* which requirements are common vs unusually strict
* whether the role is unusually broad
* whether the title and requirement mix are internally inconsistent

That gives you the "market state" angle you were considering, but it keeps the benchmark on the **job**, not on real competing candidates. That is a cleaner scope and a better systems demo.

I would keep this to a curated snapshot dataset, not live web scraping in v1.

## The demo flow I would design for

A strong demo for this project would look like this:

1. Upload a JD packet and a CV packet.
2. Watch the system build a requirement graph and evidence graph.
3. Click one requirement and inspect the exact supporting resume spans.
4. Ask a natural-language question in chat.
5. Trigger a low-confidence or risky conclusion and show the review interrupt.
6. Open the trace/eval pane and show why the output was or was not trusted.
7. Optionally show the same system exposed as an MCP tool inside Claude or Cursor.

That is a serious demo. It shows UX, agent orchestration, evidence grounding, governance, observability, and reuse.

## My bottom-line recommendation

Build **RoleGraph: an evidence-grounded role-fit workbench**.

Do **not** build "AI that ranks candidates against each other."
Do build:

* requirement graph extraction
* evidence graph extraction
* evidence-backed fit workspace
* openings Q&A
* review/guardrail layer
* eval + trace cockpit
* MCP/API exposure

That combination is focused enough to finish, tactile enough to demo, and deep enough to signal exactly the kind of advanced AI systems engineering that creates leverage for a senior/staff IC.

The next useful step is to turn this into a **concrete flow spec**: entities, states, tools, Langflow canvases, eval datasets, and the exact UI panes.

[1]: https://ai-act-service-desk.ec.europa.eu/en/ai-act/annex-3 "Annex III | AI Act Service Desk"
[2]: https://docs.langchain.com/oss/python/langchain/structured-output?utm_source=chatgpt.com "Structured output - Docs by LangChain"
[3]: https://docs.langflow.org/api-files?utm_source=chatgpt.com "Files endpoints | Langflow Documentation"
[4]: https://docs.langchain.com/oss/python/langchain/guardrails "Guardrails - Docs by LangChain"
[5]: https://docs.langchain.com/langsmith/evaluation?utm_source=chatgpt.com "LangSmith Evaluation - Docs by LangChain"
[6]: https://docs.langchain.com/oss/python/langchain/multi-agent "Multi-agent - Docs by LangChain"
[7]: https://docs.langflow.org/components-agents "Agents | Langflow Documentation"
