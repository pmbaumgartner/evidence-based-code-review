# MCP Playbook

Use this reference when executing an evidence-based review. Tool schemas can evolve; inspect current descriptors if a call is rejected.

## Independent lane discipline

Treat SCC and codebase-memory as complementary discovery lanes, not a serial filter:

1. Establish shared scope, production roots, exclusions, and snapshot caveats.
2. Complete and validate the SCC ranking independently.
3. Complete and validate the graph ranking independently.
4. Only then synthesize the union by file/symbol, preserving SCC-only, graph-only, overlapping, and contradictory signals.
5. Source-validate a bounded selection that includes material lane-unique candidates; overlap is corroboration, not an admission requirement.

Run the lanes concurrently when practical. If calls must be serial, do not use early results from one lane to narrow, suppress, or rerank the other. Do not combine unlike metrics into a pseudo-precise score.

## Runtime tool syntax

Discover the tools, then confirm the exact invocation form with a harmless call such as `list_projects` or a tool descriptor. Record the successful form in the review basis. Do not assume namespacing from the displayed server name. For example, one gateway may accept a flat tool name with a separate server selector:

```text
tool="list_projects", server="codebase-memory-mcp"
tool="analyze", server="scc"
```

Another runtime may expose prefixed names. Use only the form returned and accepted by that runtime; if `server/tool` fails, rediscover rather than guessing.

For `query_graph`, probe a harmless bounded call with `format:"json"`. Current upstream accepts it and emits `columns` plus row arrays even though some published input schemas omit the `format` property. If the client accepts the argument, prefer this structured mode for every query that will be ranked or merged.

## SCC calls

Use one explicit history depth. Keep history calls at the repository root, but choose file-analysis paths from the Git/filesystem inventory and graph-coverage reconciliation.

For a small, homogeneous repository with bounded output:

```text
analyze({
  path: "/absolute/repo",
  by_file: true,
  cognitive: true,
  sort: "complexity",
  limit: 100
})
```

For a large, polyglot, monorepo, or artifact-heavy repository, an optional compact root summary can establish scale and languages:

```text
analyze({path: "/absolute/repo", by_file: false, cognitive: true})
```

Then analyze the discovered owned categories or components separately; these paths are examples, not conventions:

```text
analyze({path: "/absolute/repo/component-a", by_file: true, cognitive: true, sort: "complexity", limit: 100})
analyze({path: "/absolute/repo/packages/service-b", by_file: true, cognitive: true, sort: "complexity", limit: 100})
analyze({path: "/absolute/repo/owned-tools", by_file: true, cognitive: true, sort: "complexity", limit: 100})
```

Always keep history analysis at the repository root:

```text
hotspots({path: "/absolute/repo", depth: 1000, limit: 50})
coupling({path: "/absolute/repo", depth: 1000, limit: 50})
coupling({path: "/absolute/repo", file: "relative/path.ext", depth: 1000, limit: 30})
```

Interpretation:

- `analyze` reads the filesystem snapshot, including eligible modified or untracked files; it is not necessarily a HEAD snapshot. History calls cannot rank uncommitted files. Record dirty state and reconcile the inputs.
- When analysis is partitioned, merge all owned production rows client-side into one global numeric ranking. Keep separate rankings for tests, benchmarks, migrations, and owned scripts/tooling. Keep fixtures, generated code, and vendored code separate or excluded.
- Capture SCC candidates before consulting graph architecture. After both lanes complete, architecture may help explain component boundaries, but it is never proof that the file inventory is complete.
- File complexity is partly a size signal. Source inspection and post-synthesis symbol evidence can refine it, but lack of graph corroboration does not discard an SCC candidate.
- `hotspots.score` is normalized complexity × commits, not defect probability. Preserve `commits`, `linesChanged`, `authors`, and the returned date window as context.
- Repo-wide coupling uses symmetric Jaccard-like degree. File-specific coupling reports directional `couple` and `reverse`; a large gap suggests a hub relationship rather than peer coupling.
- Common source↔test and manifest↔lockfile pairs are expected. Unexpected cross-module policy coupling deserves inspection.

SCC's `no_duplicates` option concerns duplicate-file accounting in line counts; it is not a source-clone detector.

## Index setup and coverage

```text
list_projects({include_details: true})
index_repository({
  repo_path: "/absolute/repo",
  name: "stable-project-name",
  mode: "moderate",
  persistence: false
})
index_status({project: "stable-project-name", verbose: true})
get_graph_schema({project: "stable-project-name"})
```

Reuse an index only when its root matches the repository. Use `index_repository` when absent or after a material external update. Before citing files:

```text
check_index_coverage({
  project: "stable-project-name",
  paths: ["src/a.py", "src/b.py"]
})
```

Before saying a bounded area has no cycles, duplicates, callers, dead code, or other matches:

```text
check_index_coverage({project: "stable-project-name", scopes: ["src/domain/"]})
```

Inspect skipped and parse-partial paths directly. `indexed_no_recorded_gap` is still not proof of completeness.

