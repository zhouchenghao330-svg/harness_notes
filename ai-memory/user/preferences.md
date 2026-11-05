# Preferences

- Prefer evidence-backed answers tied to the real repository or runtime state.
- For narrow edits and lookups, execute directly; discuss design-heavy or cross-cutting changes first. Decide autonomously: simple work can proceed directly; complex work should follow a written spec.
- 对明确的窄范围改动，只改用户点名的文件和行为；不要擅自扩展为防复发重构、补脚本、加跨层校验或额外测试。若发现可能需要扩大范围，先用一句话说明原因并等用户确认。
- Explain unfamiliar technical ideas in plain Chinese and layers before jargon.
- Before a commit or push, inspect `git status --short` and include only relevant files; keep unrelated local noise out.
- For “latest code”, answer upstream freshness first and separately report whether the working tree is dirty.
- For “local commits”, inspect the current branch/`HEAD`; say explicitly if an answer instead covers all refs.
- For verification or audits, provide concrete findings or a clear no-issue verdict; test when asked whether something was tested.
- For paper writing, work section-by-section when requested; keep formal prose separate from TODO notes and avoid overclaiming readiness.
- For short form writing (for example, a salary-review reason), provide a concise paste-ready version first.
