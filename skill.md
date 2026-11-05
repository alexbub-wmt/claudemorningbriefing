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
compatibility: "Requires Microsoft 365 connector (Outlook + Teams) and the Syncro connector (list_tickets, get_ticket)"
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
- The briefing also carries **Unifi alerts** (8-hour window, from the `anthropic-skills:unifi-alerts` skill, which owns its own folder location) and a **Syncro ticket list** (new in the last 10 hours, plus every ticket in Customer Reply status regardless of age, pulled from the **Syncro API**, not from mail). Step 7 reconciles the API pull against the `__SYNCRO ALERTS` folder and reports any gap as a footer disclaimer.
- The summary (step 5) is created as a draft and moved to Inbox. It is **never sent** — no `outlook_send_mail` call at any point. The only address used is alex@westmaintech.com.

**Publishing changes to this file:** `git push` and the GitHub MCP tools can be denied (403 "Claude doesn't have GitHub access") in scheduled and cloud sessions. When that happens, push through the **Zapier connector** instead — do not leave commits unpushed:
1. `inspect_zapier_actions` with `selected_api` for GitHub (`GitHubCLIAPI`), then `tool_name: github_create_or_update_file` for the exact params.
2. Repo `alexbub-wmt/claudemorningbriefing`, branch = the session's designated branch (create it first with `github_create_branch` from `main` if it does not exist on the remote).
3. Get the current file SHA with `github_get_file_contents` (needed when updating an existing file), then call `github_create_or_update_file` (`execute_zapier_write_action`) with the **full** file content, path `skill.md`, and a clear commit message.
4. Verify by reading the file back with `github_get_file_contents` and comparing it to the local copy before reporting success. Report the push method used in the run summary.

## Step-by-Step Workflow

### 1. Gather data in parallel (or sequentially if parallel isn't available)

Run all searches simultaneously. Each source is pulled **once** per run and reused by later steps.

**Inbox** — emails from the last 48 hours (feeds both the briefing and Step 4 filing):
```
outlook_email_search(folderName="Inbox", afterDateTime="2 days ago", limit=25)
```
**Paginate until exhausted.** The API caps results at 25 per call, so if the response indicates more results exist (`moreResults: true`, or a `totalResultCount` higher than what you received), call again with `offset` 25, then 50, and so on until every email in the window has been retrieved. **Do not process a partial set** — emails past the first 25 are silently skipped otherwise, which has caused missed filings before. Store full results (sender, recipient, subject, body snippet, messageId, categories).

**_SYNCRO ALERTS** — last 2 days, read directly by folder ID (`read_resource` with `mail:///folders/{folderId}`, paginating if needed). Used for Rule 1 reminder counts and Step 7 verification.

**Unifi alerts** — invoke `anthropic-skills:unifi-alerts` with the Skill tool, window **8 hours**, using the skill's default folder (_TO REVIEW > _ALERTS > CloudUnifi). Collect device offline/reconnect pairs, reboots, firmware updates and security alerts, grouped by customer site. Pair each offline with its matching reconnect so a resolved blip is reported once, as resolved.

**Syncro tickets** — pull from the **Syncro API**, not from mail. Read-only; this files nothing. Make **two** calls and merge the results, de-duplicating by ticket `number`:

1. **New tickets** — `list_tickets` with `created_after` set to **10 hours before the run time**, RFC 3339 with a UTC offset (e.g. `2026-11-02T00:00:00Z`), `per_page: 100`.
2. **Customer replies, any age** — `list_tickets` with `status: "Customer Reply"`, `per_page: 100`. **No date filter.** A customer who replied on a ticket opened months ago is still waiting on us, so age is irrelevant here.

Page each until `pagination.has_more` is false.

The 10-hour window on the first call against an 8-hour run cadence is deliberate: the 2-hour overlap means a ticket opened near a run boundary is never missed. Repeats between runs are expected and are not duplicates to suppress.

**Drop tickets that are already handled.** Before anything else, discard any ticket whose `status` is `REVIEW` or whose status resolves the ticket (`Resolved`, or any status `list_ticket_statuses` returns with `resolves_ticket: true`). That work is finished: those tickets never appear in the ticket list and never appear in Action needed, however recently they were created or updated. Call `list_ticket_statuses` once if you need to confirm which statuses resolve.

For each ticket keep: `number`, `subject`, `status`, `priority`, `created_at`, `updated_at`, `customer_business_then_name`, `problem_type`, `ticket_type_name`, `user_id` (null = unassigned), `customer_reply`, `sla_breached`, `sla_breaching_soon`, `due_date`. Ignore the embedded `comments[].rich_text_preview` — it is large and adds nothing; use `comments[].body` only when you need the substance of an update.

