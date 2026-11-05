# ai-human

Repository: `/Users/zhouchenghao/python_file/wj/ai-human`

- `mini_runtime`: chat is ReAct-first; routing and writing are workflow-based. Wire no-thinking explicitly for workflow clients rather than relying on defaults.
- `required_sources` is a runtime control contract, not merely metadata.
- The current public business-data interface is the composite `query_current_business_data(intent)`; deterministic internals handle search, get, and traversal. Surface multi-match ambiguity instead of guessing.
- The real writing API is `/api/v1/runs` with SSE; `/api/v1/mini/execute` is retired.
- For no-selection document edits, use locate-then-rewrite: resolve the target before rewriting rather than treating cursor or active-block state as a selection.
- `ContextManager.build(state, profile)` centralizes context assembly; raw-query routing should not inherit generic chat/editor context.
- Keep runtime memory-pipeline work separate from the Pi/Obsidian personal knowledge base.
- 2026-07: delivered configuration-driven business-ontology resource mentions: Agent business config declares one primary resource type; users can multi-select up to five real objects as structured `resource_mention` blocks, while trusted backend templates turn their codes into the LangGraph query. This lets ReAct query and compare real entities/relations rather than infer referents from text. Preserve the latest four business-query user turns (not four calls), so all calls for one multi-object comparison remain available.

## Related
- [[user/preferences]]
- [[environments/local-dev]]
- [[environments/server-75]]
