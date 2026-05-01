---
name: interview
description: Two-mode interview skill. Use when prepping for an upcoming interview (pull candidate bio, find your focus area in the interview guide, compile tailored questions, drop into a notes doc) OR debriefing after one (read raw notes, generate write-up, append to master doc, send via Slack or Greenhouse). Triggers on: "prep for interview", "who am I interviewing", "interview questions", "debrief", "interview write-up", "candidate feedback", "scorecard".
allowed-tools:
  - Bash(uv:*)
  - Bash(~/.claude/skills/slack/scripts/slack-cli:*)
  - Bash(sq:*)
  - Bash(python3:*)
  - Bash(cat:*)
  - Bash(mkdir:*)
  - Read
  - Write
  - Edit
metadata:
  author: skill-owner
  version: "3.0"
---

# Interview Skill

> **Maintainer:** Skill owner controls this file. Don't push edits back without their sign-off.

Two modes. Always ask upfront — do not try to detect from context:

> "Are you prepping for an upcoming interview, or debriefing after one?"

Role configs are stored at `~/.claude/skills/interview/role-configs.json`.

---

## Prep Mode

### P1: Role Setup

Ask:
> "What role is this interview for?"

Read the role configs:
```bash
cat ~/.claude/skills/interview/role-configs.json
```

**If role exists in config:** Show stored values (hiring manager, recruiter, interview guide URL, notes doc URL) and ask if anything changed. Update if needed.

**If new role:** Ask:
1. Hiring manager — name and Slack handle?
2. Recruiter — name and Slack handle?
3. Interview guide Google Doc URL?
4. Notes doc URL (existing multi-candidate doc), or should we create one?

If creating a new notes doc, use the gdrive skill to create "[Role Name] Interview Notes - [Month Year]" with sections for each candidate and a blank template at the bottom.

Save the config:
```python3
import json
from pathlib import Path

config_path = Path("~/.claude/skills/interview/role-configs.json")
configs = json.loads(config_path.read_text())

configs["[ROLE_NAME]"] = {
    "hiring_manager_name": "[NAME]",
    "hiring_manager_email": "[EMAIL]",
    "hiring_manager_slack": "[HANDLE]",
    "recruiter_name": "[NAME]",
    "recruiter_email": "[EMAIL]",
    "recruiter_slack": "[HANDLE]",
    "interview_guide_id": "[DOC_ID]",
    "interview_guide_url": "[DOC_URL]",
    "notes_doc_id": "[DOC_ID]",
    "notes_doc_url": "[DOC_URL]",
    "writeups_doc_id": "[DOC_ID]",
    "writeups_doc_url": "[DOC_URL]",
    "panel_channel_id": ""
}

config_path.write_text(json.dumps(configs, indent=2))
print("Saved.")
```

---

### P2: Candidate Bio

Ask:
> "What's the source for the candidate — LinkedIn URL, Slack channel/thread, or a PDF or document you can share?"

**If Slack channel or thread URL:** Read the thread using `sq agent-tools slack read-thread`. Look for a candidate report PDF attachment and read it via `sq agent-tools slack view-attachment --file-id [FILE_ID]`. Also check for a LinkedIn URL in the thread.

**If LinkedIn URL:** Fetch the profile with WebFetch. Note: LinkedIn without auth returns limited data — extract what's visible (current role, past roles, education) and note where data is thin.

**If none of the above:** Ask:
> "No problem — do you know where I can find the candidate's background? You can share a LinkedIn link, point me to a Slack thread, or paste a resume or bio directly."

Do not proceed to P3 without at least a basic candidate summary. A name alone is not enough.

Extract and summarize:
- Current role and company
- Career timeline (most recent 3–4 roles)
- Education
- Any strengths or flags noted in recruiter materials (if provided)

---

### P3: Focus Area & Questions

Read the interview guide using the gdrive skill:
```bash
cd ~/.claude/skills/gdrive && uv run gdrive-cli.py read [INTERVIEW_GUIDE_ID] 2>&1
```

**Find the row in the Interview Process table where the user's name appears.** Ask the user their name if not already known, or check who is running the session. Do not hardcode a name. That row identifies the assigned focus area.

Extract the full focus area section: the "What to look for" guidance, all questions, and all probes (→ lines).

Then generate a tailored question list by crossing the focus area questions against the candidate's actual background:
- Lead with the questions most likely to reveal signal given what this candidate has done
- Flag questions where the candidate's background suggests a potential gap (extra probing time)
- Flag questions where the candidate looks strong (confirm and move on)
- Keep all probes — they're the real test

Format:
```
FOCUS AREA: [Name]

WHAT TO LOOK FOR
[paste the guide's guidance verbatim]

QUESTIONS (tailored to [Candidate Name])

1. [Question text]
   → [Probe]
   * [Optional note: e.g., "Strong signal expected — confirm depth."]

2. [Question text]
   → [Probe]
   * [Optional note: e.g., "Gap risk — push hard here."]
...
```

Show the full output in the conversation first. Then ask:
> "Want me to drop this into your notes doc, or is the conversation view enough for now?"

---

### P4: Drop Into Notes Doc (if confirmed)

Only proceed if the user confirms they want it in a doc.

