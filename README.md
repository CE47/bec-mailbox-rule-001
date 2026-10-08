# CTIA Scenario Lab — bec-mailbox-rule-001

A self-contained SOC analyst training scenario. Investigate it in Kibana using the
step-by-step guide, then write up and submit your findings.

## The scenario

You have been handed an alert about someone's mailbox and told to work out what
happened:

> **Suspicious mailbox activity — possible Business Email Compromise.**
> A user account signed in to the corporate email tenant from an unfamiliar foreign IP,
> and shortly afterwards that mailbox had an inbox rule created and external mail
> forwarding enabled.

Your job is to decide one thing: **is this a real attack that must be escalated and
cleaned up, or a false alarm that can be closed?** You answer it by following the
evidence from the first search to the last and writing down what you find as you go.

Two synthetic, deterministic log sources are loaded into Elasticsearch:

| Data view | Documents | What it covers |
|---|---|---|
| `auth-*` | 271 | Windows security-style authentication events: sign-ins, logoffs, failures, lockouts, privilege grants, program starts |
| `cloud-*` | 149 | Microsoft 365 audit: sign-ins, mailbox opens, sent mail, and the mailbox configuration changes |

The whole incident occupies **twelve minutes** on **2026-09-22**, inside a day of
ordinary tenant traffic. Two of the day's sign-in records belong to the attacker's
address — and a lookalike address one digit away, belonging to five ordinary
accounts, sits close enough that a report built on the wrong address still looks
entirely plausible.

Event times are stored in UTC and Kibana displays them in your own timezone — read
the **gaps** between events, not the absolute clock times.

---

## How to run the lab

### 1. Clone the repository

```bash
git clone https://github.com/CE47/bec-mailbox-rule-001.git
cd bec-mailbox-rule-001
```

### 2a. Windows — set up Docker Desktop

1. Install **WSL** (Windows Subsystem for Linux). In an elevated PowerShell window:

   ```powershell
   wsl --install
   ```

   Restart Windows when it asks, then verify with `wsl --status`.

2. Install **Docker Desktop for Windows** from <https://www.docker.com/products/docker-desktop/>.
   Let it finish installing the WSL 2 backend updates, then launch Docker Desktop and
   wait until the whale icon reports **Engine running**.

### 2b. Linux / macOS — set up Docker

