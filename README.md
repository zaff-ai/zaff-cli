# Zaff CLI

> Your personal knowledge graph from the command line.

## Install

```bash
pipx install zaff
```

(or `pip install zaff` inside a venv). It installs the `zaff` command.

## Quick start

```bash
zaff login                              # opens browser, OAuth flow
zaff memo search "..."                  # full-text + semantic search
zaff memo get <SID>                     # fetch a memo by sid
zaff run <SID>                          # execute a Python script memo
zaff media upload <PATH> [--tag <SID>]  # upload a file (image, PDF, etc.)
```

`--json` on the read commands emits machine-readable output for piping into `jq`.

## Uploading files

```bash
zaff media upload ~/photo.jpg                       # auto-tagged under System > Uploads
zaff media upload ~/scan.pdf --tag <leaf-tag-sid>   # attach to a specific tag
zaff media upload ~/big.mp4 --json                  # machine-readable response
```

`zaff media upload` runs a three-step dance: it asks the server for a presigned
S3 URL, PUTs the bytes directly to S3 (the Zaff server never sees the
file body), and finalizes by creating a Media memo. Re-uploading a file
you've already uploaded short-circuits at the first step (no PUT, no wasted
bandwidth) — the client SHA-256s the file once and the server matches it
against existing memos.

## Running script memos

`zaff run <sid>` fetches the memo, extracts the first ` ```python ` fenced
block, and runs the code via `uv run --python $(which python3) <tmpfile>`
with `ZAFF_TOKEN` and `ZAFF_URL` set in the env. Stdout and
stderr stream live; the script's exit code propagates.

```bash
zaff run <SID>                   # run with whatever token the CLI is using
zaff run <SID> --no-log          # skip writing the run-audit memo
```

Inside the script, `from zaff.client import Client` picks up the env
vars set by the runner. Requires [`uv`](https://docs.astral.sh/uv/) on `PATH`.

The CLI does **not** mint its own per-run token. The script subprocess
inherits the same identity (`ZAFF_TOKEN`) the CLI is using. To get a
narrower scope, shorter TTL, or `script_sid`-bound audit attribution, mint
a cli token via the **`mint_cli_token`** MCP tool (for example from Claude)
first and set its output as `ZAFF_TOKEN` before invoking
`zaff run`.

### Audit trail

After the script exits, `zaff run` posts a memo titled `Run of <script-title>
— rc=<N>` with rc, duration, started_at, and a captured stdout excerpt,
plus an `instance_of` ref pointing to the script. Pass `--no-log` to skip
the memo. If your token was minted with `script_sid=<sid>`, every API call
the script makes is also attributed in server logs.

## Configuration


- **`ZAFF_URL`** — server URL (default `https://zaff.ai`).
- **`--base-url`** — per-command override.

Credentials are stored at `~/.config/zaff/config.json` (mode 0600).

## Auth

`zaff login` runs the OAuth 2.0 authorization-code flow with PKCE: it registers a
client via Dynamic Client Registration (RFC 7591), opens your browser to the
authorize endpoint, and captures the callback on a one-shot loopback server.
Tokens auto-refresh on 401.

`zaff logout` clears stored credentials.

## License

MIT
