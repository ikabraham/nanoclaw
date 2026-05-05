# Sub-Agent Creation Playbook

## Architecture: Two Databases

| Database | Location | Purpose |
|----------|----------|---------|
| PostgreSQL | onecli-postgres-1 (Docker) | OneCLI: agents, credentials, rules |
| SQLite | /home/ubuntu/nanoclaw/data/v2.db | NanoClaw: routing, sessions, pairings |

**Key fact:** Agent containers cannot access the host filesystem. All commands below must be run on the Ubuntu host terminal.

---

## Convention: Always Deliver All Four Scripts Together

When creating a new agent, provide ALL FOUR scripts in a single message. Never split them across turns.

- **Script 1** — pairing code registration (Step 4)
- **Script 2** — post-wiring fix: SQLite routing + container.json update (Step 7)
- **Script 3** — directory scaffold + identity write (Step 8)
- **Script 4** — OrchaYah contacts.md update (Step 9)

Instruct the user: run Script 1 → send pairing code in Telegram group → run Script 2 → run Script 3 → run Script 4.

---

## Step-by-Step

### 1. Ensure sqlite3 is installed
```bash
which sqlite3 || sudo apt-get install -y sqlite3
```

### 2. Create the agent via OrchaYah
Use `create_agent({ name: "agentname", instructions: "..." })` — creates the folder under `/home/ubuntu/nanoclaw/groups/<name>/` and adds a row to `agent_groups` in SQLite.

**⚠️ Known issue:** The container.json created by `create_agent` does NOT include `agentGroupId`. It must be written back manually (Step 7).

### 3. Verify the group folder exists
```bash
ls /home/ubuntu/nanoclaw/groups/
cat /home/ubuntu/nanoclaw/groups/<agentname>/container.json
```

### 4. Script 1 — Add the pairing code (use next available from contacts.md)

```bash
python3 << 'EOF'
import json
from datetime import datetime, timezone

path = "/home/ubuntu/nanoclaw/data/telegram-pairings.json"
CODE = "XXXX"       # next available from memory/contacts.md
FOLDER = "agentname"

with open(path) as f:
    doc = json.load(f)

data = doc["pairings"]
now = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.000Z")
new = {"code": CODE, "intent": {"kind": "wire-to", "folder": FOLDER}, "status": "pending", "createdAt": now}

if not any(x["code"] == CODE for x in data):
    data.append(new)
    print(f"Added: {CODE} -> {FOLDER}")
else:
    print(f"Already exists: {CODE}")

doc["pairings"] = data
with open(path, "w") as f:
    json.dump(doc, f, indent=2)
print("Done.")
EOF
```

### 5. Wire the Telegram group
- Ensure @OrchaYahBot is added to the new Telegram group
- Send the 4-digit code in that group
- Verify consumption: `grep "XXXX" /home/ubuntu/nanoclaw/data/telegram-pairings.json`

### 6. Diagnose and fix SQLite routing

**⚠️ Known issue:** NanoClaw sometimes maps a new group to the wrong agent. Always verify after wiring.

```bash
# Check consumed pairings
python3 << 'EOF'
import json
with open("/home/ubuntu/nanoclaw/data/telegram-pairings.json") as f:
    doc = json.load(f)
for p in doc["pairings"]:
    if p.get("status") == "consumed" and isinstance(p.get("intent"), dict):
        print(f"  {p['code']} ({p['intent']['folder']}): {p['consumed']['platformId']}")
EOF

# Check current routing
sqlite3 /home/ubuntu/nanoclaw/data/v2.db "SELECT mga.id, mg.platform_id, ag.folder, mga.agent_group_id FROM messaging_group_agents mga JOIN messaging_groups mg ON mga.messaging_group_id=mg.id JOIN agent_groups ag ON mga.agent_group_id=ag.id ORDER BY mga.created_at DESC LIMIT 10;"

# Fix wrong mapping if needed
sqlite3 /home/ubuntu/nanoclaw/data/v2.db "UPDATE messaging_group_agents SET agent_group_id='<correct-ag-id>' WHERE messaging_group_id='<mg-id>';"
```

