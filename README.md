# End-to-End Lab Guide: Copilot Updates Monitoring with Dataverse, Power Automate, and Copilot Studio

## 1. What you will build

You will build an end-to-end solution in Microsoft Power Platform that:

1. Collects Copilot-related updates from:
   - Microsoft Roadmap API
   - Microsoft 365 Message Center (via Microsoft Graph)
2. Stores and upserts those updates in Dataverse
3. Detects changes daily
4. Sends an email summary when changes exist
5. Exposes the data through a Copilot Studio agent

This guide is written for newcomers and includes both **what to do** and **why**.

---

## 2. Architecture used in this lab

We used a two-flow backend plus one Copilot Studio agent:

- **Flow A (Collector):** `Copilot Updates - Collector`
  - Runs on schedule
  - Calls Roadmap + Message Center
  - Upserts records into Dataverse
- **Flow B (Notifier):** `Copilot Updates - Daily Change Notifier`
  - Runs daily
  - Finds records changed in last 24h
  - Sends email summary
- **Copilot Studio Agent:** `Copilot Updates Analyst`
  - User-facing conversational layer
  - Reads Dataverse data through tools

Why this split:
- Scheduled ETL and change tracking are more reliable in Power Automate.
- Copilot Studio is best as the interaction layer, not heavy ingestion ETL.

---

## 3. Environment and tenant context used

Use your own values, but these are the values used during this build:

- **Account:** `AadiK@DIAx72705336.OnMicrosoft.com`
- **Tenant ID:** `a199beb0-50ca-457a-86f2-46d83e0cb767`
- **Environment:** `Default-a199beb0-50ca-457a-86f2-46d83e0cb767` (Zava default)
- **Dataverse Web API:** `https://orga16d493e.api.crm.dynamics.com/api/data/v9.2`

---

## 4. Prerequisites

You need:

- Access to Power Apps, Power Automate, and Copilot Studio in target tenant
- Permission to create/edit Dataverse tables and cloud flows
- Permission to register Entra app (or admin assistance)
- Ability to grant Graph application permission:
  - `ServiceMessage.Read.All`

---

## 5. Dataverse design (table and key)

### 5.1 Use one canonical table

We validated duplicate-similar tables existed and standardized on:

- **Table display name:** `Copilot Update`
- **Logical name:** `cra2f_copilotupdate`

Do not split writes across two similar tables.

### 5.2 Add `UpdateKey` column

Create a text column:

- **Display name:** `UpdateKey`
- **Purpose:** deterministic unique identity per source item

Recommended key format:

- Roadmap item: `roadmap-{id}`
- Message Center item: `mc-{id}`

### 5.3 Create Dataverse alternate key on `UpdateKey`

Create key:

- **Key name:** `UpdateKey Key`
- **Column:** `UpdateKey`

Why this matters:
- Prevents duplicate records
- Enables stable upsert behavior
- Makes collector idempotent

---

## 6. Entra app registration for Message Center (Graph app-only)

### 6.1 Create app registration

Create:

- **App name:** `Copilot-Updates-Collector`

### 6.2 Add Graph application permission

Add:

- `ServiceMessage.Read.All` (Application permission)

Then grant admin consent.

### 6.3 Create client secret

Create one active secret and securely store:

- Tenant ID
- Client ID
- Secret value

---

## 7. Build Flow A: `Copilot Updates - Collector`

### 7.1 Create flow

Create a **Scheduled cloud flow**:

- Name: `Copilot Updates - Collector`
- Frequency: every 6 hours (or your preferred frequency)

### 7.2 Add Roadmap HTTP action

Add HTTP GET to Microsoft roadmap endpoint (Copilot-filtered query).

### 7.3 Add Message Center HTTP action

Add second HTTP action (Graph) using app-only OAuth settings:

- Tenant/Authority: your tenant
- Audience: Graph
- Client ID: app registration client id
- Secret: app secret

Endpoint:

- `https://graph.microsoft.com/v1.0/admin/serviceAnnouncement/messages?$top=100`

### 7.4 Build loop input safely (critical)

A key failure we fixed was malformed loop input becoming **string** instead of **array**.

Use a single array-safe expression pattern in **Apply to each**:

```text
@union(coalesce(body('HTTP')?['value'], json('[]')), coalesce(body('HTTP_1')?['value'], json('[]')))
```

Why:
- Avoids null-array failures
- Combines both sources
- Prevents invalid `@@{...}` and string coercion issues

### 7.5 Inside Apply to each: compute identity + hash

Add **Compose** actions:

1. `UpdateKey`
   - Build from source + item id
