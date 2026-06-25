---
name: forget
description: "Reset your password via DM — sends a password reset link to your registered email"
user-invocable: true
metadata:
  {
    "openclaw": {
      "emoji": "🔓"
    }
  }
---

# Forget Skill

> **STRICT WIZARD — follow this script exactly.**
> - Execute steps in the order written.
> - Use the user's language (Thai if they write Thai).

When the user runs `/forget`, help them reset their password by sending a magic link to their registered email. The link is valid for 1 hour.

---

## Step 0 — Validate: DM only

From the conversation metadata, check `is_group_chat`:

- `is_group_chat = false` (DM) → continue to Step 1
- `is_group_chat = true` (Guild channel) → stop immediately and send:

> ❌ `/forget` ใช้ได้เฉพาะใน DM นะครับ — เพื่อความปลอดภัย
>
> กรุณาเปิด DM กับ bot แล้วพิมพ์ `/forget your@email.com` ที่นั่นแทนครับ

---

## Step 1 — Extract email from arguments

Check if the user provided an email argument with the command (e.g. `/forget user@example.com`).

- If email provided → hold as `EMAIL`, continue to Step 2
- If no email → reply:

> 📧 กรุณาระบุ email ที่ใช้ลงทะเบียนครับ
>
> Usage: `/forget your@email.com`

Stop here and wait for user to retry with email.

---

## Step 2 — Confirm with user

Send this confirmation message and wait for user to reply "yes":

> 🔑 จะส่งลิงก์รีเซ็ตรหัสผ่านไปที่ **{EMAIL}** ใช่ไหมครับ?
>
> พิมพ์ **yes** เพื่อยืนยัน หรือ **no** เพื่อยกเลิก

- User replies "yes" (or "ใช่", "ยืนยัน", "ok", "y") → continue to Step 3
- User replies anything else → reply "ยกเลิกแล้วครับ" and stop

---

## Step 3 — Call forgot-password API

Read AUTH_CENTER from environment. Replace `PASTE_EMAIL_HERE` with the actual email, then run:

```bash
python3 - <<'PYEOF'
import subprocess, json, os, pathlib

email = "PASTE_EMAIL_HERE"

env_file = pathlib.Path.home() / ".openclaw/.env"
auth_center = "http://localhost:8787"
if env_file.exists():
    for line in env_file.read_text().splitlines():
        line = line.strip()
        if line.startswith("AUTH_CENTER="):
            auth_center = line.split("=", 1)[1].strip().rstrip("/")
            break

result = subprocess.run(
    ["curl", "-s", "-w", "\nHTTP_STATUS:%{http_code}",
     "-X", "POST", f"{auth_center}/auth/users/forgot-password",
     "-H", "Content-Type: application/json",
     "-d", json.dumps({"email": email})],
    capture_output=True, text=True
)
parts = result.stdout.rsplit("\nHTTP_STATUS:", 1)
body   = parts[0].strip()
status = parts[1].strip() if len(parts) == 2 else "000"
print(f"STATUS={status}")
print(f"BODY={body}")
PYEOF
```

**Map the output:**

- `STATUS=200` → continue to Step 4
- `STATUS=500` → reply:
  > ⚠️ เกิดข้อผิดพลาดในการส่ง email ครับ — กรุณาลองใหม่ในอีกสักครู่ หรือติดต่อ admin
  Stop here.
- Other status → reply:
  > ❌ เกิดข้อผิดพลาด (status: {STATUS}) — กรุณาติดต่อ admin
  Stop here.

---

## Step 4 — Report success

Send this message:

> ✅ **ส่งแล้วครับ!**
>
> ถ้า email นี้ลงทะเบียนไว้ในระบบ เราจะส่งลิงก์รีเซ็ตรหัสผ่านให้
>
> 📬 กรุณาตรวจ **inbox** และ **spam folder**
>
> คลิกลิงก์ในอีเมล **(มีอายุ 1 ชั่วโมง)** เพื่อตั้งรหัสผ่านใหม่ครับ
