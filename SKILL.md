---
name: evidence-based-code-review
description: Assess repository or change-set design complexity using independent SCC history/complexity evidence, codebase-memory structural evidence, and focused source inspection. Use for explicitly SCC-backed, graph-backed, hotspot, coupling, duplication, blast-radius, or A Philosophy of Software Design reviews. Do not use for a full defect hunt or generic PR review.
---

# Evidence-Based Design Complexity Review

Use SCC and codebase-memory to find design-complexity candidates, then validate claims in source.

## Review philosophy

Complexity is anything that makes software harder to understand or modify.

- Its symptoms are **change amplification**, **cognitive load**, and **unknown unknowns**.
- Its common causes are dependencies and obscurity.
- Complexity matters in proportion to how often developers encounter or change it.
- Metrics find candidates; source inspection establishes design problems.
- Tool agreement strengthens a signal but is not required.

Keep the review bounded. Do not use subagents, run broad test suites, or turn it into a general defect hunt unless the user expands the scope.

## Setup

Record repository identity, review scope, dirty state, production roots, exclusions, SCC history window, and graph freshness and coverage.

Confirm SCC exposes `analyze`, `hotspots`, and `coupling`. Call `list_projects` as the first graph operation and reuse a matching fresh index. If either tool is unavailable, disclose the reduced evidence basis before continuing.

Read [references/mcp-playbook.md](references/mcp-playbook.md) when executing MCP calls. It contains tool syntax, query safeguards, coverage checks, and ranking rules.

## Workflow

### 1. Run independent evidence lanes

Run in parallel when practical. If calls must be serial, do not let one lane narrow, suppress, or rerank the other before both finish.

**SCC lane**

Use file analysis, history hotspots, and change coupling to find:

- concentrated implementation complexity
- complexity weighted by change frequency
- repeated co-change and possible change amplification
- tactical accretion or knowledge scattered across files

**Codebase-memory lane**

Use architecture, symbol metrics, fan-in/out, cycles, similarity, and change traces to find:

- central interfaces and dependency concentration
- hidden call paths, boundary crossings, and blast radius
- pass-through layers or adjacent layers with similar abstractions
- duplicated policy, algorithmic risk, and structural unknown unknowns

Validate each lane's ordering, completeness, snapshot caveats, and coverage independently. Failed or partial queries are evidence gaps, not negative evidence.

### 2. Synthesize by union

After both lanes finish:

- preserve SCC-only, graph-only, overlapping, and contradictory candidates
- key candidates by file and symbol without forcing unlike evidence into one score
- treat overlap as corroboration and disagreement as a question to inspect
- select 4–6 candidates from the union, including material lane-unique signals

### 3. Validate in source

Inspect each candidate's formal interface, informal contract, implementation, representative callers, relevant history, and comments or names that define the abstraction.

Ask five questions:

1. **Complexity burden:** How much must developers know, and how often do they pay that cost?
2. **Module depth:** Does a simple interface hide substantial knowledge and implementation complexity?
3. **Complexity direction:** Is complexity handled once below the interface or repeatedly pushed onto callers?
4. **Strategic direction:** Is the design becoming easier to change, or accumulating tactical exceptions?
5. **Simplification counterfactual:** Could a small change to ownership, the state model, or the default flow delete branches, modes, wrappers, or coordination rather than merely relocate them?

Before this stage, read [references/design-lenses.md](references/design-lenses.md) for the detailed *A Philosophy of Software Design* checks and interpretation cautions.

Run a focused reproduction only when an obvious high-impact failure is visible and cheap to check. Otherwise report design risk, not a verified defect.

### 4. Report and stop

Stop after the leading signals and material discrepancies from both lanes have been explained or rejected. Do not manufacture findings.

## Findings

Report only source-validated observations. For each finding include:

- **Complexity symptom:** change amplification, cognitive load, or unknown unknowns
- **Design issue:** the responsible design pattern
- **Provenance:** SCC-only, graph-only, both, or discrepancy
- **Evidence:** tool signal plus `path:line` observations
- **Leaked knowledge or obscurity:** what readers or callers must know or discover
- **Impact:** unnecessary complexity weighted by likely exposure
- **Direction:** the smallest strategic improvement that removes rather than relocates complexity, not an unsupported redesign

## Report shape

Keep the response concise:

1. **Summary:** overall complexity shape in 2–4 bullets.
2. **Independent evidence map:** SCC results, graph results, then overlaps and discrepancies.
3. **Validated findings:** at most 3–5 findings.
4. **Rejected signals:** misleading or contradictory evidence worth noting.
5. **Review basis:** repository identity, scope, dirty state, SCC window, graph coverage, and exclusions.

If no concern survives validation, say so and list the leading candidates from each lane to inspect next.
