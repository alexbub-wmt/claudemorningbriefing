---
name: morning-briefing
description: >
  Review the user's inbox, calendar, and Teams messages to prepare a morning briefing.
  Use this skill whenever the user asks to "get ready for the day", "review my inbox",
  "what do I have today", "morning briefing", "catch me up", "what did I miss",
  "what's on my plate", "what's happening today", or any variation of wanting
  a daily summary of emails, meetings, and messages. Trigger even if the user
  only mentions one of the three (email/calendar/Teams) — always pull all three
  for a complete picture. This skill uses Microsoft 365 (Outlook + Teams).
compatibility: "Requires Microsoft 365 connector (Outlook + Teams)"
---

# Morning Briefing Skill

Prepares the user for their day by summarizing their inbox, today's calendar, and recent Teams messages.

**Implementation notes for Claude:**
- This skill is best run with **Sonnet 5** for accuracy and reliability in pattern-matching complex email rules (sender/subject/body matching across 10 rules).
- The filing workflow (step 4) is **email-first, rule-ordered**: pull inbox once, process each email through rules 1–10 in order, apply first match, move to next email. This ensures every email is checked against every rule systematically.
- **Processing guarantee**: batch tag and move operations atomically; every email gets evaluated; no rule is skipped.
- Maintain a filing registry (persistent across runs) to prevent duplicate processing and enable tracking of reminder fires.
- Always include the filing summary footer so the user can verify what was processed and spot any errors. Report counts by rule.
- Draft creation (step 5) happens after filing is complete. The draft is a self-reminder for the user and is **never sent automatically** — the user reviews and sends manually.

## Step-by-Step Workflow

### 1. Gather data in parallel (or sequentially if parallel isn't available)

Run all three searches simultaneously:

**Inbox** — unread/recent emails from the last 48 hours:
```
outlook_email_search(folderName="Inbox", afterDateTime="2 days ago", limit=30)
```

**Calendar** — today's meetings and events:
```
outlook_calendar_search(query="*", afterDateTime="today", beforeDateTime="tomorrow", limit=20)
```

**Teams** — recent chat messages from the last 3 days:
```
chat_message_search(query="*", afterDateTime="3 days ago", limit=50)
```

### 2. Read full content selectively

- For emails that look **urgent or important** based on subject/sender, call `read_resource` with the email URI to get the full body before summarizing.
- Don't read every email — use subject lines and senders to prioritize.
- For calendar events, the search result usually has enough detail (time, attendees, location). Only call `read_resource` if the body seems important (e.g., agenda for a key meeting).

### 3. Synthesize and present the briefing

Structure the output as a clear, scannable morning briefing. Use this format:

---

## 🌅 Good morning! Here's your day at a glance.

**📅 Today's Meetings** *(chronological)*
List each meeting with: time, title, attendees (shortened), and location/link if present.
Flag any meetings in the next 1–2 hours as "⚡ Coming up soon."
If no meetings: "You have a clear calendar today."

