---
name: bobby-cli
description: "จัดการบัญชี bobby-cli ของคุณผ่าน Discord — login (เข้าสู่ระบบส่วนตัว), setup (เชื่อมต่อ server กับ shared team token), logout (ลบ token ที่บอทเก็บไว้). พิมพ์ /bobby_cli login|setup|logout หรือพิมพ์คำว่า login/setup/logout ตรง ๆ ก็ได้"
user-invocable: true
metadata:
  {
    "openclaw": {
      "emoji": "🔑"
    }
  }
---

# bobby-cli Skill

> **STRICT WIZARD — follow this script exactly.**

## Step 0 — Determine the requested action

**Who actually types this matters (2026-07-24 amendment 2 — corrects
amendment 1's "first word only" rule, which was itself a fix for a real
bug but wrong for the real audience).** The intended end users of this
bot are non-technical staff — e.g. housekeeping, front-desk/counter
service — not developers. They will not reliably type bare commands.
They will type full, polite, natural sentences, usually in Thai: *"อยากจะ
login ค่ะ"*, *"ขอเข้าสู่ระบบหน่อยครับ"*, *"ช่วย setup ให้หน่อยได้ไหมคะ"*.
A first-word-only rule fails every one of these (the first word is
"อยากจะ"/"ขอ"/"ช่วย", not the action word) and would make the
disambiguation question fire on the single most common real phrasing of
a perfectly clear request — worse UX than what it was fixing.

**Matching rule:** determine intent from the **whole message**, in
whatever position and grammatical wrapping the action word appears —
Thai or English, formal or casual. The table below is a set of
**illustrative anchor words**, not an exhaustive literal whitelist or a
positional rule; if the message clearly expresses one of these three
intents using different wording than the examples, that still counts as
a match. This deliberately leans on the model's own language
understanding rather than fighting it with rigid string matching —
appropriate here specifically because the executor is an LLM reading a
script, not a regex parser.

| Anchor words (illustrative, not exhaustive) | Action |
|---|---|
| "login", "เข้าสู่ระบบ", "ล็อกอิน", "เข้าระบบ" | LOGIN |
| "setup", "ตั้งค่า", "เชื่อมต่อ", "ผูก server" | SETUP |
| "logout", "ออกจากระบบ", "ล็อกเอาท์", "ลบ token" | LOGOUT |

**Credential safety is unchanged from amendment 1 and still absolute:**
whatever surrounds the action word in the message is never interpreted as
a credential, even if it looks like one (e.g. an email address typed in
the same message as "login"). LOGIN's Step 2 always asks for email/
password as its own separate follow-up turn (spec 14 § 3.3) — nothing
about broadening Step 0's matching changes that; there is still no path
by which text elsewhere in the message gets treated as, echoed as, or
logged as a password.

