# SOC Investigation Brief — Alert: Foreign sign-in to the email tenant, then a stealth inbox rule and external forwarding

**Scenario ID:** bec-mailbox-rule-001
**Alert name:** Suspicious mailbox activity — possible Business Email Compromise (BEC)
**Severity:** High (escalate on confirmation)

---

## 1. Scenario briefing

Your organisation uses Microsoft 365 for corporate email. Sign-in and audit telemetry is
shipped into Elastic, and a detection rule has fired:

> A user account signed in to the email tenant from an **unfamiliar foreign IP**, and
> shortly afterwards the mailbox had an **inbox rule created** and **external mail
> forwarding enabled**.

This is the classic shape of **Business Email Compromise (BEC)**. An attacker who has
stolen a user's password logs into their mailbox and then:

- **Creates a hidden inbox rule** — often named to look boring (e.g. "Move to
  Conversation History") — that quietly files or hides replies (for example messages
  about *invoices* or *payments*) so the real user never notices the fraud.
- **Turns on external forwarding** — silently copying all incoming mail to an
  attacker-controlled address, so they keep reading the victim's email even after the
  session ends.

BEC leads to invoice fraud and data theft, and the mailbox rules make it stealthy. Your
job is to confirm the suspicious sign-in, find the inbox rule and the forwarding change,
identify the forwarding destination, build a timeline, and write an incident report.

**Do not read the solution file.** Investigate as if this were a live incident.

---

## 2. Setup — required files and data import

This lab needs a real Elastic Stack (Elasticsearch + Kibana). Your lab folder contains
exactly six files; run every command inside that folder:

| File | Purpose |
|---|---|
| `auth.ndjson` | authentication events (271 docs) |
| `cloud.ndjson` | Microsoft 365 audit events (149 docs) |
| `auth.json` | index template — field types for the `auth-2026.09.22` index |
| `cloud.json` | index template — field types for the `cloud-2026.09.22` index |
| `guided-walkthrough.md` | the step-by-step guide |
| `instructions.md` | this brief |

**Step 1 — install the index templates** against a running Elasticsearch so fields are
typed correctly (`event.code` as a number, IP fields as IPs, the `event.action` /
`email.*` / `o365.*` fields as exact keywords, `message` as text — never let
Elasticsearch guess):

```bash
curl -s -XPUT localhost:9200/_index_template/auth  -H 'Content-Type: application/json' --data-binary @auth.json
curl -s -XPUT localhost:9200/_index_template/cloud -H 'Content-Type: application/json' --data-binary @cloud.json
```

**Step 2 — import the NDJSON.** Elasticsearch's bulk API needs one action line before
each document line, so add it on the fly:

```bash
for s in auth cloud; do
  while read -r l; do
    echo '{"index":{"_index":"'$s'-2026.09.22"}}'
    echo "$l"
  done < $s.ndjson | curl -s -XPOST localhost:9200/_bulk -H 'Content-Type: application/x-ndjson' --data-binary @-
done
```

Verify with `curl -s localhost:9200/_cat/indices/auth*,cloud*` — expect 271 and 149 docs.

**Step 3 — create the data views** in Kibana: **Stack Management → Data Views → Create
data view**, name `auth` with pattern `auth-*` and `cloud` with pattern `cloud-*`,
timestamp field `@timestamp`.

---

## 3. What to query

Two data views (index patterns) are available:

| Data view | Content |
|---|---|
| `auth-*` | Windows security / sign-in events: successful logons (4624), failed logons (4625), logoffs (4634), privilege assignment (4672), process starts (4688), lockouts (4740), Kerberos failures (4771), with `user.name` and `source.ip` |
| `cloud-*` | Microsoft 365 audit events: `event.action` (e.g. `user_login`, `mailbox_rule_created`, `external_forwarding_enabled`), `o365.operation` (e.g. `New-InboxRule`, `Set-Mailbox`), `email.rule.name`, `email.rule.condition`, `email.forwarding.target`, `user.name`, `source.ip` |

---

## 4. The investigation query

Run this in **Discover** against the **`cloud-*`** data view, with the time range set to
**2026-09-22 00:00:00.000 → 2026-09-22 23:59:59.999 (UTC)**:

```kql
(event.action: "mailbox_rule_created" or event.action: "external_forwarding_enabled") and user.name: "kpatel"
```

- `event.action: "mailbox_rule_created"` — an inbox rule was created
- `event.action: "external_forwarding_enabled"` — external mail forwarding was turned on
- `user.name: "kpatel"` — the mailbox named in the alert

This query isolates the two malicious mailbox-configuration changes on the flagged
account. Add the `o365.operation`, `email.rule.name`, and `email.forwarding.target`
columns: you should see the inbox rule's name and the address that mail is being
forwarded to.

---

## 5. Expected shape of the answer

Your findings should take a specific shape. Do **not** expect a number here — you will
derive it — but the pattern should be:

1. In `auth-*`, a **failed logon** (`event.code: 4625`) then a **successful logon**
   (`event.code: 4624`) for `kpatel` from the **same foreign IP** within a few minutes.
2. In `cloud-*`, an **inbox rule created** (`event.action: "mailbox_rule_created"`) on
   that mailbox — note its `email.rule.name` and the condition it matches on.
3. In `cloud-*`, **external forwarding enabled**
   (`event.action: "external_forwarding_enabled"`) — note the
   `email.forwarding.target` (the destination address).
4. All of the above by the **same user from the same foreign IP**, in a short window,
   ending with a logoff.

**What a suspicious answer looks like:** a foreign sign-in immediately followed by a
hidden inbox rule (hiding invoice/payment mail) and forwarding to an **external,
non-corporate** address. A **benign** pattern looks different — sign-ins from expected
locations, no new inbox rules, and no forwarding to outside domains.

---

## 6. Report template

Write your report using exactly these sections.

### Summary
*(2–4 sentences: what happened, which mailbox, one-line verdict.)*

### Timeline
*(List each stage with the timestamp you found, oldest first.)*

| Time (UTC) | Event |
|---|---|
|  |  |
|  |  |

### Evidence
*(The queries you ran and the concrete facts each one proved.)*

- KQL query / observation → what it proved

### Verdict
*(State clearly: True Positive / False Positive, and the attack chain in one paragraph.)*

### Recommendations
*(Concrete, short, actionable remediations, e.g. revoke sessions, remove rule, disable forwarding.)*

---
Once complete, submit your report; your instructor compares it against the answer key.
