# Architecture Details

## Pipeline Overview

The automation pipeline follows an event-driven, modular design where a
single orchestrator coordinates specialized sub-workflows. Each sub-workflow
is independently deployable and communicates through Shuffle's sub-workflow
execution mechanism.

Three principles guide the design:

- Single responsibility — one workflow, one job
- Loose coupling — sub-workflows receive self-contained payloads and return
  self-contained results
- Parallel execution — independent operations run concurrently to minimize
  total processing time

---

## Detailed Data Flow

### Phase 1 — Ingestion

Wazuh generates an alert based on its detection rules and sends it as a
JSON payload via webhook to the Shuffle orchestrator (SIEM_to_ticket).
The webhook captures the full Wazuh alert structure: rule metadata (ID,
level, description, MITRE ATT&CK mapping), agent information (name, IP,
OS), and the raw log that triggered the rule.

### Phase 2 — Parsing and Normalization

The orchestrator immediately delegates to Parse_and_Normalize_Observables.

IOC Extraction applies Shuffle's built-in parser to identify observables
embedded in the raw alert payload: IP addresses, file hashes (MD5/SHA1/
SHA256), URLs, and domain names.

Normalization restructures the extracted IOCs into a typed schema:

```json
{
  "ip":     { "values": ["203.0.113.42"] },
  "hash":   { "values": ["d41d8cd98f00b204e9800998ecf8427e"] },
  "url":    { "values": ["https://example.com/payload"] },
  "domain": { "values": ["malicious.example.com"] }
}
```

This schema is the interface contract between the parsing layer and all
downstream enrichment workflows. Adding a new observable type requires
updating only the normalization action.

### Phase 3 — Enrichment (Parallel)

Two enrichment workflows execute simultaneously, each covering a different
intelligence scope.

**Cortex Enrichment — External APIs**

| API | Observable types | Data returned |
|-----|-----------------|---------------|
| VirusTotal | IP, Hash, URL, Domain | Detection ratio, scan results, reputation |
| AbuseIPDB | IP | Confidence score, ISP, report count |
| MalwareBazaar | Hash | Malware family, tags, first/last seen |
| URLScan.io | URL, Domain | Verdict, DOM analysis |

Each API is called through Shuffle's HTTP action. A sleep action manages
rate limiting for asynchronous analyzers (URLScan). All responses are merged
and normalized into a uniform report via a custom Python action.

**OpenCTI Enrichment — Internal TIP**

| Query | Observable type | Data returned |
|-------|----------------|---------------|
| Run_IP_Search | IP | Matching indicators, labels, confidence |
| Run_Hash_Search | Hash | Threat actor, campaign, marking |
| Run_URL_Search | URL | Indicators, creation date |
| Run_Domain_Search | Domain | Indicators, marking definitions |

Each query uses the OpenCTI GraphQL API with filterGroups for precise
indicator lookups by name. Results are merged and normalized identically
to the Cortex workflow output.

The dual strategy matters: an IP can have a clean VirusTotal record but
match a known C2 indicator in OpenCTI. Or a hash may be unknown externally
but flagged internally from a previous incident. Both sources are needed
for complete coverage.

### Phase 4 — Merge

The orchestrator receives results from both enrichment branches and
consolidates them through three merge actions:

- Merge_Alert_Data — original Wazuh alert metadata
- Merge_Observables — parsed and typed IOC list
- Merge_Enrichments — combined Cortex and OpenCTI reports

The result is a single unified JSON object carrying the full alert context.

### Phase 5 — Ticket Creation

The merged object is passed to Create_IRIS_ticket, which maps it to the
DFIR-IRIS alert schema and creates the ticket via API. The resulting alert
in IRIS contains:

- Alert title and severity derived from the Wazuh rule level
- Source agent details: hostname, IP, operating system
- Full observable list with per-source enrichment summaries
- MITRE ATT&CK technique references from Wazuh rule metadata

Analysts open the ticket with complete triage context already in place.

---

## Error Handling

