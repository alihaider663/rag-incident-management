# Test Cases — RAG Incident Management Platform (WF1–WF12)

Replace `{{N8N_BASE_URL}}` with your instance URL. All workflows must be **Active** (see README §3 for order) before running these. Each test case lists: request, expected HTTP response, and how to verify in Supabase.

---

## WF1 — Knowledge Base Ingestion

**TC-1.1 — Ingest a text document**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/kb-ingest \
  -F "file=@sample_runbook.txt" \
  -F "fileName=sample_runbook.txt" \
  -F "mimeType=text/plain"
```
- Expect: `200`, JSON with a success indicator and chunk count.
- Verify: `SELECT * FROM rag_multimodal_incidents_documents ORDER BY created_at DESC LIMIT 5;` shows new rows with non-null `content` and `embedding`.

**TC-1.2 — Ingest a PDF document**
Same as above with a `.pdf` file and `mimeType=application/pdf`.
- Verify: same table, `metadata->>'blobType'` reflects `application/pdf`.

**TC-1.3 — Malformed/empty upload**
Send the request with no `file` field.
- Expect: a 4xx/5xx error, not a silent 200 with 0 chunks stored unnoticed.

---

## WF2 — Incident Intake

**TC-2.1 — Valid incident, full chain**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/incident/create \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-2001","title":"Payment API timeout","description":"Checkout API returning 504 for all requests","service":"Billing","environment":"Production","reporter":"Monitoring","source":"Synthetic Monitor"}'
```
- Expect: `200`, response body contains `status`, `ticket_id`, and a nested `classification_result` with `classification`/`routing`.
- Verify: `SELECT * FROM incidents WHERE ticket_id='TEST-2001';` → row exists, `status` progresses to `ROUTED` shortly after.

**TC-2.2 — Missing required field**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/incident/create \
  -H "Content-Type: application/json" \
  -d '{"title":"No ticket id"}'
```
- Expect: `400` with an error message naming the missing fields.
- Verify: no row inserted into `incidents`.

**TC-2.3 — Duplicate ticket_id**
Re-send TC-2.1's payload with the same `ticket_id`.
- Expect: `200`, no duplicate row (upsert-safe on `ticket_id` unique constraint).

---

## WF3 — AI Classification Agent (direct call)

**TC-3.1 — Classify in isolation**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/ai/classify \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-3001","title":"Database connection pool exhausted","description":"App servers reporting connection pool exhaustion errors","service":"Database","environment":"Production"}'
```
- Expect: `200`, response contains `classification.category`, `.priority` (P1–P4), `.confidence` (0–100 integer, **not** a 0–1 fraction), and `.routing`.
- Verify: `incident_classifications` and `ai_logs` each have a new row for `TEST-3001`.

**TC-3.2 — Low-confidence triggers human review**
Use a vague/nonsensical description with no KB match, e.g. `"description":"asdkjhaskjdh random text"`.
- Expect: `classification.category = "HUMAN_REVIEW_REQUIRED"` when confidence lands below 70.

**TC-3.3 — Missing required field**
Omit `description`.
- Expect: error thrown (`Missing required field: description`), no downstream calls fire.

---

## WF4 — Team Routing Engine (direct call)

**TC-4.1 — Known category routes correctly**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/team/route \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-4001","team_category":"Database","priority":"P2","category":"Database","title":"Test","description":"Test"}'
```
- Expect: `200`, `{ticket_id, assigned_team, team_email, routing_status:"SUCCESS"}` — **flat shape**, not a raw incidents row.
- Verify: `incidents` row for `TEST-4001` (create one first, or expect the update to no-op if it doesn't exist) shows `assigned_team` populated.

**TC-4.2 — Unknown category falls back**
Use `"team_category":"NonexistentCategory"`.
- Expect: `routing_status:"FALLBACK"`, `assigned_team:"Unassigned - L1 Triage"`.

---

## WF5 — Root Cause Analysis Agent (direct call)

**TC-5.1 — RCA with KB context**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/rca/analyze \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-5001","title":"CRM Login Failure","description":"500 users unable to login","category":"Application","priority":"P1","assigned_team":"CRM Support"}'
```
- Expect: `200`, `observations[]`, `evidence[]`, `root_causes[]` (each with `cause` + `confidence` 0–100), `overall_confidence` 0–100.
- Verify: `incident_rca` has a new row; `incident_recommendations` gets a row shortly after (WF6 fires automatically).

**TC-5.2 — No KB match → human investigation**
Use an obscure, unrelated description.
- Expect: `review_status` effectively `HUMAN_INVESTIGATION_REQUIRED` reasoning when `overall_confidence < 70`.

---

## WF6 — Resolution Recommendation Agent (direct call)

