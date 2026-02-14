## Project Overview

**secure-codex** is a Linux-only Bash script that provides a secure wrapper for running Codex CLI. It decrypts `age`-encrypted auth credentials into RAM (`/dev/shm` tmpfs), keeping plaintext secrets off disk. Sessions are cached across invocations and cleared on reboot, via `SECURE_CODEX_CLEANUP=true`, or automatically after a configurable TTL (default: 1 day).

## Running

```bash
SECURE_CODEX_AUTH_ENC=... SECURE_CODEX_REAL_CODEX_HOME=... SECURE_CODEX_CLEANUP=true ./secure-codex <codex args...>
```

Linux only — requires `/dev/shm` tmpfs. No build step; the script is the entire project.

## External Tool Dependencies

`codex`, `age`, `jq`, `ln`, `rm`, `mkdir`, `mktemp`, `chmod`, `uname`, `basename`, `readlink`, `stat`, `date`

## Architecture

Single script (`secure-codex`) with four phases:

1. **Validation** — OS check, argument parsing, tool availability, `/dev/shm` writability
2. **RAM home setup** — Creates `/dev/shm/codex-home-$(id -u)` with ownership/permission validation; guards against symlink attacks
3. **Session management** — Auto-cleans stale sessions exceeding `SECURE_CODEX_TTL` (default 86400s/1 day); reuses existing decrypted session if valid, otherwise creates fresh: symlinks non-auth files from `~/.codex`, decrypts `auth.json.age` via `age` into a temp file, validates JSON with `jq`, atomically moves into place
4. **Execution** — Sets `CODEX_HOME` to RAM directory, runs `codex` with passthrough args

## Key Conventions

- `set -euo pipefail` strict mode throughout
- `umask 077` for all file creation
- Atomic writes via temp file + `mv` with `trap` cleanup
- Exit codes: 0 = success, 1 = initialization/runtime error, 2 = argument error
- Safety guards: refuses `rm` outside `/dev/shm`, validates directory ownership (`-O`), prevents symlink traversal
