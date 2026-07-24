---
name: login
description: "Log in via DM — links your personal token to your Discord identity for session-memory access"
user-invocable: true
metadata:
  {
    "openclaw": {
      "emoji": "🔑"
    }
  }
---

# Login Skill

> **STRICT WIZARD — follow this script exactly.**
> - Execute steps in the order written.
> - Before running any bash command, verify it appears verbatim in the current step.
> - Use the user's language (Thai if they write Thai).

When the user runs `/login`, guide them through authenticating and linking their personal token to their Discord identity.

---

## Step 0 — Validate: DM only

From the conversation metadata, check `is_group_chat`:

- `is_group_chat = false` (DM) → continue to Step 1
- `is_group_chat = true` (Guild channel) → stop immediately and send:

> ❌ `/login` ใช้ได้เฉพาะใน DM นะครับ — เพื่อความปลอดภัยของรหัสผ่าน
>
> กรุณาเปิด DM กับ bot แล้วพิมพ์ `/login` ที่นั่นแทนครับ

---

## Step 1 — Extract sender ID from conversation metadata

From the conversation info at the start of this turn, extract:
- `SENDER_ID` — value of `sender_id` (your Discord user ID)

Hold it as `SENDER_ID`. Do **not** call any tool for this step — the value is already in context.

---

## Step 2 — Get credentials

Ask the user for their email and password:

> กรุณาพิมพ์ email และ password ของคุณในรูปแบบ: `email@example.com yourpassword`
>
> _(ข้อมูลนี้จะใช้เพื่อสร้าง API token ให้คุณเท่านั้น — จะไม่ถูกบันทึกไว้)_

Wait for the user to provide credentials. Parse `email` and `password` from their reply.

---

## Step 3 — Authenticate and create API token

Replace `PASTE_EMAIL_HERE`, `PASTE_PASSWORD_HERE`, and `PASTE_SENDER_ID_HERE` with actual values, then run:

```bash
python3 - <<'PYEOF'
import subprocess, json, os, sys, pathlib, datetime

email     = "PASTE_EMAIL_HERE"
password  = "PASTE_PASSWORD_HERE"
sender_id = "PASTE_SENDER_ID_HERE"

env_file = pathlib.Path.home() / ".openclaw/.env"
auth_center = "http://localhost:3000"
if env_file.exists():
    for line in env_file.read_text().splitlines():
        line = line.strip()
        if line.startswith("AUTH_CENTER="):
            auth_center = line.split("=", 1)[1].strip().rstrip("/")
            break

# Step A: Login to get session token
login_result = subprocess.run(
    ["curl", "-s", "-w", "\nHTTP_STATUS:%{http_code}",
     "-X", "POST", f"{auth_center}/auth/token",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({"email": email, "password": password})],
    capture_output=True, text=True
)
parts = login_result.stdout.rsplit("\nHTTP_STATUS:", 1)
login_body   = parts[0].strip()
login_status = parts[1].strip() if len(parts) == 2 else "000"

if login_status != "200":
    print(f"LOGIN_FAILED status={login_status}")
    sys.exit(1)

try:
    login_data = json.loads(login_body)
    session_token = login_data.get("accessToken") or login_data.get("token")
    if not session_token:
        session_token = login_data.get("user", {}).get("token") or login_data.get("sessionToken")
    if not session_token:
        print("LOGIN_NO_TOKEN")
        sys.exit(1)
except Exception as e:
    print(f"LOGIN_PARSE_ERROR: {e}")
    sys.exit(1)

# Step B: Create a named API token for this Discord user
token_result = subprocess.run(
    ["curl", "-s", "-w", "\nHTTP_STATUS:%{http_code}",
     "-X", "POST", f"{auth_center}/auth/tokens",
     "-H", f"Authorization: Bearer {session_token}",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({
         "label": f"discord-dm-{sender_id}",
         "resource": "session-memory",
         "scopes": ["memory:read", "memory:write", "memory:delete"]
     })],
    capture_output=True, text=True
)
parts2 = token_result.stdout.rsplit("\nHTTP_STATUS:", 1)
token_body   = parts2[0].strip()
token_status = parts2[1].strip() if len(parts2) == 2 else "000"

if token_status not in ("200", "201"):
    print(f"TOKEN_FAILED status={token_status}")
    sys.exit(1)

try:
    token_data = json.loads(token_body)
    raw_token = token_data.get("rawToken") or token_data.get("token", {}).get("raw")
    user_id   = token_data.get("token", {}).get("userId") or token_data.get("userId") or ""
    if not raw_token:
        print("TOKEN_NO_RAW")
        sys.exit(1)
except Exception as e:
    print(f"TOKEN_PARSE_ERROR: {e}")
    sys.exit(1)

# Step C: Save to user-sessions file keyed by Discord user ID
sessions_dir = pathlib.Path.home() / ".openclaw/user-sessions"
sessions_dir.mkdir(parents=True, exist_ok=True)

session_file = sessions_dir / f"{sender_id}.json"
session_data = {
    "discordUserId": sender_id,
    "userId": user_id,
    "email": email,
    "apiToken": raw_token,
    "tokenLabel": f"discord-dm-{sender_id}",
    "createdAt": datetime.datetime.utcnow().isoformat() + "Z"
}
session_file.write_text(json.dumps(session_data, indent=2))
session_file.chmod(0o600)

print("OK")
PYEOF
```

**Map the output:**

- `OK` → continue to Step 4
- `LOGIN_FAILED` → tell the user:
  > ❌ Login ไม่สำเร็จครับ — กรุณาตรวจสอบ email และ password แล้วลองใหม่
  Stop here.
- `TOKEN_FAILED` → tell the user:
  > ❌ สร้าง token ไม่สำเร็จครับ — กรุณาติดต่อ admin
  Stop here.
- Any other error → show the error and stop.

---

## Step 4 — Verify and report

Confirm the session file was created:

```bash
python3 - <<'PYEOF'
import pathlib, json

sender_id    = "PASTE_SENDER_ID_HERE"
session_file = pathlib.Path.home() / ".openclaw/user-sessions" / f"{sender_id}.json"

if session_file.exists():
    data  = json.loads(session_file.read_text())
    token = data.get("apiToken", "")
    print("VERIFIED")
else:
    print("FILE_MISSING")
PYEOF
```

- `VERIFIED` → proceed to success message
- `FILE_MISSING` → tell the user:
  > ⚠️ บันทึก session ไม่สำเร็จ — กรุณาลองใหม่

---

## Completion

Send this message to the user:

> ✅ **Login สำเร็จแล้ว!**
>
> ตอนนี้ personal memory ของคุณพร้อมใช้งานใน DM นี้แล้วครับ — ครั้งต่อไปที่ DM มาฉันจะจำคุณได้ทันที
>
> ถ้าต้องการใช้ memory ใน guild channel ให้ admin ของ server รัน `/setup` แยกต่างหาก

Then immediately start a new session:

```json
{ "action": "new" }
```

Do NOT use session-memory tools after this point. The new session will pick up the token automatically via the Discord Identity Gate.
