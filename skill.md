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
- This skill is best run with **Sonnet 5** for accuracy and reliability in pattern-matching complex email rules (sender/recipient/subject/body matching across the full rule set).
- The filing workflow (step 4) is **email-first, rule-ordered**: pull the inbox (paginating until exhausted), then process each email through every defined rule in order, apply the first match, and move to the next email. This ensures every email is checked against every rule systematically.
- Rules are ordered **most specific first** so a broad rule never shadows a narrow one (e.g. CUSTOMER REPLY REMINDER precedes the general Syncro ticket rule, since both come from `support@westmaintech.com`).
- **Processing guarantee**: batch tag and move operations atomically; every email gets evaluated; no rule is skipped.
- Maintain a filing registry (persistent across runs) to prevent duplicate processing and enable tracking of reminder fires.
- Always include the filing summary footer so the user can verify what was processed and spot any errors. Report counts by rule.
- After the same-day briefing, run the extended 7-day lookback (Step 3.5) to surface unresolved items that scrolled out of the daily view — this is read-only and never triggers new filing actions.
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

**⏳ Long-Term Follow-Up** *(only if items found — see Step 3.5)*
List items still open from the extended 7-day lookback that haven't been resolved. For each: sender/subject, how many days it's been sitting, and why it's still flagged (e.g., "3rd CUSTOMER REPLY REMINDER, ticket #47758, unresolved since Aug 8").
If nothing found in the lookback: omit this section entirely (don't show an empty header).

**🎯 Suggested Focus**
Based on what you've seen, offer 2–3 bullet points on what the user might want to tackle first. Keep it practical, not generic.

---

### 3.5 Extended lookback (up to 7 days)

After the same-day briefing (Step 3) is built, run a second, separate check to catch items that scrolled out of the 24-hour window but are still unresolved.

**Scope:** search Inbox and relevant filing-destination folders (e.g., _SYNCRO ALERTS) for items received in the **last 7 days** (not just today).

**What counts as "still flagged for follow-up":**
- A CUSTOMER REPLY REMINDER (or similar recurring alert) for a ticket that has fired multiple times across the 7-day window without an apparent resolution (check registry for repeat fires on the same ticket #).
- Any email tagged "Claude Auto" and left in Inbox (per rules that say "tag only, leave in inbox") that is still present and unactioned after more than 1 day.
- Any item explicitly called out as "action needed" in a prior day's briefing that is still present in Inbox today.

**What does NOT count:**
- Items already filed/moved out of Inbox as part of normal rule processing — those are handled, not "open."
- Low-priority/newsletter-type items — this section is for things that need a decision or reply, not backlog volume.
- Anything already surfaced in today's same-day 🔴 Action needed section — don't duplicate; only show items *older* than today that are still lingering.

**Output:** populate the **⏳ Long-Term Follow-Up** section in the briefing (Step 3 format above) with what's found. If nothing qualifies, omit the section — don't force it to appear empty.

**Note:** this lookback is read-only — it does not trigger new filing actions. If an old item matches a filing rule and simply wasn't processed in a prior run (a genuine miss), flag it in this section AND note it as needing manual reprocessing, but don't auto-file it silently as part of the lookback — that keeps this step from becoming a second uncontrolled filing pass.

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

1. **Pull inbox — paginate until exhausted.** Search inbox from midnight today, limit 25, newest first. The API caps results at 25 per call, so if the response indicates more results exist (`moreResults: true`, or a `totalResultCount` higher than what you received), call again with `offset` 25, then 50, and so on until every email in the window has been retrieved. **Do not process a partial set** — emails past the first 25 are silently skipped otherwise, which has caused missed filings before. Store full results (sender, recipient, subject, body snippet, messageId).

2. **Process each email in order** — for each email in the results:
   - Check **every defined rule** in sequence (priority order below), starting at rule 1 and continuing until a match is found or the rule list is exhausted
   - On **first rule match**, immediately apply that rule's action (tag ± move)
   - Log the result: `{ messageId, rule matched, action taken, timestamp }`
   - Move to next email
   - If no rule matches, leave untouched

3. **Batch tag and move operations** — collect all tag operations and all move operations (grouped by destination folder), then execute via atomic `outlook_batch_modify_labels` calls. Capture success/failure for each batch.

4. **Update filing registry** — for each successfully filed email, append to `/filing-registry.md`: messageId, timestamp, folder destination (if moved), rule name matched, ticket # (if applicable).

5. **Report results** — include filing summary by rule (counts and specific emails per rule) and any errors in the briefing footer (see step 6, below).

**Error handling:** If tagging fails for an email, do not attempt to move it. Log as error and skip to next email. If a move fails after successful tagging, log that too (email is already tagged/flagged even if move didn't complete).

**Processing guarantee:** Every email in the inbox — across all pagination pages — is checked against every defined rule in order until a match is found. No email is skipped; no rule is bypassed. When rules are added or removed, this guarantee still means "all of them," so do not rely on a hardcoded rule count anywhere in execution.

**Defined rules (auto-executed, in priority order — most specific first):**

1. **CUSTOMER REPLY REMINDER alerts** (sender `support@westmaintech.com`, subject contains "CUSTOMER REPLY REMINDER"):
   - Scan body for ticket #, customer/contact name, reply content.
   - Check filing registry: count how many times this ticket's reminder has fired in last 7 days; surface count + details in briefing as flagged item.
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
    - Then tag Claude Auto and move to Deleted Items via `outlook_trash_thread` (soft delete — recoverable, not permanent). Never trash before the content has been read and reflected in the briefing.

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

After filing is complete, create the briefing summary as a message and place it directly in your Inbox — not sent over the network, just created as a draft and then relocated into Inbox. This keeps the summary out of Drafts clutter and avoids any actual email send.

**Message content:**
1. **Subject:** "Daily Briefing Summary — [date, e.g., Thursday, Aug 6, 2026]"
2. **To:** alex@westmaintech.com
3. **Body:** Must include BOTH of the following sections, in this order — the filing footer is not optional and not chat-only, it must be part of the email body itself:
   - **Briefing content:**
     - Today's meetings (times, titles, key attendees)
     - Action items from email (top 3–5 from 🔴 section)
     - FYI highlights (top 2–3 from 📌 section)
     - Any flagged Syncro reminders (CUSTOMER REPLY REMINDER with ticket counts)
     - Long-term follow-up items from the 7-day lookback (Step 3.5), if any were found
     - Suggested focus areas (2–3 from the briefing)
   - **Filing Actions footer** (per Step 6 format below) — count of tags/moves by rule, any errors, registry confirmation, move verification results (per Step 7)
   - **Delivery confirmation line:** "Delivered to inbox by morning briefing at [timestamp]"

**Execution:**
1. Build the full body — briefing + filing footer + verification results + delivery timestamp — as ONE combined message before calling any tool. Do not create the message first and add the footer later; the footer's content (filing/verification results) is only known after Steps 4 and 7 run, so Step 5 must execute after Step 7, not before it.
2. **Format the body as HTML** (`bodyType: "html"`), matching the same visual structure as the chat briefing — emoji section headers (🔴 Action needed, 📌 FYI, 🗃️ Low priority, 💬 Teams, 🎯 Suggested Focus, 📂 Filing actions, 🔍 Move verification), bullet lists (`<ul><li>`), and bold (`<strong>`) for emphasis. Do not send as plain text — plain text strips the section structure and makes the email harder to scan than the chat version.
3. Call `outlook_create_draft` with the combined HTML body (this creates the message in Drafts).
4. Immediately call `outlook_modify_labels` on the newly created message ID, with `moveToFolderId` set to the **Inbox** folder ID (`AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwAAAgEMAAAA`). This is a folder move only — no network send, no email leaves the mailbox.
5. **No `outlook_send_mail` call at any point.** This step never sends email — it only creates and relocates a message within the mailbox.
6. Confirm in the chat response that the summary (including footer) was delivered to Inbox.

**Note on step order:** Because the filing footer must be part of the email body, Step 5 (create + deliver message) must run AFTER Step 4 (filing) and Step 7 (verification) are complete — not concurrently, and not before. The chat-visible briefing can still be presented early for the user's immediate reading, but the emailed copy is only created once all filing/verification data exists to populate its footer.

**Footer note:** Include a timestamp ("Delivered to inbox by morning briefing at [time]") so you know when it was generated.

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