## Architecture and changed-code scope

```text
get_architecture({
  project: "stable-project-name",
  aspects: ["overview", "hotspots", "boundaries", "cycles", "clusters"]
})

detect_changes({
  project: "stable-project-name",
  scope: "impact",
  direction: "both",
  depth: 3,
  base_branch: "main",
  limit: 200
})
```

Use `since` instead of `base_branch` when the user gives an exact revision. Record merge base, changed files, impacted total, truncation, and direction/depth.

Architecture hotspot output can include builtins and test utilities even when index metadata suggests those areas are excluded. Prefer path-scoped architecture calls for production roots, reconcile returned paths with `index_status`, and filter to owned production symbols before prioritizing. Clusters reveal de-facto seams, not necessarily desired architecture.

## Symbol complexity and hot-path queries

Check the schema first. Query `Function` and `Method` labels separately if the backend handles label unions poorly.

Fetch bounded production symbols and verify the returned numeric ordering. Current `query_graph` projects at most `max_rows` rows before applying `ORDER BY` for a simple non-aggregate return. Therefore, setting `max_rows=N` on a top-`N` query sorts only an initial subset and can return the wrong ranking. Cypher `LIMIT`, by contrast, is applied after ordering when `max_rows` has not already truncated projection.

Use this primary ranking recipe:

1. Query the bounded production labels/roots with explicit numeric metric columns.
2. Use Cypher `ORDER BY` with projected aliases, a deterministic tie-break, and `LIMIT N`.
3. Omit `max_rows`; its default 100k ceiling is then the projection budget. If it must be supplied, set it at or above the entire candidate universe, never merely `N`.
4. Request `format:"json"` when accepted. Current upstream supports this compatibility mode even though some published tool schemas omit it. In `mcpScript`, consume `result.data.structuredContent.rows`; direct MCP calls may expose the parsed object directly.
5. Parse rendered metric cells as numbers and verify the final ranking is monotonically descending. Treat a warning, timeout, omission, unexpected shape, insertion-ordered output, unexplained empty result, or results skewed to one label as failure.
6. On ranking failure, rerun `Function` and `Method` separately. If needed, partition further by production root or descending metric threshold, merge the complete partitions numerically, and verify the final ordering.

Do not request the entire candidate universe merely to sort it inside `mcpScript`. MCP output guarding can summarize a large `structuredContent.rows` array before the script receives it, even when the underlying tool call succeeded. Keep each tool result below the guard by using label, root, and metric-threshold partitions. If `rows` is absent, represented only by preservation metadata, or otherwise omitted, discard that result and rerun smaller partitions; never rank from the visible preview.

For example:

```cypher
MATCH (f:Function|Method)
WHERE f.file_path STARTS WITH 'src/'
RETURN f.qualified_name AS symbol,
       f.file_path AS file,
       f.start_line AS line,
       f.complexity AS cyclomatic,
       f.cognitive AS cognitive
ORDER BY cognitive DESC, cyclomatic DESC, symbol ASC
LIMIT 20
```

Use separate `Function` and `Method` queries and merge numerically when label alternation is unsupported or suspect.

The default ceiling and current in-memory sort make a broad query unsuitable for very large candidate sets. If the bounded universe may approach 100k rows, the query times out, or a result is truncated, use this fallback:

1. Query `Function` and `Method` separately, then partition by production root/module or add a descending metric threshold such as `f.cognitive >= threshold`.
2. For adaptive thresholds, try values such as `[100, 50, 20, 10, 5, 0]` until the complete combined result contains at least `N` candidates. Once it does, every excluded symbol is below the threshold, so the global top `N` is present.
3. Request JSON, parse metrics numerically, merge partitions, sort client-side with a deterministic tie-break, and take `N`.
4. Verify every partition is complete and the final values are monotonically descending.

If `format:"json"` is rejected by an older server, parse the fixed-column text table only as a last resort. Inspect the returned shape and fail closed on an unexpected row, omission, or count mismatch.

For broader metric collection, query a fixed column order such as:

```cypher
MATCH (f:Function)
WHERE f.file_path STARTS WITH 'src/'
RETURN f.qualified_name AS symbol,
       f.file_path AS file,
       f.start_line AS line,
       f.end_line AS end_line,
       f.complexity AS cyclomatic,
       f.cognitive AS cognitive,
       f.loop_count AS loop_count,
       f.loop_depth AS loop_depth,
       f.transitive_loop_depth AS transitive_loop_depth,
       f.linear_scan_in_loop AS linear_scan_in_loop,
       f.alloc_in_loop AS alloc_in_loop,
       f.recursion_in_loop AS recursion_in_loop,
       f.unguarded_recursion AS unguarded_recursion,
       f.param_count AS param_count,
       f.max_access_depth AS max_access_depth
```

Replace the path predicate with the repository's production roots. Query `Function` and `Method` separately. If a result is too broad, query one root/module at a time rather than accepting truncation. If output is truncated or saved to a file by the gateway, parse the saved complete result rather than ranking the visible prefix.

