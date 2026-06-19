# ClinicPulse — System Design Document (v0.2, elaborate)

> **WhatsApp-first patient engagement + a lightweight clinical record for small Indian clinics.**
> Three purpose-built interfaces: a **WhatsApp experience for elderly patients**, a **web console for the receptionist**, and a **prescription + patient-history console for the doctor**.
>
> Status: v0.2 design draft (supersedes and expands `DESIGN.md` v0.1) · Owner: Gokul · Last updated: 2026-06-19

---

## How v0.2 differs from v0.1 (read this first)

`DESIGN.md` (v0.1) deliberately excluded prescriptions and any EMR-like feature. This document (v0.2) **adds a deliberately small clinical layer** because of two explicit product decisions:

1. **The patient is the elderly end-user, and WhatsApp is their only digital surface.** Everything the patient touches must work inside WhatsApp with zero app installs, large readable messages, simple numeric replies, and regional-language support.
2. **The doctor needs a real reason to open the product every consultation.** That reason is: *"the second time a patient comes, search her by phone number or name and instantly see her past problems and the medicines previously prescribed, then write today's prescription in under a minute."*

So v0.2 keeps the v1 wedge (no-shows, follow-up recovery, billing) **and** adds a **lightweight clinical record**: chief complaint, optional vitals, optional diagnosis, and a structured prescription, all tied to a searchable patient timeline.

