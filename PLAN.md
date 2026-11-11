# DRCC Contact Centre: plan of action

DRCC Contact Centre is the in-house contact centre and case system for **Debt Review Cancellation Centre (DRCC, drcc.co.za, NCR reg. NCRDC3239)**.

There are no external CRM, helpdesk or inbox tools. Everything below lives in one Node app with one database:
- the dialer
- the CRM
- case management
- the WhatsApp/email/web inbox
- documents
- reports

Only the carrier (SIP trunk), Meta's WhatsApp Cloud API and an SMTP/IMAP mailbox sit outside it, because nobody can self-host those.

Research behind this plan:
- DRCC's public site and service flow.
- The National Credit Act (NCA) rescission and clearance process.
- The 2024–2026 POPIA and CPA direct-marketing changes.
- Open-source dialers, CRMs and inboxes on GitHub.
- The earlier Capterra ZA teardown (1,135 reviews).

Reddit could not be read: automated fetches are blocked, so there are no Reddit findings in this plan.

---

## 1. What DRCC actually does (the workflow we must model)

DRCC takes consumers **out of** debt review. It is not a debt counsellor that puts people into it. The customer journey on drcc.co.za:

| Stage | What happens | DRCC's promise | System object |
|---|---|---|---|
| 1. Enquiry | WhatsApp, phone, email or the web callback form | Reply within **1 business hour** (Mon–Fri 08:00–17:00) | `lead` + SLA timer |
| 2. Free assessment | Agent qualifies: under debt review? since when? court order? accounts paid up? | Same day | `assessment` form → score |
| 3. Sign-up and documents | ID, debt review order or 17.1/17.2 notices, latest statements, mandate | — | `case` + document vault |
| 4. Legal application | Path chosen per case (see below) | 7–14 days | case stage + legal tasks |
| 5. Court / order | Rescission order or clearance certificate issued | — | document upload, stage gate |
| 6. Bureau clearance | Each bureau (TransUnion, Experian, XDS, Compuscan) removes the flag separately | 14–21 days | 4 bureau checkboxes with dates |
| 7. Closed | Client confirmed clear; review request | — | case closed, NPS |

**Legal paths the case manager picks** (the case record stores which one):
- **Clearance certificate (Form 19, s71):** all debts except the home loan are settled. The debt counsellor must issue it.
- **Rescission (s87):** a court rescinds the debt review order, usually when the consumer is no longer over-indebted under s79.
- **Withdrawal before an order (s86(10)/17.W):** used before a court or tribunal order exists. Form 17.W was amended in March 2022, and 17.W(a) was removed.

**Things the system must make easy, because they drive trust:**
- Each client has a dedicated case manager, shown on every screen and in every WhatsApp message.
- Status updates are proactive. The NCR warns publicly about "removal" scams, so clients will ask "is this real?" Every stage change can trigger a WhatsApp template with the case reference.
- Fees and NCR registration are visible in every client-facing template.
- English and Afrikaans scripts and templates throughout.

---

## 2. Architecture: one app, all native

```
Browser (agent / case manager / supervisor / admin)
   │  HTTPS + SSE
Node 22 app ── node:sqlite (WAL)  → later Postgres when >30 seats
   ├─ dialer engine (pacing, retries, callbacks)          src/dialer.js
   ├─ telephony adapter (ARI → Asterisk/FreePBX)          src/ari.js
   ├─ CRM: contacts, leads, cases, tasks, notes           src/crm/*
   ├─ inbox: WhatsApp Cloud API, email, web form, SMS     src/inbox/*
   ├─ documents: local disk + hash, per-case vault        src/docs.js
   ├─ automations: stage → template/task/SLA rules        src/rules.js
   ├─ reports + CSV export                                src/reports.js
   └─ workers: transcription (faster-whisper), bureau reminders
```

