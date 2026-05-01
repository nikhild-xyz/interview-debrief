---
name: interview-debrief
description: Structured interview debrief workflow — reads raw notes, generates a candidate write-up using a standard template, appends to a per-role master doc, then either sends a Slack debrief to the hiring manager and recruiter or uploads a scorecard to Greenhouse.
allowed-tools:
  - Bash(uv:*)
  - Bash(/Users/ndixit/.claude/skills/slack/scripts/slack-cli:*)
  - Bash(python3:*)
  - Bash(cat:*)
  - Bash(mkdir:*)
  - Read
  - Write
  - Edit
metadata:
  author: ndixit
  version: "1.0"
---

# Interview Debrief Skill

> **Maintainer:** @ndixit owns this skill. To suggest changes, ping him directly on Slack. You're welcome to modify a local copy for personal use, but please do not push edits back to the shared skill file without his sign-off.

You are helping the user create a structured interview write-up for a candidate, append it to a per-role master Google Doc, and communicate it to stakeholders via Slack or Greenhouse. Be efficient and confirm before any irreversible actions (sending Slack, uploading to Greenhouse).

Role configs are stored at `/Users/ndixit/.claude/skills/interview-debrief/role-configs.json`.

---

## Step 1: Role Setup

Ask:
> "What role is this interview for?"

Read the role configs file:
```bash
cat /Users/ndixit/.claude/skills/interview-debrief/role-configs.json
```

**If the role already exists in config:**
Show the stored values and ask:
> "Here's what I have on file for [Role]:
> - Hiring manager: [name] (@[slack_handle])
> - Recruiter: [name] (@[slack_handle])
> - Master doc: [doc_url]
>
> All still correct, or any changes?"

Wait for confirmation. Update the config if anything changed.

**If the role is new:**
Ask:
> "Looks like this is a new role. A few quick setup questions — these are stored per role, so different roles can have different hiring managers and recruiters:
> 1. Hiring manager for this role — name and Slack handle?
> 2. Recruiter for this role — name and Slack handle?
> 3. Do you already have a master Google Doc for this role's write-ups, or should we create one?"

If creating a new doc, use the gdrive skill to create a new Google Doc titled "[Role Name] — Interview Write-Ups" and share/note the URL.

Save the config:
```bash
python3 << 'EOF'
import json
from pathlib import Path

config_path = Path("/Users/ndixit/.claude/skills/interview-debrief/role-configs.json")
configs = json.loads(config_path.read_text())

configs["[ROLE_NAME]"] = {
    "hiring_manager_name": "[NAME]",
    "hiring_manager_slack": "[HANDLE]",
    "recruiter_name": "[NAME]",
    "recruiter_slack": "[HANDLE]",
    "doc_id": "[DOC_ID]",
    "doc_url": "[DOC_URL]"
}

config_path.write_text(json.dumps(configs, indent=2))
print("Saved.")
EOF
```

---

## Step 2: Candidate & Raw Notes

Ask:
> "Who are we debriefing on?"

Then ask:
> "For the raw notes — do you have a Google Doc link, or would you like to paste them directly here? A Doc link is cleaner for longer notes; paste works great for shorter ones."

**If Google Doc link:** Use the gdrive skill to read the document content. Find and extract only the section relevant to this candidate if the doc contains multiple candidates.

**If pasted:** Accept the text as-is.

---

## Step 3: Generate Write-Up

Using the raw notes, generate a structured candidate write-up in this exact format. Be honest and direct — preserve the user's actual assessments and flags, don't soften them:

```
[Candidate Name]

**Overall Takeaway**
[2-3 sentence honest summary. Lead with the headline — what is this person's overall profile and the single most important thing to know.]

**Background & Experience**
[3-5 bullets covering relevant experience, tenure, scope of past roles, and anything that directly maps to this role's requirements.]

**Operational Strengths**
[3-5 bullets on what they do well. Specific and grounded in what came up in the interview — not generic.]

**Things to Watch / Push On**
[2-4 bullets on flags, open questions, or areas that need more digging. Be candid. Name the concern clearly.]

**Bottom Line**
[1-2 sentences. Clear recommendation stance — e.g. advance, advance with push on X, do not advance. Include the key open question if there is one.]
```

Present the write-up and ask:
> "Here's the draft write-up — any edits before I add it to the doc?"