2. `RecordHash`
   - Hash/fingerprint from fields such as title/status/modified/published

### 7.6 Dataverse lookup

Add **List rows** against `Copilot Update` table filtered by `UpdateKey`.

### 7.7 Upsert branching

Add **Condition**:

- If row not found -> **Add a new row** (`Change Type = New`)
- If row exists:
  - If hash changed -> **Update row** (`Change Type = Updated`, move prior hash)
  - If hash unchanged -> **Update row** (`Change Type = Unchanged`, refresh `LastSeenOn`)

Recommended mapped fields:

- `UpdateKey`
- `External Id`
- `Title`
- `Source`
- `Record Hash`
- `Previous Hash`
- `Change Type`
- `Last Seen On`
- `Last Change On`
- `Last Notified On`

---

## 8. Build Flow B: `Copilot Updates - Daily Change Notifier`

### 8.1 Create flow

Create scheduled flow:

- Name: `Copilot Updates - Daily Change Notifier`
- Frequency: daily

### 8.2 Query changed rows

Use Dataverse **List rows** with time-window filter (last 24h) and/or `LastNotifiedOn` logic.

### 8.3 Condition

If changed rows exist:

- Build summary (HTML table/text)
- Send email

### 8.4 Update notification tracking

After send, update each row’s `LastNotifiedOn` so records are not repeatedly re-sent.

---

## 9. Copilot Studio integration

### 9.1 Create agent

Create draft agent:

- **Name:** `Copilot Updates Analyst`

### 9.2 Add instructions

Define behavior:

- Summarize changes by source/product/date
- Do not hallucinate
- Use connected tools/data only
- Return concise business summaries

### 9.3 Add tools

Attach Dataverse/workflow tools (for example, list rows or flow-trigger tools) so the agent can answer “what changed in the last 24 hours?” with real data.

---

## 10. Validation checklist

Use this exact sequence:

1. Run collector manually.
2. Confirm run succeeds end-to-end.
3. Confirm rows are inserted/updated in `Copilot Update`.
4. Re-run collector with same data:
   - Should not create duplicates.
5. Change source data (or wait for new updates) and run again:
   - Should mark updated records correctly.
6. Run notifier:
   - Email should contain only changed/new records for the intended window.
7. Ask Copilot Studio agent:
   - Verify response reflects Dataverse-backed data.

---

## 11. Common issues and fixes

### Issue A: `Apply_to_each` fails with “result is type String”

Cause:
- Loop source was stored as literal text instead of expression token.

Fix:
- Re-enter loop input as valid expression using array-safe union/coalesce pattern.

### Issue B: Flow saves with strange `@@{...}` behavior

Cause:
- Expression typed in wrong mode (text vs expression builder).

Fix:
- Use Expression tab / correct expression entry path.
- Re-save and re-open to verify value persists correctly.

### Issue C: Designer read-only or unstable panels

Fix:
- Save often.
- Refresh page.
- Re-open from flow details.
- If action state is corrupted, rebuild only the broken section.

### Issue D: Graph Message Center permission errors

Fix:
- Verify app permission is **Application** (not delegated).
- Re-check admin consent and client secret validity.

---

## 12. Security and operations guidance

- Keep one active client secret and rotate periodically.
- Never hard-code secrets in shared docs/screenshots.
- Use least privilege.
- Add run-failure notifications on both flows.
- Export solution/flows for backup.

---

## 13. What was completed so far in this build

Completed:

- Tenant/environment aligned
- Entra app created and Graph permission configured
- Dataverse table selection finalized (`cra2f_copilotupdate`)
- `UpdateKey` strategy and key setup performed
- Collector flow built and iteratively repaired
- Notifier flow scaffolded and started
- Copilot Studio agent created (`Copilot Updates Analyst`) and tool wiring started

Most recent technical fix:

- Repaired collector loop-expression behavior and restored flow to a saved “ready to go” state during latest pass.

---

## 14. Suggested next actions (to finish production readiness)

1. Re-run collector and verify latest run result for full success.
2. Validate dual-source normalization (roadmap + message center) with real payloads.
3. Finalize notifier’s “already-notified” gating logic.
4. Complete Copilot Studio tool wiring and publish.
5. Run a 2–3 day observation cycle before broad rollout.

---

## 15. Appendix: Artifact names used in this lab

- Dataverse table: `Copilot Update` (`cra2f_copilotupdate`)
- Collector flow: `Copilot Updates - Collector`
- Notifier flow: `Copilot Updates - Daily Change Notifier`
- Copilot Studio agent: `Copilot Updates Analyst`
- Entra app: `Copilot-Updates-Collector`