### 7. Script 2 — Post-wiring fix (SQLite routing + container.json)

```bash
python3 << 'EOF'
import json, sqlite3, time

FOLDER = "agentname"        # e.g. "creativere"
DISPLAY_NAME = "AgentName"  # e.g. "CreativeRe"
DB = "/home/ubuntu/nanoclaw/data/v2.db"
PAIRINGS = "/home/ubuntu/nanoclaw/data/telegram-pairings.json"

with open(PAIRINGS) as f:
    doc = json.load(f)
platform_id = None
for p in doc["pairings"]:
    intent = p.get("intent", {})
    if isinstance(intent, dict) and intent.get("folder") == FOLDER and p.get("status") == "consumed":
        platform_id = p["consumed"]["platformId"]
        print(f"Found pairing code {p['code']} -> {platform_id}")
        break
if not platform_id:
    print(f"ERROR: No consumed pairing found for folder '{FOLDER}'")
    exit(1)

con = sqlite3.connect(DB)
cur = con.cursor()
mg_id = cur.execute("SELECT id FROM messaging_groups WHERE platform_id=?", (platform_id,)).fetchone()
ag_id = cur.execute("SELECT id FROM agent_groups WHERE folder=?", (FOLDER,)).fetchone()
if not mg_id or not ag_id:
    print(f"ERROR: messaging_group={mg_id}, agent_group={ag_id}")
    exit(1)
mg_id, ag_id = mg_id[0], ag_id[0]
print(f"messaging_group_id: {mg_id}")
print(f"agent_group_id: {ag_id}")

existing = cur.execute("SELECT id FROM messaging_group_agents WHERE messaging_group_id=?", (mg_id,)).fetchone()
if existing:
    cur.execute("UPDATE messaging_group_agents SET agent_group_id=? WHERE messaging_group_id=?", (ag_id, mg_id))
    print(f"Updated existing entry {existing[0]}")
else:
    mga_id = f"mga-{int(time.time()*1000)}-{FOLDER[:5]}"
    cur.execute("""INSERT INTO messaging_group_agents
        (id, messaging_group_id, agent_group_id, session_mode, priority, created_at,
         engage_mode, engage_pattern, sender_scope, ignored_message_policy)
        VALUES (?,?,?,'shared',0,datetime('now'),'pattern','.','known','accumulate')""",
        (mga_id, mg_id, ag_id))
    print(f"Inserted new entry {mga_id}")
con.commit()
con.close()

path = f"/home/ubuntu/nanoclaw/groups/{FOLDER}/container.json"
with open(path) as f:
    cj = json.load(f)
cj["agentGroupId"] = ag_id
cj["groupName"] = DISPLAY_NAME
cj["assistantName"] = DISPLAY_NAME
with open(path, "w") as f:
    json.dump(cj, f, indent=2)
print(f"Updated {FOLDER}/container.json")
print("Done.")
EOF
```

### 8. Script 3 — Directory scaffold + identity write

**This creates the required memory structure and writes the agent's identity.** Fill in the heredoc with the agent's full identity before giving to the user.

```bash
# Create directory structure
mkdir -p /home/ubuntu/nanoclaw/groups/FOLDER/conversations
mkdir -p /home/ubuntu/nanoclaw/groups/FOLDER/memory

# Write identity file
cat > /home/ubuntu/nanoclaw/groups/FOLDER/CLAUDE.local.md << 'IDENTITY'
# AgentName — Role Title

You are **AgentName**. [One sentence identity statement.]

[Full identity content — see memory/new-agent-template.md for structure]

## Memory
- `memory/` — structured memory files
- `conversations/` — past conversation logs
IDENTITY

echo "Directory scaffold and identity written for FOLDER"
```

**Verify the identity loaded:** Send `Reintroduce yourself.` in the agent's Telegram group after the next message. If it responds generically as "Claude", the CLAUDE.local.md wasn't picked up — re-run the write and try again.

### 9. Script 4 — Update OrchaYah's contacts.md