> **Important scope guardrail.** This is **not** a full EMR. We add the *minimum* clinical structure needed for (a) the doctor to see history and prescribe fast, and (b) the patient to receive her prescription on WhatsApp. We do **not** build drug-interaction engines, ICD-10 coding, lab integrations, insurance, or hospital IPD. Those remain explicitly out of scope.

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [What the clinical market is actually asking for](#2-what-the-clinical-market-is-actually-asking-for)
3. [Product thesis and the three-interface model](#3-product-thesis-and-the-three-interface-model)
4. [Problem statement and validated wedge](#4-problem-statement-and-validated-wedge)
5. [Product scope — in and out (v0.2)](#5-product-scope--in-and-out-v02)
6. [Personas, ICP, and core user flows](#6-personas-icp-and-core-user-flows)
7. [Interface 1 — Patient (WhatsApp, elderly-first)](#7-interface-1--patient-whatsapp-elderly-first)
8. [Interface 2 — Receptionist console (web)](#8-interface-2--receptionist-console-web)
9. [Interface 3 — Doctor console (prescription + history)](#9-interface-3--doctor-console-prescription--history)
10. [Patient history & search — the core clinical feature](#10-patient-history--search--the-core-clinical-feature)
11. [System architecture (high level)](#11-system-architecture-high-level)
12. [Backend design — modular monolith](#12-backend-design--modular-monolith)
13. [Domain model and database schema](#13-domain-model-and-database-schema)
14. [REST API surface (v0.2)](#14-rest-api-surface-v02)
15. [Background jobs and scheduler design](#15-background-jobs-and-scheduler-design)
16. [WhatsApp integration design](#16-whatsapp-integration-design)
17. [Payment integration design](#17-payment-integration-design)
18. [Frontend design — all three surfaces](#18-frontend-design--all-three-surfaces)
19. [Multi-tenancy and security](#19-multi-tenancy-and-security)
20. [DPDP and healthcare compliance (clinical data)](#20-dpdp-and-healthcare-compliance-clinical-data)
21. [Observability, logging, and audit](#21-observability-logging-and-audit)
22. [Testing strategy](#22-testing-strategy)
23. [Infrastructure and DevOps](#23-infrastructure-and-devops)
24. [Cost and unit economics](#24-cost-and-unit-economics)
25. [Phased build plan and backend-first roadmap](#25-phased-build-plan-and-backend-first-roadmap)
26. [Risk register](#26-risk-register)
27. [Glossary](#27-glossary)
28. [Appendices](#28-appendices)

---

## 1. Executive summary

ClinicPulse is a **WhatsApp-first patient engagement product with a lightweight clinical record**, built for **small Indian clinics** (1–5 doctors/chairs, 20–80 patients/day) where:

- **Patients are often elderly and only know WhatsApp.** No patient app, ever. Everything reaches them as simple WhatsApp messages in their language.
- **Receptionists run the front desk** on a fast, mobile-first web console.
- **Doctors prescribe and review history** on a separate console designed for sub-60-second prescriptions and instant patient lookup by phone/name.

The product does four jobs:

1. **Reduce no-shows** — WhatsApp confirmations + reminders.
2. **Recover follow-ups** — automated recall over WhatsApp (the revenue wedge).
3. **Give the doctor memory** — searchable patient history (past problems + medicines) so the second visit is informed.
4. **Get paid faster** — WhatsApp payment links + billing.

Architecture: **Spring Boot 3 modular monolith** on **PostgreSQL 16**, WhatsApp via a **BSP** (Interakt/Gupshup), payments via **Razorpay/Cashfree**, hosted in **AWS Mumbai / Render**. Frontend: **Next.js + React + TypeScript**, three role-scoped apps from one codebase.

**Backend is built first.** The concierge pilots run on Postman + the internal admin console while the customer-facing UIs are built.

---

## 2. What the clinical market is actually asking for

A grounded read of how Indian small-clinic software actually works in 2026, from market scans:

| Observation | Source signal | Design consequence for us |
|---|---|---|
| **"Rx in 30 seconds" is the doctor's headline expectation.** | HealthPlix markets its mobile EMR (SPOT) around writing prescriptions in 30 seconds with a drug **search-suggestion dropdown** and regional-language Rx. | Our doctor console must center on a **fast drug autocomplete** with saved favorites and one-tap dosage presets. Speed is the product. |
| **Doctors want patient data "on the go" and profile switching.** | HealthPlix SPOT is mobile-first; supports switching between doctor/nurse/admin profiles. | Doctor console must be **mobile/tablet-first**, with a clean role boundary from the receptionist console. |
| **Regional language is non-negotiable.** | HealthPlix supports Rx in 14 languages; Eka Care UI ships in 12 Indian languages. | Both the **patient WhatsApp templates** and the **printed prescription** must support regional languages (start: English + Hindi + 1 local). |
| **Patients discover and stay via WhatsApp.** | Multiple BSPs sell WhatsApp appointment reminders, report delivery, follow-up flows to clinics. | WhatsApp is the patient surface; "WhatsApp-first" must be real, not an add-on. |
| **A prescription is a regulated legal artifact, not free text.** | NMC governs Registered Medical Practitioners; a valid Indian prescription should be **legible/printed**, carry the doctor's **name, qualification, and registration number**, the **date**, the **patient identity**, and the **drug + dose + frequency + duration + route**; generic-name prescribing is encouraged. | Our prescription must be **structured** (not a notes blob) and the printed/PDF output must include doctor registration number, clinic details, and a clear dose regimen. |
| **Dentists prescribe within oral-health scope; physiotherapists in India generally do NOT prescribe medicines.** | Prescribing authority is scope-limited; physios prescribe exercise/therapy plans, not drugs (unlike some other countries). | The clinical layer must support **two prescription modes**: a **medicine prescription** (dental/GP) and a **therapy/exercise plan** (physio) — same structure, different item types. |
| **Free/cheap competitors exist (Eka Care free; tools from ₹999).** | Eka Care offers free clinic software; budget tools advertise ₹999/mo. | We **do not compete on price or feature count**. We win on (a) elderly-friendly WhatsApp recall that recovers revenue, and (b) a doctor history/Rx flow that is faster and simpler than a full EMR. |
| **Legibility/safety conventions matter.** | E-prescribing reduces dispensing errors from illegible handwriting; conventions: leading zeros, `mcg` not `μg`, written directions, specify timing & route. | Our structured Rx enforces these conventions by construction (dropdowns for unit, frequency, route, timing). |

**Net:** the market wants *fast prescribing*, *patient history at a glance*, *regional-language WhatsApp engagement*, and *legally clean prescriptions* — delivered simply enough that a small clinic with elderly patients can adopt it in a day. That is exactly the gap ClinicPulse targets.

---

## 3. Product thesis and the three-interface model

There are three humans in the loop and they could not be more different. We design **three separate surfaces**, each optimized for its user, all backed by the same API.

```
                          ┌──────────────────────────────┐
                          │      ClinicPulse Backend      │
                          │   (one API, role-scoped)      │
                          └───────────────┬──────────────┘
                  ┌───────────────────────┼───────────────────────┐
                  ▼                       ▼                        ▼
      ┌───────────────────┐   ┌────────────────────┐   ┌────────────────────┐
      │  PATIENT (elderly) │   │   RECEPTIONIST      │   │      DOCTOR        │
      │  Surface: WhatsApp  │   │  Surface: web app   │   │  Surface: tablet/  │
      │  - confirmations    │   │  - today's queue    │   │   mobile web app   │
      │  - reminders        │   │  - add/search pt    │   │  - patient search  │
      │  - follow-up recall │   │  - mark arrived/done│   │  - history timeline│
      │  - payment link     │   │  - billing          │   │  - write Rx (<60s) │
      │  - prescription PDF  │   │  - send reminders   │   │  - set follow-up   │
      │  - simple "1/2" reply│   │  - assign to doctor │   │  - vitals/diagnosis │
      └───────────────────┘   └────────────────────┘   └────────────────────┘
        ZERO app install         1-hour training            <60s per Rx
        large text, local lang   call-center-like board     fast drug autocomplete
```

### Design priority order
**Patient experience reliability > Doctor speed > Receptionist efficiency > Owner reporting.**
(The patient is the one who cannot be retrained; if WhatsApp confusion happens, value collapses. The doctor must get instant value or won't reopen it. The receptionist can be trained.)

### Why three surfaces and not one
- The patient must never see a dashboard — only WhatsApp.
- The doctor must never wade through billing/reception clutter — only history + Rx.
- The receptionist must never see clinical depth she shouldn't (role-scoped).
- One backend, three thin frontends keeps the codebase small while the UX stays sharp.

---

## 4. Problem statement and validated wedge

### 4.1 The problem (validated)
Small Indian clinics lose revenue and continuity because **appointments, follow-ups, billing, patient communication, and the patient's own history are scattered** across phone calls, paper registers, Excel, and personal WhatsApp. Two specific pains:

- **Revenue leakage** — no-shows and un-recovered follow-ups (the v1 wedge).
- **Lost memory** — when a patient returns months later, the doctor has no quick way to recall *what was wrong last time and what was prescribed*. Paper files are slow; memory is unreliable; elderly patients often can't recount their own medicines.

### 4.2 The wedge (validated + extended)
> **We help small clinics recover missed follow-ups and reduce no-shows over WhatsApp, and we give the doctor instant patient history so the next visit is faster and safer — without changing how the clinic works.**

We sell **recovered revenue**, **receptionist time saved**, and now **doctor confidence on the second visit** (history at a glance + a clean prescription in under a minute).

### 4.3 What we are still NOT
- A full EMR/EHR (no ICD coding, no clinical decision support).
- A telemedicine platform.
- A drug-interaction / pharmacology engine.
- A lab / pharmacy / inventory / insurance system.
- A patient discovery marketplace.
- A hospital ERP / IPD system.
- A patient mobile app (WhatsApp only).

---

## 5. Product scope — in and out (v0.2)

### 5.1 In scope — MVP v1.0 (engagement core, unchanged from v0.1)
| # | Capability |
|---|---|
| 1 | Clinic registration + users (owner, doctor, receptionist, admin) |
| 2 | Patient registry (name, phone, age, gender, notes) |
| 3 | Appointment scheduling + status lifecycle |
| 4 | WhatsApp confirmation, reminder, reply handling (1=confirm, 2=reschedule) |
| 5 | Visit completion + follow-up date |
| 6 | Follow-up recovery automation (the wedge) |
| 7 | Simple billing (line items, discount, paid/unpaid) |
| 8 | Razorpay/Cashfree payment links + status |
| 9 | Owner report (appointments, no-shows, follow-ups recovered, revenue, pending) |
| 10 | CSV import of existing patients |
| 11 | Patient consent capture (audit-logged) |
| 12 | Audit log of all PHI access |
| 13 | Tenant isolation |

### 5.2 In scope — NEW clinical layer (v1.0c, the "doctor + patient history" addition)
| # | Capability | Why it's now in |
|---|---|---|
| C1 | **Doctor console** (separate role + UI) | Doctors need their own surface |
| C2 | **Patient search by phone or name** returning a **history timeline** | The headline request: "has she come before?" |
| C3 | **Consultation record per visit**: chief complaint/problem, optional vitals (BP, weight, pulse, blood sugar), optional diagnosis label | Minimum clinical context |
| C4 | **Structured prescription**: drug + strength + form + dosage (e.g., 1-0-1) + timing (before/after food) + duration + route + instructions | The doctor's daily value |
| C5 | **Drug autocomplete** from a system catalog + clinic favorites | "Rx in under a minute" |
| C6 | **Therapy/exercise-plan prescription** (physio mode) | Physios don't prescribe drugs |
| C7 | **Prescription PDF** with doctor name, qualification, NMC registration number, clinic header | Legal validity |
| C8 | **Send prescription to patient via WhatsApp** (as a document, with consent) | Elderly patients keep their Rx in WhatsApp |
| C9 | **Previous-medicine recall**: one tap to "repeat last prescription" | Chronic/elderly patients on stable regimens |

### 5.3 In scope — v1.1 fast-follow (≤60 days)
- Google Review automation (post-visit ask)
- Dental recall pack (6-month cleaning) and physio session-package tracker
- Receipt + prescription PDF in regional languages (Hindi + 1 local)
- "Repeat prescription" request flow from patient over WhatsApp (forwarded to doctor for approval)

### 5.4 Explicitly OUT of scope (do not build)
- ICD-10 / SNOMED coding, clinical decision support, drug-interaction checks
- Lab/radiology integration, pharmacy dispensing, inventory
- Telemedicine / video consults
- AI diagnosis / AI prescription (Phase 3+ only as *doctor-approved drafts*)
- Insurance / TPA / claims
- ABDM HIP/HIU integration (Phase 3+)
- Patient-facing mobile app
- Multi-branch hospital ERP / IPD

### 5.5 Future (Phase 3+, conditional on ≥20 paying clinics)
- ABDM HFR/HPR registration assist, then optional HIP for care-context linking
- AI visit-summary and AI follow-up drafts (always doctor-approved)
- Specialty-targeted marketing recalls
- Multi-branch, native apps (only if mobile web pain is proven)

---

## 6. Personas, ICP, and core user flows

### 6.1 ICP
| Attribute | Value |
|---|---|
| Specialty | Dental (primary), GP/physician (clinical layer fits well), Physiotherapy (therapy-plan mode) |
| Size | 1–5 doctors/chairs |
| Daily volume | 20–80 patient interactions |
| Patient base | **Skews elderly / low digital literacy; WhatsApp is their only app** |
| Location | Urban / semi-urban tier-1 & tier-2 |
| Staffing | At least one full-time receptionist |
| Current tools | Personal WhatsApp, paper register/Excel, maybe a basic appointment app |
| Buyer | Owner-doctor; one-meeting decision |

### 6.2 Personas
| Persona | Surface | Primary needs | Pays? |
|---|---|---|---|
| **Elderly patient** (e.g., Mrs. Lakshmi, 67) | **WhatsApp only** | Understand when to come, confirm with one tap, get her medicine list she can show family | No (but is the value gate) |
| **Owner-doctor** (Dr. Suresh, 38) | Doctor console + owner report | History at a glance, fast Rx, revenue/no-show/follow-up numbers | **Yes** |
| **Associate doctor** | Doctor console | Today's queue, patient history, write Rx | No |
| **Receptionist** | Receptionist console | Fast lookup, queue, mark status, bill, send reminders | No |

### 6.3 The end-to-end clinic day (all three surfaces)
1. **Receptionist** opens the console → sees today's queue.
2. Patient (or family) **WhatsApps** the clinic / calls → receptionist books appointment → patient gets **WhatsApp confirmation** in her language → replies **1** to confirm.
3. Reminder fires 24h and 2h before (WhatsApp).
4. Patient arrives → receptionist taps **Arrived** → patient appears in the **doctor's queue**.
5. **Doctor** taps the patient → sees **history timeline** (past problems + medicines). For a returning patient she can **search by phone/name** even before arrival.
6. Doctor records **chief complaint**, optional **vitals/diagnosis**, writes a **structured prescription** (drug autocomplete, dosage presets) in under a minute, sets a **follow-up date**.
7. Doctor taps **Finish** → prescription **PDF** generated → optionally **sent to patient's WhatsApp** (with consent) → receptionist notified to bill.
8. **Receptionist** creates the **bill**, takes payment or **sends a WhatsApp payment link**.
9. The **follow-up** is auto-scheduled; recall reminders fire later over WhatsApp.
10. Monday morning, the **owner** gets a WhatsApp summary (no-shows, recovered follow-ups, revenue, pending dues).

---

## 7. Interface 1 — Patient (WhatsApp, elderly-first)

The patient never logs in, never installs anything, never sees a dashboard. The **only** patient surface is WhatsApp. This section is a first-class design area because the patient is elderly and low-literacy.

### 7.1 Design principles for elderly WhatsApp UX
1. **Zero install, zero login.** WhatsApp is the app.
2. **One message = one idea.** Never combine confirmation + payment + follow-up in one message.
3. **Big, simple, local language.** Short sentences, the patient's preferred language (set per patient), minimal English.
4. **Numeric/echo replies only.** "Reply **1** to confirm, **2** to change." Avoid free-text expectations. Accept common variants (1/y/yes/ஆம்/हाँ).
5. **Name the clinic and the doctor every time** so the elderly patient recognizes who is messaging.
6. **No sensitive medical detail by default.** Appointment date/time and "follow-up due" are fine; diagnosis is not sent unsolicited.
7. **The prescription is the exception** — the patient *wants* her medicine list. It is sent **as a PDF/image document**, only after consent, because she (and her family) benefit from holding it.
8. **Human fallback.** Any confused reply routes to the receptionist's queue; a person follows up. Never leave the elderly patient stuck in a bot loop.
9. **Family-proxy aware.** Often a son/daughter manages the elderly patient's WhatsApp. Messages must make sense to a proxy too (always include patient name).

### 7.2 What the patient receives (message catalogue)
| Trigger | Message (rendered in patient's language) | Reply handling |
|---|---|---|
| Appointment booked | "Namaste {{name}}. Your appointment with Dr. {{doctor}} at {{clinic}} is on **{{date}} at {{time}}**. Reply **1** to confirm, **2** to change." | 1→confirm, 2→reschedule queue |
| 24h before | "Reminder: Dr. {{doctor}} tomorrow **{{date}} {{time}}**. Please come 10 minutes early." | optional 1/2 |
| 2h before | "Reminder: your visit is today at **{{time}}**." | — |
| Follow-up due | "Namaste {{name}}, Dr. {{doctor}} asked you to come for a check-up around **{{date}}**. Reply **1** and we will book a time." | 1→follow-up request queue |
| Prescription ready (consented) | "Namaste {{name}}, here is your prescription from Dr. {{doctor}}. Please follow the medicines as written." **+ PDF document** | — |
| Payment | "Namaste {{name}}, your amount is **₹{{amount}}**. Pay safely here: {{link}}" | pay via link |
| Feedback / review | "Thank you for visiting {{clinic}}. If you were happy, please rate us: {{link}}" | — |

### 7.3 Accessibility specifics
- **Font/size:** WhatsApp controls rendering, so we keep messages short and avoid dense paragraphs; bold only the key fact (date/time/amount).
- **Language per patient:** `patient.preferred_language`; templates are pre-approved per language with the BSP.
- **Voice-note tolerance (later):** if a patient sends a voice note, route to receptionist (we do not transcribe in v1).
- **Document over link for Rx:** elderly users struggle with links; send the prescription as a WhatsApp **document (PDF)** or **image** so it stays in the chat. Payment must be a link (unavoidable) — keep it last and clearly labeled.

### 7.4 Consent for the patient surface
- No message is sent until consent is recorded (verbal-at-desk captured by receptionist, or WhatsApp opt-in).
- Every message carries a BSP-managed "Reply STOP to stop" footer.
- Sending the **prescription document** requires a specific consent purpose (`RX_DELIVERY`), separate from appointment reminders.

---

## 8. Interface 2 — Receptionist console (web)

The receptionist is the workhorse user. The console must feel like a **call-center board**, not an EMR.

### 8.1 Design principles
- **Mobile-first**, works on a cheap Android tablet or phone at the desk.
- **One-hand, few-tap** operations; a 40-patient day must be runnable with one hour of training.
- **Phone-number-first search** (Indian receptionists think in phone numbers).
- **Status board** as the home screen.

### 8.2 Screens
1. **Login** (phone + password; optional clinic PIN).
2. **Today** — appointments grouped by status: *Pending · Confirmed · Arrived · With doctor · Done · No-show*. Big status chips, one-tap transitions. Floating **+ Appointment** button.
3. **Add / search patient** — search by phone/name; if not found, 1-tap add (name + phone minimum).
4. **Book appointment** — pick patient, doctor, date/time, reason. WhatsApp confirmation auto-sent.
5. **Patient detail** — contact, consent status, visit/appointment timeline (read-only clinical summary: dates, problems, "Rx sent"), billing history.
6. **Queue hand-off** — mark **Arrived** to push patient into the doctor's queue; reassign doctor.
7. **Billing** — create invoice (line items, discount), record cash/UPI, or **send WhatsApp payment link**.
8. **Reminders** — daily lists: *Follow-ups due*, *Pending payments*; bulk-send with one tap (rate-limited).
9. **Settings (limited)** — clinic hours, templates preview (no clinical config).

### 8.3 What the receptionist can and cannot see
- **Can:** demographics, appointment/visit dates, problem labels, "prescription sent" status, billing.
- **Cannot:** full clinical notes, vitals detail, or edit prescriptions (doctor-owned). Role-scoped by the permission matrix (§19.3).

---

## 9. Interface 3 — Doctor console (prescription + history)

The doctor console exists to do two things extremely well: **show history fast** and **write a prescription in under a minute.** Everything else is secondary.

### 9.1 Design principles
- **Tablet/mobile-first** (doctors move between chairs/rooms).
- **Search-first home**: a single search box ("phone or name") plus today's queue.
- **History on top, prescription below** — the doctor reads, then writes, on one screen.
- **Autocomplete everything**: drug name, strength, form, frequency, duration, timing, route are pickers, not free text. This enforces NMC legibility/safety conventions by construction.
- **Favorites & repeat**: the doctor's most-used drugs are one tap; "repeat last Rx" clones the previous prescription for chronic/elderly patients.
- **Two prescription modes**: **medicine** (dental/GP) and **therapy/exercise plan** (physio).

### 9.2 Screens
1. **Login** (doctor role).
2. **Doctor home** — big search box ("Search patient by phone or name") + **My queue today** (arrived patients).
3. **Patient summary card** — name, age, gender, **allergies & chronic conditions banner** (red if present), last visit date, visit count.
4. **History timeline** — reverse-chronological visits; each card shows **date · chief complaint/problem · diagnosis (if any) · medicines prescribed · follow-up**. Expand a card to see the full prescription.
5. **Consultation screen** (the workhorse):
   - **Chief complaint / problem** (free text + quick chips like "tooth pain", "cleaning", "back pain").
   - **Vitals** (optional, collapsible): BP, pulse, weight, blood sugar.
   - **Diagnosis** (optional short label).
   - **Prescription builder**:
     - Add drug → autocomplete from catalog/favorites.
     - For each item: **strength** (e.g., 500 mg), **form** (tab/cap/syrup), **dosage** (morning-noon-night, e.g., `1-0-1`), **timing** (before/after food), **duration** (e.g., 5 days), **route** (oral/topical/…), **instructions** (free text).
     - **Repeat last prescription** button.
   - **Advice** (free text) and **follow-up date**.
6. **Finish** → generates **prescription PDF** → choices: **Print**, **Send to patient WhatsApp** (consent-gated), **Done**. Hands off to receptionist for billing.

### 9.3 The "Rx in under a minute" interaction model
- Drug autocomplete with the **doctor's favorites pinned at top**.
- Selecting a favorite **pre-fills its usual strength/form/dosage/duration**, which the doctor can tweak.
- Dosage entered as a **1-0-1 picker** (morning-noon-night) plus before/after-food toggle — matches how Indian doctors actually write.
- **Repeat last Rx** for returning chronic patients (very common with elderly).
- The whole consultation can be completed with taps + minimal typing.

### 9.4 Prescription validity (NMC-aligned)
The generated prescription PDF includes, by construction:
- Clinic name, address, phone (header).
- **Doctor name, qualification, and NMC/State Medical Council registration number** (from the doctor's profile).
- **Date**, patient name, age, gender.
- The **℞** block: each drug with strength, form, dose regimen (dose, frequency, duration, route, timing).
- Generic name shown where the catalog provides it (generic-prescribing encouraged).
- Advice and follow-up date.
- A digital "issued via ClinicPulse" footer and a unique prescription number.

### 9.5 Physio mode (therapy plan)
Same structure, different item type: instead of drugs, the doctor adds **therapy items** (e.g., "IFT 15 min", "lower-back stretches x10, 2 sets") with **frequency**, **duration**, and **home-exercise notes**. The PDF renders a "Therapy / Exercise Plan" instead of a drug ℞. Physios in India generally don't prescribe medicines, so the medicine builder is hidden for physio clinics.

---

## 10. Patient history & search — the core clinical feature

This is the feature the user explicitly asked for: *"the second time a patient comes, the doctor should search by mobile number or name and see her past problems and the medicines prescribed."* It deserves its own section.

### 10.1 The search
- **Endpoint:** `GET /api/v1/patients?q={phoneOrName}` (already in the API), plus a clinical-history endpoint `GET /api/v1/patients/{id}/clinical-history`.
- **Match logic:** phone is normalized to E.164 and matched exactly first; name is matched case-insensitively with prefix + fuzzy fallback. Results are **clinic-scoped** (a doctor only ever sees her own clinic's patients).
- **Speed:** indexed on `(clinic_id, phone_e164)` and `(clinic_id, lower(full_name))`. Sub-100ms for a clinic of tens of thousands of patients.

### 10.2 What the history returns
A reverse-chronological **timeline** built by joining the patient's visits with their consultation records and prescriptions:

```
Patient: Lakshmi R. (67, F)   Allergies: Penicillin   Chronic: Type-2 Diabetes
─────────────────────────────────────────────────────────────────────────────
2026-06-19  Visit (Dr. Suresh)
  Problem:    Tooth pain, upper left
  Diagnosis:  Irreversible pulpitis 26
  Rx:         Amoxicillin 500mg  1-0-1  after food  5 days
              Ibuprofen 400mg    1-0-1  after food  3 days
  Follow-up:  2026-06-26  (RCT continuation)
─────────────────────────────────────────────────────────────────────────────
2025-12-02  Visit (Dr. Suresh)
  Problem:    Routine cleaning
  Rx:         (none)
  Follow-up:  2026-06-02  (6-month recall)
─────────────────────────────────────────────────────────────────────────────
2025-07-15  Visit (Dr. Meera)
  Problem:    Sensitivity lower right
  Rx:         Sensodyne toothpaste; SDF application advised
  Follow-up:  —
```

### 10.3 Data that powers it
- `visit` (the encounter) → links appointment, patient, doctor, date.
- `consultation` (clinical record for that visit) → chief complaint, vitals, diagnosis.
- `prescription` + `prescription_item` (what was prescribed).
- `patient_problem` (a longitudinal **problem list** — chronic conditions/allergies surfaced as the red banner).

### 10.4 "Repeat last prescription"
From any history card, **Repeat** clones that prescription's items into a new draft for today's visit, which the doctor can edit and finish. This is the single biggest time-saver for elderly patients on stable regimens.

### 10.5 Privacy guardrails on history
- History is visible to **doctors and the owner-doctor only**; the receptionist sees a redacted summary (dates + problem labels + "Rx sent"), not full prescriptions or vitals.
- Every history view writes an **audit event** (`PATIENT_HISTORY_VIEW`).
- Cross-clinic access is impossible (tenant isolation).

---

## 11. System architecture (high level)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                  CLIENTS                                     │
│  Patient: WhatsApp     Receptionist: web app     Doctor: tablet/mobile web   │
│        (BSP)                (Next.js)                   (Next.js)            │
└───────────────┬────────────────────┬───────────────────────┬───────────────┘
                │ WA webhooks          │ HTTPS (JWT)            │ HTTPS (JWT)
┌───────────────▼──────────────────────────────────────────────▼─────────────┐
│                          ClinicPulse Backend                                │
│              Spring Boot 3 modular monolith (single deployable)             │
│  ┌────────┬────────┬────────┬─────────┬──────────┬───────────┬───────────┐  │
│  │ auth   │ clinic │ patient│  appt   │  visit   │ clinical  │ followup  │  │
│  ├────────┼────────┼────────┼─────────┼──────────┼───────────┼───────────┤  │
│  │ billing│  comms │payments│reporting│  audit   │  catalog  │  (jobs)   │  │
│  └────────┴────────┴────────┴─────────┴──────────┴───────────┴───────────┘  │
│  Scheduler (Spring Scheduling + ShedLock → Quartz later)                    │
│  PDF renderer (prescription + receipt)                                      │
└───────┬───────────────┬───────────────┬───────────────┬────────────────────┘
        ▼               ▼               ▼               ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │PostgreSQL│    │  Redis   │    │ WA BSP   │    │ Razorpay │
  │ (RDS)    │    │ (queue+  │    │(Interakt │    │ /Cashfree│
  │          │    │  cache)  │    │ /Gupshup)│    │          │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘
        │
        ▼
  ┌──────────────┐
  │  S3-compat   │  (prescription PDFs, receipts, optional uploads)
  └──────────────┘
```

New since v0.1: a **`clinical`** module (consultations, prescriptions, problem list), a **`catalog`** module (drug/therapy master + clinic favorites), and a **PDF renderer** used by both prescriptions and receipts.

### Why modular monolith (unchanged rationale)
One repo, one deploy, one DB; cheap; right for a solo dev and ≤200 clinics; module boundaries keep a future microservices split open (extract `comms` or `clinical` if they ever get hot).

---

## 12. Backend design — modular monolith

### 12.1 Tech stack
| Layer | Choice | Notes |
|---|---|---|
| Language | Java 21 LTS | Existing Spring Boot expertise |
| Framework | Spring Boot 3.3+ | Web, Security, Data JPA, Validation, Actuator |
| Build | Maven | Workspace convention |
| DB | PostgreSQL 16 | JSONB for flexible clinical/specialty fields |
| Migration | Flyway | Versioned SQL |
| Persistence | Spring Data JPA (Hibernate) | jOOQ later if reporting gets heavy |
| Auth | Spring Security + JWT (access+refresh) | Stateless, role + clinic claims |
| Validation | Jakarta Validation | DTO-level |
| Cache/queue | Redis (Phase 1.5) | Jobs + inbound WA queue |
| Scheduler | `@Scheduled` + ShedLock → Quartz | Distributed-safe |
| HTTP client | Spring `RestClient`/`WebClient` | BSP + Razorpay |
| PDF | openhtmltopdf or iText community | Prescription + receipt |
| Docs | springdoc-openapi | Swagger UI |
| Testing | JUnit 5, Testcontainers, MockMvc, REST Assured, WireMock | Real Postgres in tests |
| Container | Docker multi-stage | Convention |
| Observability | Micrometer + Prometheus + Grafana | stdout logs shipped to Loki |

### 12.2 Package structure (adds `clinical` and `catalog`)
```
com.clinicpulse
├── ClinicPulseApplication.java
├── common/            # api, exception, security, tenancy, audit, time, util, pdf
│
├── auth/              # login, refresh, users, roles (OWNER, DOCTOR, RECEPTIONIST, ADMIN)
├── clinic/            # clinic, settings, subscription, doctor profile (reg no, qualification)
├── patient/           # patient, consent, import, problem-list
├── appointment/       # appointment + status lifecycle
├── visit/             # visit (the encounter that ties appt → consultation)
│
├── clinical/          # NEW — the lightweight clinical layer
│   ├── api/           # ConsultationController, PrescriptionController, HistoryController
│   ├── domain/        # Consultation, Vitals, Diagnosis, Prescription, PrescriptionItem
│   ├── repo/
│   ├── service/       # ConsultationService, PrescriptionService, HistoryService
│   ├── pdf/           # PrescriptionPdfRenderer
│   └── dto/
│
├── catalog/           # NEW — drug & therapy master + clinic favorites
│   ├── api/
│   ├── domain/        # DrugCatalogItem, TherapyCatalogItem, DoctorFavorite
│   ├── repo/
│   ├── service/       # CatalogService (autocomplete)
│   └── dto/
│
├── followup/          # followup_task + reminder jobs
├── billing/           # invoice, invoice_line, payment
├── payments/          # gateway abstraction (razorpay, cashfree)
├── comms/             # WhatsApp/SMS provider abstraction, templates, message log
├── reporting/         # owner dashboard read models
└── audit/             # append-only audit events
```

### 12.3 Module boundary rules (unchanged, still enforced)
1. No cross-module entity references — pass IDs + DTOs, call the other module's `*.api` bean.
2. Each module owns its tables; no cross-module repository calls.
3. Public API per module in `*.api`; everything else package-private where possible.
4. DTOs cross boundaries, not entities.
5. Cross-module orchestration (e.g., *finish consultation → create prescription → schedule follow-up → notify receptionist to bill*) lives in a thin `application` service inside one transaction.

---

## 13. Domain model and database schema

### 13.1 ER overview (v0.2)
```
Clinic (1) ── (N) User[doctor has reg_no, qualification]
   │
   ├── (N) Patient ── (N) Consent
   │       │        └── (N) PatientProblem        (longitudinal problem/allergy list)
   │       │
   │       └── (N) Appointment ── (0/1) Visit
   │                                   │
   │                                   ├── (0/1) Consultation ── (0/1) Vitals
   │                                   │                         └── (0/N) Diagnosis
   │                                   ├── (0/1) Prescription ── (N) PrescriptionItem
   │                                   ├── (0/N) FollowUpTask
   │                                   └── (0/1) Invoice ── (N) Payment
   │
   ├── (1) ClinicSettings
   ├── (1) Subscription
   ├── (N) DrugCatalogItem / TherapyCatalogItem / DoctorFavorite
   └── (N) MessageLog
```

### 13.2 Conventions (all tables)
`id UUID PK DEFAULT gen_random_uuid()`, `clinic_id UUID NOT NULL` (except `clinic`), `created_at/updated_at TIMESTAMPTZ`, `created_by/updated_by UUID`. Money = `NUMERIC(10,2)` INR. Time = `TIMESTAMPTZ` (UTC stored, `Asia/Kolkata` rendered). Phone = E.164.

### 13.3 Engagement tables (from v0.1 — summarized)
`clinic`, `clinic_settings`, `subscription`, `app_user`, `patient`, `consent`, `appointment`, `visit`, `followup_task`, `invoice`, `invoice_line`, `payment`, `message_template`, `message_log`, `audit_event`.
(Full DDL for these is in `DESIGN.md` §7 and carries over unchanged, except the additions below.)

**Additions to existing tables:**
```sql
-- app_user: add doctor clinical identity (nullable; only doctors fill these)
ALTER TABLE app_user ADD COLUMN qualification        TEXT;   -- "BDS, MDS"
ALTER TABLE app_user ADD COLUMN registration_number  TEXT;   -- State Medical/Dental Council reg no.
ALTER TABLE app_user ADD COLUMN registration_council TEXT;   -- e.g., "Karnataka State Dental Council"
ALTER TABLE app_user ADD COLUMN signature_image_key  TEXT;   -- optional S3 key for Rx signature

-- patient: add preferred language for WhatsApp + clinical flags
ALTER TABLE patient ADD COLUMN preferred_language TEXT NOT NULL DEFAULT 'en'; -- en, hi, ta, ...
ALTER TABLE patient ADD COLUMN proxy_contact_phone TEXT;  -- family member who manages WhatsApp

-- consent: ensure RX_DELIVERY is an allowed purpose (no schema change, value convention)
```

### 13.4 NEW clinical tables
```sql
-- consultation : one clinical record per visit
CREATE TABLE consultation (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  visit_id        UUID NOT NULL UNIQUE REFERENCES visit(id) ON DELETE CASCADE,
  patient_id      UUID NOT NULL REFERENCES patient(id),
  doctor_id       UUID NOT NULL REFERENCES app_user(id),
  chief_complaint TEXT,                          -- "tooth pain upper left"
  clinical_notes  TEXT,                          -- free text, doctor-only
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_consultation_patient ON consultation (clinic_id, patient_id, created_at DESC);

-- vitals : optional, 0/1 per consultation
CREATE TABLE vitals (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  consultation_id UUID NOT NULL UNIQUE REFERENCES consultation(id) ON DELETE CASCADE,
  bp_systolic     INT,
  bp_diastolic    INT,
  pulse_bpm       INT,
  weight_kg       NUMERIC(5,2),
  blood_sugar_mgdl INT,
  temperature_c   NUMERIC(4,1),
  extra           JSONB                          -- specialty-specific extras
);

-- diagnosis : optional short labels (NOT ICD-coded in v1)
CREATE TABLE diagnosis (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  consultation_id UUID NOT NULL REFERENCES consultation(id) ON DELETE CASCADE,
  label           TEXT NOT NULL,                 -- "Irreversible pulpitis 26"
  note            TEXT
);
CREATE INDEX idx_diagnosis_consultation ON diagnosis (consultation_id);

-- patient_problem : longitudinal problem/allergy list (powers the red banner)
CREATE TABLE patient_problem (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  patient_id      UUID NOT NULL REFERENCES patient(id) ON DELETE CASCADE,
  type            TEXT NOT NULL,                 -- ALLERGY, CHRONIC, NOTE
  label           TEXT NOT NULL,                 -- "Penicillin allergy", "Type-2 Diabetes"
  active          BOOLEAN NOT NULL DEFAULT TRUE,
  noted_by        UUID REFERENCES app_user(id),
  noted_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_problem_patient ON patient_problem (clinic_id, patient_id, active);

-- prescription : one per consultation (header)
CREATE TABLE prescription (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id          UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  consultation_id    UUID NOT NULL UNIQUE REFERENCES consultation(id) ON DELETE CASCADE,
  patient_id         UUID NOT NULL REFERENCES patient(id),
  doctor_id          UUID NOT NULL REFERENCES app_user(id),
  rx_number          TEXT NOT NULL,              -- RX-2026-000123
  mode               TEXT NOT NULL DEFAULT 'MEDICINE', -- MEDICINE, THERAPY
  advice             TEXT,
  followup_date      DATE,
  pdf_object_key     TEXT,                       -- S3 key once rendered
  sent_to_patient_at TIMESTAMPTZ,                -- when WA-delivered
  issued_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (clinic_id, rx_number)
);
CREATE INDEX idx_rx_patient ON prescription (clinic_id, patient_id, issued_at DESC);

-- prescription_item : the lines
CREATE TABLE prescription_item (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  prescription_id UUID NOT NULL REFERENCES prescription(id) ON DELETE CASCADE,
  item_type       TEXT NOT NULL DEFAULT 'DRUG',  -- DRUG, THERAPY
  drug_name       TEXT NOT NULL,                 -- "Amoxicillin" (or therapy name)
  generic_name    TEXT,                          -- "Amoxicillin" generic
  strength        TEXT,                          -- "500 mg"
  form            TEXT,                          -- TAB, CAP, SYRUP, TOPICAL, THERAPY
  dosage          TEXT,                          -- "1-0-1" (morning-noon-night)
  timing          TEXT,                          -- BEFORE_FOOD, AFTER_FOOD, NA
  route           TEXT,                          -- ORAL, TOPICAL, ...
  duration_days   INT,                           -- 5
  quantity        TEXT,                          -- "#10 (ten)"
  instructions    TEXT,                          -- free text
  sort_order      INT NOT NULL DEFAULT 0
);
CREATE INDEX idx_rx_item_rx ON prescription_item (prescription_id, sort_order);
```

### 13.5 NEW catalog tables (autocomplete + favorites)
```sql
-- drug_catalog_item : system defaults (clinic_id NULL) + clinic-specific
CREATE TABLE drug_catalog_item (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id     UUID REFERENCES clinic(id) ON DELETE CASCADE,  -- NULL = system default
  brand_name    TEXT NOT NULL,
  generic_name  TEXT,
  default_strength TEXT,
  default_form  TEXT,
  default_dosage TEXT,
  default_duration_days INT,
  active        BOOLEAN NOT NULL DEFAULT TRUE
);
CREATE INDEX idx_drug_search ON drug_catalog_item (clinic_id, lower(brand_name));
CREATE INDEX idx_drug_generic ON drug_catalog_item (lower(generic_name));

-- therapy_catalog_item : for physio mode
CREATE TABLE therapy_catalog_item (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id     UUID REFERENCES clinic(id) ON DELETE CASCADE,
  name          TEXT NOT NULL,                  -- "IFT", "Lower-back stretch"
  default_frequency TEXT,                        -- "2 sets x10"
  default_duration_days INT,
  active        BOOLEAN NOT NULL DEFAULT TRUE
);

-- doctor_favorite : a doctor's pinned items for one-tap Rx
CREATE TABLE doctor_favorite (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id     UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  doctor_id     UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
  item_type     TEXT NOT NULL,                  -- DRUG, THERAPY
  catalog_item_id UUID,                          -- references one of the catalog tables
  use_count     INT NOT NULL DEFAULT 0,
  UNIQUE (doctor_id, item_type, catalog_item_id)
);
```

### 13.6 Schema rules (extended)
- Same tenant/money/time/phone rules as v0.1.
- **Clinical rows are only writable by `DOCTOR`/`OWNER`.** Enforced in service + permission matrix.
- **A consultation is immutable after `prescription` is issued** except for an explicit "amend" that writes a new audit event (no silent edits to a legal artifact).
- **System catalog rows (`clinic_id NULL`) are read-only** to clinics; clinics add their own rows or favorites.

---

## 14. REST API surface (v0.2)

Conventions from v0.1 carry over (`/api/v1`, JWT, `{data,meta}`/`{error}`, pagination, idempotency, tenant from JWT). New/changed groups below.

### 14.1 Doctor identity (clinic module)
```
GET    /api/v1/users/{id}/doctor-profile
PUT    /api/v1/users/{id}/doctor-profile      # qualification, registrationNumber, council, signature
```

### 14.2 Patient search & clinical history
```
GET    /api/v1/patients?q={phoneOrName}&page=0&size=20      # phone-first, name fuzzy
GET    /api/v1/patients/{id}/clinical-history               # the timeline (doctor/owner)
GET    /api/v1/patients/{id}/problems                       # problem/allergy list
POST   /api/v1/patients/{id}/problems                       # add allergy/chronic/note
PUT    /api/v1/patients/{id}/problems/{pid}                 # deactivate/edit
```

`GET /clinical-history` response (shape):
```json
{
  "data": {
    "patient": { "id":"...", "name":"Lakshmi R.", "age":67, "gender":"F" },
    "problems": [
      { "type":"ALLERGY", "label":"Penicillin", "active":true },
      { "type":"CHRONIC", "label":"Type-2 Diabetes", "active":true }
    ],
    "timeline": [
      {
        "visitId":"...", "date":"2026-06-19", "doctor":"Dr. Suresh",
        "chiefComplaint":"Tooth pain upper left",
        "diagnoses":["Irreversible pulpitis 26"],
        "prescription": {
          "rxNumber":"RX-2026-000123",
          "items":[
            {"drug":"Amoxicillin","strength":"500 mg","form":"CAP","dosage":"1-0-1","timing":"AFTER_FOOD","durationDays":5},
            {"drug":"Ibuprofen","strength":"400 mg","form":"TAB","dosage":"1-0-1","timing":"AFTER_FOOD","durationDays":3}
          ]
        },
        "followUpDate":"2026-06-26"
      }
    ]
  }
}
```

### 14.3 Consultation & prescription (clinical module)
```
POST   /api/v1/visits/{visitId}/consultation               # create/start consultation
GET    /api/v1/consultations/{id}
PUT    /api/v1/consultations/{id}                           # chief complaint, notes, vitals, diagnosis
POST   /api/v1/consultations/{id}/prescription              # create prescription + items
PUT    /api/v1/consultations/{id}/prescription              # edit before finalize
POST   /api/v1/consultations/{id}/prescription/repeat-last  # clone patient's previous Rx
POST   /api/v1/consultations/{id}/finish                    # finalize → render PDF → hand off to billing
GET    /api/v1/prescriptions/{id}/pdf                       # download PDF
POST   /api/v1/prescriptions/{id}/send-whatsapp            # consent-gated WA document send
```

### 14.4 Catalog (autocomplete + favorites)
```
GET    /api/v1/catalog/drugs?q={prefix}                     # autocomplete (system + clinic)
GET    /api/v1/catalog/therapies?q={prefix}
GET    /api/v1/catalog/favorites                            # current doctor's favorites
POST   /api/v1/catalog/favorites                            # pin a drug/therapy
DELETE /api/v1/catalog/favorites/{id}
POST   /api/v1/catalog/drugs                                # clinic adds a custom drug
```

### 14.5 Doctor queue
```
GET    /api/v1/doctor/queue?date=2026-06-19                 # arrived patients for this doctor
```

### 14.6 Unchanged groups (from v0.1)
Auth, clinic/settings, users, appointments, visits, follow-ups, billing, reporting, webhooks (`/webhooks/wa/{provider}`, `/webhooks/payments/{gateway}`), admin. See `DESIGN.md` §8.

---

## 15. Background jobs and scheduler design

Same engine as v0.1 (Spring `@Scheduled` + ShedLock → Quartz later). Job catalogue is unchanged plus one addition:

| Job | Cadence | Purpose |
|---|---|---|
| `AppointmentReminderJob` | 5 min | T-24h / T-2h WhatsApp reminders |
| `FollowUpReminderJob` | 30 min | Recall reminders at configured offsets |
| `NoShowMarkerJob` | 15 min | Flip overdue BOOKED → NO_SHOW |
| `PaymentReminderJob` | daily 11:00 IST | Pending dues reminder once, then flag |
| `OwnerWeeklyReportJob` | Mon 08:30 IST | WhatsApp summary to owner |
| `MessageRetryJob` | 2 min | Retry failed messages (≤3) |
| **`PrescriptionDeliveryRetryJob`** (NEW) | 5 min | Retry consent-gated Rx WhatsApp document sends that failed |
| `ConsentExpiryJob` | daily 02:00 | Optional re-opt-in after 12 months |
| `AuditRetentionJob` | weekly | Archive old audit events |

Reliability rules unchanged: every outbound message goes through `MessagingService.send()` (writes `message_log` first), jobs are idempotent, providers never called directly from jobs.

---

## 16. WhatsApp integration design

Same as v0.1 (BSP abstraction; do **not** use Cloud API directly in v1) with elderly-first and prescription-delivery additions.

### 16.1 Provider abstraction
```java
public interface MessagingProvider {
    SendResult sendTemplate(SendTemplateRequest req);     // utility templates
    SendResult sendDocument(SendDocumentRequest req);     // NEW: prescription PDF
    void verifyAndProcessWebhook(WebhookPayload payload);
    String name();   // "INTERAKT", "GUPSHUP"
}
```

### 16.2 Template catalogue (per language)
`APPT_CONFIRM`, `APPT_REMINDER`, `FOLLOWUP_DUE`, `PAYMENT_LINK`, `REVIEW_ASK` (from v0.1) **+** `RX_READY` (NEW: "your prescription is ready" carrier message that accompanies the PDF document). Every template is submitted per language (English + Hindi + 1 local to start) and pre-approved by Meta via the BSP.

### 16.3 Prescription delivery over WhatsApp
1. Doctor finishes consultation → PDF rendered → stored in S3.
2. If patient has `RX_DELIVERY` consent → `MessagingService.sendDocument(patient, rxPdf, RX_READY)`.
3. The PDF is delivered as a WhatsApp **document** so it persists in the elderly patient's chat (no link to click).
4. `prescription.sent_to_patient_at` set; `message_log` row written.
5. No diagnosis text is placed in the message body — the clinical content lives only inside the consented PDF.

### 16.4 Inbound replies (elderly-tolerant)
- `1/y/yes/ஆம்/हाँ` → confirm appointment or accept follow-up booking.
- `2/n/no` → route to receptionist reschedule queue.
- Anything else (including voice notes) → mark unread, route to receptionist. **Never trap an elderly patient in a bot loop.**

### 16.5 Opt-in / DPDP
First message only after consent (verbal-at-desk captured by receptionist, or WA opt-in). Separate consent purpose for `RX_DELIVERY`. STOP suppresses all future messages and records `consent.status = REVOKED`.

---

## 17. Payment integration design

Unchanged from v0.1. Gateway abstraction (`PaymentGateway`), Razorpay first (Cashfree v1.5), payment-link flow, signed webhooks, daily reconciliation job, **no payment cut taken in v1** (avoids RBI aggregator licensing). Payment links are the **only** unavoidable link sent to elderly patients; it is sent as its own message, last, clearly labeled with the amount.

---

## 18. Frontend design — all three surfaces

One Next.js + React + TypeScript codebase, three role-scoped app areas, shared design system. Tailwind CSS + Headless UI, TanStack Query for server state, JWT in HTTP-only cookie.

### 18.1 Shared foundations
- **Design system:** large touch targets (min 44px), high contrast, one primary action per screen.
- **Auth:** login by phone + password; JWT carries `role` + `clinicId`; the app shell routes by role.
- **Offline tolerance:** autosave drafts (consultation, invoice) to local storage; graceful "reconnecting…" banner; daily lists exportable.
- **i18n:** UI strings in English first; patient-facing PDF + WhatsApp in English/Hindi/local.

### 18.2 Patient surface — there is NO frontend
The patient surface is **WhatsApp only** (see §7). We build **no patient web app and no patient mobile app**. The only patient-facing rendered artifacts are:
- WhatsApp templates (text), and
- the **prescription/receipt PDF** (a document, not a page).

This is a deliberate, central product decision for the elderly ICP.

### 18.3 Receptionist app (`/app/reception/*`)
Mobile-first, call-center board. Screens (from §8): Login · Today (status board) · Add/Search patient · Book appointment · Patient detail (redacted clinical summary) · Queue hand-off · Billing · Reminders · Limited settings.
**North star:** a receptionist runs a 40-patient day after one hour of training.

### 18.4 Doctor app (`/app/doctor/*`)
Tablet/mobile-first, search-first. Screens (from §9): Login · Doctor home (search box + my queue) · Patient summary card (allergy/chronic banner) · **History timeline** · **Consultation screen** (complaint, vitals, diagnosis, **prescription builder with autocomplete + favorites + repeat-last**) · Finish (Print / Send to WhatsApp / Done).
**North star:** patient history visible in one tap; a prescription written in under a minute.

Component highlights:
- `PatientSearchBar` — debounced phone/name search.
- `HistoryTimeline` — virtualized list of visit cards; expand for full Rx.
- `PrescriptionBuilder` — drug autocomplete, `1-0-1` dosage picker, before/after-food toggle, duration stepper, favorites rail, "Repeat last Rx" button.
- `AllergyBanner` — red, sticky, shown whenever active allergies/chronic problems exist.

### 18.5 Owner/admin app (`/app/owner/*`)
Reporting and configuration: overview, no-shows, follow-ups recovered, revenue, pending dues, message volumes; user management; clinic settings; template previews. Plus the **internal admin console** (founder-only) for onboarding clinics and viewing audit logs.

---

## 19. Multi-tenancy and security

### 19.1 Tenancy
Shared schema, `clinic_id` discriminator on every table; `TenantContext` (ThreadLocal) set from JWT `clinicId`; Hibernate `@Filter("tenant")` enabled globally; a dedicated **cross-tenant test suite** must stay green (clinic A can never read clinic B). Appropriate for ≤200 clinics; schema-per-tenant only if a large customer demands it.

### 19.2 Auth
bcrypt(12); JWT access 15 min, refresh 7 days (rotated, family-tracked); claims `sub, clinicId, role, jti`; refresh tokens stored hashed, revocable by `jti`.

### 19.3 Authorization — role matrix (extended for clinical layer)
| Capability | OWNER | DOCTOR | RECEPTIONIST | ADMIN |
|---|---|---|---|---|
| View patient demographics | ✅ | ✅ | ✅ | ✅ |
| Edit patient | ✅ | — | ✅ | ✅ |
| Delete patient (DPDP erasure) | ✅ | — | — | ✅ |
| **View full clinical history (Rx, vitals, notes)** | ✅ | ✅ | — | — |
| **View redacted history (dates, problems, "Rx sent")** | ✅ | ✅ | ✅ | ✅ |
| **Write consultation / prescription** | ✅ | ✅ | — | — |
| **Manage problem/allergy list** | ✅ | ✅ | — | — |
| Create appointment | ✅ | ✅ | ✅ | ✅ |
| Complete visit | ✅ | ✅ | ✅ | — |
| Create invoice / send payment link | ✅ | — | ✅ | ✅ |
| Send prescription to patient WhatsApp | ✅ | ✅ | ✅* | ✅ |
| View revenue reports | ✅ | — | — | ✅ |
| Manage users / clinic settings | ✅ | — | — | ✅ |
| View audit log | ✅ | — | — | ✅ |

\* Receptionist may *trigger* the WA send of an already-finalized Rx PDF but can never view/edit its clinical content.

### 19.4 Transport & storage
TLS 1.2+/HSTS; DB connections over TLS; secrets in env + secret manager (never in repo/logs); WA API keys + prescription PDFs encrypted at rest (KMS envelope); backups encrypted, 30-day retention, quarterly restore drill. Prescription PDFs are stored with **per-clinic key prefixes** and short-lived signed URLs.

### 19.5 Rate limiting
Bucket4j on auth; per-clinic outbound WhatsApp cap (default 60/min) to protect BSP quality rating.

---

## 20. DPDP and healthcare compliance (clinical data)

Adding clinical data raises the compliance bar. This section extends v0.1 §13.

### 20.1 Data classification
- **Clinical data (consultations, vitals, diagnoses, prescriptions, problem list) is sensitive personal data.** Treat it with the strictest controls: doctor/owner-only access, audit on every read, encryption at rest, no third-party sharing.

### 20.2 Roles
We remain a **Data Processor**; the clinic is the **Data Fiduciary**. Each clinic signs a DPA at onboarding that now explicitly covers clinical records and prescription delivery over WhatsApp.

### 20.3 Consent (now multi-purpose)
Separate consent purposes: `APPT_REMINDERS`, `FOLLOWUPS`, `PAYMENTS`, `RX_DELIVERY`, `MARKETING`. **Prescription delivery over WhatsApp requires explicit `RX_DELIVERY` consent**, captured at the desk and recorded with evidence. Revocable via STOP and via a public link.

### 20.4 Data minimization for the elderly-WhatsApp surface
- WhatsApp message bodies never contain diagnosis or medicine names.
- Clinical content travels **only inside the consented PDF document**, not as chat text.
- We collect the minimum: name, phone, age/gender, complaint, prescription. No Aadhaar/ABHA in v1.

### 20.5 Prescription as a legal artifact
- Generated PDFs carry the doctor's **name, qualification, registration number, council**, date, patient identity, and a structured dose regimen (NMC-aligned).
- Prescriptions are **immutable once finalized**; an amendment creates a new versioned PDF and an audit trail (no silent edits).
- We do **not** provide clinical decision support or drug-interaction checks, and the UI states that the doctor is solely responsible for the prescription.

### 20.6 Data-subject rights
- **Export:** `GET /patients/{id}/export` returns demographics + appointments + visits + consultations + prescriptions + messages as a bundle.
- **Erasure:** `DELETE /patients/{id}` hard-deletes patient + clinical records + messages; invoices retained for tax law (7 yrs) with personal fields blanked and `patient_id` nulled. Audit-logged with reason.

### 20.7 Retention & residency
All data (DB, PDFs, backups) in India region (`ap-south-1`/Mumbai). Clinical records retained per clinic policy (default 7 years), then archived/anonymized. No model training on patient data. No cross-clinic joins. No data sale.

### 20.8 Breach handling
72-hour clinic + Data Protection Board notification runbook (`SECURITY-INCIDENT.md`); severity matrix; on-call.

---

## 21. Observability, logging, and audit

- **Structured JSON logs** with `requestId, clinicId, userId, action`. **PII and clinical content are never logged** at INFO/DEBUG — only entity IDs.
- **Metrics (Micrometer→Prometheus):** `messages.sent{template,status}`, `followups.due`, `followups.recovered`, `payments.received`, `appointments.no_shows`, **`prescriptions.issued`**, **`prescriptions.sent_whatsapp`**, **`consultation.duration_seconds`** (to track the "Rx in under a minute" goal).
- **Audit (`audit_event`)** is append-only; **every clinical read/write is audited** (`PATIENT_HISTORY_VIEW`, `PRESCRIPTION_CREATE`, `PRESCRIPTION_SEND`, `PROBLEM_ADD`). DB app role has no UPDATE/DELETE on the audit table.

---

## 22. Testing strategy

| Layer | Tool | Target |
|---|---|---|
| Unit | JUnit 5 + Mockito | Services ≥80% on domain logic incl. prescription rules |
| Integration | Spring Boot Test + Testcontainers (Postgres) | Repos + controllers on real DB |
| Contract | WireMock | Razorpay + BSP request/signature shapes |
| E2E | REST Assured | book→remind→arrive→**consult→prescribe→send PDF**→bill→follow-up |
| **Clinical correctness** | Dedicated suite | Rx structure validity, immutability after finalize, repeat-last cloning, history timeline ordering |
| **Multi-tenancy** | Dedicated suite | Clinic A can't read clinic B (incl. clinical tables) |
| **Authorization** | Dedicated suite | Receptionist cannot read full Rx/vitals; doctor can |
| Security | OWASP ZAP baseline in CI | Catch misconfig |

CI gate: PR cannot merge if integration, multi-tenancy, or authorization suites fail.

---

## 23. Infrastructure and DevOps

- **Envs:** `local` (Docker Compose: app+Postgres+Redis), `staging` (Render/Railway), `production` (Render → AWS Mumbai ECS Fargate + RDS).
- **Repo layout:**
```
clinic-pulse/
├── backend/          # Spring Boot monolith
├── frontend/         # Next.js (reception, doctor, owner areas; NO patient app)
├── infra/            # Terraform (AWS phase)
├── ops/              # runbooks, onboarding scripts
└── docs/
    ├── DESIGN.md            # v0.1 (engagement core)
    ├── SYSTEM-DESIGN.md     # v0.2 — this file
    ├── API.md               # generated from OpenAPI
    ├── DPDP.md              # compliance specifics
    └── ONBOARDING.md        # clinic onboarding checklist
```
- **CI/CD:** GitHub Actions — PR: build+unit+integration+lint+security+OpenAPI diff; merge→staging; manual approve→prod. Flyway migrations reviewed in PR.
- **Backups/DR:** RDS automated backups (30 days) + daily `pg_dump` to S3 (90 days) + monthly restore drill. Prescription PDFs versioned in S3.

---

## 24. Cost and unit economics

### 24.1 Per-clinic monthly COGS (small scale)
Assume 750 appts/mo, 3 WA messages each = 2,250 utility messages, plus ~400 prescription-document sends.

| Item | Cost |
|---|---|
| WA utility messages (2,250 × ₹0.16) | ₹360 |
| WA prescription documents (400 × ~₹0.16) | ₹64 |
| BSP platform fee (allocated) | ₹250 |
| Hosting + Postgres + S3 (allocated) | ₹350 |
| Support/ops (allocated) | ₹400 |
| **Total COGS / clinic** | **~₹1,424** |

### 24.2 Pricing tiers
| Tier | Price | Target | Notes |
|---|---|---|---|
| Pilot (30 days) | ₹999 | First touch | break-even |
| Standard | ₹3,499/mo | Most clinics | engagement + clinical layer |
| Growth (managed) | ₹6,999/mo | Done-for-you recalls | ~80% margin |
| Pro (multi-doctor + specialty) | ₹12,999/mo | 3+ chairs | ~85% margin |

Setup/data-migration: ₹5,000 one-time after pilot. We do **not** charge per-prescription (keeps doctor adoption frictionless).

### 24.3 Year-1 target
10 paying clinics × ~₹4,500 ARPU = **₹45,000 MRR**; COGS ~₹14,000; gross profit ~₹31,000/mo; founder salary zero in year 1.

---

## 25. Phased build plan and backend-first roadmap

**Backend first is confirmed.** Concierge pilots run on Postman + the internal admin console while customer UIs are built. The clinical layer is added *after* the engagement core proves it sends and recovers.

### 25.1 Phases (first 150 days)
| Phase | Days | Goal | Output |
|---|---|---|---|
| 0. Validation | 1–14 | 25–30 clinic interviews | CRM filled, 5 pilot commits |
| 1. Concierge | 15–44 | Run 3 clinics manually | Recovered-revenue reports |
| 2. Backend engagement core | 30–80 | v1 capabilities 1–13 | Postman-usable API + admin console |
| 3. Backend clinical layer | 70–110 | C1–C9 (doctor, history, Rx, PDF, WA send) | Doctor flow usable via API + admin |
| 4. Frontend (reception + doctor) | 90–130 | Two role apps | Live for 3 pilots |
| 5. First 10 paying | 110–160 | Founder-led sales | ₹45K MRR |

### 25.2 Backend-first sprint roadmap (1 sprint ≈ 1 week, solo dev)
- **S0 Scaffolding** — Spring Boot 3, Java 21, Flyway, Testcontainers, Docker Compose, CI. *DoD:* `/actuator/health` 200.
- **S1 Auth + clinic + tenancy** — tables, JWT, tenant filter, cross-tenant suite green. *DoD:* clinic A can't read clinic B.
- **S2 Patient + consent + import** — CSV import, phone normalization, multi-purpose consent (incl. `RX_DELIVERY`). *DoD:* import 200 patients; consent capture/revoke.
- **S3 Appointments + visits** — status state machine, audit on completion. *DoD:* book→confirm→arrive→complete via Postman.
- **S4 Comms (WhatsApp)** — provider abstraction (Interakt), `sendTemplate`, webhook, seed templates. *DoD:* real WA confirm + delivery webhook updates log.
- **S5 Scheduler** — ShedLock, appointment + no-show jobs, idempotency, metrics. *DoD:* exactly one T-24h + one T-2h reminder, restart-safe.
- **S6 Follow-ups** — `followup_task`, recall job, "1" reply → receptionist queue. *DoD:* recall fires at T-3/T-1/T-0; reply recovers.
- **S7 Clinical layer I — consultation + history** — `consultation`, `vitals`, `diagnosis`, `patient_problem`; **search by phone/name**; **clinical-history timeline**. *DoD:* returning patient searched by phone shows past problems.
- **S8 Clinical layer II — prescription + catalog** — `prescription(_item)`, `drug/therapy_catalog`, `doctor_favorite`, autocomplete, **repeat-last**, **PDF render**, **WA document send** (consent-gated). *DoD:* doctor writes Rx via API, PDF generated, sent to test WhatsApp.
- **S9 Billing + Razorpay** — invoice/payment, payment link + WA, webhook, reconciliation, receipt PDF. *DoD:* link paid in test mode flips invoice to PAID.
- **S10 Reporting + owner weekly** — read models, weekly WA summary, dashboards incl. `consultation.duration` and `prescriptions.issued`. *DoD:* owner gets a real Monday summary.
- **S11 Audit + DPDP hardening** — AOP audit on clinical read/write, export, erasure with invoice anonymization, rate limiting, ZAP baseline. *DoD:* reviewer exports + deletes a patient, confirmed in audit.
- **S12 Internal admin + onboarding** — founder-only console: add clinic, seed catalog/templates, import, activate. *DoD:* onboard a clinic in <30 min, no DB access.

After S12 the backend is feature-complete for v1.0 + clinical layer. Then build the **reception** and **doctor** Next.js apps (Phase 4).

---

## 26. Risk register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Scope creep: clinical layer drifts toward full EMR | High | High | Hard guardrails in §5.4; no ICD/CDS/labs; "minimum clinical structure" rule |
| 2 | Elderly patients confused by WhatsApp flows | Medium | High | One-idea messages, numeric replies, local language, human fallback, family-proxy aware |
| 3 | WhatsApp template/document rejected by Meta | High | High | BSP pre-approved templates; clinical content only inside consented PDF, never chat text |
| 4 | BSP quality-rating drop → throttling | Medium | High | Strict opt-in, per-clinic rate limit, daily quality monitoring |
| 5 | Prescription legal validity challenged | Medium | High | NMC-aligned PDF (reg no, qualification, date, dose regimen); immutability + audit; doctor-responsibility notice |
| 6 | Doctor finds Rx flow slower than paper | Medium | High | Autocomplete + favorites + repeat-last; measure `consultation.duration`; target <60s |
| 7 | Free competitor (Eka Care) undercuts | High | Medium | Win on elderly WhatsApp recall ROI + simpler doctor history/Rx, not features/price |
| 8 | DPDP rules tighten on clinical data | Medium | Medium | Compliance-first; doctor-only clinical access; audit everything; legal review pre-Phase 3 |
| 9 | Receptionist sees data she shouldn't | Medium | High | Role matrix (§19.3); redacted history; authorization test suite |
| 10 | Founder bandwidth (solo dev + sales) | High | High | Backend-first; clinical layer after engagement proves; phased sprints |
| 11 | Clinic cash-culture resists digital billing | High | Low | "Cash, not billed" visit option; don't force every visit through invoicing |

---

## 27. Glossary

| Term | Meaning |
|---|---|
| ABDM / ABHA | Ayushman Bharat Digital Mission / Health Account |
| BSP | Business Solution Provider (WhatsApp Business Platform reseller) |
| Chief complaint | The patient's stated reason for the visit |
| Consultation | The clinical record for one visit (complaint, vitals, diagnosis) |
| DPDP | Digital Personal Data Protection Act, 2023 (India) |
| Dose regimen | dose size + frequency + duration + route |
| EMR / EHR | Electronic Medical/Health Record (we are deliberately NOT this) |
| ICP | Ideal Customer Profile |
| MRR | Monthly Recurring Revenue |
| NMC | National Medical Commission (regulates RMPs in India) |
| OPD | Outpatient Department |
| PHI / PII | Protected Health Info / Personally Identifiable Info |
| Problem list | Longitudinal record of a patient's chronic conditions/allergies |
| RCT | Root Canal Treatment (dental) |
| RMP | Registered Medical Practitioner |
| Rx / ℞ | Prescription |
| Utility template | WhatsApp transactional message category |

---

## 28. Appendices

### Appendix A — Open decisions awaiting input
1. **Specialties for v1.0c clinical layer:** dental + GP confirmed; physiotherapy uses therapy-plan mode. Confirm whether GP is an active beachhead or dental-only first.
2. **Prescription delivery default:** PDF document vs image for elderly readability — recommend **PDF document**; confirm.
3. **Regional languages at launch:** English + Hindi + which local (Tamil/Kannada/Telugu/Marathi)? Tied to first city.
4. **Drug catalog source:** seed a curated common-drug list (a few hundred items) vs license a fuller Indian drug DB later. Recommend curated seed in v1.
5. **BSP:** Interakt vs Gupshup vs AiSensy — 1-day spike on pricing + document-message support + template approval speed.
6. **Hosting:** Render (Phase 1) → AWS Mumbai (Phase 2).
7. **Brand/first city:** "ClinicPulse" working name; Bengaluru recommended first.

### Appendix B — Reference repos in this workspace
- `healthcare-hospital-system/` — Spring Boot microservices; reuse entity/exception/Docker patterns (as a library, not architecture).
- `fhir-api-application/` — Spring Boot + GraphQL; JPA/`application.yaml` conventions transfer. FHIR data shapes are a useful reference if/when ABDM is added in Phase 3.
- `spring-rag-app/` — relevant only for Phase 3 AI summaries.

New project lives at: `c:\Users\320219651\spring-projects-hub\clinic-pulse\` (`backend/`, `frontend/`).

### Appendix C — Sample structured prescription (rendered intent)
```
────────────────────────────────────────────────────────────
  SMILE DENTAL CLINIC                      Bengaluru · 080-xxxx
  Dr. Suresh Kumar, BDS MDS                Reg: KSDC-12345
────────────────────────────────────────────────────────────
  Patient: Lakshmi R.   Age: 67   Sex: F     Date: 19-Jun-2026
  Rx No:   RX-2026-000123
  Allergy: Penicillin        Chronic: Type-2 Diabetes
────────────────────────────────────────────────────────────
  Chief complaint: Tooth pain, upper left
  Diagnosis:       Irreversible pulpitis 26

  ℞
  1) Amoxicillin 500 mg (CAP)   1-0-1   after food   5 days   #10
  2) Ibuprofen   400 mg (TAB)   1-0-1   after food   3 days   #6

  Advice:    Warm saline rinse twice daily.
  Follow-up: 26-Jun-2026 (RCT continuation)
────────────────────────────────────────────────────────────
  Issued via ClinicPulse · Doctor is responsible for this Rx.
────────────────────────────────────────────────────────────
```

---

**End of v0.2 system design document.**
