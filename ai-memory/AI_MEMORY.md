# AI Memory

## User
- Prefer evidence-backed answers tied to the real repository or runtime state.
- For narrow edits and lookups, execute directly; discuss design-heavy or cross-cutting changes first.
- Explain unfamiliar technical ideas in plain Chinese and layers before jargon.
- Before a commit or push, inspect `git status --short` and include only relevant files; keep cleanup separate from a feature publish.
- For “latest code”, distinguish upstream freshness from a dirty working tree. For “local commits”, inspect the current branch (`HEAD`), not all refs unless requested.
- For verification or audits, provide concrete findings or a clear no-issue verdict; test when asked whether something was tested.
- Details: [[user/preferences]] and [[lessons/git-workflow]].

## Active projects
- [[projects/ai-human]]: chat is ReAct-first; routing and writing are workflow-based. `required_sources` is a runtime control contract. The real writing API is `/api/v1/runs` with SSE. Development and testing normally use Docker Compose; do not assume a host-local application environment is available. For no-selection document edits, use locate-then-rewrite rather than guessing a target.
- [[projects/ecoli-dual-evidence]]: active research/paper project; write and review section-by-section, keep formal manuscript text separate from TODO notes, and do not overclaim readiness.
- [[projects/pi-obsidian-memory]]: a minimal Pi package that injects this file at session start.

## Key decision
- [[decisions/obsidian-as-agent-memory]]: Obsidian is Pi's long-term knowledge base. Do not build a separate memory database, vector retrieval layer, automatic extraction pipeline, or dedicated memory tool. At a natural stopping point, ask before saving durable cross-session knowledge; after approval, maintain the Vault and links with native tools.

## Reference
- Local development details: [[environments/local-dev]].