The notes doc is named after the interviewer — e.g. "[Name] only" or "[Role] Interview Notes - [Month Year] [[Name] only]". If no notes doc exists yet for this role, create one and save the ID to role config.

Find the candidate's section in the notes doc (from role config `notes_doc_id`). If the candidate already has a section, update it. If not, add a new section using the template at the bottom of the doc.

Insert the focus area + tailored questions under a "Questions" heading, before the "Raw Notes" area:

```bash
cd ~/.claude/skills/gdrive && echo '[MARKDOWN CONTENT]' | uv run gdrive-cli.py docs insert-markdown [NOTES_DOC_ID] --at-index [INDEX]
```

Return the doc link once done.

---

## Debrief Mode

### Step 1: Role Setup

Ask:
> "What role is this interview for?"

Read the role configs file:
```bash
cat ~/.claude/skills/interview/role-configs.json
```

**If role exists in config:** Show stored values and confirm they're current. Update if anything changed.

**If new role:** Ask for hiring manager (name, email, Slack handle), recruiter (name, email, Slack handle), and write-ups doc (create one if needed). Save to config.

---

### Step 2: Candidate & Raw Notes

Ask:
> "Who are we debriefing on?"

Then ask:
> "For the raw notes — do you have a Google Doc link, or would you like to paste them directly here?"

**If Google Doc link:** Use the gdrive skill to read the document. Extract only the raw notes section for this candidate — ignore any pre-interview content (interview questions, focus area framing, recruiter or search firm summaries, candidate reports, or any other material that existed before the interview). If the doc contains multiple candidates, extract only the relevant section.

**If pasted:** Accept as-is.

---

### Step 3: Generate Write-Up

Source the write-up exclusively from the interviewer's own raw notes. Do not pull from:
- Resumes, candidate reports, or search firm summaries
- Recruiter notes or pre-interview briefings
- Interview question framing or focus area guidance
- Anything that existed before the interviewer sat down with the candidate

If a question area has no notes, assume it was not covered — do not flag it or ask about it. Simply omit it from the write-up.

Generate a structured write-up in this format. Preserve the interviewer's honest voice — don't soften flags or polish away candor:

```
[Candidate Name]

**Overall Takeaway**
[2-3 sentence honest summary. Lead with the headline — use the interviewer's own words and framing where possible.]

**Strengths**
[Bullets grounded in what the interviewer noted. Specific, not generic.]

**Areas to Validate**
[Only include if the interviewer explicitly flagged something. Do not invent open questions.]

**Bottom Line**
[1-2 sentences. Clear stance — advance, advance with X to validate, do not advance.]
```

Show the draft in the conversation and ask:
> "Here's the draft — any edits before I add it to the doc?"

Incorporate edits before proceeding.

---

### Step 4: Append to Write-Ups Doc

The write-ups doc is the [Shared] doc (`writeups_doc_id` in role config). Never write to the interviewer's raw notes doc.

Append the finalized write-up:

```bash
cd ~/.claude/skills/gdrive && echo '――――――――――――――――――――

[WRITE-UP CONTENT]' | uv run gdrive-cli.py docs insert-markdown [WRITEUPS_DOC_ID]
```

Confirm once appended. Then ask:
> "Ready to share doc access with [hiring manager] and [recruiter]?"

If yes, update access (note: the gdrive CLI does not support suppressing notification emails — a Google Drive notification will be sent):
```bash
cd ~/.claude/skills/gdrive && uv run gdrive-cli.py share add [WRITEUPS_DOC_ID] --email [HM_EMAIL] --role writer 2>&1
uv run gdrive-cli.py share add [WRITEUPS_DOC_ID] --email [RECRUITER_EMAIL] --role writer 2>&1
```

---

### Step 5: Communicate — Slack or Greenhouse

Draft a Slack message grounded in the write-up. Casual, conversational — not formatted like a memo. Then ask where to send it:

> "Where should I send this — is there a panel channel, or should I DM [hiring manager] and [recruiter] directly?"

**If a `panel_channel_id` is already stored in role config:** Confirm it first:
> "I have [channel] on file — send it there, or somewhere else?"

**If sending to a channel:** Post to that channel ID using `sq agent-tools slack post-message`.

**If DM to HM and recruiter:** Look up their Slack user IDs first — search by email or handle, confirm the IDs before sending. Open a group DM and post.

Footer on all messages: `*(Sent via Claude Code on behalf of [User])*`

**Option 2: Greenhouse**

Check credentials:
```bash
cat ~/.claude/skills/interview/greenhouse-config.json 2>/dev/null || echo "NOT_CONFIGURED"
```

If not configured, ask for API key and user ID, save to `greenhouse-config.json`.

If configured, find the candidate via the Harvest API and post the scorecard. Confirm the upload URL before submitting.

---

## Notes

- Always ask prep vs. debrief upfront — never auto-detect
- Role config JSON is source of truth; always read it first
- The focus area is found by matching the user's name in the interview guide process table — never hardcode a name
- Write-ups are sourced only from the interviewer's raw notes — not resumes, reports, or pre-interview materials
- Questions with no notes = not covered; omit silently, don't flag
- [Shared] write-ups doc and [interviewer-only] notes doc are always kept separate
- Confirm Slack user IDs before sending any message; never guess
- Share doc access silently — suppress notification emails
