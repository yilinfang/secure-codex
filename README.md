# secure-codex

Codex CLI stores OAuth tokens in a plaintext `auth.json` on disk. OpenAI currently provides no way to revoke these tokens ([openai/codex#2557](https://github.com/openai/codex/issues/2557)) — running `codex logout` only deletes the local file while the token remains valid on their servers. If `auth.json` is leaked, the token is permanently compromised.

**secure-codex** mitigates this by keeping `auth.json` encrypted on disk ([`age`](https://github.com/FiloSottile/age)) and decrypting it only into RAM (`/dev/shm` tmpfs) at runtime. Plaintext credentials never touch persistent storage. Sessions are cached across invocations and automatically cleared on reboot, after a configurable TTL (default: 1 day), or via `SECURE_CODEX_CLEANUP=true`.

_Currently only Linux is supported and tested._

## Installation

**One-liner (curl):**

```bash
curl -fsSL https://raw.githubusercontent.com/yilinfang/secure-codex/main/secure-codex -o ~/.local/bin/secure-codex && chmod +x ~/.local/bin/secure-codex
```

**One-liner (wget):**

```bash
wget -qO ~/.local/bin/secure-codex https://raw.githubusercontent.com/yilinfang/secure-codex/main/secure-codex && chmod +x ~/.local/bin/secure-codex
```

**From source:**

```bash
git clone https://github.com/yilinfang/secure-codex.git
cd secure-codex
chmod +x secure-codex
ln -s "$(pwd)/secure-codex" ~/.local/bin/secure-codex
```

## Prerequisites

The following tools must be available on `PATH`:

- [`codex`](https://github.com/openai/codex) — the CLI being wrapped
- [`age`](https://github.com/FiloSottile/age) — for decrypting `auth.json.age`
- [`jq`](https://jqlang.org/) — for JSON validation
- Standard coreutils: `ln`, `rm`, `mkdir`, `mktemp`, `chmod`, `uname`, `basename`, `readlink`, `stat`, `date`

## Setup

1. Install Codex CLI and configure it normally (`~/.codex/auth.json`).
2. Encrypt your auth file with `age`:
   ```bash
   age -e -R ~/.age/recipients.txt -o ~/.codex/auth.json.age ~/.codex/auth.json
   ```
3. Optionally remove the plaintext `~/.codex/auth.json`.

## Usage

```bash
secure-codex <codex args...>
```

All CLI arguments are passed through directly to `codex` without wrapper-specific flag parsing. The script has no flags of its own (including `--help`); configuration is done entirely through environment variables.

### Environment Variables

| Variable                       | Description                       | Default                  |
| ------------------------------ | --------------------------------- | ------------------------ |
| `SECURE_CODEX_AUTH_ENC`        | Path to encrypted auth file       | `~/.codex/auth.json.age` |
| `SECURE_CODEX_REAL_CODEX_HOME` | Path to real Codex home directory | `~/.codex`               |
| `SECURE_CODEX_CLEANUP`         | Remove RAM session and exit       | `false`                  |
| `SECURE_CODEX_TTL`             | Session TTL in seconds            | `86400` (1 day)          |

`SECURE_CODEX_CLEANUP` accepts: `true`, `false`, `1`, `0`, `yes`, `no`, `on`, `off`.

When a cached session's `auth.json` is older than `SECURE_CODEX_TTL` seconds, it is automatically wiped and re-decrypted on the next invocation.

### Examples

```bash
# Normal usage — decrypt and run codex
secure-codex chat

# Custom encrypted file location
SECURE_CODEX_AUTH_ENC=/path/to/auth.json.age secure-codex chat

# Clean up the RAM session
SECURE_CODEX_CLEANUP=true secure-codex
```

## How It Works

1. **Validates** the environment: Linux OS, required tools, `/dev/shm` writability
2. **Creates a RAM home** at `/dev/shm/codex-home-<uid>` with strict ownership and permission checks
3. **Manages sessions** — reuses an existing decrypted session if valid, otherwise symlinks non-auth files from `~/.codex` and decrypts `auth.json.age` into RAM via atomic temp-file-then-move
4. **Runs Codex** with `CODEX_HOME` pointing to the RAM directory

## Security

- Plaintext credentials exist only in RAM (`/dev/shm` tmpfs), never on disk
- `umask 077` and `chmod 600`/`700` enforce restrictive permissions
- Atomic writes via temp file + `mv` prevent partial-state exposure
- Ownership validation (`-O`) and symlink attack prevention on the RAM home directory
- `rm` operations are confined to `/dev/shm` paths
- Sessions are automatically cleared on reboot or after TTL expiry (default: 1 day)