**TC-6.1 — Recommendation from RCA output**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/resolution/recommend \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-6001","root_causes":[{"cause":"Authentication Service Stopped","confidence":90}],"observations":["Auth service errors detected"]}'
```
- Expect: `200`, `immediate_actions[]`, `resolution_steps[]`, `validation_steps[]`, `preventive_actions[]` all arrays (possibly empty, never null).
- Verify: `incident_recommendations` row exists; `approvals` gets a row shortly after (WF9 fires automatically).

---

## WF7 — Notification Engine (direct call)

**TC-7.1 — New incident notification**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/notify/send \
  -H "Content-Type: application/json" \
  -d '{"notification_type":"new_incident","ticket_id":"TEST-7001","priority":"P1","category":"Application","assigned_team":"CRM Support","status":"NEW"}'
```
- Expect: `200`, `{ticket_id, notification_type, email_sent:true, teams_attempted:<bool>}`.
- Verify: `notification_logs` has a `channel='email'` row with `status='SENT'`; inbox receives the email.

**TC-7.2 — Escalation notification**
Same as above with `"notification_type":"escalation","level":"Lead Escalation"`.
- Expect: subject line reflects "SLA Escalation Alert".

**TC-7.3 — Unknown team falls back to default recipient**
Use `"assigned_team":"NonexistentTeam"`.
- Expect: `200`, email still sends to the fallback `l1.triage@company.com`.

**TC-7.4 — Missing required field**
Omit `notification_type`.
- Expect: error, no rows written to `notification_logs`.

---

## WF8 — SLA Engine

**TC-8.1 — SLA record auto-created after routing**
Run TC-2.1 (full incident submission), then check:
```sql
SELECT * FROM sla_tracking WHERE ticket_id = 'TEST-2001';
```
- Expect: row exists, `status='OPEN'`, `current_level=0`, `next_escalation_at` ≈ now + first tier offset for that priority (P1=15min, P2=30min, P3=60min).

**TC-8.2 — Escalation fires on schedule**
Manually backdate a test row to force an immediate check:
```sql
UPDATE sla_tracking SET next_escalation_at = now() - interval '1 minute' WHERE ticket_id = 'TEST-2001';
```
Wait up to 2 minutes (poll interval), then check:
- Expect: `current_level` incremented to `1`, `last_notification` updated, `next_escalation_at` pushed to the next tier.
- Verify: `notification_logs` has a new `notification_type='escalation'` row.

**TC-8.3 — Resolved tickets stop escalating**
Set `status='RESOLVED'` on a test row with a past `next_escalation_at`.
- Expect: row is excluded from the next poll cycle (query filters `status='OPEN'`), no further escalation.

**TC-8.4 — Final tier marks breach**
Manually set `current_level` to the max tier for that priority (e.g. 3 for P1) with a past `next_escalation_at`.
- Expect: next poll cycle sets `breached=true` and pushes `next_escalation_at` far into the future (stops repeat-firing).

---

## WF9 — Approval Workflow

**TC-9.1 — LOW risk auto-approves**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/approval/assess \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-9001","resolution_steps":["Restart service","Clear cache"],"immediate_actions":["Health check"]}'
```
- Expect: `200`, `{risk_level:"LOW", approval_required:false, approval_status:"AUTO_APPROVED"}`.
- Verify: `approvals` row with `approval_status='AUTO_APPROVED'`; WF10 fires automatically (`self_healing_logs` gets a row shortly after).

**TC-9.2 — HIGH risk requires approval**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/approval/assess \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-9002","resolution_steps":["Restart production database","Apply firewall rule change"]}'
```
- Expect: `200`, `{risk_level:"HIGH", approval_required:true, approval_status:"PENDING"}`.
- Verify: an email was sent (WF7 fired); `approvals` row shows `PENDING`.

**TC-9.3 — Unknown action defaults to HIGH (fail-safe)**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/approval/assess \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-9003","resolution_steps":["Do something unusual and unclassified"]}'
```
- Expect: `risk_level:"HIGH"` — the keyword classifier defaults to maximum scrutiny when nothing matches.

**TC-9.4 — Approve a pending ticket**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/approval/decision \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-9002","decision":"APPROVED","approver":"Test Manager"}'
```
- Expect: `200`, `{ticket_id, decision:"APPROVED", approver}`.
- Verify: `approvals.approval_status='APPROVED'`, `approved_at` set; WF10 fires (`self_healing_logs` new row for `TEST-9002`).

**TC-9.5 — Reject a pending ticket**
Same as TC-9.4 with `"decision":"REJECTED"`.
- Expect: `approvals.approval_status='REJECTED'`; **no** `self_healing_logs` row created for this ticket.