**Flag these three states** and carry the flags into Step 3. They are facts about a ticket's current state, not a list of important subjects:
- **Customer waiting** — `status` is `Customer Reply`, or `customer_reply` is true. The customer has answered and the ball is back with us. **Always goes in Action needed**, whatever the ticket's age.
- **Urgent** — `priority` contains "urgent" or "high" (case-insensitive), or `sla_breached` is true. Note `priority` is often an empty string in this account, so its absence means nothing.
- **Unacknowledged** — `status` is exactly `New` **and** `created_at` is more than **2 hours** before the run time. Nobody has picked it up.

Apart from Customer waiting, a flag is evidence for Step 3's ranking test, not a verdict: an unflagged ticket can still belong in Action needed, and a flagged one can be routine. Say which flag applies when you list the ticket.

If the Unifi skill or the Syncro API is unavailable or returns an error, keep its section in the briefing with a one-line note saying what failed. Never omit the section silently, and never invent entries. For a Syncro API failure specifically, fall back to the mail folder per Step 7 item 8 and label the section as mail-only.

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

Built **after** Steps 4, 7 and 3.5 are complete (see execution order).

Everything gathered this run — inbox, calendar, Teams, Unifi alerts, Syncro tickets — is **one pool**. Rank it first, then list it.

**The ranking test.** An item belongs in 🔴 Action needed if either is true:
- **Waiting makes it worse** — the damage, cost, exposure or disruption grows the longer it sits.
- **Someone is blocked on you** — a person or a job cannot move until you answer or act.

Apply this test to every item, whatever its source. Judge it by what it actually describes, not by which section it arrived in, what words it uses, or whether it came as an email. **Do not keep a list of important categories.** Something you have never seen before can be the most important item of the run, and a familiar-looking item can be routine. If you can state why waiting costs something, it qualifies; if you cannot, it does not.

One standing rule sits on top of the test: **every ticket flagged Customer waiting in Step 1 goes in Action needed**, regardless of age or subject. This is not an exception to the test — a customer who has replied is blocked on you — it is settled in advance so it never has to be re-argued.

Order Action needed worst-first by the same test. An item that qualifies is listed there **once**, and its inventory section cross-references it ("see Action needed") rather than repeating it.

Use this format:

---

## 🌅 Here's your day at a glance.

