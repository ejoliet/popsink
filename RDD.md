# popsink

> Run a command, keep using it normally, and when it finishes get the same **structured completion event** on your terminal and your phone — no notification framework installed. Events are JSON envelopes; sinks are URIs; the whole tool is one stdlib-only Python file.

![Python](https://img.shields.io/badge/python-3.11+-blue)
![Deps](https://img.shields.io/badge/third--party%20deps-0-brightgreen)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## Purpose

**Problem**: Knowing when a long command finishes means babysitting a terminal. Existing tools notify but don't emit machine-readable events you can pipe, filter, and fan out.

**Differentiator** (vs `ntfy --wait-cmd`, `noti`, Apprise): those deliver notifications. popsink emits a **structured completion event** (stable JSON envelope) and fans it out to URI-addressed sinks — composable with `jq`, future filter stages, and future emitters — with zero third-party dependencies and zero required accounts (Slack webhooks, when used, require a Slack workspace).

**v0.1 scope**: local dev and cron on macOS/Linux. **CI and Airflow are explicitly out of scope** until process-group and SIGTERM behavior is implemented and tested (see Signals).

**Gates**:
- **Gate 0 (spike, GO/NO-GO)**: `wrap` → console JSON + ntfy, correct exit status, output tee.
- **Gate 1 (usable tool)**: `send`, generic HTTPS + Slack formatting, parallel delivery, retry, env expansion.

---

## Envelope v0 — PROVISIONAL

Do not treat as frozen. Freeze as v1 only after Gate 1 plus real usage with at least one more emitter.

```json
{
  "v": 0,
  "id": "b1946ac92492d2347c6235b4d2611184",
  "source": "popsink",
  "kind": "command.finished",
  "ts": "2026-08-14T17:03:22Z",
  "title": "make build",
  "status": "failure",
  "exit_code": 2,
  "duration_s": 341.2,
  "body": "last lines of output...",
  "meta": { "host": "obiwan-kenobi", "cwd": "/home/e/proj" }
}
```

| Field | Type | Rules |
|-------|------|-------|
| `v` | int | `0` while provisional |
| `id` | str | `uuid.uuid4().hex`, set at creation |
| `source` | str | Emitting tool; `wrap` sets `"popsink"` |
| `kind` | str | Dotted event type; `wrap` sets `"command.finished"` |
| `ts` | str | ISO 8601 UTC, `Z` suffix |
| `title` | str | Reconstructed display command via `shlex.join(argv)` (original shell quoting is unrecoverable) |
| `status` | enum | `success` \| `failure` \| `info` |
| `exit_code` | int \| null | Null for non-command events |
| `duration_s` | float \| null | Wall time |
| `body` | str | Output tail, capped at **4096 UTF-8 bytes** (not characters — matches ntfy's message limit); truncate at a character boundary, prefix `…` when truncated |
| `meta` | object | **All extensions live here.** No arbitrary top-level keys. |

**`send` input rules (Gate 1)**: each stdin line must be a JSON **object** (bare numbers, strings, arrays are invalid records). `title` required. Defaults filled: `v:0`, `id` new, `source:"stdin"`, `kind:"event"`, `ts:now`, `status:"info"`. Unknown top-level keys are relocated under `meta` (lossless, keeps the eventual Go struct clean).

**Conditional rendering**: sinks must omit exit/duration segments when null. Never render `exit None`.

---

## Process Transparency & Exit Status — the hard problem

### What `wrap` actually does

- Runs the child with `stdout=PIPE`, `stderr=PIPE` (argv exec, **no shell**).
- Two reader threads, one per stream: each tees bytes back to the parent's matching stream (child stdout → popsink stdout, child stderr → popsink stderr) **and** appends lines to one lock-protected bounded tail (`collections.deque`). Exact inter-stream ordering is not guaranteed.
- So `popsink wrap -- gen-json > out.json` keeps stdout clean: child stdout goes to `out.json`, child stderr stays on the terminal, and the popsink summary goes to stderr (see console sink).

### What `wrap` does NOT do

> ⚠️ **wrap is not TTY-transparent.** The child sees pipes, not a terminal (`isatty()` is false). Programs may change buffering, disable colors/progress bars, or refuse interactive prompts. A PTY mode is a possible later feature; not v0.1. Document this in `--help`.

### Exit status mapping (POSIX: signal death = negative returncode)

| Child result | popsink exits |
|---|---|
| `rc >= 0` | `rc` unchanged |
| `rc == -N` (killed by signal N) | `128 + N` (e.g. SIGINT → 130) |
| Spawn failure (not found / not executable) | `127`, with envelope `status:"failure"`, `exit_code:127` |

### Signals

- **SIGINT**: do **not** forward. The terminal delivers Ctrl-C to the whole foreground process group, so the child already received it; forwarding would signal it twice. popsink catches its own SIGINT, waits for the child, emits the envelope (child's mapped exit status), exits with that status.
- **SIGTERM forwarding, process groups**: required for CI/Kubernetes/Airflow cancellation — **deferred**, which is why those environments are out of scope for v0.1.

---

## Repository Layout

```
popsink/
├── popsink.py        # Single readable file, stdlib only. No line-count target.
├── test_popsink.py   # stdlib unittest
├── README.md
├── IDEAS.md          # Non-binding future directions. Must not influence spike architecture.
└── LICENSE           # MIT
```

stdlib only: `argparse`, `subprocess`, `threading`, `collections`, `json`, `urllib.request`, `uuid`, `shlex`, `datetime`, `socket`, `os`, `sys`, `signal`, and (Gate 1) `concurrent.futures`.

---

## Quick Start

```bash
git clone https://github.com/ejoliet/popsink.git && cd popsink

# 1. No sink → JSON envelope on stdout
python3 popsink.py wrap -- sleep 2

# 2. Phone push — create a strong topic ONCE, subscribe, reuse it.
#    ntfy topics are effectively passwords; never use low-entropy names.
TOPIC="ej-$(python3 -c 'import secrets; print(secrets.token_urlsafe(12))')"
echo "Subscribe to this topic in the ntfy app, then save it: $TOPIC"
python3 popsink.py wrap --sink "ntfy://ntfy.sh/$TOPIC" -- make build

# 3. stdout stays clean for pipes (summary goes to stderr)
python3 popsink.py wrap --sink "ntfy://ntfy.sh/$TOPIC" -- ./gen-report > report.json
```

Gate 1 additions:

```bash
# Pipe events from anywhere
echo '{"title":"backup done","status":"success"}' | python3 popsink.py send --sink "ntfy://ntfy.sh/$TOPIC"

# Slack, secret kept out of shell-expanded argv — note the SINGLE quotes:
export SLACK_SINK='https://hooks.slack.com/services/T00/B00/xxx?format=slack'
python3 popsink.py wrap --only-failures --sink '${SLACK_SINK}' -- pytest
```

---

## CLI Contract

```
Gate 0:  popsink.py wrap [--sink URI]... [--tail N] -- CMD [ARGS...]
Gate 1:  popsink.py wrap [--sink URI]... [--only-failures] [--tail N] -- CMD...
         popsink.py send [--sink URI]... [--only-failures]
```

Rules:

1. `wrap` requires `--`; missing → usage error, exit 2, before anything runs.
2. Unknown sink scheme → error naming it and listing supported schemes, exit 2, before running the child.
3. `--sink` repeatable; default `console://` when none given.
4. `--tail N`: lines kept for `body` (default 5), then the 4096-byte cap applies.
5. Sink failures print one redacted stderr warning each (`popsink: sink failed: https://hooks.slack.com/…`) and **never** change `wrap`'s exit status.
6. `send` exit status: `0` only if every input record was valid **and** every delivery succeeded; otherwise `1`. Processing continues past bad lines/sinks either way.
7. Env expansion in sink URIs (Gate 1): `os.path.expandvars` applied by popsink. Users should single-quote placeholders so secrets stay out of shell-expanded argv.

---

## Sink Reference

| Scheme | Gate | Delivery |
|--------|------|----------|
| `console://` | 0 | **Compact JSON envelope to stdout** (default; pipe-friendly, `jq`-ready). `console://?format=pretty` → human summary (glyph ✓/✗, title, exit, duration, body) to **stderr**. |
| `ntfy://host/topic` | 0 | `POST https://<host>/<topic>`, body = envelope `body` (or `(no output)`); headers: `Title:` `<title> — <status>` + ` (exit N, Xs)` only when non-null; `Priority: high` on failure else `default`; `Tags: x` / `white_check_mark`. **Always HTTPS.** Verify exact header names against https://docs.ntfy.sh/publish/ at build time. |
| `https://...` | 1 | `POST` full envelope JSON, `Content-Type: application/json`. |
| `https://...?format=slack` | 1 | `POST {"text": "<glyph> *<title>* — <status>[ , exit N, Xs]\n```<body>```"}`; strip `format` from the forwarded URL. |
| `http://...` | 1 | Generic JSON webhook, identical to `https://`, with a plaintext warning on stderr. **Never** ntfy semantics. |

Plaintext self-hosted ntfy: **unsupported in v0.1** (a `ntfy+http://` scheme may come later on demand). This resolves former Q1; console stream routing above resolves former Q2.

---

## Delivery Semantics

**Best-effort. Duplicate delivery is possible.** (A lost HTTP response after a successful POST + a retry = two notifications. The `id` field enables receiver-side dedup but popsink cannot guarantee it.) Never claim "at-most-once".

| | Gate 0 | Gate 1 |
|---|---|---|
| Attempts | 1, no retry | Up to 2 |
| Retry on | — | Connection errors, timeouts, HTTP 408, 429, 5xx only. **Never retry other 4xx** (malformed payload, disabled webhook — retries can't fix these). |
| Concurrency | Sequential (≤2 sinks in scope, 5 s timeout each) | Parallel across sinks via `ThreadPoolExecutor` — one dead sink must not serially block the rest |
| Per-attempt timeout | 5 s | 5 s, 1 s backoff before retry |

---

## Security Notes

- **Data-leak boundary**: envelopes carry the command line, output tail, hostname, and cwd. Command args and build logs can contain credentials. Only point sinks at services you trust with that data; this warning belongs in `--help` for `--sink`.
- **Secrets in sink URIs** (webhook paths are bearer tokens): pass single-quoted placeholders (`--sink '${SLACK_SINK}'`) so popsink expands them internally; double quotes put the full secret in argv where any local process can read it.
- **Redaction**: any warning or error that prints a sink URI truncates after the hostname.
- **ntfy topics are capability URLs**: generate with `secrets.token_urlsafe`, treat like passwords.
- Never write secrets to any file (env-var-only, the numwatch/cullroom invariant).

---

## Testing

```bash
python3 -m unittest test_popsink.py -v
```

| Test | Covers |
|------|--------|
| `test_exit_code_passthrough` | `sh -c 'exit 3'` → 3 |
| `test_signal_exit_mapping` | Child killed by SIGTERM → popsink exits 143 (`128+15`) |
| `test_spawn_failure` | Nonexistent binary → 127 + failure envelope |
| `test_stream_separation` | Child stdout → parent stdout, child stderr → parent stderr; pretty summary on stderr only |
| `test_tail_bytes_cap` | `body` ≤ 4096 UTF-8 bytes, multibyte-safe truncation |
| `test_title_shlex_join` | argv with spaces → quoted display title |
| `test_status_enum` | Only success/failure/info accepted |
| `test_envelope_meta_relocation` | Unknown top-level input keys moved under `meta` (Gate 1) |
| `test_send_rejects_non_objects` | `12`, `"hello"`, `[]` → invalid records, final exit 1 (Gate 1) |
| `test_retry_classification` | 400 not retried; 429/timeout retried once (Gate 1) |
| `test_sink_uri_dispatch` | Scheme routing; unknown scheme exits 2 |

HTTP tested by monkeypatching `urllib.request.urlopen`; no network in tests.

---

## Non-Goals (v0.1)

No config files, profiles, filter stages, ledger/SQLite/S3 sinks, observability sinks, sparklines, shell hooks, approval gates, daemon mode, PTY mode, SIGTERM forwarding, plugin API, or delivery-order guarantees. Future directions live in **IDEAS.md** and are explicitly non-binding: nothing there may complicate the Gate 0/1 design. The only forward-compatibility commitments are: extensions under `meta`, sinks behind a single `deliver(envelope, uri)` dispatch (the seam for a possible Go port), and the envelope being provisional until proven.

---

## Agent Build Instructions

> Implement from this README only. Python 3.11+, stdlib only — any third-party import is a build failure. Build Gate 0 completely before touching Gate 1.

### Gate 0 build order

| Phase | Deliverable | Done when |
|-------|-------------|-----------|
| 0 | Skeleton: argparse, Envelope dataclass (+ `to_json`) | `--help` correct |
| 1 | Child exec, dual reader threads, tee, bounded tail, exit mapping, SIGINT handling | `test_exit_*`, `test_signal_*`, `test_stream_separation`, `test_tail_bytes_cap` pass |
| 2 | `console://` (json→stdout, pretty→stderr) + `ntfy://` behind `deliver()` | `test_sink_uri_dispatch` passes; manual phone demo works |

### Gate 0 GO / NO-GO — manual demo, all three required

- [ ] The wrapped command remains usable: live output visible, `> file` redirection captures only child stdout
- [ ] Exit status is unchanged (including a signal-kill case)
- [ ] The phone receives a useful notification containing the final output tail

NO-GO on any failure → fix or kill; do not proceed to Gate 1.

### Gate 1 build order

`send` (stdin JSONL, defaults, meta-relocation, non-object rejection, exit-status rule) → `https://` + Slack renderer + `http://` warning → parallel delivery + classified retry → env expansion + `--only-failures` → full test suite green.

### Constraints

- Single readable file. No line-count target — never compress error handling to hit one.
- Typed signatures. `AIDEV-` comments for non-obvious decisions only.
- Sink functions: pure `(Envelope, ParseResult) -> None`, raise `SinkError`. No dynamic imports, no metaclasses.
- No `TODO`/`FIXME`/bare `pass` in code paths.

---

## Next Steps

1. [ ] Agent builds Gate 0; run the manual GO/NO-GO demo (real ntfy topic on a phone)
2. [ ] GO → Gate 1; then dogfood `wrap` on real builds for a few days
3. [ ] Freeze Envelope v1 only after a second emitter uses it in anger
4. [ ] Decide final name (candidates in IDEAS.md) and run `ship-check` before the repo goes public

---

## References

- ntfy publish API and limits: https://docs.ntfy.sh/publish/ (verify headers, 4096-byte message limit)
- Python `subprocess` pipe semantics and negative returncodes: https://docs.python.org/3/library/subprocess.html
- Prior art, differentiate honestly: `ntfy --wait-cmd`, `noti` (process-wrapping notifications), Apprise (URI fan-out library). popsink's angle: structured events + pipes + stdlib-only single file.