**Why not glue in Chatwoot, EspoCRM or Twenty?** They are good, and we borrow their data models. But every integration is a second login, a second contact record and a sync that can fail silently, which is the top CRM complaint in the Capterra reviews. One schema means:
- The agent screen, the case board and the inbox show the **same contact row**.
- A WhatsApp reply, a call and a document upload all land on one timeline.

### 2.1 Core schema (new tables)

```
contacts(id, first_name, last_name, id_number_hash, phone, alt_phone, email,
         language['en'|'af'], whatsapp_opt_in_at, consent_source, created_at)
leads(id, contact_id, source['call'|'whatsapp'|'web'|'email'|'referral'|'import'],
      campaign_id, status, owner_id, first_response_due_at, first_response_at)
assessments(id, lead_id, agent_id, under_review bool, review_since, has_court_order bool,
            debt_counsellor, accounts_settled_except_home bool, monthly_income,
            monthly_debt, recommended_path['clearance'|'rescission'|'withdrawal'|'not_eligible'],
            score, notes, created_at)
cases(id, ref 'DRCC-2026-00412', contact_id, case_manager_id, path, stage, fee_cents,
      fee_paid_cents, opened_at, stage_entered_at, target_close_at, closed_at, outcome)
case_events(id, case_id, at, actor_id, kind, from_stage, to_stage, body)   -- audit trail
bureau_status(case_id, bureau['transunion'|'experian'|'xds'|'compuscan'],
              requested_at, cleared_at, proof_doc_id)
documents(id, case_id, kind['id','dr_order','form_17_1','form_17_2','form_17w',
          'form_19','court_order','statement','mandate','bureau_report','other'],
          filename, sha256, size, uploaded_by, uploaded_at, verified_by, verified_at)
tasks(id, case_id, assignee_id, title, due_at, done_at, auto_rule_id)
conversations(id, contact_id, channel, external_id, last_inbound_at, window_expires_at, assignee_id)
messages(id, conversation_id, direction, body, template_name, media_doc_id, status, at)
templates(id, name, channel, language, category['utility'|'marketing'|'service'], body, meta_status)
rules(id, trigger['stage_enter'|'sla_breach'|'doc_missing'|'idle_days'], condition_json, action_json)
targets(agent_id, date, calls_target, contacts_target, assessments_target, signups_target)
```

The existing `leads` and `attempts` tables (PoC) map onto `contacts`, `leads` and `attempts`.

### 2.2 Case pipeline (Kanban + list)

Stages: **New lead → Assessed → Docs pending → Ready to file → Filed → Order received → Bureau clearance → Closed**, plus a **Not eligible / Lost** side lane.
- **Stage gates (enforced):**
  - "Ready to file" needs ID, the debt review order or 17.x notice, and a mandate, all verified.
  - "Order received" needs a court order or Form 19 document.
  - "Closed" needs all 4 bureaux cleared, or a manual override with a reason.
- **Ageing:** each card shows days in stage. It turns amber past the stage's target (Filed > 14 d, Bureau > 21 d) and red at 1.5×.
- **Bureau tracker:** four chips per case. A reminder task is created 7 days after a request with no clearance.
- Drag to move on the board. Every move writes a `case_events` row and fires any rules for that stage.

### 2.3 Lead capture and the 1-hour SLA

- **Web form endpoint:**
  - `POST /api/capture/web` (embed snippet for drcc.co.za).
  - A honeypot field plus a per-IP rate limit.
  - Creates the contact and lead, sets `first_response_due_at = +1 business hour` using the 08:00–17:00 calendar and SA public holidays, and pushes the lead to the **top of the dial queue** ("speed to lead").
- **WhatsApp:** an inbound message on the Cloud API webhook creates the lead the same way. The 24-hour service window is tracked per conversation.
- **SLA board:** time remaining per new lead. Breaches alert the supervisor on the wallboard and count in reports.

### 2.4 Dialer (from the previous plan, unchanged in substance)