**📬 Inbox Highlights** *(prioritized)*
Group into:
- 🔴 **Action needed** — emails requiring a reply or decision
- 📌 **FYI / notable** — important updates, announcements, things to be aware of
- 🗃️ **Low priority** — newsletters, automated notifications (mention count only, don't list)

For each notable email: sender name, subject, and a 1-sentence summary of what it's about.

**💬 Teams Highlights**
Summarize notable messages or threads. Skip automated bot messages.
If nothing notable: "No urgent Teams messages."

**🎯 Suggested Focus**
Based on what you've seen, offer 2–3 bullet points on what the user might want to tackle first. Keep it practical, not generic.

---

## Tone & Style Guidelines

- Be **concise** — the goal is a quick scan, not a wall of text.
- Use **plain language** — avoid corporate speak.
- **Prioritize ruthlessly** — surface the 3–5 things that actually matter.
- If something is time-sensitive or requires action before a meeting, call it out clearly.
- If the inbox is empty or quiet, say so cheerfully — that's good news.
- Use the user's **local context** — if you know their timezone or location, note the time naturally.
- Keep the total briefing readable in under 2 minutes.

### 4. File recurring "update" emails (Claude Auto category)

After presenting the briefing, apply filing rules to recurring automated senders using **email-first, rule-ordered processing**.

**Filing Workflow:**

1. **Pull inbox** — search inbox from midnight today, limit 25, newest first. Store full results (sender, subject, body snippet, messageId).

2. **Process each email in order** — for each email in the results:
   - Check rules 1–10 in sequence (priority order below)
   - On **first rule match**, immediately apply that rule's action (tag ± move)
   - Log the result: `{ messageId, rule matched, action taken, timestamp }`
   - Move to next email
   - If no rule matches, leave untouched

3. **Batch tag and move operations** — collect all tag operations and all move operations (grouped by destination folder), then execute via atomic `outlook_batch_modify_labels` calls. Capture success/failure for each batch.

4. **Update filing registry** — for each successfully filed email, append to `/filing-registry.md`: messageId, timestamp, folder destination (if moved), rule name matched, ticket # (if applicable).

5. **Report results** — include filing summary by rule (counts and specific emails per rule) and any errors in the briefing footer (see step 6, below).

**Error handling:** If tagging fails for an email, do not attempt to move it. Log as error and skip to next email. If a move fails after successful tagging, log that too (email is already tagged/flagged even if move didn't complete).

**Processing guarantee:** Every email in the inbox is checked against all 10 rules in order until a match is found. No email is skipped; no rule is bypassed.

**Defined rules (auto-executed, in priority order):**

1. **Syncro ticket reply notifications** (sender `no-reply@syncromsp.com` OR `support@westmaintech.com`, subject contains "A Ticket Reply came in"):
   - If body contains **"REPLY ABOVE THIS LINE TO SEND A RESPONSE"** → tag only, leave in inbox.
   - Otherwise → tag, move to **_SYNCRO ALERTS**.

2. **Syncro Community digests** (sender `notifications@syncro.discoursemail.com`, subject "[Syncro Community] Summary"):
   - Scan body for security/RMM keywords; mention in briefing FYI if found.
   - Tag, move to **_TO REVIEW > NEWS** folder.

3. **CUSTOMER REPLY REMINDER alerts** (sender `support@westmaintech.com`, subject "CUSTOMER REPLY REMINDER", body contains "NEEDS IMMEDIATE ATTENTION"):
   - Scan body for ticket #, customer/contact name, reply content.
   - Check filing registry: count how many times this ticket's reminder has fired in last 7 days; surface count + details in briefing as flagged item.
   - Tag, move to **_SYNCRO ALERTS**.

4. **WMT internal ticket copies** (sender `support@westmaintech.com`, sent to `tech@westmaintech.com`, subject contains ticket reference like "j&j" or "Website form Issue"):
   - If body contains **"REPLY ABOVE THIS LINE TO SEND A RESPONSE"** → tag only, leave in inbox.
   - Otherwise → out of scope, do not touch (skip to next email).

5. **Axcient case emails** (sender `support@axcient.com`, subject contains "Case #"):
   - Tag only, leave in inbox (manual review needed).

6. **Blackpoint Cyber marketing** (sender `marketing@blackpointcyber.com`):
   - Tag, move to **_TO REVIEW > NEWS**.

7. **Syncro webinar/promotional** (sender `webinars@syncrosecure.com`):
   - Tag, move to **_TO REVIEW > NEWS**.

8. **Syncro "Tickets Due Tomorrow"** (sender `no-reply@syncromsp.com`, subject "Tickets Due Tomorrow"):
   - Tag, move to **_SYNCRO ALERTS**.

9. **Comcast payment notifications** (sender `online.communications@alerts.comcast.net`, subject contains "payment"):
   - Tag, move to **VENDORS > Comcast**.

10. **Bitwarden invoices/receipts** (sender `no-reply@bitwarden.com` OR `invoice+statements@bitwarden.com`, subject contains "invoice" or "receipt"):
    - Tag, move to **VENDORS > Bitwarden**.
    - Note: other Bitwarden emails (login alerts, member confirmation requests) do NOT match this rule — only billing-related subjects.

11. **Teams message notifications** (sender `no-reply@teams.mail.microsoft`):
    - Read and analyze content; include relevant details in the briefing (💬 Teams Highlights section).
    - Tag Claude Auto, then move to Deleted Items via `outlook_trash_thread` (soft delete — recoverable, not permanent). Do this only after the content has been read and reflected in the briefing.

12. **Any other sender** → do not touch (rules not yet defined; confirm with user first).

### 5. Create Summary Email Draft

After filing is complete, create an email draft for yourself as a self-reminder and save it to your Drafts folder.

**Draft content:**
1. **Subject:** "Daily Briefing Summary — [date, e.g., Thursday, Aug 6, 2026]"
2. **To:** alex@westmaintech.com
3. **Body:** Condensed version of the briefing:
   - Today's meetings (times, titles, key attendees)
   - Action items from email (top 3–5 from 🔴 section)
   - FYI highlights (top 2–3 from 📌 section)
   - Any flagged Syncro reminders (CUSTOMER REPLY REMINDER with ticket counts)
   - Suggested focus areas (2–3 from the briefing)

**Execution:**
- Call `outlook_create_draft` with the above structure
- Save to your Drafts folder (inbox name: Drafts)
- Do NOT send automatically — user reviews and sends manually
- Report in the footer that draft was created

**Draft footer note:** Include a timestamp ("Created by morning briefing at [time]") so you know when it was generated.

### 6. Filing summary footer & audit trail

At the bottom of every briefing/digest, include a **Filing Actions** section with:
- Count of successful tags and moves (by rule)
- Any tagging or move failures (email ID, rule, error detail)
- Confirmation that registry was updated (e.g., "✓ Registry updated: 4 emails logged")
- Any candidates that were examined but skipped (e.g., "already in registry, skipped")

Example format:

```
---
📂 Filing actions this run:
✓ Tagged 4, moved 3:
  • 3 Syncro ticket replies → moved to _SYNCRO ALERTS
  • 2 Syncro ticket replies (awaiting response) → tagged, left in inbox
  • 1 Syncro Community digest → moved to _TO REVIEW > NEWS
  • 1 CUSTOMER REPLY REMINDER (Ticket #47738, 2nd reminder, no follow-up) → moved to _SYNCRO ALERTS

✓ Registry updated: 4 emails logged

✉️ Draft created: "Daily Briefing Summary — Thursday, Aug 6, 2026" saved to Drafts (review and send manually)

⚠️ Issues (1):
  • Tagging failed for email ID [xyz123]: "Rich Stewart new computer" — left untouched, no move attempted
```

If nothing was filed, state: "No filing actions this run. ✓ Registry up to date."

**Draft creation note:** Always include the draft confirmation line in the footer, even if nothing was filed — the draft is independent of filing actions.

**Registry file location:** The filing registry is maintained at `/filing-registry.md` (or similar persistent location). Each entry includes: messageId, timestamp, destination folder, rule name. This allows future runs to:
- Check whether an email was already processed (avoid duplicate actions)
- Count reminder fires for the same ticket (for CUSTOMER REPLY REMINDER tracking)
- Audit what was filed and when

### 7. Verify moves completed successfully

**Do not trust the batch API response alone** — a "success" status doesn't always mean the email landed in the destination folder, and for recurring senders (Syncro, CUSTOMER REPLY REMINDER, etc.) subject-line matching is not sufficient verification, since multiple similar-subject emails arrive per day. Verification must match on **exact messageId**.

**Verification workflow:**

1. **Track messageIds through the move, not just subjects** — from the filing results in step 4, build a list of `{ messageId (post-move, if returned by the API), expectedFolder, originalSubject }` for every email that was supposed to move.

2. **Read the destination folder directly by folder ID** — use `read_resource` with `mail:///folders/{folderId}` for each destination folder used this run. Do NOT rely on `outlook_email_search` for verification — it has shown stale/cached results and folder-name lookups can silently fail. A direct folder read reflects true current state.

3. **Compare by exact messageId, never by subject alone** — check whether each expected messageId (or the new ID returned by the move operation, if the API changes it) appears in the destination folder's message list. Subject-line or sender matching is **not sufficient** — recurring senders produce many similar emails per day, and a similar-looking match is not proof the specific email moved.

4. **Also check the source folder (Inbox) directly** — read the Inbox folder by ID the same way. If a messageId that was supposed to move is still present in Inbox, the move failed regardless of what the batch API reported.

5. **Compare expected vs. actual:**
   - If a messageId is found in its expected destination folder → ✅ verified.
   - If a messageId is **not found** in the destination folder AND is still in Inbox → ❌ move did not complete. Flag it explicitly with subject + messageId.
   - If a messageId is not found in either location → ⚠️ flag as needing manual investigation (could have moved somewhere unexpected).

6. **Report verification results** in the footer, separate from the initial filing summary:

```
🔍 Move verification (by exact messageId, folders read directly):
✓ Verified 3/3 moves to _SYNCRO ALERTS
✓ Verified 1/1 move to _TO REVIEW > NEWS
⚠️ 2 emails still in Inbox despite reported "success": "CUSTOMER REPLY REMINDER" [messageId], "A Ticket Reply came in" [messageId]. Filing did not actually complete — retrying required.
```

7. **If any verification fails, do not silently retry with the same batch call** — first confirm the tag/move batch call itself returned success for that messageId (check the batch API's per-message `categoriesFailed` / `moveFailed` arrays from step 4). If the batch call itself reported failure for that message, retry the specific failed operation. If the batch call reported success but the email is still in Inbox, treat this as a deeper sync/API issue — report it clearly rather than looping retries, since blind retries could cause duplicate moves.

**Why this matters:** the CUSTOMER REPLY REMINDER and Syncro Community incidents both involved emails that were tagged but not actually moved, even though the tagging step appeared to succeed. This verification step catches that failure mode before the user finds out the hard way.

## Edge Cases

- **No emails**: "Your inbox is quiet — enjoy it while it lasts."
- **No meetings**: "You have a free day — great for deep work."
- **Lots of unread email (20+)**: Summarize by theme/sender cluster rather than listing each one.
- **Meeting in <30 min**: Prominently call it out at the top with a ⚡ warning.
- **Tool errors**: If one data source fails (e.g., Teams is unavailable), proceed with the others and note what's missing.

## Example Output Shape

```
🌅 Good morning! Here's your day at a glance.

📅 Today's Meetings
• ⚡ 9:00 AM – Standup with Engineering (30 min) — Zoom
• 11:00 AM – 1:1 with Sarah — Teams call
• 2:00 PM – Product Review (1 hr) — Conf Room B

📬 Inbox Highlights
🔴 Action needed:
• [Boss] "Q3 Budget Approval" — Needs your sign-off by EOD
• [Client] "Follow-up on proposal" — They have questions about pricing

📌 FYI:
• [IT] "Scheduled maintenance Friday 6–8 PM" — Plan accordingly
• [HR] "New PTO policy update" — Minor changes to rollover rules

🗃️ Low priority: 8 newsletters, 3 automated alerts

💬 Teams Highlights
• Design team is asking for feedback on the new mockups (pinged you twice)
• Nothing else urgent

🎯 Suggested Focus
• Reply to the client email before your 11 AM — they seem to be waiting
• Review budget doc before EOD to hit the approval deadline
• Quick Teams reply to Design so they're unblocked

---
📂 Filing actions & summary:
✓ Tagged 3, moved 2 (Syncro alerts, news folder)
✓ Registry updated: 2 emails logged
✉️ Draft created: "Daily Briefing Summary — Thursday, Aug 6, 2026" → Drafts (review and send)

🔍 Move verification:
✓ Verified 2/2 moves
```
