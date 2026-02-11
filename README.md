# secure-codex

Codex CLI stores OAuth tokens in a plaintext `auth.json` on disk. OpenAI currently provides no way to revoke these tokens ([openai/codex#2557](https://github.com/openai/codex/issues/2557)) — running `codex logout` only deletes the local file while the token remains valid on their servers. If `auth.json` is leaked, the token is permanently compromised.

**secure-codex** mitigates this by keeping `auth.json` encrypted on disk ([`age`](https://github.com/FiloSottile/age)) and decrypting it only into RAM (`/dev/shm` tmpfs) at runtime. Plaintext credentials never touch persistent storage. Sessions are cached across invocations and automatically cleared on reboot or via `--cleanup`.

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
- Standard coreutils: `ln`, `rm`, `mkdir`, `mktemp`, `chmod`, `uname`, `basename`, `readlink`

## Setup

1. Install Codex CLI and configure it normally (`~/.codex/auth.json`).
2. Encrypt your auth file with `age`:
   ```bash
   age -e -R ~/.age/recipients.txt -o ~/.codex/auth.json.age ~/.codex/auth.json
   ```
3. Optionally remove the plaintext `~/.codex/auth.json`.

## Usage

```bash
./secure-codex [OPTIONS] -- <codex args...>
```

### Options

| Flag                     | Description                       | Default                  |
| ------------------------ | --------------------------------- | ------------------------ |
| `--auth-enc PATH`        | Path to encrypted auth file       | `~/.codex/auth.json.age` |
| `--real-codex-home PATH` | Path to real Codex home directory | `~/.codex`               |
| `--cleanup`              | Remove RAM session and exit       | —                        |
| `-h`, `--help`           | Show usage and exit               | —                        |

### Examples

```bash
# Normal usage — decrypt and run codex
./secure-codex -- chat

# Custom encrypted file location
./secure-codex --auth-enc /path/to/auth.json.age -- chat

# Clean up the RAM session
./secure-codex --cleanup
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
- Sessions are automatically cleared on reboot
