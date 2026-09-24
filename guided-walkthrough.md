# Guided Walkthrough — SOC Investigation (Absolute Beginner)

This guide walks you through the whole investigation one click at a time. If you follow
every step literally, you cannot fail. Do your own report before asking your instructor
for the answer key.

---

## Step 1 — Words you need to know

- **Alert** — a rule in the security tool that fires when something looks dangerous. It is a *hint*, not a *fact*. Investigators confirm or dismiss it.
- **Index / data view** — a named collection of log records with the same shape. We have `auth-*` (sign-in events) and `cloud-*` (Microsoft 365 audit events). In Kibana this is called a **data view** (older versions call it an index pattern).
- **NDJSON** — the text format the log data is stored in: one JSON object per line. Our log files are `auth.ndjson` and `cloud.ndjson`. "NDJSON" just means "Newline-Delimited JSON".
- **Index template** — a configuration that tells Elasticsearch which *type* every field has (that an IP field is an IP, that `event.code` is a number, that the `event.action` / `email.*` fields are exact keywords, etc.) before any data arrives. Our templates are `auth.json` and `cloud.json`. If you skip this step, Elasticsearch guesses and searches can silently break.
- **Elasticsearch** — the database that stores and searches the logs. It answers URLs like `http://localhost:9200`.
- **Kibana** — the web interface on top of Elasticsearch where you will do the investigation. It lives at `http://localhost:5601`.
- **IOC (Indicator of Compromise)** — a piece of evidence that something bad happened, e.g. an IP address, a username, a forwarding address. The address `kpatel.backup@mail-exfil.example` would be an IOC if you find it.
- **External vs internal / foreign IP** — a sign-in from an address your users don't normally come from (here, an external `203.0.113.x` documentation IP) is a **foreign** sign-in and deserves scrutiny.
- **Microsoft 365 (M365) audit log** — a record of actions taken in the cloud email/office tenant: who signed in, who created mailbox rules, who changed forwarding, etc. Each record has an **operation** (like `New-InboxRule`, `Set-Mailbox`) and an **action**.
- **Inbox rule** — an automatic rule in a mailbox (e.g. "move messages containing 'invoice' to a folder"). Attackers create these to **hide** replies so the victim never sees the fraud.
- **External forwarding** — automatically sending copies of incoming mail to another address. Attackers turn this on to keep reading the victim's mail from **outside** the company.
- **Business Email Compromise (BEC)** — an attack where someone takes over a mailbox (usually with a stolen password) to commit fraud (fake invoices/payments) or steal information, often hiding their tracks with inbox rules and forwarding.
- **ECS field** — a standardised name for a piece of data, written with dots, e.g. `user.name` = the account, `source.ip` = where a sign-in came from, `event.code` = the numeric Windows sign-in code, `event.action` = the audited action, `email.rule.name` = the inbox rule's name, `email.forwarding.target` = the forwarding destination.
- **False positive** — the alert fired but nothing bad actually happened.
- **True positive** — the alert fired and something bad actually did happen. Your job is to decide which one this is.

---

## Step 2 — What files you need

Your lab folder contains **exactly six files** — nothing else. Your instructor
pre-generated the log data with a deterministic generator, so you never need to
regenerate or verify anything:

| File | What it is |
|---|---|
| `auth.ndjson` (271 documents) | The sign-in log data: successful/failed logons, logoffs, process starts |
| `cloud.ndjson` (149 documents) | The Microsoft 365 audit data: sign-ins, inbox-rule creation, forwarding changes, file/mail ops |
| `auth.json` | Index template that fixes the field types for the `auth-2026.09.22` index |
| `cloud.json` | Index template that fixes the field types for the `cloud-2026.09.22` index |
| `guided-walkthrough.md` | This guide |
| `instructions.md` | The shorter companion brief |

- **What to do:** keep all six files together in one folder, open a terminal, and go into
  that folder. Every command below is run from there.
- **Type:** `ls`
- **What you should see:** the six files above.
- **Why this matters:** these two `.ndjson` files are the evidence dataset. Nothing in
  Kibana will appear until they are loaded into Elasticsearch (next step).

---

## Step 3 — Load the logs into Elasticsearch

This lab needs a real Elastic Stack (Elasticsearch + Kibana). Offline validation of the
data is done on the instructor's side and is **not** part of your package. Make sure
Elasticsearch is running first:

- **What to do:** in the terminal, from your lab folder, run:
- **Type:** `curl -s localhost:9200/_cluster/health`
- **What you should see:** a JSON line with `"status":"green"` or `"status":"yellow"`.
  If the command fails, start your Elasticsearch service first.