- Predictive, progressive, preview and manual modes, with a 3% abandon target.
- Pacing explainer, retry rules, callbacks, caller-ID pool health, and connect-latency p50/p95.

**DRCC-specific additions:**
- **Campaign types:**
  - *Speed-to-lead* (new web and WhatsApp leads, progressive, highest priority).
  - *Assessment follow-up* (assessed, not signed).
  - *Docs chase* (cases stuck in Docs pending; preview mode so the agent sees the missing documents).
  - *Bureau confirmation* (case managers call clients when a bureau clears).
- **Preview mode for case calls:** existing clients are never predictive-dialled. That avoids abandoned calls on paying clients.
- **Pre-connect card:** shows case ref, stage and case manager before the line connects.

### 2.5 Agent desktop for DRCC

- **Assessment form inline:** the script panel and the form sit side by side. Answers compute the recommended path live:
  - clearance if all accounts except the home loan are settled
  - rescission if the consumer is no longer over-indebted (income − living costs > debt repayments)
  - otherwise not eligible
- **Dispositions:** Assessed – eligible, Assessed – not eligible, Signed up, Callback, Send info on WhatsApp, No answer, Wrong number, Do not contact. Hotkeys 1–8 as before.
- **EN/AF script toggle:** defaults to the contact's language.
- **Quick WhatsApp:** approved templates, one click, written to the timeline.
- **Daily targets bar:** calls, contacts, assessments and sign-ups against the targets table.

### 2.6 Case manager workspace

- **My cases:** list with SLA and ageing, next task, and last client contact (any channel).
- **Case view:**
  - header (ref, path, stage, fee balance)
  - stage stepper
  - bureau tracker
  - document checklist with upload and verify
  - unified timeline (calls with recordings, WhatsApp, email, notes, stage changes)
  - tasks
- **"Send update" button:** picks the stage's template (EN/AF), fills the merge fields and sends on WhatsApp, or by email if there is no WhatsApp opt-in.

### 2.7 Inbox (native, modelled on Chatwoot's data model)

- **WhatsApp Cloud API direct:**
  - Webhook verify and receive, plus send for text, templates and media. Media is stored straight into the case's document vault.
  - Use the official Cloud API, **not** unofficial WhatsApp-Web bridges (Evolution API, WAHA): those risk number bans.
- **Email:** IMAP poll and SMTP send, threaded by Message-ID onto the contact.
- **Web form:** as above. SMS is optional (provider HTTP API).
- **Assignment:** to the case manager if the contact has an open case, otherwise round-robin to agents.
- **Cost guard:** show "free reply window: 14 h left" on each conversation. Outside the window, only approved templates are allowed. South Africa July 2026 rates were roughly US$0.0076 utility and $0.0379 marketing per message excluding VAT. Pricing changed again on 1 Oct 2026, so confirm with Meta before budgeting.

### 2.8 Compliance (configurable; off in the PoC as requested)

These stay as **settings**, because going live without them is a legal risk for DRCC as an NCR-registered business:
- **POPIA s69:** direct-marketing calls need opt-in (Information Regulator guidance note, Dec 2024). Store `consent_source` and the timestamp, and support a Form 4 opt-in record. Inbound enquiries are not cold marketing; that is the basis for calling web/WhatsApp leads back.
- **CPA amendment regulations (Apr 2026):** NCC opt-out registry scrub and calling hours. The toggle exists but is off for the PoC.
- **Recording notice:** a "calls are recorded" prompt. Recording access is limited to the role, and every playback is logged.
- **ID numbers:** store hash plus last 4 digits only. Keep the documents themselves encrypted at rest.

### 2.9 Reports DRCC needs

- Speed-to-lead: median first-response minutes, % within 1 business hour, by source.
- Funnel: leads → assessed → eligible → signed → filed → order → cleared → closed, by source and agent.
- Case ageing per stage, plus bureau turnaround per bureau.
- Agent daily targets against actuals.
- Revenue: fees signed, fees collected, outstanding.
- Plus all dialer reports from the earlier plan (calls per hour, dispositions, abandon, latency, caller-ID).