Incorporate any edits.

---

## Step 4: Append to Master Doc

Append the finalized write-up to the master Google Doc using the gdrive skill:

```bash
cd /Users/ndixit/.claude/skills/gdrive && cat << 'EOF' | uv run gdrive-cli.py docs insert-markdown [DOC_ID] --tab t.0
――――――――――――――――――――

[WRITE-UP CONTENT]
EOF
```

Confirm once appended.

---

## Step 5: Communicate — Slack or Greenhouse

Ask:
> "How would you like to communicate this? Options:
> 1. Slack — draft a debrief DM to [hiring manager] and [recruiter]
> 2. Greenhouse — upload as a scorecard
> 3. Both"

### Option 1: Slack

Draft a tight Slack message using this structure:
- Opening line: "Hey [HM]/[Recruiter] — quick debrief on [Candidate Name] for the [Role] role."
- One bullet for the candidate covering: overall stance, key strengths (2-3), and the main flag if any
- Closing: "Let me know if any questions"
- Footer: "*(Sent via Claude Code on behalf of Nikhil)*"

Present the draft and ask:
> "Here's the Slack draft — any tweaks, or good to send?"

After confirmation, look up user IDs for both the hiring manager and recruiter, open a group DM, and send:

```python3
import json, urllib.request, urllib.parse

TOKEN = "[READ FROM ~/.config/slack-skill/credentials.json]"

# Open group DM
data = urllib.parse.urlencode({"users": "[HM_USER_ID],[RECRUITER_USER_ID]"}).encode()
req = urllib.request.Request(
    "https://slack.com/api/conversations.open",
    data=data,
    headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/x-www-form-urlencoded"}
)
with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())
channel_id = result["channel"]["id"]
print(channel_id)
```

Then post using:
```bash
/Users/ndixit/.claude/skills/slack/scripts/slack-cli post-message --channel-id [CHANNEL_ID] --workspace block --format markdown "[MESSAGE]"
```

### Option 2: Greenhouse

Check if Greenhouse API credentials are configured:
```bash
cat /Users/ndixit/.claude/skills/interview-debrief/greenhouse-config.json 2>/dev/null || echo "NOT_CONFIGURED"
```

**If not configured**, tell the user:
> "Greenhouse isn't connected yet. To set it up I'll need:
> 1. Your Greenhouse API key (found in Greenhouse → Settings → Dev Center → API Credential Management)
> 2. Your Greenhouse user ID
>
> Once you have those I can save them and handle uploads automatically going forward."

Save credentials when provided:
```bash
python3 << 'EOF'
import json
from pathlib import Path

config = {
    "api_key": "[API_KEY]",
    "user_id": "[USER_ID]",
    "base_url": "https://harvest.greenhouse.io/v1"
}
Path("/Users/ndixit/.claude/skills/interview-debrief/greenhouse-config.json").write_text(json.dumps(config, indent=2))
print("Saved.")
EOF
```

**If configured**, upload the scorecard via the Greenhouse Harvest API:
```python3
import json, urllib.request, urllib.parse, base64
from pathlib import Path

config = json.loads(Path("/Users/ndixit/.claude/skills/interview-debrief/greenhouse-config.json").read_text())
api_key = config["api_key"]
user_id = config["user_id"]

# Encode API key
credentials = base64.b64encode(f"{api_key}:".encode()).decode()

# First: find the candidate/application ID in Greenhouse
# GET /v1/candidates?email= or search by name
search_url = f"https://harvest.greenhouse.io/v1/candidates?full_name=[CANDIDATE_NAME]"
req = urllib.request.Request(
    search_url,
    headers={"Authorization": f"Basic {credentials}", "On-Behalf-Of": user_id}
)
with urllib.request.urlopen(req) as resp:
    candidates = json.loads(resp.read())
print(json.dumps(candidates[:2], indent=2))
```

After finding the candidate and application, post the scorecard with the write-up content as the overall notes. Confirm the upload URL with the user before submitting.

---

## Notes

- Always confirm before sending Slack or submitting to Greenhouse
- Preserve the user's honest voice in write-ups — don't sanitize flags or concerns
- The role config JSON is the source of truth for per-role setup; always read it first
- If a doc doesn't exist yet for a role, create it before appending
- For group DMs: read the Slack token from `~/.config/slack-skill/credentials.json` under the `token` key