**Ambiguity — the case that must still ask, not guess:** a message that
gives a clear signal for exactly one of the three actions (regardless of
where in the sentence, or how it's phrased) is not ambiguous, and must
not trigger a re-ask. A message is ambiguous, and must trigger the
clarifying question below, when it (a) gives no recognizable signal for
any of the three, or (b) gives signals for **more than one** (e.g.
mentions both login and setup) with no clear indication which is the
current request — never silently pick one when the message points at two:
> ต้องการ **login** (เข้าสู่ระบบส่วนตัว), **setup** (เชื่อมต่อ server), หรือ **logout** (ออกจากระบบ) ครับ? พิมพ์คำใดคำหนึ่ง

This is the single highest-risk step in this skill — misrouting `login`
intent into the SETUP branch (or vice versa) points a Discord user's
password-derived token at the wrong shared identity. Genuine ambiguity
(no signal, or competing signals) always asks; a clearly-expressed single
intent, however phrased, must not be second-guessed back to the user.

**Structural risk this amplifies, worth stating plainly rather than
leaving implicit:** for an audience unlikely to ever type `/bobby_cli`
explicitly, the plain-text model-invocation path (see "Why this exists"
above — every skill is listed in `<available_skills>` unless opted out,
and the agent decides on its own whether to load a skill from plain
conversation) stops being a fallback and becomes the **primary** way this
skill gets triggered at all. That path was already flagged as weaker for
a consolidated multi-action skill than for single-purpose skills, and
fails outright in the loader's compact-prompt fallback mode. This spec
does not change that structural fact — it only makes Step 0 behave well
*once the skill has been loaded*. Whether the skill reliably gets loaded
in the first place from a natural Thai sentence with no slash is a
separate concern or, more precisely, one that success criterion 10 below
must be tested against realistically, not with bare keywords.

---

## Action: LOGIN (DM only)

**Step 0 — Validate: DM only.** From the conversation metadata, check
`is_group_chat`:
- `is_group_chat = false` (DM) → continue.
- `is_group_chat = true` (Guild) → stop, send:
  > ❌ `/bobby_cli login` ใช้ได้เฉพาะใน DM นะครับ — เพื่อความปลอดภัยของรหัสผ่าน
  >
  > กรุณาเปิด DM กับ bot แล้วพิมพ์ `/bobby_cli login` ที่นั่นแทนครับ

**Step 1 — Extract `SENDER_ID`** (value of `sender_id`) from conversation
metadata. Do not call any tool for this — the value is already in context.

**Step 2 — Get credentials.** Ask:
> กรุณาพิมพ์ email และ password ของคุณในรูปแบบ: `email@example.com yourpassword`
>
> _(ข้อมูลนี้จะใช้เพื่อสร้าง API token ให้คุณเท่านั้น — จะไม่ถูกบันทึกไว้)_

Wait for the reply; parse `email` and `password` from it. Never echo the
password back, never log it, never treat any other part of the
conversation (including Step 0's original message) as a credential.

**Step 3 — Authenticate:**
```bash
BOBBY_CLI_PROFILES_DIR=~/.openclaw/user-sessions \
BOBBY_CLI_EMAIL="<email>" BOBBY_CLI_PASSWORD="<password>" \
  bobby-cli auth login --profile "$SENDER_ID" --label "discord-dm-$SENDER_ID" --json
```
Env vars, never flags — argv is visible in the process list. On `ok:true`,
continue to Step 4. On `ok:false`, follow the returned `hint` field; if
that's not actionable, tell the user:
> ❌ Login ไม่สำเร็จครับ — กรุณาตรวจสอบ email และ password แล้วลองใหม่

**Step 4 — Verify:**
```bash
BOBBY_CLI_PROFILES_DIR=~/.openclaw/user-sessions \
  bobby-cli auth show --profile "$SENDER_ID" --json
```
`loggedIn:true` → proceed to Completion. `loggedIn:false` → tell the user:
> ⚠️ บันทึก session ไม่สำเร็จ — กรุณาลองใหม่

**Completion:**
> ✅ **Login สำเร็จแล้ว!**
>
> ตอนนี้ personal memory ของคุณพร้อมใช้งานใน DM นี้แล้วครับ — ครั้งต่อไปที่ DM มาฉันจะจำคุณได้ทันที
>
> ถ้าต้องการใช้ memory ใน guild channel ให้ admin ของ server รัน `/bobby_cli setup` แยกต่างหาก

Then start a new session (`{ "action": "new" }`). Do NOT use
session-memory tools after this point.

---

## Action: SETUP (guild, admin-only by convention — not enforced)

**Step 1 — Guard.** If not in a guild, reply:
> ❌ คำสั่งนี้ใช้ได้เฉพาะใน server ครับ

**Step 2 — Idempotency check:**
```bash
BOBBY_CLI_PROFILES_DIR=~/.openclaw/server-sessions \
  bobby-cli auth show --profile "$GUILD_ID" --json
```
- `loggedIn:false` (or the profile file doesn't exist) → continue to Step 3.
- `loggedIn:true` → ask:
  > ℹ️ Server นี้ถูก setup ไว้แล้วครับ ต้องการเปลี่ยนเป็น token ล่าสุดไหม? (ใช่ / ไม่)
  - "ไม่" → stop:
    > ✅ ยังคงใช้ token เดิมอยู่ครับ ไม่มีการเปลี่ยนแปลง
  - "ใช่" → continue to Step 3 (overwrite).

**Step 3 — Check the shared team token exists:**
```bash
bobby-cli auth show --json
```
- `loggedIn:false` → stop:
  > ❌ ยังไม่มี shared team token ครับ — กรุณาให้ผู้ดูแลระบบรัน `/setup_chawengburi` ก่อน
- `loggedIn:true` → continue to Step 4.

**Step 4 — Copy team credentials:**
```bash
cp ~/.bobby-cli/credentials.json ~/.openclaw/server-sessions/$GUILD_ID.json
```

**Step 5 — Verify:**
```bash
BOBBY_CLI_PROFILES_DIR=~/.openclaw/server-sessions \
  bobby-cli memory show --profile "$GUILD_ID" --json
```

**Completion:**
> ✅ เชื่อมต่อ server กับ shared memory แล้วครับ
>
> _(หมายเหตุ: ถ้า admin รัน `/setup_chawengburi` ใหม่ในอนาคต ทุก server ที่เคย `/bobby_cli setup` ไว้ต้องรัน `/bobby_cli setup` ซ้ำอีกครั้ง เพราะ token ที่ใช้ร่วมกันจะถูก rotate)_

---

## Action: LOGOUT (DM only)

**Step 0 — Validate: DM only** (same guard as LOGIN Step 0, substituting
`/bobby_cli logout` for `/bobby_cli login` in the message text).

**Step 1 — Extract `SENDER_ID`** (same as LOGIN Step 1).

**Step 2 — Confirm:**
> 🔓 ต้องการออกจากระบบ (ลบ token ที่บอทเก็บไว้) ใช่ไหมครับ? พิมพ์ **yes** เพื่อยืนยัน

**Step 3 — On "yes":**
```bash
BOBBY_CLI_PROFILES_DIR=~/.openclaw/user-sessions \
  bobby-cli auth forget --profile "$SENDER_ID" --json
```

**Completion (security caveat is mandatory, not optional — `bobby-cli auth
forget` only deletes the local profile file; it does not call any
server-side revoke endpoint):**
> ✅ ลบข้อมูล login ที่บอทเก็บไว้แล้วครับ ครั้งต่อไปต้องพิมพ์ `/bobby_cli login` ใหม่
>
> _(หมายเหตุ: นี่คือการลบสำเนาที่บอทเก็บไว้เท่านั้น ไม่ใช่การยกเลิก token ที่ auth-center — หากสงสัยว่า token รั่วไหลจริง ให้ติดต่อ admin เพื่อ revoke จากฝั่ง server)_
