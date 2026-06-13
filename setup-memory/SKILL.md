---
name: setup-memory
description: Bootstrap session-memory for this OpenClaw instance — configures env, installs helper, writes compact AGENTS.md, and verifies the connection
user-invocable: true
metadata:
  {
    "openclaw": {
      "emoji": "🧠"
    }
  }
---

# Setup Memory Skill

> **STRICT WIZARD — follow this script exactly.**
> - Execute steps in the order written. Do not skip decision points.
> - Use the user's language.
> - Do not print raw tokens, token previews, session JSON, Bearer headers, or local session paths in the final user-facing reply.
> - This skill configures the helper-based design. It does **not** configure a session-memory MCP server in `openclaw.json`.

When the user runs `/setup-memory`, bootstrap this OpenClaw instance so Discord memory works like the development machine.

Target state:
- `~/.openclaw/.env` has `AI_TOKEN`, `SESSION_MEMORY_URL`, and optionally `AUTH_CENTER`.
- `~/.openclaw/scripts/session-memory-call.py` exists and is executable.
- `~/.openclaw/workspace/AGENTS.md` contains the compact Discord identity and memory routing policy.
- `/login` creates DM personal sessions.
- `/setup` creates guild/shared sessions.

---

## Step 1 — Check environment

Run:

```bash
python3 - <<'PYEOF'
import pathlib

env_file = pathlib.Path.home() / ".openclaw/.env"
keys = {"AI_TOKEN": "", "SESSION_MEMORY_URL": "", "AUTH_CENTER": ""}

if env_file.exists():
    for line in env_file.read_text().splitlines():
        line = line.strip()
        if not line or line.startswith("#") or "=" not in line:
            continue
        k, v = line.split("=", 1)
        if k in keys:
            keys[k] = v.strip()

for key in ["AI_TOKEN", "SESSION_MEMORY_URL", "AUTH_CENTER"]:
    value = keys[key]
    if key == "AI_TOKEN":
        print(f"{key}={'set len=' + str(len(value)) if value else 'missing'}")
    elif value:
        print(f"{key}=set")
    else:
        print(f"{key}=missing")

missing = [k for k in ["AI_TOKEN", "SESSION_MEMORY_URL"] if not keys[k]]
print("MISSING=" + ",".join(missing) if missing else "ALL_SET")
PYEOF
```

If `ALL_SET`, continue to Step 2.

If anything is missing, explain only the missing items and ask the user to provide them:
- `AI_TOKEN`: machine token from auth-center for session-memory, starts with `sm_live_`.
- `SESSION_MEMORY_URL`: session-memory Worker URL ending with `/mcp`.
- `AUTH_CENTER`: optional auth-center URL used by `/login`; recommended for cloud.

Accept these user replies:
- Token starting with `sm_live_` → run Token Save Flow.
- URL starting with `http` and intended for session-memory → run URL Save Flow.
- Auth-center URL → run Auth Center Save Flow.
- "done" / "พร้อมแล้ว" → rerun Step 1.

---

## Token Save Flow

```bash
python3 - <<'PYEOF'
import pathlib, re

token = "PASTE_FULL_TOKEN_HERE"

if not token.startswith("sm_live_") or len(token) < 20:
    print("TOKEN_INVALID")
    raise SystemExit(1)

env_file = pathlib.Path.home() / ".openclaw/.env"
env_file.parent.mkdir(parents=True, exist_ok=True)
content = env_file.read_text() if env_file.exists() else ""
content = re.sub(r"^AI_TOKEN=.*\n?", "", content, flags=re.MULTILINE)
content = content.rstrip("\n") + "\nAI_TOKEN=" + token + "\n"
env_file.write_text(content)
env_file.chmod(0o600)
print("TOKEN_SAVED")
PYEOF
```

Then return to Step 1.

---

## URL Save Flow

```bash
python3 - <<'PYEOF'
import pathlib, re

url = "PASTE_FULL_SESSION_MEMORY_URL_HERE"
url = url.strip().rstrip("/")
if not url.startswith("http"):
    print("URL_INVALID")
    raise SystemExit(1)
if not url.endswith("/mcp"):
    url += "/mcp"

env_file = pathlib.Path.home() / ".openclaw/.env"
env_file.parent.mkdir(parents=True, exist_ok=True)
content = env_file.read_text() if env_file.exists() else ""
content = re.sub(r"^SESSION_MEMORY_URL=.*\n?", "", content, flags=re.MULTILINE)
content = content.rstrip("\n") + "\nSESSION_MEMORY_URL=" + url + "\n"
env_file.write_text(content)
env_file.chmod(0o600)
print("URL_SAVED")
PYEOF
```