Use this focused query for algorithmic-risk candidates:

```cypher
MATCH (f:Function)
WHERE f.transitive_loop_depth >= 3
   OR f.linear_scan_in_loop >= 1
   OR f.recursion_in_loop >= 1
   OR f.unguarded_recursion = true
RETURN f.qualified_name, f.file_path, f.start_line,
       f.complexity, f.cognitive, f.loop_depth,
       f.transitive_loop_depth, f.linear_scan_in_loop,
       f.alloc_in_loop, f.recursion_in_loop,
       f.unguarded_recursion
```

Validate complexity against source. A high transitive loop depth can be intended numerical work; `alloc_in_loop` often needs workload and allocation-lifetime evidence before it is actionable.

## Fan-in and fan-out

Run separate queries to avoid accidental Cartesian multiplication:

```cypher
MATCH (caller)-[:CALLS]->(f:Function)
WHERE f.file_path STARTS WITH 'src/'
RETURN f.qualified_name AS symbol, f.file_path AS file,
       f.start_line AS line, count(DISTINCT caller) AS fan_in
ORDER BY fan_in DESC
LIMIT 50
```

```cypher
MATCH (f:Function)-[:CALLS]->(callee)
WHERE f.file_path STARTS WITH 'src/'
RETURN f.qualified_name AS symbol, f.file_path AS file,
       f.start_line AS line, count(DISTINCT callee) AS fan_out
ORDER BY fan_out DESC
LIMIT 50
```

Repeat for `Method` when present, or use label alternation when the current server handles it correctly. Request JSON and do not set `max_rows` to the desired top count; let Cypher `LIMIT` run after aggregation and ordering. Parse degree values numerically and verify ordering. These count resolved graph edges, not all runtime calls. Builtins, dynamic dispatch, callbacks, macros, reflection, and FFI can distort the result.

## Duplicate candidates

Query graph-generated token-similarity edges:

```cypher
MATCH (a)-[r:SIMILAR_TO]->(b)
WHERE a.file_path STARTS WITH 'src/'
  AND b.file_path STARTS WITH 'src/'
RETURN a.qualified_name AS symbol_a, a.file_path AS file_a,
       a.start_line AS line_a, b.qualified_name AS symbol_b,
       b.file_path AS file_b, b.start_line AS line_b,
       r.jaccard AS jaccard, r.same_file AS same_file
ORDER BY r.jaccard DESC
LIMIT 100
```

Request JSON and parse `jaccard` numerically before ranking; decimal properties may otherwise be ordered lexically by the current executor. Do not set restrictive `max_rows` on the ordered query. No rows means no indexed `SIMILAR_TO` edges, not no duplication. The returned `total` is only the post-limit row count, not the number of all matching edges. Inspect both snippets. Strong evidence of harmful duplication requires shared behavior or policy plus a plausible divergence cost; similar tests or boundary adapters often should remain explicit.

Repeated symbol names can identify parallel implementations worth comparison, but are weaker evidence than `SIMILAR_TO`:

```cypher
MATCH (f:Function)
WITH f.name AS name, collect(f) AS defs
WHERE size(defs) > 1
UNWIND defs AS f
RETURN name, f.qualified_name, f.file_path, f.start_line
ORDER BY name
```

## Tracing and source evidence

Discover the exact qualified name before tracing:

```text
search_graph({
  project: "stable-project-name",
  name_pattern: ".*CandidateName.*",
  label: "Function",
  fields: ["complexity", "cognitive", "signature"],
  limit: 50
})

trace_path({
  project: "stable-project-name",
  function_name: "exact.qualified.name",
  direction: "both",
  depth: 3,
  risk_labels: true,
  include_evidence: true,
  include_tests: false,
  limit: 200
})
```

Follow `has_more`, `next`, or cursor fields. Use `get_code_snippet` for exact symbols, then direct file reads when more context is needed or coverage is partial. Treat trace `risk_labels` only as prioritization hints: they are not finding severity, and transitive risk propagation can label harmless accessors, builtins, or wrappers as `CRITICAL` or recursive.

## Dead-code candidates

```text
search_graph({
  project: "stable-project-name",
  label: "Function",
  max_degree: 0,
  exclude_entry_points: true,
  limit: 100
})
```

Paginate. Never report dead code from graph isolation alone. Verify exports, framework registration, reflection, plugin hooks, callbacks, FFI, macros, scripts, and direct textual references.

## Union evidence matrix

After both independent lanes complete, maintain a small working table before source validation:

| Candidate | Provenance | SCC file rank | History/coupling | Graph rank/role | Discrepancy or complement | Source verification | Coverage | Decision |
|---|---|---:|---|---|---|---|---|---|

`Provenance` is SCC-only, graph-only, both, or discrepancy. Preserve leading lane-unique candidates instead of requiring populated cells from both tools. Treat disagreement as a question to inspect, not a reason to average ranks or discard the candidate.

The table is an analysis aid. Keep it out of the final report unless the user asks for the full audit trail.
