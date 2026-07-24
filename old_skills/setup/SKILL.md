---
name: setup
description: "Admin-only: connect this Discord server to the shared team memory — saves machine token to server-sessions"
user-invocable: true
metadata:
  {
    "openclaw": {
      "emoji": "⚙️"
    }
  }
---

# Setup Skill

> **STRICT WIZARD — follow this script exactly.**
> - Execute steps in the order written.
> - Before running any bash command, verify it appears verbatim in the current step.
> - Use the user's language (Thai if they write Thai).

When the user runs `/setup`, connect this Discord server to the shared team memory so all members can use the bot immediately without logging in.

---

## Step 0 — Validate: Guild channel only

From the conversation metadata, check `is_group_chat`:

- `is_group_chat = true` (Guild channel) → continue to Step 1
- `is_group_chat = false` (DM) → stop immediately and send:

> ❌ `/setup` ใช้ได้เฉพาะใน guild channel นะครับ
>
> กรุณาพิมพ์ `/setup` ใน channel ของ Discord server ที่ต้องการ setup

---

## Step 1 — Extract guild ID from conversation metadata

From the conversation info at the start of this turn, extract:
- `GUILD_ID` — value of `group_space` (Discord guild/server ID)
- `SENDER_ID` — value of `sender_id` (Discord user ID of admin running this command)

Hold both. Do **not** call any tool for this step — the values are already in context.

---

## Step 2 — Check if already set up

```bash
python3 - <<'PYEOF'
import pathlib, json

guild_id     = "PASTE_GUILD_ID_HERE"
session_file = pathlib.Path.home() / ".openclaw/server-sessions" / f"{guild_id}.json"

if session_file.exists():
    data = json.loads(session_file.read_text())
    token = data.get("apiToken", "")
    setup_by = data.get("setupBy", "unknown")
    created_at = data.get("createdAt", "unknown")
    if token:
        print(f"ALREADY_SETUP setupBy={setup_by} createdAt={created_at}")
    else:
        print("FILE_EXISTS_NO_TOKEN")
else:
    print("NOT_SETUP")
PYEOF
```

Replace `PASTE_GUILD_ID_HERE` with `GUILD_ID` from Step 1.

**If `NOT_SETUP` or `FILE_EXISTS_NO_TOKEN`** → continue to Step 3.

**If `ALREADY_SETUP`** → tell the user:

> ℹ️ Server นี้ถูก setup ไว้แล้วครับ (ตั้งค่าเมื่อ `{createdAt}`)
>
> ต้องการ reset token ใหม่ไหม? (ใช่ / ไม่)

- ผู้ใช้ตอบ **ใช่ / yes / reset** → continue to Step 3 (overwrite)
- ผู้ใช้ตอบ **ไม่ / no** → stop, send:
  > ✅ ยังคงใช้ token เดิมอยู่ครับ ไม่มีการเปลี่ยนแปลง

---

## Step 3 — Read machine token from environment

```bash
python3 - <<'PYEOF'
import pathlib

env_file = pathlib.Path.home() / ".openclaw/.env"
token = ""
url = ""
if env_file.exists():
    for line in env_file.read_text().splitlines():
        line = line.strip()
        if line.startswith("AI_TOKEN="):
            token = line.split("=", 1)[1].strip()
        if line.startswith("SESSION_MEMORY_URL="):
            url = line.split("=", 1)[1].strip()

if token:
    print(f"SESSION_MEMORY_URL={url}")
    print("TOKEN_FOUND=true")
else:
    print("TOKEN_FOUND=false")
PYEOF
```

**If `TOKEN_FOUND=false`** → stop and send:

> ❌ ไม่พบ AI_TOKEN ใน `.env` ครับ
>
> กรุณา run `/setup-memory` ก่อนเพื่อตั้งค่า machine token

**If `TOKEN_FOUND=true`** → continue to Step 4. Do not print or expose the full token; later scripts read it directly from `.env`.

---

## Step 4 — Verify token against session-memory

```bash
python3 - <<'PYEOF'
import subprocess, pathlib

env_file = pathlib.Path.home() / ".openclaw/.env"
token = ""
url = ""
for line in env_file.read_text().splitlines():
    if line.strip().startswith("AI_TOKEN="):
        token = line.strip().split("=", 1)[1].strip()
    if line.strip().startswith("SESSION_MEMORY_URL="):
        url = line.strip().split("=", 1)[1].strip()

base_url = url.rstrip("/")
if base_url.endswith("/mcp"):
    base_url = base_url[:-4]
if not base_url or not token:
    print("CONFIG_MISSING")
else:
    result = subprocess.run(
        ["curl", "-s", "-o", "/dev/null", "-w", "%{http_code}",
         f"{base_url}/list?n=1",
         "-H", f"Authorization: Bearer {token}"],
        capture_output=True, text=True
    )
    print(f"HTTP={result.stdout.strip()}")
PYEOF
```

- `HTTP=200` → token valid → continue to Step 5
- `HTTP=401` → token invalid or revoked → stop and send:
  > ❌ AI_TOKEN ไม่ valid ครับ — กรุณา rotate token ใน auth-center แล้วอัปเดตใน `.env` ก่อน
- `HTTP=000` หรือ error → URL ผิดหรือ Worker ไม่ตอบ → stop and send:
  > ❌ เชื่อมต่อ session-memory ไม่ได้ครับ — กรุณาตรวจสอบ SESSION_MEMORY_URL ใน `.env`

---

## Step 5 — Save server session file

```bash
python3 - <<'PYEOF'
import pathlib, json, datetime

guild_id  = "PASTE_GUILD_ID_HERE"
sender_id = "PASTE_SENDER_ID_HERE"

env_file = pathlib.Path.home() / ".openclaw/.env"
token = ""
for line in env_file.read_text().splitlines():
    if line.strip().startswith("AI_TOKEN="):
        token = line.strip().split("=", 1)[1].strip()

sessions_dir = pathlib.Path.home() / ".openclaw/server-sessions"
sessions_dir.mkdir(parents=True, exist_ok=True)

session_file = sessions_dir / f"{guild_id}.json"
session_data = {
    "guildId": guild_id,
    "apiToken": token,
    "tokenLabel": f"discord-guild-{guild_id}",
    "setupBy": sender_id,
    "createdAt": datetime.datetime.utcnow().isoformat() + "Z"
}
session_file.write_text(json.dumps(session_data, indent=2))
session_file.chmod(0o600)

print("OK")
PYEOF
```

Replace `PASTE_GUILD_ID_HERE` with `GUILD_ID` and `PASTE_SENDER_ID_HERE` with `SENDER_ID` from Step 1.

**Map the output:**

- `OK` → continue to Completion
- Any error → show the error and stop

---

## Completion

Send this message:

> ✅ **Setup สำเร็จแล้ว!**
>
> ตอนนี้ทุกคนใน server นี้สามารถพูดคุยกับ bot ได้ทันทีโดยไม่ต้อง login ครับ
> Bot จะใช้ shared team memory สำหรับทุกคนใน server นี้
>
> **หมายเหตุ:** ผู้ที่ต้องการ personal memory แยกสำหรับตัวเองให้ DM bot แล้วพิมพ์ `/login`

Then immediately start a new session:

```json
{ "action": "new" }
```

Do NOT use session-memory tools after this point. The new session will pick up the machine token automatically via the Discord Identity Gate.
