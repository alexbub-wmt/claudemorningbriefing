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

Prepares the user for their day by summarizing their inbox, today's calendar, and recent Teams messages, filing recurring automated emails, and delivering one combined summary to Inbox.

**Implementation notes for Claude:**
- This skill is best run with **Sonnet 5** for accuracy and reliability in pattern-matching complex email rules (sender/recipient/subject/body matching across the full rule set).
- **Execution order (fixed):** Step 1 (gather) → Step 2 (read selectively) → Step 4 (file) → Step 7 (verify) → Step 3.5 (lookback) → Step 3 (build briefing, using Step 6 footer format) → Step 5 (deliver). The briefing is built **once**, after filing and verification, because the email body needs their results. Do not produce a separate chat version — the final chat response is only the Step 5 confirmation plus the Step 6 footer.
- The filing workflow (step 4) is **email-first, rule-ordered**: process each email from the Step 1 inbox pull through every defined rule in order, apply the first match, and move to the next email.
- Rules are ordered **most specific first** so a broad rule never shadows a narrow one (e.g. CUSTOMER REPLY REMINDER precedes the general Syncro ticket rule, since both come from `support@westmaintech.com`).
- **Processing guarantee**: batch tag and move operations atomically; every email gets evaluated; no rule is skipped.
- **No filing registry.** The mailbox is the record: the **Claude Auto** category marks an email as already processed, `_SYNCRO ALERTS` holds reminder history, and each Daily Briefing Summary email is the audit trail. Do not write a registry file — scheduled runs start in a fresh workspace, so it would not persist.
- Always include the filing summary footer so the user can verify what was processed and spot any errors. Report counts by rule.
- The extended 7-day lookback (Step 3.5) is **Inbox only** and read-only — it never triggers new filing actions.
- The summary (step 5) is created as a draft and moved to Inbox. It is **never sent** — no `outlook_send_mail` call at any point. The only address used is alex@westmaintech.com.

## Step-by-Step Workflow

### 1. Gather data in parallel (or sequentially if parallel isn't available)

Run all searches simultaneously. Each source is pulled **once** per run and reused by later steps.

**Inbox** — emails from the last 48 hours (feeds both the briefing and Step 4 filing):
```
outlook_email_search(folderName="Inbox", afterDateTime="2 days ago", limit=25)
```
**Paginate until exhausted.** The API caps results at 25 per call, so if the response indicates more results exist (`moreResults: true`, or a `totalResultCount` higher than what you received), call again with `offset` 25, then 50, and so on until every email in the window has been retrieved. **Do not process a partial set** — emails past the first 25 are silently skipped otherwise, which has caused missed filings before. Store full results (sender, recipient, subject, body snippet, messageId, categories).

**_SYNCRO ALERTS** — last 7 days, read directly by folder ID (`read_resource` with `mail:///folders/{folderId}`, paginating if needed). Used for Rule 1 reminder counts and Step 7 verification.

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
- Don't read every email — use subject lines and senders to prioritize. Rules 10 and 11 also require reading (see Step 4).
- For calendar events, the search result usually has enough detail (time, attendees, location). Only call `read_resource` if the body seems important (e.g., agenda for a key meeting).

### 3. Synthesize the briefing

Built **after** Steps 4, 7 and 3.5 are complete (see execution order). Structure it as a clear, scannable briefing. Use this format:

---

## 🌅 Here's your day at a glance.

**📅 Today's Meetings** *(chronological)*
List each meeting with: time, title, attendees (shortened), and location/link if present.
Flag any meetings in the next 1–2 hours as "⚡ Coming up soon."
If no meetings: "You have a clear calendar today."