**TC-9.6 — Invalid decision value**
Send `"decision":"MAYBE"`.
- Expect: error (`decision must be APPROVED or REJECTED`).

---

## WF10 — Self-Healing Engine (direct call)

**TC-10.1 — Safe action matched in KB → simulated**
Requires WF1 to have ingested a runbook containing an exact command (e.g. `systemctl restart auth-service`).
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/selfheal/execute \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-10001","risk_level":"LOW","resolution_steps":["Restart Authentication Service"]}'
```
- Expect: `200`, `execution_status:"SIMULATED"` (never `"SUCCESS"` in this build — no live infra wired), `validation_status:"NOT_RUN"`.
- Verify: `self_healing_logs` row created.

**TC-10.2 — No matching runbook command → skipped**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/selfheal/execute \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-10002","risk_level":"LOW","resolution_steps":["Do something never documented anywhere"]}'
```
- Expect: `execution_status:"SKIPPED"` — the model must not invent a command.

**TC-10.3 — HIGH risk never auto-executes, even if "safe"**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/selfheal/execute \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-10003","risk_level":"HIGH","resolution_steps":["Restart Authentication Service"]}'
```
- Expect: `execution_status:"SKIPPED"` regardless of whether a matching command exists — this is the hard safety backstop.

---

## WF11 — Feedback Learning (direct call)

**TC-11.1 — Generate and store a knowledge article**
Requires `incidents`/`incident_classifications` rows to exist for the ticket (e.g. from TC-2.1 / TC-3.1).
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/feedback/learn \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"TEST-2001"}'
```
- Expect: `200`, `{status:"stored", ticket_id, chunks_stored:<N>}` (N ≥ 1).
- Verify: `rag_multimodal_incidents_documents` has new rows with `metadata->>'source' = 'feedback_learning'` and `metadata->>'ticket_id' = 'TEST-2001'`.

**TC-11.2 — Duplicate detection prevents re-storage**
Re-send the exact same request from TC-11.1 immediately after.
- Expect: `{status:"duplicate_skipped", ticket_id, top_similarity_score:>0.95}` — no new chunks stored.

**TC-11.3 — Ticket with no downstream data**
Use a `ticket_id` that only exists in `incidents` (no classification/RCA/recommendation rows).
- Expect: still returns `200` and generates an article using whatever partial data is available (fields fall back to "N/A"/"Not available").

---

## WF12 — Audit & Analytics

**TC-12.1 — Each metric returns data**
Run once per metric and confirm `200` + non-error `data[]`:
```bash
for m in mttr sla_compliance incident_volume category_trends team_workload escalation_rate approval_rate self_healing_success rca_confidence_trend top_recurring; do
  echo "=== $m ==="
  curl -s -X POST {{N8N_BASE_URL}}/webhook/analytics/query -H "Content-Type: application/json" -d "{\"metric\":\"$m\"}"
  echo
done
```
- Expect: every call returns `{"metric": "<name>", "data": [...]}`, no 500s.

**TC-12.2 — Invalid metric name**
```bash
curl -X POST {{N8N_BASE_URL}}/webhook/analytics/query -H "Content-Type: application/json" -d '{"metric":"not_a_real_metric"}'
```
- Expect: error listing the allowed metric values.

---

## End-to-end lifecycle test

**TC-E2E-1 — Full happy path, LOW risk (fully automatic)**
1. Run TC-1.1 with a runbook that documents a LOW-risk action (e.g. cache clear).
2. Submit an incident (TC-2.1 style) whose resolution will plausibly match that action.
3. Poll over ~30–60 seconds:
   - `incidents.status` → `NEW` → `ROUTED`
   - `sla_tracking` row created
   - `incident_classifications`, `ai_logs` rows created
   - `incident_rca` row created
   - `incident_recommendations` row created
   - `approvals` row created with `AUTO_APPROVED`
   - `self_healing_logs` row created with `SIMULATED` or `SKIPPED`
   - `notification_logs` has at least one `new_incident` row
4. Manually call `/feedback/learn` for the ticket and confirm a new KB chunk is stored.
5. Call `/analytics/query` with `metric=incident_volume` and confirm the new ticket is reflected.

**TC-E2E-2 — Full happy path, HIGH risk (human-in-the-loop)**
Same as above, but craft the incident so resolution steps include HIGH-risk keywords (e.g. "production deployment"). Confirm the pipeline **pauses** at `approvals.approval_status = PENDING` and does **not** proceed to `self_healing_logs` until `/approval/decision` is called with `APPROVED`.

**TC-E2E-3 — SLA breach path**
Submit an incident, then backdate its `sla_tracking.next_escalation_at` repeatedly (per TC-8.2) to walk it through all escalation tiers and confirm `breached=true` at the final tier, with a corresponding notification logged at each step.
