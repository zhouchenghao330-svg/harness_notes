# Git workflow

- Before staging or pushing, inspect `git status --short --branch`; keep local configuration and unrelated artifacts out of commits.
- For “latest code”, separately check upstream divergence and working-tree cleanliness.
- For “local commits”, inspect the current branch/`HEAD`; `git log --all` answers a broader question and can overcount.
- If asked to inspect a remote without changing local state, use read-only remote commands; do not fetch or pull merely to answer.
- When a remote has advanced, check divergence before choosing a rebase-based recovery path.

## Related
- [[user/preferences]]
- [[projects/ai-human]]