- **Why this matters:** the log data has nowhere to go until Elasticsearch answers.

**A1 — Install the two index templates** (mandatory: it fixes the field types so queries
behave correctly). The templates are in the same folder as the data:

- **Type:**
```bash
curl -s -XPUT localhost:9200/_index_template/auth  -H 'Content-Type: application/json' --data-binary @auth.json
curl -s -XPUT localhost:9200/_index_template/cloud -H 'Content-Type: application/json' --data-binary @cloud.json
```
- **What you should see:** both return `{"acknowledged":true}`.
- **Why this matters:** every field in the log is now typed (IPs as IPs, `event.code` as a
  number, the `event.action` / `email.*` fields as exact keywords, `message` as text).
  Without this, Elasticsearch would guess, and terms like `user.name: "kpatel"` could
  return wrong results.

**A2 — Import the data** (the bulk API needs one "action" line + one data line per
document, so we add the action line on the fly):

- **Type:**
```bash
for s in auth cloud; do
  while read -r l; do
    echo '{"index":{"_index":"'$s'-2026.09.22"}}'
    echo "$l"
  done < $s.ndjson | curl -s -XPOST localhost:9200/_bulk -H 'Content-Type: application/x-ndjson' --data-binary @-
done
```
- **What you should see:** two JSON responses from the bulk API, each ending in `"errors":false`.
- **Why this matters:** this copies every line of each `.ndjson` file into its own index
  (`auth-2026.09.22` and `cloud-2026.09.22`). The patterns you will use in Kibana are
  `auth-*` and `cloud-*`.
- **Verify it worked**, then type:
```bash
curl -s localhost:9200/_cat/indices/auth*,cloud*
```
- **What you should see:** both index names with `271` and `149` documents.
- **Continue with Step 4.**

---

## Step 4 — Create the two data views in Kibana

Kibana will not show any data until you tell it which indices exist.

- **What to do:** open Kibana in your browser.
- **Type in the address bar:** `http://localhost:5601`
- **What you should see:** the Kibana home screen.
- **Type/click:** click the **hamburger menu** (☰) in the top-left corner, then **Stack Management**, then **Data Views**, then the **Create data view** button.
- **Type:** in **Name** type `auth`, in **Index pattern** type `auth-*`, and under **Timestamp field** choose `@timestamp`. Click **Save data view to Kibana**.
- **Repeat** for a second data view: **Name** `cloud`, **Index pattern** `cloud-*`, **Timestamp field** `@timestamp`, save it.
- **What you should see:** two data views in the list, `auth` and `cloud`.
- **Why this matters:** without these, the "Select data view" step later in this guide has nothing to select, and every query would return *"no results"*. Getting this right now avoids hours of confusion later.

---

## Step 5 — Open Kibana

- **What to do:** open your web browser (again, or continue where you are).
- **Type/click:** go to `http://localhost:5601`.
- **What you should see:** the Kibana login / home screen.
- **Why this matters:** Kibana is the tool that lets you search the logs without touching a database.

---

## Step 6 — Open Discover

- **What to do:** open the Discover search screen.
- **Type/click:** click the **hamburger menu** (three horizontal lines ☰) in the top-left corner, then click **Discover** (it lives under "Analytics" in newer versions).
- **What you should see:** a search bar at the top, a field list on the left, and a list of documents below.
- **Why this matters:** Discover is where you will type your queries and read the answers.

---

## Step 7 — Select the auth data view first

- **What to do:** tell Kibana to look at the sign-in logs first.
- **Type/click:** at the top of Discover, click the dropdown that shows the current index. In the list, choose **`auth`** (the data view you created in Step 4).
- **What you should see:** the index name at the top now reads `auth`, and the left sidebar shows fields like `event.code`, `user.name`, `source.ip`.
- **Why this matters:** you will confirm the suspicious sign-in here, then pivot to the cloud audit log for the mailbox changes.

---

## Step 8 — Set the time range

- **What to do:** make sure you are looking at the correct day.
- **Type/click:** click the **time picker** at the top right (it says something like "Last 15 minutes"). Click the **Absolute** tab. Set **From** to `Sep 22, 2026 @ 00:00:00.000` and **To** to `Sep 22, 2026 @ 23:59:59.999`. Click **Apply**.
- **What you should see:** the search field at the top now shows `@timestamp` ranging across the whole of September 22, 2026.
- **Why this matters:** all the relevant events happened on that day, in UTC. A too-narrow time window silently hides them.

---

## Step 9 — First query: the suspicious sign-in

