# Local development

- Primary `ai-human` checkout: `/Users/zhouchenghao/python_file/wj/ai-human`.
- For Python tests in that repository, prefer `.venv/bin/pytest` if `pytest` is not on `PATH`.
- When macOS bytecode-cache permissions interfere, use `PYTHONPYCACHEPREFIX=/tmp/...` with targeted checks.
- For Docker Compose registry credential errors, inspect Docker Desktop credentials before changing repository image configuration; a clean temporary `DOCKER_CONFIG` can isolate credential-helper failures.
- When checking Mac thermal spikes, inspect Docker/virtualization workload before assuming hardware damage; re-check live signals rather than relying on old findings.
- `ai-human` Git auth: `origin` (`ssh://git@192.168.130.75:16003/dev-data/ai-human.git`) uses the registered key whose local public half is `~/.ssh/id_ed25519_gitee.pub`. Its private key has a non-default filename, so OpenSSH will not discover it automatically when a fresh Pi/Orca session's ssh-agent has no identities. For fetch/pull, use `GIT_SSH_COMMAND='ssh -i ~/.ssh/id_ed25519_gitee -o IdentitiesOnly=yes -o BatchMode=yes'`; a prior `ssh-add` does not guarantee a later session is populated. If a persistent fix is wanted, add a host-specific SSH config with `IdentityFile` (and optionally `AddKeysToAgent`/`UseKeychain`) rather than changing repository settings.
