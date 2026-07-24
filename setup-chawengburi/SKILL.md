---
name: setup-chawengburi
description: "Bootstrap bobby-cli/session-memory for this OpenClaw instance — configures ~/.bobby-cli/.env, the admin's shared team login, writes AGENTS.md's bobby-cli command table, and verifies the connection"
user-invocable: true
metadata:
  {
    "openclaw": {
      "emoji": "🧠"
    }
  }
---

# Setup Chawengburi Skill

> **STRICT WIZARD — follow this script exactly.**
> - Execute steps in the order written. Do not skip decision points.
> - Use the user's language.
> - Do not print raw tokens, token previews, session JSON, Bearer headers, or local session/credential paths in the final user-facing reply.

When the user runs `/setup_chawengburi`, bootstrap this OpenClaw instance
so `/bobby_cli login`/`/bobby_cli setup` work.

---

## Step 0 — bobby-cli present on this host, auto-install if missing

```
bobby-cli --version
```
- Succeeds (prints a version) → continue to Step 1.
- Fails (not found / errors) → attempt automatic install:
  ```
  npm install -g bobby-cli
  ```
  Then re-check:
  ```
  bobby-cli --version
  ```
  - Now succeeds → continue to Step 1.
  - Still fails → stop, show the admin the actual error text from the
    install attempt, and suggest checking that Node.js/npm (≥18) are
    present on this host at all before retrying `/setup_chawengburi`.

This three-command sequence deliberately uses no bash-specific syntax (no
`VAR=value` prefix, no `~` expansion, no `&&`) — both `bobby-cli --version`
and `npm install -g bobby-cli` are plain program invocations, valid under
POSIX shells (Linux/macOS/Docker) and PowerShell (Windows) alike.

---

## Step 1 — Guided save-flow for `AUTH_CENTER`/`SESSION_MEMORY_URL`

Write to **`~/.bobby-cli/.env`** (not `~/.openclaw/.env` — bobby-cli never
reads that file):
```
AUTH_CENTER=<url>
SESSION_MEMORY_URL=<url>
```
`AI_TOKEN` is not written to this file — bobby-cli mints and stores its
own token via `auth login` in Step 2. Ask for each missing value, same
guided prompt/parse shape as the old wizard, until both are present.

---

## Step 1b — Mechanical refusal, not a default

Before Step 2 runs, confirm both URLs are actually present:
```bash
grep -c '^AUTH_CENTER=' ~/.bobby-cli/.env
grep -c '^SESSION_MEMORY_URL=' ~/.bobby-cli/.env
```
Either count is `0` (or the file doesn't exist yet, so `grep` exits
non-zero with no count — treat identically to a `0` count) → **stop**, do
not run Step 2, and re-prompt Step 1. This is a checkable gate the wizard
script must read and act on, not a guarantee the shell enforces on its
own — never silently fall through to Step 2 without both counts being
nonzero.

---

## Step 2 — Admin bootstrap login

```bash
BOBBY_CLI_EMAIL="<email>" BOBBY_CLI_PASSWORD="<password>" \
  bobby-cli auth login --json
```
No `--profile` — the shared team identity is bobby-cli's own default path
(`~/.bobby-cli/credentials.json`), the same one `/bobby_cli setup` reads
via `cp`. Only reachable after Step 1b's gate passes.

---

## Step 3 — Install the rewritten `AGENTS.md`

Programmatically write `~/.openclaw/workspace/AGENTS.md`, same
backup-before-overwrite behavior as the current wizard. Apply the exact
four edits below to the file currently live at
`~/.openclaw/workspace/AGENTS.md`. Everything in the current file outside
those four edited sections is carried over unchanged.

`~/.openclaw/scripts/session-memory-call.py` (the old ~180-line
hand-written JSON-RPC/SSE-parsing helper) is deleted as part of this step
— bobby-cli covers its transport responsibilities (JSON-RPC framing,
SSE-vs-JSON response parsing, `sm_live_*` redaction) plus structured
`code`/`hint` the helper's plain status strings never had.

---

## Step 4 — Confirm companion skill exists

```bash
test -f ~/.openclaw/workspace/skills/bobby-cli/SKILL.md && echo BOBBY_CLI_OK || echo BOBBY_CLI_MISSING
```

If missing, tell the user `/bobby_cli` must be installed before production
Discord use.

---

## Step 5 — Smoke test

```bash
bobby-cli memory show --json
```
No `--profile` — same default-path team identity as Step 2.

---

## Completion

Before sending success, confirm: Step 1 wrote both URLs; Step 1b's gate
passed; Step 2 returned `ok:true`; Step 3 installed the rewritten
`AGENTS.md` and deleted `session-memory-call.py`; Step 4 found
`BOBBY_CLI_OK`; Step 5 returned a successful `memory show` result.

Success message:
> ✅ ตั้งค่า bobby-cli สำหรับ OpenClaw instance นี้เรียบร้อยแล้วครับ
>
> ตอนนี้ Discord memory จะใช้ bobby-cli:
> - DM users ใช้ `/bobby_cli login`
> - Guild/shared memory ใช้ `/bobby_cli setup`
> - ทุก memory request จะผ่าน bobby-cli โดยตรง (ไม่มี Python helper อีกต่อไป)
>
> แนะนำให้รัน `openclaw doctor --non-interactive` ต่อเพื่อตรวจ version/plugin/gateway warnings ก่อนใช้งาน production

If any step failed, summarize the failed step and the next action. Do not
claim setup is complete.