After the pairing is consumed and the agentGroupId is known, update `memory/contacts.md` and the pairing code registry. Run this on the host:

```bash
python3 << 'EOF'
import json, sqlite3

FOLDER = "agentname"
DISPLAY_NAME = "AgentName"
CODE = "XXXX"
PAIRINGS = "/home/ubuntu/nanoclaw/data/telegram-pairings.json"
DB = "/home/ubuntu/nanoclaw/data/v2.db"

with open(PAIRINGS) as f:
    doc = json.load(f)
platform_id = next(
    (p["consumed"]["platformId"] for p in doc["pairings"]
     if isinstance(p.get("intent"), dict) and p["intent"].get("folder") == FOLDER and p.get("status") == "consumed"),
    None
)

con = sqlite3.connect(DB)
ag_id = con.execute("SELECT id FROM agent_groups WHERE folder=?", (FOLDER,)).fetchone()
con.close()

if not platform_id or not ag_id:
    print(f"ERROR: platform_id={platform_id}, ag_id={ag_id}")
    exit(1)

print(f"Add this row to memory/contacts.md Agent Registry:")
print(f"| {DISPLAY_NAME} | {FOLDER} | {ag_id[0]} | {platform_id} |")
print()
print(f"Add this row to memory/contacts.md Pairing Code Registry:")
print(f"| {CODE} | {FOLDER} | consumed |")
print()
print(f"Update 'Next available pairing code' to: {int(CODE)+1}")
EOF
```

Then edit `memory/contacts.md` to add the new rows and increment the next available code. **This must be done every time** — it is the source of truth for all agent IDs.

---

## Identity Template

See `memory/new-agent-template.md` for the standard CLAUDE.local.md structure to use for new agents.

---

## Agent Registry

| Agent | Folder | agentGroupId | Telegram Platform ID |
|-------|--------|-------------|----------------------|
| OrchaYah | dm-with-todahyah | ag-1777422689239-bwp9a1 | telegram:8711481286 |
| DevOps (AWS) | devops | ag-1777478818969-utcbg7 | telegram:-1003915761734 |
| DevOps-WordPress | devops-wordpress | ag-1777478998688-qe14pf | telegram:-1003871244983 |
| TikTok | tiktok | ag-1777625934239-ykxrow | telegram:-1003907361310 |
| CRO | cro | ag-1777840556819-n4kx4h | telegram:-5239389129 |
| Funnel | funnel | ag-1777845824166-ajgwyh | telegram:-5133633329 |
| Copy | copy | ag-1777845835235-rbup0z | telegram:-5157673366 |
| CreativeRe | creativere | ag-1777940482199-lg1j5p | telegram:-5043864914 |
| CreReYah | creyah | ag-1777942977250-ju74gl | telegram:-5245448475 |
| FetchRe | fetchre | ag-1778008982242-sgvj7a | telegram:-5066111430 |
| ComfyRe | comfyre | ag-1778012590095-jt948t | telegram:-5135986813 |

**Next available pairing code: 2011** (canonical source: `memory/contacts.md`)

---

## Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| New group responds as wrong agent | SQLite `messaging_group_agents` has wrong `agent_group_id` | Run UPDATE query in Step 6 |
| container.json missing `agentGroupId` | `create_agent` doesn't write it back | Run Script 2 |
| Agent responds generically as "Claude" | CLAUDE.local.md not written or not picked up | Re-run Script 3, verify file content, send `Reintroduce yourself.` |
| `conversations/` or `memory/` missing | Script 3 not run | Run `mkdir -p /home/ubuntu/nanoclaw/groups/FOLDER/conversations /home/ubuntu/nanoclaw/groups/FOLDER/memory` |
| contacts.md not updated | Script 4 skipped | Run Script 4 and update manually |
| "Please run /login" in group | User's Telegram not linked to NanoClaw | User sends personal pairing code in DM to @OrchaYahBot |
| Script error "string indices must be integers" | Wrong file format assumption | Use `doc = json.load(f); data = doc["pairings"]` |
| sqlite3 command not found | Not installed on host | `sudo apt-get install -y sqlite3` |