### 2.10 Optional AI layer (self-hosted, off by default)

- Transcribe recordings with **faster-whisper** on a local GPU or CPU.
- **Set `language` explicitly** (`af` or `en`): Afrikaans auto-detect drifts to Dutch or English.
- Afrikaans fine-tunes exist on Hugging Face (e.g. digiphyte "Fluister"). Test word error rate on 50 real DRCC calls before trusting any model.
- Summary and next-step extraction into the case timeline.

---

## 3. Lessons borrowed from open source

| Project | Licence | What we take (ideas and data model, not code) |
|---|---|---|
| VICIdial / GoAutoDial | GPL / AGPL | Campaign, list and lead-status model; hopper (pre-loaded dial queue); agent pause codes; per-campaign dispositions; recording naming `campaign_date_lead.wav` |
| WebDevProUS AsteriskManager | — | Proof that Node + ARI + SQLite works for a small dialer: Stasis app, originate, bridge |
| Chatwoot | MIT | Conversation / inbox / contact model; 24-hour window handling; assignment rules; canned responses |
| EspoCRM / Twenty | AGPL / GPL | Entity + timeline ("stream") pattern; Kanban by stage field; audit log per field change |
| Evolution API / WAHA | — | **Avoid** for production (unofficial WhatsApp Web); fine only for internal testing |

Copying GPL/AGPL code would force our licence. We re-implement the patterns instead.

---

## 4. Build order

1. **Schema migration:** contacts, leads, cases, case_events, documents, bureau_status, tasks, conversations, messages, templates, rules, targets. Update the importer.
2. **Lead capture plus SLA:** web form endpoint, business-hours calendar, speed-to-lead queue priority, SLA board.
3. **Agent desktop:** assessment form with path logic, DRCC dispositions, EN/AF scripts, daily targets.
4. **Cases:** pipeline board, case view, stage gates, document vault, bureau tracker, rules engine (stage → template/task).
5. **WhatsApp Cloud API:** webhook, templates (EN/AF, submitted to Meta for approval), media to vault, window timer.
6. **Engine:** predictive pacing with the abandon controller; campaign types; preview for clients.
7. **ARI on FreePBX:** bridge, recording, snoop (listen/whisper/barge), with a South African SIP trunk.
8. **Reports** and CSV.
9. **Email inbox, compliance toggles, AI transcription.**

**Done for the pilot means:**
- A web-form lead appears in the agent's queue within 5 s.
- The agent assesses it on the call and signs it up.
- The case manager moves the case through every stage with documents.
- The client gets a WhatsApp update at each stage.
- The funnel report shows it end to end.

---

## 5. Design system (unchanged)

- Glass only on the control layer, with an opaque → frost → refraction fallback ladder.
- Green OKLCH accent at hue 158.
- Type: Bricolage Grotesque, Plus Jakarta Sans (DRCC site font), IBM Plex Mono. Accent is DRCC green #157042.
- Hotkeys and states designed for every control.

## Sources

- DRCC: https://drcc.co.za/
- NCA s71/s86/s87, Forms 17.W and 19: National Credit Regulations; NCR consumer notices on debt review removal scams
- POPIA s69 guidance note (Information Regulator, Dec 2024); CPA amendment regulations (Apr 2026)
- Chatwoot WhatsApp Cloud setup: https://www.chatwoot.com/hc/user-guide/articles/1756799850-how-to-setup-a-whats_app-channel-manual-flow
- WhatsApp SA rates: https://chatmaxima.com/whatsapp-api-pricing/south-africa/ and Meta: https://developers.facebook.com/docs/whatsapp/pricing
- Afrikaans Whisper fine-tunes: https://huggingface.co/digiphyte/fluister-large-v3, https://huggingface.co/DanieClar/stillscript-whisper-large-v3-afrikaans