**📬 Inbox Highlights** *(prioritized)*
Group into:
- 🔴 **Action needed** — emails requiring a reply or decision (include flagged CUSTOMER REPLY REMINDERs with 7-day ticket counts)
- 📌 **FYI / notable** — important updates, announcements, things to be aware of (include Rule 3 keyword hits and unrecognized Rule 10 prefixes)
- 🗃️ **Low priority** — newsletters, automated notifications (mention count only, don't list)

For each notable email: sender name, subject, and a 1-sentence summary of what it's about.

**💬 Teams Highlights**
Summarize notable messages or threads from the Step 1 Teams search and Rule 11 notification emails — don't list the same message twice. Skip automated bot messages.
If nothing notable: "No urgent Teams messages."

**⏳ Long-Term Follow-Up** *(only if items found — see Step 3.5)*
List items still open from the extended 7-day lookback. For each: sender/subject, how many days it's been sitting, and why it's still flagged (e.g., "Axcient Case #12345, tagged, in Inbox 3 days").
If nothing found in the lookback: omit this section entirely (don't show an empty header).

**🎯 Suggested Focus**
Based on what you've seen, offer 2–3 bullet points on what the user might want to tackle first. Keep it practical, not generic.

**📂 Filing actions** and **🔍 Move verification** — per Step 6 format.

---

### 3.5 Extended lookback (up to 7 days)

Catches items that scrolled out of the current window but are still unresolved.

**Scope:** **Inbox only**, items received in the **last 7 days**. Reuse the Step 7 Inbox read (paginate it back to 7 days if needed). Do not read `_SYNCRO ALERTS` or any other folder for this step.

**What counts as "still flagged for follow-up":**
- Any email tagged "Claude Auto" and left in Inbox (per rules that say "tag only, leave in inbox") that is still present and unactioned after more than 1 day.
- Any item explicitly called out as "action needed" in a prior briefing that is still present in Inbox today.

**What does NOT count:**
- Items already filed/moved out of Inbox as part of normal rule processing — those are handled, not "open."
- Low-priority/newsletter-type items — this section is for things that need a decision or reply, not backlog volume.
- Anything already surfaced in this run's 🔴 Action needed section — don't duplicate; only show items *older* than today that are still lingering.

**Output:** populate the **⏳ Long-Term Follow-Up** section in the briefing (Step 3 format above). If nothing qualifies, omit the section.

**Note:** this lookback is read-only — it does not trigger new filing actions. If an old item matches a filing rule and simply wasn't processed in a prior run (a genuine miss), flag it in this section AND note it as needing manual reprocessing, but don't auto-file it.

## Tone & Style Guidelines

- Be **concise** — the goal is a quick scan, not a wall of text.
- Use **plain language** — avoid corporate speak.
- **Prioritize ruthlessly** — surface the 3–5 things that actually matter.
- If something is time-sensitive or requires action before a meeting, call it out clearly.
- If the inbox is empty or quiet, say so cheerfully — that's good news.
- Use the user's **local context** (America/New_York) — note the time naturally. This runs several times a day, so don't assume it's morning.
- Keep the total briefing readable in under 2 minutes.

### 4. File recurring "update" emails (Claude Auto category)

Apply filing rules to recurring automated senders using **email-first, rule-ordered processing**, on the inbox results from Step 1 (do not pull the inbox again).

**Filing Workflow:**

1. **Skip already-processed emails** — if an email already has the **Claude Auto** category, skip it and count it as "already processed." **Exception:** an email tagged Claude Auto but still in Inbox that should have been moved (Rules 1, 3–8, 10 MMC/WMT, 11) is a failed move from a prior run — queue its move again without re-tagging, and note the retry in the footer.

2. **Process each remaining email in order** — for each email:
   - Check **every defined rule** in sequence (priority order below), starting at rule 1 and continuing until a match is found or the rule list is exhausted
   - On **first rule match**, queue that rule's action (tag ± move)
   - Log the result: `{ messageId, rule matched, action taken, timestamp }`
   - Move to next email
   - If no rule matches, leave untouched

3. **Batch tag and move operations** — collect all tag operations and all move operations (grouped by destination folder), then execute via atomic `outlook_batch_modify_labels` calls. Capture per-message success/failure (`categoriesFailed` / `moveFailed`).

4. **Report results** — include filing summary by rule (counts and specific emails per rule) and any errors in the footer (see step 6, below).

**Error handling:** If tagging fails for an email, do not attempt to move it. Log as error and skip to next email. If a move fails after successful tagging, log that too (email is already tagged even if move didn't complete — the next run will retry it per item 1).

**Processing guarantee:** Every email in the inbox — across all pagination pages — is checked against every defined rule in order until a match is found. No email is skipped; no rule is bypassed. When rules are added or removed, this guarantee still means "all of them," so do not rely on a hardcoded rule count anywhere in execution.

**Defined rules (auto-executed, in priority order — most specific first):**

1. **CUSTOMER REPLY REMINDER alerts** (sender `support@westmaintech.com`, subject contains "CUSTOMER REPLY REMINDER"):
   - Scan body for ticket #, customer/contact name, reply content.
   - Count how many times this ticket's reminder has fired in the last 7 days (this email plus CUSTOMER REPLY REMINDER emails with the same ticket # in the Step 1 `_SYNCRO ALERTS` read); surface count + details in briefing as flagged item.
   - Tag, move to **_SYNCRO ALERTS**.
   - *Ordered first because it is the most specific `support@westmaintech.com` case — Rule 2 would otherwise shadow it.*

2. **Syncro ticket correspondence** (sender `no-reply@syncromsp.com` OR `support@westmaintech.com`, matching ANY of: subject contains "A Ticket Reply came in"; subject contains "(message id:"; recipient is `tech@westmaintech.com`):
   - If body contains **"REPLY ABOVE THIS LINE TO SEND A RESPONSE"** → tag only, leave in inbox (an open thread the user may need to answer).
   - Otherwise → tag, move to **_SYNCRO ALERTS**.
   - *Match on sender + recipient + body string. Do NOT match on specific past subject text (e.g. "j&j", "Website form Issue") — those were examples, not a pattern, and matching them produces false negatives on new ticket titles.*

3. **Syncro Community digests** (sender `notifications@syncro.discoursemail.com`, subject "[Syncro Community] Summary"):
   - Scan body for security/RMM keywords; mention in briefing FYI if found.
   - Tag, move to **_TO REVIEW > NEWS**.

4. **Syncro "Tickets Due Tomorrow"** (sender `no-reply@syncromsp.com`, subject "Tickets Due Tomorrow"):
   - Tag, move to **_SYNCRO ALERTS**.

5. **Syncro webinar/promotional** (sender `webinars@syncrosecure.com`):
   - Tag, move to **_TO REVIEW > NEWS**.

6. **Blackpoint Cyber marketing** (sender `marketing@blackpointcyber.com`):
   - Tag, move to **_TO REVIEW > NEWS**.
   - Note: `support@blackpointcyber.com` (renewals, billing reports) does NOT match — no rule defined yet.

7. **Comcast payment notifications** (sender `online.communications@alerts.comcast.net`, subject contains "payment"):
   - Tag, move to **VENDORS > Comcast**.
   - Note: Comcast outage/service notices from `noreply@alerts.comcast.net` and non-payment subjects do NOT match — no rule defined yet.

8. **Bitwarden invoices/receipts** (sender `no-reply@bitwarden.com` OR `invoice+statements@bitwarden.com`, subject contains "invoice" or "receipt"):
   - Tag, move to **VENDORS > Bitwarden**.
   - Note: other Bitwarden emails (login alerts, member confirmation requests) do NOT match — only billing-related subjects.

9. **Axcient case emails** (sender `support@axcient.com`, subject contains "Case #"):
   - Tag only, leave in inbox (manual review needed).
   - Note: Axcient x360Cloud digests and org-attention notices do NOT match — no rule defined yet.

10. **Sophos client daily reports** (sender `wmt.notifications@gmail.com`, subject matches "Events - Daily", "Traffic Dashboard - Daily", or "SSL VPN - Daily"):
    - **This rule requires opening the email** — matching can't be done from subject/sender alone, since the client is identified only by the PDF attachment filename. Call `read_resource` on the email to get the attachment filename before deciding.
    - If the PDF attachment filename **starts with "MMC"** (case-insensitive, any separator or none — e.g. `MMC_AUTH_EVENTS_DAILY.pdf`, `MMC_TRAFFIC_DASHBOARD.pdf`, `MMC SSL VPN.pdf` all match): tag, move to **Clients > MMC > _SOPHOS REPORTS** (folder ID: `AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwAEifQaawAAAA==`).
    - If the PDF attachment filename **starts with "WMT"** (case-insensitive, any separator or none — e.g. `WMT_AUTH_EVENTS.pdf`): tag, move to **_WMT > COMPLIANCE** (folder ID: `AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwABPCPpnQAAAA==`). Note: this is the COMPLIANCE folder nested under _WMT — not the separate top-level COMPLIANCE folder that also exists in the mailbox.
    - If the PDF attachment filename starts with a **different, unrecognized** prefix (any client other than "MMC" or "WMT"): do NOT move. Tag Claude Auto, leave in inbox, and surface in the briefing as an unrecognized client prefix that needs a new rule — do not guess a destination folder for it.
    - Do not match on subject or recipient alone — always confirm via the actual attachment filename, since these reports are otherwise identical across clients.
    - Note: this is distinct from `do-not-reply@central.sophos.com` "[HIGH] Alert for Sophos Central" real-time alert emails — those are a separate, still-undefined case and fall through to Rule 12.

11. **Teams message notifications** (sender `no-reply@teams.mail.microsoft`):
    - Read and analyze content FIRST; include relevant details in the briefing (💬 Teams Highlights section).
    - Then tag Claude Auto and move to Deleted Items via `outlook_trash_thread` (soft delete — recoverable, not permanent). Never trash before the content has been captured for the briefing.

12. **Any other sender** → do not touch (no rule defined; surface in briefing and confirm with user before adding a rule).

**Quick reference — destinations:**

| Destination | Rules |
|---|---|
| _SYNCRO ALERTS | 1, 2 (no "REPLY ABOVE"), 4 |
| _TO REVIEW > NEWS | 3, 5, 6 |
| VENDORS > Comcast | 7 |
| VENDORS > Bitwarden | 8 |
| Clients > MMC > _SOPHOS REPORTS | 10 (MMC prefix) |
| _WMT > COMPLIANCE | 10 (WMT prefix) |
| Tag only, stays in Inbox | 2 (with "REPLY ABOVE"), 9, 10 (unrecognized prefix) |
| Deleted Items (soft) | 11 |
| Untouched | 12 |

### 5. Create Summary Email and Deliver to Inbox

Runs **last**, after Steps 4, 7, 3.5 and 3. Create the briefing as a message and place it directly in Inbox — created as a draft and then relocated into Inbox, never sent over the network.

**Message content:**
1. **Subject:** "Daily Briefing Summary — [date and time, e.g., Thursday, Aug 6, 2026 12:31 PM]"
2. **To:** alex@westmaintech.com
3. **Body:** the full Step 3 briefing, including the Step 6 **📂 Filing actions** and **🔍 Move verification** sections, ending with the delivery line: "Delivered to inbox by morning briefing at [timestamp]"

**Execution:**
1. **Format the body as HTML** (`bodyType: "html"`) — emoji section headers (📅 Meetings, 🔴 Action needed, 📌 FYI, 🗃️ Low priority, 💬 Teams, ⏳ Long-Term Follow-Up, 🎯 Suggested Focus, 📂 Filing actions, 🔍 Move verification), bullet lists (`<ul><li>`), and bold (`<strong>`) for emphasis. Do not use plain text.
2. Call `outlook_create_draft` with the complete HTML body (this creates the message in Drafts).
3. Immediately call `outlook_modify_labels` on the newly created message ID, with `moveToFolderId` set to the **Inbox** folder ID (`AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwAAAgEMAAAA`). This is a folder move only — no network send, no email leaves the mailbox.
4. **No `outlook_send_mail` call at any point.**
5. Final chat response: "✓ Summary delivered to Inbox at [time]" followed by the Step 6 footer.

### 6. Filing summary footer & audit trail

At the bottom of every briefing, include a **Filing Actions** section with:
- Count of successful tags and moves (by rule)
- Any tagging or move failures (email ID, rule, error detail)
- Emails skipped because they were already tagged Claude Auto
- Prior-run failed moves that were retried
- Delivery line (summary is delivered to Inbox, not left in Drafts)

Example format:

```
---
📂 Filing actions this run:
✓ Tagged 4, moved 3:
  • 1 CUSTOMER REPLY REMINDER (Ticket #47738, 2nd reminder in 7 days, no follow-up) → moved to _SYNCRO ALERTS
  • 2 Syncro ticket replies → moved to _SYNCRO ALERTS
  • 1 Syncro ticket reply (awaiting response) → tagged, left in inbox
↻ Retried 1 prior-run failed move → _TO REVIEW > NEWS
Skipped 5 (already tagged Claude Auto)

⚠️ Issues (1):
  • Tagging failed for email ID [xyz123]: "Rich Stewart new computer" — left untouched, no move attempted

✉️ Delivered to inbox by morning briefing at 12:31 PM
```

If nothing was filed, state: "No filing actions this run."

**Audit trail:** there is no registry file. Each Daily Briefing Summary email in Inbox is the record of what was filed and when; the Claude Auto category prevents duplicate processing; `_SYNCRO ALERTS` provides CUSTOMER REPLY REMINDER history.

### 7. Verify moves completed successfully

**Do not trust the batch API response alone** — a "success" status doesn't always mean the email landed in the destination folder, and for recurring senders (Syncro, CUSTOMER REPLY REMINDER, etc.) subject-line matching is not sufficient verification, since multiple similar-subject emails arrive per day. Verification must match on **exact messageId**.

**Verification workflow:**

1. **Track messageIds through the move, not just subjects** — from the filing results in step 4, build a list of `{ messageId (post-move, if returned by the API), expectedFolder, originalSubject }` for every email that was supposed to move.

2. **Read the destination folder directly by folder ID** — use `read_resource` with `mail:///folders/{folderId}` for each destination folder used this run (re-read `_SYNCRO ALERTS` only if emails were moved there this run). Do NOT rely on `outlook_email_search` for verification — it has shown stale/cached results and folder-name lookups can silently fail. A direct folder read reflects true current state.

3. **Compare by exact messageId, never by subject alone** — check whether each expected messageId (or the new ID returned by the move operation, if the API changes it) appears in the destination folder's message list. Subject-line or sender matching is **not sufficient**.

4. **Also check the source folder (Inbox) directly** — read the Inbox folder by ID the same way. If a messageId that was supposed to move is still present in Inbox, the move failed regardless of what the batch API reported. **Keep this Inbox read — Step 3.5 reuses it.**

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

7. **If any verification fails, do not silently retry with the same batch call** — first check the per-message `categoriesFailed` / `moveFailed` arrays from step 4. If the batch call itself reported failure for that message, retry the specific failed operation once. If the batch call reported success but the email is still in Inbox, treat this as a deeper sync/API issue — report it clearly rather than looping retries, since blind retries could cause duplicate moves. (The next run's Step 4 item 1 will pick it up again.)

**Why this matters:** the CUSTOMER REPLY REMINDER and Syncro Community incidents both involved emails that were tagged but not actually moved, even though the tagging step appeared to succeed. This verification step catches that failure mode before the user finds out the hard way.

## Edge Cases

- **No emails**: "Your inbox is quiet — enjoy it while it lasts."
- **No meetings**: "You have a free day — great for deep work."
- **Lots of unread email (20+)**: Summarize by theme/sender cluster rather than listing each one.
- **Meeting in <30 min**: Prominently call it out at the top with a ⚡ warning.
- **Tool errors**: If one data source fails (e.g., Teams is unavailable), proceed with the others and note what's missing.

## Example Output Shape

```
🌅 Here's your day at a glance.

📅 Today's Meetings
- ⚡ 9:00 AM – Standup with Engineering (30 min) — Zoom
- 2:00 PM – Product Review (1 hr) — Conf Room B

📬 Inbox Highlights
🔴 Action needed:
- [Client] "Follow-up on proposal" — They have questions about pricing
- CUSTOMER REPLY REMINDER — Ticket #47738, 2nd reminder in 7 days, no follow-up

📌 FYI:
- [IT] "Scheduled maintenance Friday 6–8 PM" — Plan accordingly

🗃️ Low priority: 8 newsletters, 3 automated alerts

💬 Teams Highlights
- Design team is asking for feedback on the new mockups (pinged you twice)

⏳ Long-Term Follow-Up
- Axcient "Case #12345" — tagged, in Inbox 3 days

🎯 Suggested Focus
- Reply to the client email before your 11 AM — they seem to be waiting
- Clear ticket #47738

---
📂 Filing actions this run:
✓ Tagged 3, moved 2 (Syncro alerts, news folder)
Skipped 4 (already tagged Claude Auto)

🔍 Move verification:
✓ Verified 2/2 moves

✉️ Delivered to inbox by morning briefing at 12:31 PM
```