Install **Docker Desktop** (<https://www.docker.com/products/docker-desktop/>) or
Docker Engine with the Compose plugin, make sure it is running, and skip to step 3.

### 3. Build the lab

Double-click **`setup-lab.cmd`** (Windows), or run `./setup-lab.sh` in a terminal in
this folder (Linux / macOS; `bash setup-lab.sh` always works).

> Do not close the command window. The first run downloads over a gigabyte of
> Elasticsearch and Kibana images — allow **5 to 20 minutes**. Later runs take seconds.

The script starts Elasticsearch and Kibana in Docker, installs the two index
templates, imports both datasets, creates the `auth-*` and `cloud-*` data views, and
installs the detection rule — then waits until the alerts it produces are really
there. It finishes by reading **every number the guide quotes** back out of the live
index and refuses to declare itself finished if any of them disagrees. Then it prints:

```
The lab is ready.
Elasticsearch : http://127.0.0.1:9200
Kibana        : http://127.0.0.1:5601
Kibana login  lab_kibana / LabKibana001
```

Your browser opens by itself on **`Security → Alerts`**, which is where the
investigation starts. Sign in with the lab credentials if prompted — they are
throwaway credentials and protect nothing.

### 4. Perform the investigation

Open **`guided-walkthrough.html`** in any browser (no server and no internet
required) and work through it beside Kibana. It is a 19-card deck that takes a
complete beginner from an empty screen to a signed-off verdict. Every picture in it
is a **screenshot of this exact lab**, and after every click it tells you the exact
number or text you should now be seeing.

The walk teaches the three things that actually trip people up in a real BEC
investigation: that a big number can belong to nobody, that an address which *looks*
like the attacker's can belong to five ordinary colleagues, and that the same words
typed in the wrong query language return a different answer with no error message.

The setup script also prints the six investigation queries and the number of results
each one must return. That list is your reference sheet, and it is checked against
the live index on every run.

> **No answer key is printed anywhere in the guide.** Nothing is filled in for you and
> there is no "check your answers" card. If you cannot answer a question without
> opening Kibana again, you have not finished the investigation.

### 5. Write it up and submit

The **`Write it up`** card in `guided-walkthrough.html` is the deliverable — the only
place in the guide you type. Fill in your **name**, **batch number** and **verdict**
(true positive or false positive), then the six report sections: Summary, Timeline,
Evidence, Impact, Verdict, Recommendations.

When it is done, press **`EXPORT MY SUBMISSION`**.

A PDF of your typed report is built in your browser and saved to your **Downloads**
folder — nothing is uploaded and no internet connection is needed. **That PDF is your
submission for this activity.**

The card ends with a seven-point checklist; work through it before you hand the file
in.

### 6. Clean up

When you have exported and submitted your report:

- **Windows:** double-click `teardown.cmd`
- **Linux / macOS:** run `./teardown.sh` in this folder

The teardown prints the exact list of what it will remove and asks you to confirm
(`Y`/`N` on Windows, `y`/`n` on macOS and Linux). Every name in that list is derived
from this folder, so **no other container, volume or lab on your machine is touched**,
no Docker image is removed, and your own files stay in the folder. Running it twice is
harmless.

To stop the lab but keep the data, run `docker compose -f docker-compose.yml down`
and bring it back later with `docker compose -f docker-compose.yml up -d`.

---

## Files in this repository

| File | Description |
|---|---|
| `README.md` | this file |
| `guided-walkthrough.html` | **the investigation guide and the write-up / submission form — open this in a browser** |
| `auth.ndjson` / `auth.json` | auth events (271 docs) and the `auth-*` index template |
| `cloud.ndjson` / `cloud.json` | cloud events (149 docs) and the `cloud-*` index template |
| `rule.json` | the detection rule the lab installs, so the Alerts page is not empty |
| `setup-lab.cmd` / `setup-lab.sh` | **Windows / Linux-macOS.** One-click lab build, fully verified |
| `teardown.cmd` / `teardown.sh` | **Windows / Linux-macOS.** Safe, scenario-scoped removal of exactly what setup created |

`docker-compose.yml` is generated by the setup script; delete it and re-run setup to
rebuild it.

Running setup twice is safe — it skips work already done and will not import the data
twice. `setup-lab.sh reset` (or `setup-lab.cmd reset`) rebuilds the data from scratch
if you ever need a clean slate.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `docker was not found` | Docker is not installed. Install it and reopen the terminal |
| `Docker is installed but the engine is not responding` | Open Docker Desktop and wait until it says **Engine running**, then run setup again |
| Setup fails with a container error | `docker logs bec-mailbox-rule-001-elasticsearch` or `docker logs bec-mailbox-rule-001-kibana` |
| Another lab owns port 9200 or 5601 | Only one lab runs at a time. Run that lab's teardown from its own folder — this script will not touch it |
| Discover shows no data | Press **Search entire time range** — the default of "Last 15 minutes" excludes the data |
| A query returns 0 unexpectedly | Check the **data view**: the sign-in queries need `auth-*`, the mailbox queries need `cloud-*` |
| Query 5 returns 4 instead of 2 | You are in **Lucene**. Switch the query language back to **KQL** |
| The Alerts page is empty | An alert is stamped with the time the *rule ran*, so set that page's time picker to **Last 24 hours** |
| Every field starts with `kibana.alert.` | You are in Kibana's Default security data view. Switch to `auth-*` |
| The sidebar says `0 available fields` | Still a time-range problem. Press **Search entire time range** |
| Every number is double | The data was imported twice. Run the setup file with `reset` |