- **What to do:** look at sign-ins for the alerted account.
- **Type:** `user.name: "kpatel" and (event.code: 4624 or event.code: 4625)`
- **What you should see:** a small set of sign-in events for `kpatel`. Add `event.code`, `event.outcome`, and `source.ip` as columns (in the left sidebar, hover over each field and click **⊕ Add**).
- **What you should notice:** a **failed** logon (`event.code: 4625`) then a **successful** logon (`event.code: 4624`) a few minutes later, both from the **same external IP** (an address that is not one of your internal `10.0.4.x` hosts). The failure-then-success from a foreign IP is the account takeover.
- **Why this matters:** `4625` = "an account failed to log on", `4624` = "an account was successfully logged on". This establishes *who* was compromised and *from where*. Write down the user, the source IP, and both timestamps.

---

## Step 10 — Switch to the cloud audit log

- **What to do:** pivot to the Microsoft 365 audit events.
- **Type/click:** click the index dropdown at the top of the page (currently `auth`) and choose **`cloud`**.
- **Type:** `user.name: "kpatel"`
- **What you should see:** the M365 audit records for that mailbox. Add `event.action`, `o365.operation`, and `source.ip` as columns.
- **What you should notice:** among the sign-in records there are two **configuration-change** actions — one creating an inbox rule, one enabling forwarding — from the **same foreign IP** as the sign-in.
- **Why this matters:** the cloud audit log is where mailbox tampering shows up. This is where BEC attackers give themselves away.

---

## Step 11 — Isolate the two malicious mailbox changes

- **What to do:** narrow to just the inbox-rule and forwarding actions on this mailbox.
- **Type:** `(event.action: "mailbox_rule_created" or event.action: "external_forwarding_enabled") and user.name: "kpatel"`
- **What you should see:** exactly the mailbox-tampering events. Read the count at the top right.
- **What to do now:** add `email.rule.name`, `email.rule.condition`, and `email.forwarding.target` as columns.
- **What you should notice:**
  - An inbox rule was created — note its **name** (it looks deliberately boring) and the **condition** it matches (messages about invoices/payments — hiding finance replies).
  - External forwarding was enabled — note the **`email.forwarding.target`**: an address on an **outside** domain, not your company's.
- **Why this matters:** this is the core of the alert, and it is the **same single query** from the companion brief (`instructions.md`) — Steps 9 → 10 → 11 build up to it. A stealth rule plus forwarding to an outside address is textbook BEC. **Write down the rule name, its condition, and the forwarding target address.**

---

## Step 12 — Build the timeline

- **What to do:** open a blank note (paper or text file) and list your findings oldest → newest.
- **Type:** one line per stage with the exact `@timestamp` you found:
  1. (time) — failed logon for `kpatel` from the foreign IP
  2. (time) — successful logon for `kpatel` from the same foreign IP (account takeover)
  3. (time) — inbox rule created (hiding invoice/payment mail)
  4. (time) — external forwarding enabled to the outside address
  5. (time) — logoff
- **What you should see:** a clean sequence — get in, hide replies, forward mail out, leave.
- **Why this matters:** a timeline **is** your evidence of cause and effect. It is also the heart of the report you are about to write.

---

## Step 13 — Write the report

Use this fill-in-the-blank template. Each line has an example sentence.

### Summary
*Fill in:* what happened, which mailbox, verdict.
> Example: "On 22 September 2026 the account 'kpatel' was accessed from a foreign IP after a failed-then-successful sign-in, and the attacker created a hidden inbox rule and enabled external mail forwarding to an outside address. This is assessed as a true positive Business Email Compromise."

### Timeline
*Fill in:* your rows from Step 12.

### Evidence
*Fill in:* each query you ran and what it proved.
> Example: "`user.name: \"kpatel\" and (event.code: 4624 or event.code: 4625)` proved the failed-then-successful sign-in from a foreign IP. `(event.action: \"mailbox_rule_created\" or event.action: \"external_forwarding_enabled\") and user.name: \"kpatel\"` proved the stealth inbox rule and the external forwarding, including the destination address."

### Verdict
*Fill in:* True Positive or False Positive, and why.
> Example: "TRUE POSITIVE — a foreign sign-in on a stolen credential, followed immediately by a hidden inbox rule (to bury invoice/payment replies) and external forwarding to an attacker-controlled address, form a coherent Business Email Compromise chain."

### Recommendations
*Fill in:* three or more concrete actions.
> Example: "Immediately revoke all active sessions for 'kpatel' and reset the password with MFA. Delete the malicious inbox rule and disable the external forwarding. Block the forwarding destination domain and the foreign sign-in IP. Search all mailboxes for similar rules/forwarding and review sent/finance mail for fraud."

When you are done, submit your report. Your instructor will compare your findings
against the answer key.