Then return to Step 1.

---

## Auth Center Save Flow

```bash
python3 - <<'PYEOF'
import pathlib, re

url = "PASTE_FULL_AUTH_CENTER_URL_HERE"
url = url.strip().rstrip("/")
if not url.startswith("http"):
    print("AUTH_CENTER_INVALID")
    raise SystemExit(1)

env_file = pathlib.Path.home() / ".openclaw/.env"
env_file.parent.mkdir(parents=True, exist_ok=True)
content = env_file.read_text() if env_file.exists() else ""
content = re.sub(r"^AUTH_CENTER=.*\n?", "", content, flags=re.MULTILINE)
content = content.rstrip("\n") + "\nAUTH_CENTER=" + url + "\n"
env_file.write_text(content)
env_file.chmod(0o600)
print("AUTH_CENTER_SAVED")
PYEOF
```

Then return to Step 1.

---

## Step 2 — Verify machine token against session-memory

Run:

```bash
python3 - <<'PYEOF'
import pathlib, subprocess

env_file = pathlib.Path.home() / ".openclaw/.env"
token = ""
url = ""
if env_file.exists():
    for line in env_file.read_text().splitlines():
        if line.startswith("AI_TOKEN="):
            token = line.split("=", 1)[1].strip()
        if line.startswith("SESSION_MEMORY_URL="):
            url = line.split("=", 1)[1].strip()

base_url = url.rstrip("/")
if base_url.endswith("/mcp"):
    base_url = base_url[:-4]

if not token or not base_url:
    print("CONFIG_MISSING")
else:
    result = subprocess.run(
        ["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}", f"{base_url}/list?n=1", "-H", f"Authorization: Bearer {token}"],
        capture_output=True,
        text=True,
        timeout=15,
    )
    print(f"HTTP={result.stdout.strip()}")
PYEOF
```

Map result:
- `HTTP=200` → continue to Step 3.
- `HTTP=401` → token is invalid/revoked; ask the user to rotate the machine token in auth-center and return to Step 1.
- `HTTP=000`, timeout, or any other result → session-memory URL or network is wrong; ask the user to fix it and return to Step 1.

---

## Step 3 — Install safe session-memory helper

Run:

```bash
python3 - <<'PYEOF'
import pathlib

helper = pathlib.Path.home() / ".openclaw/scripts/session-memory-call.py"
helper.parent.mkdir(parents=True, exist_ok=True)
helper.write_text("""#!/usr/bin/env python3
import argparse
import json
import os
import sys
import urllib.error
import urllib.request
from pathlib import Path


HOME = Path.home()
ENV_FILE = HOME / ".openclaw" / ".env"
DEFAULT_URL = ""


def load_env():
    env = {}
    if ENV_FILE.exists():
        for line in ENV_FILE.read_text(encoding="utf-8", errors="replace").splitlines():
            line = line.strip()
            if not line or line.startswith("#") or "=" not in line:
                continue
            key, value = line.split("=", 1)
            env[key.strip()] = value.strip().strip('"').strip("'")
    return env


def session_path(context, subject_id):
    base = HOME / ".openclaw"
    if context == "dm":
        return base / "user-sessions" / f"{subject_id}.json"
    return base / "server-sessions" / f"{subject_id}.json"


def emit(status, payload=None):
    print(status)
    if payload:
        print(payload)


def parse_response(raw):
    text = raw.decode("utf-8", errors="replace")
    stripped = text.strip()
    if not stripped:
        return None
    try:
        return json.loads(stripped)
    except json.JSONDecodeError:
        pass

    messages = []
    for line in stripped.splitlines():
        line = line.strip()
        if not line.startswith("data:"):
            continue
        data = line[5:].strip()
        if not data or data == "[DONE]":
            continue
        try:
            messages.append(json.loads(data))
        except json.JSONDecodeError:
            continue
    for message in messages:
        if isinstance(message, dict) and ("result" in message or "error" in message):
            return message
    return messages[-1] if messages else None


def result_text(message):
    result = message.get("result")
    if isinstance(result, dict):
        content = result.get("content")
        if isinstance(content, list):
            parts = []
            for item in content:
                if isinstance(item, dict) and isinstance(item.get("text"), str):
                    parts.append(item["text"])
            if parts:
                return "\\n".join(parts)
        return json.dumps(result, ensure_ascii=False)
    if result is not None:
        return json.dumps(result, ensure_ascii=False)
    return ""


def main():
    parser = argparse.ArgumentParser(description="Safe session-memory caller for OpenClaw Discord turns")
    parser.add_argument("--context", choices=["dm", "guild"], required=True)
    parser.add_argument("--id", required=True, help="Discord user id for DM or guild id for guild")
    parser.add_argument("--tool", choices=["recall", "list_recent", "remember", "append", "forget"], required=True)
    parser.add_argument("--arguments", default="{}", help="JSON object passed to tools/call.arguments")
    args = parser.parse_args()

    try:
        tool_args = json.loads(args.arguments)
        if not isinstance(tool_args, dict):
            raise ValueError("arguments must be a JSON object")
    except Exception as exc:
        emit("MEMORY_ARGUMENT_ERROR", str(exc))
        return 2

    session_file = session_path(args.context, args.id)
    if not session_file.exists():
        emit("SESSION_FILE_MISSING")
        return 10

    try:
        session = json.loads(session_file.read_text(encoding="utf-8"))
    except Exception:
        emit("SESSION_FILE_CORRUPT")
        return 11

    token = session.get("apiToken")
    if not isinstance(token, str) or not token.startswith("sm_live_"):
        emit("SESSION_FILE_CORRUPT")
        return 11

    env = load_env()
    base_url = env.get("SESSION_MEMORY_URL") or DEFAULT_URL
    if not base_url:
        emit("MEMORY_CONFIG_MISSING")
        return 17
    payload = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {
            "name": args.tool,
            "arguments": tool_args,
        },
    }
    request = urllib.request.Request(
        base_url,
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json",
            "Accept": "application/json, text/event-stream",
            "User-Agent": "curl/8.7.1",
        },
        method="POST",
    )

    try:
        with urllib.request.urlopen(request, timeout=12) as response:
            message = parse_response(response.read())
    except urllib.error.HTTPError as exc:
        if exc.code == 401:
            try:
                session_file.unlink()
            except OSError:
                pass
            emit("SESSION_EXPIRED")
            return 12
        if exc.code == 403:
            emit("MEMORY_FORBIDDEN")
            return 13
        emit("MEMORY_UNAVAILABLE", f"HTTP {exc.code}")
        return 14
    except Exception as exc:
        emit("MEMORY_UNAVAILABLE", exc.__class__.__name__)
        return 14

    if not isinstance(message, dict):
        emit("MEMORY_PARSE_ERROR")
        return 15
    if "error" in message:
        error = message.get("error")
        if isinstance(error, dict):
            emit("MEMORY_ERROR", str(error.get("message") or error.get("code") or "unknown"))
        else:
            emit("MEMORY_ERROR", "unknown")
        return 16

    emit("MEMORY_OK", result_text(message))
    return 0


if __name__ == "__main__":
    sys.exit(main())\n""", encoding="utf-8")
helper.chmod(0o700)
print("HELPER_INSTALLED")
PYEOF
```

Then verify syntax:

```bash
python3 -c "compile(open('$HOME/.openclaw/scripts/session-memory-call.py', encoding='utf-8').read(), '$HOME/.openclaw/scripts/session-memory-call.py', 'exec'); print('HELPER_SYNTAX_OK')"
```

Continue only if output includes `HELPER_SYNTAX_OK`.

---

## Step 4 — Install compact AGENTS.md policy

Run:

```bash
python3 - <<'PYEOF'
import pathlib

agents = pathlib.Path.home() / ".openclaw/workspace/AGENTS.md"
agents.parent.mkdir(parents=True, exist_ok=True)
backup = agents.with_suffix(".md.bak.setup-memory")
if agents.exists() and not backup.exists():
    backup.write_text(agents.read_text(encoding="utf-8", errors="replace"), encoding="utf-8")
agents.write_text("""# OpenClaw Workspace Instructions

This file is intentionally short. OpenClaw injects only part of large AGENTS.md files, so critical Discord identity, memory, and token-safety rules must stay near the top.

## Startup Priority

1. Preserve user intent and answer naturally.
2. For Discord turns, run the Discord Identity Gate before using memory.
3. Never reveal credentials, token previews, session JSON, local paths that contain secrets, or Bearer headers.
4. Do not read project files to answer Discord user memory questions. Discord memory must come from session-memory through the safe helper.

## Discord Identity Gate

Apply this gate to every Discord-originated message before memory recall, memory storage, or memory listing.

1. Extract Discord metadata from the runtime message:
   - DM context: use sender/user id as `SENDER_ID`.
   - Guild context: use guild/server id as `GUILD_ID`.
   - Do not guess ids from the text itself.
2. DM memory boundary:
   - Session file: `~/.openclaw/user-sessions/{SENDER_ID}.json`.
   - Personal memory is allowed only after this file exists and contains a valid `apiToken`.
3. Guild memory boundary:
   - Session file: `~/.openclaw/server-sessions/{GUILD_ID}.json`.
   - Guild/public memory requires server setup by an admin.
   - Never use a DM personal token as a substitute for a guild token.
4. If the required session file is missing, explain the next action:
   - DM: ask the user to run `/login`.
   - Guild: ask an admin to run `/setup`.
5. If a helper reports the session is expired or corrupt, ask for `/login` or `/setup` again as appropriate.

## Safe Session-Memory Helper

For Discord memory operations, invoke:

```bash
python3 ~/.openclaw/scripts/session-memory-call.py \\
  --context dm|guild \\
  --id "$SENDER_ID_OR_GUILD_ID" \\
  --tool recall|list_recent|remember|append|forget \\
  --arguments '{"query":"..."}'
```

The helper reads the local session file, calls session-memory, and prints only safe status codes plus memory results. It must be the default route for Discord memory. Do not print or summarize the token/session file content.

Status handling:

- `MEMORY_OK`: use the following payload as memory context.
- `SESSION_FILE_MISSING`: ask for `/login` in DM or `/setup` in guild.
- `SESSION_EXPIRED`: say the session expired and ask the user/admin to authenticate again.
- `SESSION_FILE_CORRUPT`: say authentication state is invalid and ask to authenticate again.
- `MEMORY_FORBIDDEN`: explain that this account/server lacks permission for that memory.
- `MEMORY_CONFIG_MISSING`: explain that `SESSION_MEMORY_URL` is not configured and ask an admin to run `/setup-memory`.
- `MEMORY_UNAVAILABLE`, `MEMORY_PARSE_ERROR`, `MEMORY_ERROR`: answer gracefully and avoid the generic crash message.

## Memory Behavior

**Recall before answering** any question that could be answered by stored notes. This includes:

- Questions about specific places, rooms, or equipment (e.g. "ห้อง X ต้องทำอะไร", "ใช้ Y แล้วต้องปิดอะไรไหม").
- Questions about procedures, rules, or instructions ("ต้องทำอะไร", "มีกฎอะไร", "ควรทำยังไง").
- Their name, preferences, plans, tasks, history, or prior conversations.
- Public announcements, shared notes, or current remembered state.
- Whether memory/login/setup/session-memory is connected.

When in doubt, run recall first — a fast empty result is better than answering incorrectly from general knowledge.

Use at most two recall/list calls per Discord turn. Prefer:

- `recall` for semantic questions and procedure/location lookups.
- `list_recent` for "show memory", "public memory", "recent memory", status checks, and browsing announcements.
- `remember` for new durable facts — always include `"source":"discord-dm"` or `"source":"discord-guild"` depending on context.
- `append` for updates to an existing fact.
- `forget` only when the user clearly asks to remove memory.

**After calling `remember`, confirm with the returned entry ID.** Example: "จำแล้วครับ (ID: abc123)" — this proves the tool was called and the entry was stored. Never say "จำแล้ว" without an actual tool call and ID.

For "public memory", use `list_recent` and filter/label entries marked public or shared. If no dedicated public-memory tool exists, say that you are showing the public/shared entries visible to the current session.

**After every memory tool call, always generate a final text reply.** Do not stop silently after a tool returns. If the result is empty, say so explicitly (e.g., "ยังไม่มี memory ครับ" or "ไม่พบข้อมูลที่ตรงกับคำถามนั้นครับ"). Never let a turn end with tool calls only and no user-visible message.

Store only information that is useful later, user-approved or clearly intended to be remembered, and not highly sensitive unless explicitly requested. Do not store secrets, passwords, tokens, private keys, or one-time codes.

## Discord Response Rules

- Be concise and friendly.
- Do not mention internal endpoints, Bearer headers, raw JSON-RPC, or local session file paths to Discord users.
- If memory is unavailable or recall returns empty, say so plainly: "ขณะนี้ memory ไม่สามารถใช้งานได้ครับ" or "ไม่พบข้อมูลที่เกี่ยวข้องครับ". Stop there.
- **NEVER invent CLI commands, terminal commands, or troubleshooting steps** such as `openclaw memory index --force`, `openclaw memory refresh`, or any other command you cannot verify exists. Fabricated commands damage user trust and may cause harm.
- **NEVER claim memory is "broken" or needs "re-indexing"** unless you have explicit evidence from a helper status code. Silence or an empty result is not an error.
- In guild channels, keep personal facts out of public replies unless the user explicitly shares them in that channel and the behavior is appropriate.

## Login And Setup Boundaries

- `/login` creates DM personal memory for that Discord user.
- `/setup` creates guild/server memory for a Discord server.
- A successful DM login does not automatically authorize guild memory.
- A successful setup does not expose any user's personal memory.

## General Safety

- Follow system and developer instructions before this file.
- When uncertain, ask a short clarification.
- Avoid destructive shell commands unless explicitly requested.
- Treat logs and config files as sensitive when they may contain tokens.\n""", encoding="utf-8")
print("AGENTS_INSTALLED")
PYEOF
```