API failures in one enrichment source do not block the pipeline. If
VirusTotal is unreachable, the remaining sources still complete and their
results are merged normally. Missing enrichment fields are simply absent
from the final object rather than causing workflow failure.

For asynchronous APIs such as URLScan, the Sleep action introduces a
configurable delay between job submission and report retrieval to account
for scan completion time.

## Extending the Pipeline

Additional enrichment sources can be added as new parallel branches in
the Cortex workflow, or as entirely new sub-workflows called from the
orchestrator. The normalization schema can be extended with new observable
types without modifying downstream workflows, as long as the existing
structure is preserved. Multiple Wazuh managers can send webhooks to the
same orchestrator endpoint without configuration changes.

---

## Daily Security Digest — Standalone Workflow

The Daily Security Digest is a separate, independent workflow that does not
participate in the alert pipeline. It has no data dependency on SIEM_to_ticket
or any sub-workflow, and it does not write to DFIR-IRIS.

### Trigger model

Where the alert pipeline is event-driven (Wazuh webhook fires on each alert),
the digest is time-driven: a Shuffle Webhook trigger is fired once per day by
a cron job running on the host that runs Shuffle.

Shuffle's built-in schedule feature is not used here. Instead, the trigger is
a standard unauthenticated webhook, and an OS-level cron job sends a POST
request to it at the desired time. This keeps the schedule visible and
auditable outside of Shuffle, and survives Shuffle restarts without
reconfiguration.

**Setup — host cron job**

After importing the workflow, copy the webhook URL generated by Shuffle for
the `Cron_Trigger` node. On the host running Shuffle, add a crontab entry:

```
0 7 * * * curl -s -X POST https://YOUR_SHUFFLE_INSTANCE/api/v1/hooks/webhook_YOUR_WEBHOOK_ID
```

Replace `YOUR_SHUFFLE_INSTANCE` and `YOUR_WEBHOOK_ID` with the values from
the imported workflow.

To verify the cron user can reach the Shuffle instance before committing
the schedule, run the curl command manually once:

```
curl -v -X POST https://YOUR_SHUFFLE_INSTANCE/api/v1/hooks/webhook_YOUR_WEBHOOK_ID
```

A `200 OK` response means the trigger fired successfully and the workflow
is executing.

### Data flow

```
Cron_Trigger
    │
    ├─ CISA_KEV_Feed      (GET JSON)
    ├─ The_DFIR_Report_Feed (GET RSS/XML)
    ├─ Cisco_Talos_Feed    (GET RSS/XML)
    ├─ CERT-FR_Feed        (GET RSS/XML)
    └─ Bleeping_Computer_Feed (GET RSS/XML)
            │
       Parse_Feeds  ── normalizes all feeds into a unified item schema
            │
       Classify_Feeds ── scores items against perimeter keyword stacks,
            │             cross-references CVEs with CISA KEV catalog
       Build_HTML_Report ── renders dark-mode HTML + Discord embed
            │
       Discord_Send ── posts embed + HTML file to Discord webhook
```

### Classification logic

`Classify_Feeds` maintains two keyword stacks compiled from the
organization's technology perimeter:

- **STACK_URGENT** — firewalls (Fortinet, Stormshield, OPNsense), SIEM/SOAR
  stack (Wazuh, Shuffle, OpenCTI), Active Directory attack primitives
  (Kerberoasting, DCSync, Pass-the-Hash, etc.), ransomware groups, and
  generic exploit severity markers (RCE, pre-auth, actively exploited, CVSS 9+).
- **KEV cross-reference** — any item whose CVE appears in the CISA KEV catalog
  receives an additional `in_kev` flag and the remediation due date, regardless
  of keyword match.

Items matching STACK_URGENT or flagged as KEV are sorted to the top of the
report and rendered with distinct visual indicators.

### Relationship to the alert pipeline

The digest and the alert pipeline share no data. Their only common element is
the Shuffle instance. The digest can be imported and run without any of the
other five workflows, and vice versa.