**🔴 Action needed** *(ranked worst-first, any source)*
One line each: what it is, where it came from (ticket #, sender, site), and what is waiting on you. Say plainly why it is here — what gets worse, or who is blocked.
If nothing qualifies: "Nothing needs action right now."

**📅 Today's Meetings** *(chronological)*
List each meeting with: time, title, attendees (shortened), and location/link if present.
Flag any meetings in the next 1–2 hours as "⚡ Coming up soon."
If no meetings: "You have a clear calendar today."

**📬 Inbox Highlights**
Everything from the inbox that is not already in Action needed:
- 📌 **FYI / notable** — updates, announcements, things to be aware of (include Rule 3 keyword hits and unrecognized Rule 10 prefixes)
- 🗃️ **Low priority** — newsletters, automated notifications (mention count only, don't list)

For each notable email: sender name, subject, and a 1-sentence summary of what it's about.

**💬 Teams Highlights**
Summarize notable messages or threads from the Step 1 Teams search and Rule 11 notification emails — don't list the same message twice. Skip automated bot messages.
If nothing notable: "No urgent Teams messages."

**🌐 Unifi Alerts** *(last 8 hours)*
Full inventory, grouped by customer site. Prefix anything with no resolution ⚠️ and bold the site; prefix resolved items ✅ and give the duration. An offline paired with its reconnect is one resolved line.
If nothing: "No Unifi alerts in the last 8 hours."

**🎫 Open Syncro Tickets** *(new in last 10 hours, plus all Customer Reply)*
Full inventory from the Step 1 API pull, grouped by customer, customer name bolded on its own line, then one bullet per ticket: `#XXXXX — Subject (Type · Time · Status)`. Append any Step 1 flag in brackets — `[customer waiting]`, `[urgent]`, `[new 3h, unassigned]`. Add an indented `↳ Update:` line for any ticket with activity since it was opened.
Tickets in `REVIEW` or a resolving status were dropped in Step 1 and never appear here.
Close with a count line: "N tickets across M customers, F flagged."
If nothing: "No open tickets to report."

**⏳ Long-Term Follow-Up** *(only if items found — see Step 3.5)*
List items still open from the extended 7-day lookback. For each: sender/subject, how many days it's been sitting, and why it's still flagged (e.g., "Axcient Case #12345, tagged, in Inbox 3 days").
If nothing found in the lookback: omit this section entirely (don't show an empty header).

**🎯 Suggested Focus**
2–3 bullets on what to tackle first and in what order, drawn from Action needed. This is for sequencing only — never use it to raise something that should have been in Action needed.

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

1. **Skip already-processed emails** — if an email already has the **Claude Auto** category, skip it and count it as "already processed." **Exception:** an email tagged Claude Auto but still in Inbox that should have been moved (Rules 1, 3–8, 10 MMC/WMT, 11, 12, 13) is a failed move from a prior run — queue its move again without re-tagging, and note the retry in the footer.

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

6. **Vendor marketing & informational newsletters** (sender `marketing@blackpointcyber.com`, OR any sender address at a Spiceworks domain — e.g. `spiceworks@mail.spiceworks.com`, display name "Spiceworks"; any subject):
   - Blackpoint Cyber marketing and Spiceworks informational/promotional mail (webinars, industry newsletters, "AI readiness"-style content, etc.).
   - Tag, move to **_TO REVIEW > NEWS**. No body read needed; match on sender only.
   - Note: `support@blackpointcyber.com` (renewals, billing reports) does NOT match — no rule defined yet. Blackpoint SOC threat notices from `apg@blackpointcyber.com` also do NOT match — those stay in Inbox and are surfaced in the briefing.

7. **Comcast payment notifications** (sender `online.communications@alerts.comcast.net`, subject contains "payment"):
   - Tag, move to **VENDORS > Comcast**.
   - Note: Comcast outage/service notices from `noreply@alerts.comcast.net` and non-payment subjects do NOT match — no rule defined yet.

8. **Bitwarden invoices/receipts** (sender `no-reply@bitwarden.com` OR `invoice+statements@bitwarden.com`, subject contains "invoice" or "receipt"):
   - Tag, move to **VENDORS > Bitwarden**.
   - Note: other Bitwarden emails (login alerts, member confirmation requests) do NOT match — only billing-related subjects.

9. **Axcient case emails** (sender `support@axcient.com`, subject contains "Case #"):
   - Tag only, leave in inbox (manual review needed).
   - Note: Axcient x360Cloud digests are handled by Rule 12; org-attention notices do NOT match — no rule defined yet.

10. **Sophos client daily reports** (sender `wmt.notifications@gmail.com`, subject matches "Events - Daily", "Traffic Dashboard - Daily", or "SSL VPN - Daily"):
    - **This rule requires opening the email** — matching can't be done from subject/sender alone, since the client is identified only by the PDF attachment filename. Call `read_resource` on the email to get the attachment filename before deciding.
    - If the PDF attachment filename **starts with "MMC"** (case-insensitive, any separator or none — e.g. `MMC_AUTH_EVENTS_DAILY.pdf`, `MMC_TRAFFIC_DASHBOARD.pdf`, `MMC SSL VPN.pdf` all match): tag, move to **Clients > MMC > _SOPHOS REPORTS** (folder ID: `AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwAEifQaawAAAA==`).
    - If the PDF attachment filename **starts with "WMT"** (case-insensitive, any separator or none — e.g. `WMT_AUTH_EVENTS.pdf`): tag, move to **_WMT > COMPLIANCE** (folder ID: `AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwABPCPpnQAAAA==`). Note: this is the COMPLIANCE folder nested under _WMT — not the separate top-level COMPLIANCE folder that also exists in the mailbox.
    - If the PDF attachment filename starts with a **different, unrecognized** prefix (any client other than "MMC" or "WMT"): do NOT move. Tag Claude Auto, leave in inbox, and surface in the briefing as an unrecognized client prefix that needs a new rule — do not guess a destination folder for it.
    - Do not match on subject or recipient alone — always confirm via the actual attachment filename, since these reports are otherwise identical across clients.
    - Note: this is distinct from `do-not-reply@central.sophos.com` "[HIGH] Alert for Sophos Central" real-time alert emails — those are a separate, still-undefined case and fall through to Rule 14.

11. **Teams message notifications** (sender `no-reply@teams.mail.microsoft`):
    - Read and analyze content FIRST; include relevant details in the briefing (💬 Teams Highlights section).
    - Then tag Claude Auto and move to Deleted Items via `outlook_trash_thread` (soft delete — recoverable, not permanent). Never trash before the content has been captured for the briefing.

12. **Axcient x360Cloud summary digests — CFOR** (sender `alerts@westmaintech.com` — display name "WMT ALERTS" — subject contains "x360Cloud Summary Digest", e.g. `FW: cforestart: x360Cloud Summary Digest`):
    - Tag, move to **_TO REVIEW > _ALERTS > Axcient > CFOR** (folder ID: `AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwAEifQaGQAAAA==`). No body read needed; match on sender + subject only.
    - Note: digests from other Axcient customers do NOT match this rule (mallickplumbing is Rule 13; others have no rule yet).
    - Note: `support@axcient.com` "Case #" emails still fall under Rule 9 (tag only). Other `alerts@westmaintech.com` mail (e.g. Unifi alerts) does NOT match.

13. **Axcient x360Cloud summary digests — MMC (mallickplumbing)** (sender `no-reply@axcient.com` — display name "x360Cloud" — subject contains "mallickplumbing: x360Cloud Summary Digest"):
    - Tag, move to **_TO REVIEW > _ALERTS > Axcient > MMC** (folder ID: `AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwABhIAATgAAAA==`). No body read needed; match on sender + subject only.
    - Note: digests from other Axcient customers do NOT match — no rule defined yet.

14. **Any other sender** → do not touch (no rule defined; surface in briefing and confirm with user before adding a rule).

**Quick reference — destinations:**

| Destination | Rules |
|---|---|
| _SYNCRO ALERTS | 1, 2 (no "REPLY ABOVE"), 4 |
| _TO REVIEW > NEWS | 3, 5, 6 |
| _TO REVIEW > _ALERTS > Axcient > CFOR | 12 |
| _TO REVIEW > _ALERTS > Axcient > MMC | 13 |
| VENDORS > Comcast | 7 |
| VENDORS > Bitwarden | 8 |
| Clients > MMC > _SOPHOS REPORTS | 10 (MMC prefix) |
| _WMT > COMPLIANCE | 10 (WMT prefix) |
| Tag only, stays in Inbox | 2 (with "REPLY ABOVE"), 9, 10 (unrecognized prefix) |
| Deleted Items (soft) | 11 |
| Untouched | 14 |

### 5. Create Summary Email and Deliver to Inbox

Runs **last**, after Steps 4, 7, 3.5 and 3. Create the briefing as a message and place it directly in Inbox — created as a draft and then relocated into Inbox, never sent over the network.

**Message content:**
1. **Subject:** "Daily Briefing Summary — [date and time, e.g., Thursday, Aug 6, 2026 12:31 PM]"
2. **To:** alex@westmaintech.com
3. **Body:** the full Step 3 briefing, including the Step 6 **📂 Filing actions** and **🔍 Move verification** sections, ending with the delivery line: "Delivered to inbox by morning briefing at [timestamp]"

**Execution:**
1. **Format the body as HTML** (`bodyType: "html"`) — emoji section headers (🔴 Action needed, 📅 Meetings, 📌 FYI, 🗃️ Low priority, 💬 Teams, 🌐 Unifi Alerts, 🎫 Open Syncro Tickets, ⏳ Long-Term Follow-Up, 🎯 Suggested Focus, 📂 Filing actions, 🔍 Move verification), bullet lists (`<ul><li>`), and bold (`<strong>`) for emphasis. Do not use plain text.
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
- Syncro reconciliation result (per Step 7 item 8) — a disclaimer about completeness, nothing more
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

🔁 Syncro reconciliation (disclaimer only — not reflected above):
   API returned 7 tickets, __SYNCRO ALERTS held 8 for the same window.
   ⚠️ #48861 "Printer offline at front desk" was in mail but not in the API — check by hand.
   1 API ticket had no notification email (normal).
```

7. **If any verification fails, do not silently retry with the same batch call** — first check the per-message `categoriesFailed` / `moveFailed` arrays from step 4. If the batch call itself reported failure for that message, retry the specific failed operation once. If the batch call reported success but the email is still in Inbox, treat this as a deeper sync/API issue — report it clearly rather than looping retries, since blind retries could cause duplicate moves. (The next run's Step 4 item 1 will pick it up again.)

8. **Reconcile the Syncro API pull against the mail folder.** The ticket list in Step 1 comes from the API; the `__SYNCRO ALERTS` folder is the independent second record. Check one against the other every run:
   - Read `__SYNCRO ALERTS` by folder ID (`AQMkADk0OTc2MzVkLWUyM2QtNDkwMi1hZWY0LWQ0M2JmNTA2NWQwZQAuAAADBgrMhQbA1UCWYZzkkEmHugEAAnJtQHWQ1kiJdEjgrI_4cwAEifQZqgAAAA==` — the real folder name has **two** leading underscores) and take every "A Ticket was created" email received inside the same 10-hour window. Pull the ticket number from the body's `Number:` line.
   - Compare that set of numbers against the numbers from the API pull.
   - **This check is a disclaimer, not a content source.** Discrepancies belong in the footer only. Do **not** add recovered ticket numbers to the ticket list, to Action needed, or to any other section of the briefing. The body of the email reflects the API pull; the footer says whether that pull looks complete.
   - **In mail but not in the API** → list those ticket numbers and subjects in the footer so the gap is visible and can be checked by hand. Exclude any whose status is `REVIEW` or resolving — those are finished work and are not a gap.
   - **In the API but not in mail** → report the count only. This is normal (a server-side rule may not have filed the notification) and is not an error, but a large or growing gap means the mail rule is failing and is worth saying out loud.
   - **If the two sides match**, say so in one line and move on.
   - If the API call itself failed earlier in the run, that is the one case where mail becomes the source: say so plainly, build the ticket section from the mail folder, and label the section mail-only. Never present a mail-only list as if the API had been read.

**Why this matters:** the CUSTOMER REPLY REMINDER and Syncro Community incidents both involved emails that were tagged but not actually moved, even though the tagging step appeared to succeed. This verification step catches that failure mode before the user finds out the hard way. The Step 8 reconciliation exists for the same reason in the other direction: two independent records of the same tickets, each able to catch what the other drops.

## Edge Cases

- **No emails**: "Your inbox is quiet — enjoy it while it lasts."
- **No meetings**: "You have a free day — great for deep work."
- **Lots of unread email (20+)**: Summarize by theme/sender cluster rather than listing each one.
- **Meeting in <30 min**: Prominently call it out at the top with a ⚡ warning.
- **Tool errors**: If one data source fails (e.g., Teams is unavailable), proceed with the others and note what's missing.

## Example Output Shape

```
🌅 Here's your day at a glance.

🔴 Action needed
- #48370 MMI — customer replied 9:52 AM, waiting on us
- #48199 MPH — customer replied on a ticket open since Oct 14, still waiting
- #48848 MMC — laptop stolen last night; exposure grows until it is locked or wiped
- MMC-Kenny Home — WiFi impersonation alert 8:27 AM, still live
- CUSTOMER REPLY REMINDER #47738 — 2nd reminder in 7 days, customer waiting on us
- [Client] "Follow-up on proposal" — they are waiting on pricing answers

📅 Today's Meetings
- ⚡ 9:00 AM – Standup with Engineering (30 min) — Zoom
- 2:00 PM – Product Review (1 hr) — Conf Room B

📬 Inbox Highlights
📌 FYI:
- [IT] "Scheduled maintenance Friday 6–8 PM" — Plan accordingly

🗃️ Low priority: 8 newsletters, 3 automated alerts

💬 Teams Highlights
- Design team is asking for feedback on the new mockups (pinged you twice)

🌐 Unifi Alerts (last 8 hours)
⚠️ MMC-Kenny Home — WiFi Impersonation Detected 8:27 AM — see Action needed
✅ B&R - GB — device offline 5:16 AM → reconnected 5:18 AM (2 min)

🎫 Open Syncro Tickets (new in last 10 hours, plus all Customer Reply)
MMC - Mallick Mechanical Contractors
- #48848 — Jenn's Laptop (Other · 10:00 AM · New) [new 3h, unassigned] — see Action needed
Mid-Maryland Musculoskeletal Institute
- #48364 — Corinna and Shelly cannot get into PCs after power outage (Remote Support · 9:31 AM · In Progress)
- #48370 — FW: Computer in UC needs printers installed (Other · 9:52 AM · Customer Reply) [customer waiting] — see Action needed
MPH - Mallick Plumbing & Heating
- #48199 — Website CMS file control (Other · opened Oct 14 · Customer Reply) [customer waiting] — see Action needed
- #48371 — Ian Andrews - Password Reset (Other · 11:50 AM · In Progress)
5 tickets across 3 customers, 3 flagged

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
🔁 Syncro reconciliation: API 4, mail 4 — no gaps (disclaimer only)

✉️ Delivered to inbox by morning briefing at 12:31 PM
```