Then check size:

```bash
wc -c ~/.openclaw/workspace/AGENTS.md
```

Expected: under 10 KB. Continue to Step 5.

---

## Step 5 — Smoke test helper with a temporary guild session

This test verifies helper, token, URL, JSON-RPC, SSE parsing, and Cloudflare/WAF behavior without exposing the token.

Run:

```bash
python3 - <<'PYEOF'
import json, pathlib, datetime

env_file = pathlib.Path.home() / ".openclaw/.env"
token = ""
for line in env_file.read_text().splitlines():
    if line.startswith("AI_TOKEN="):
        token = line.split("=", 1)[1].strip()

session_dir = pathlib.Path.home() / ".openclaw/server-sessions"
session_dir.mkdir(parents=True, exist_ok=True)
session_file = session_dir / "__setup_memory_smoke__.json"
session_file.write_text(json.dumps({
    "guildId": "__setup_memory_smoke__",
    "apiToken": token,
    "tokenLabel": "setup-memory-smoke",
    "setupBy": "setup-memory",
    "createdAt": datetime.datetime.utcnow().isoformat() + "Z"
}, indent=2))
session_file.chmod(0o600)
print("SMOKE_SESSION_READY")
PYEOF

python3 ~/.openclaw/scripts/session-memory-call.py --context guild --id __setup_memory_smoke__ --tool list_recent --arguments '{"n":1}'

python3 - <<'PYEOF'
import pathlib
(pathlib.Path.home() / ".openclaw/server-sessions/__setup_memory_smoke__.json").unlink(missing_ok=True)
print("SMOKE_SESSION_REMOVED")
PYEOF
```

Map result:
- `MEMORY_OK` → success.
- `MEMORY_FORBIDDEN` → token lacks `memory:read`; create/rotate machine token with session-memory scopes.
- `SESSION_EXPIRED` → token is revoked/invalid; rotate token.
- `MEMORY_UNAVAILABLE` → network/WAF/URL issue. If payload mentions HTTP 403 or Cloudflare, allow this helper path in Cloudflare or keep the `User-Agent` behavior.
- Any other result → summarize and stop.

---

## Step 6 — Confirm companion skills

Check that the Discord skills exist:

```bash
test -f ~/.openclaw/workspace/skills/login/SKILL.md && echo "LOGIN_SKILL_OK" || echo "LOGIN_SKILL_MISSING"
test -f ~/.openclaw/workspace/skills/setup/SKILL.md && echo "SETUP_SKILL_OK" || echo "SETUP_SKILL_MISSING"
```

If either is missing, tell the user that `/login` or `/setup` must be installed before production Discord use.

---

## Completion

Before sending success, confirm:
- Step 1 returned `ALL_SET`.
- Step 2 returned `HTTP=200`.
- Step 3 returned `HELPER_SYNTAX_OK`.
- Step 4 installed compact `AGENTS.md`.
- Step 5 returned `MEMORY_OK`.
- Step 6 found `login` and `setup` skills.

Success message:

> ✅ ตั้งค่า session-memory สำหรับ OpenClaw instance นี้เรียบร้อยแล้วครับ
>
> ตอนนี้ Discord memory จะใช้ helper-based flow:
> - DM users ใช้ `/login`
> - Guild/shared memory ใช้ `/setup`
> - ทุก memory request จะผ่าน `session-memory-call.py`
>
> แนะนำให้รัน `openclaw doctor --non-interactive` ต่อเพื่อตรวจ version/plugin/gateway warnings ก่อนใช้งาน production

If any step failed, summarize the failed step and the next action. Do not claim setup is complete.
