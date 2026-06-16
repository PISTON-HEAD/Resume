# ClinicPulse — System Design Document

> **WhatsApp-first follow-up, appointment recovery, and billing automation for small Indian dental and physiotherapy clinics.**
>
> Status: v0.1 design draft · Owner: Gokul · Last updated: 2026-06-17

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Problem statement and validated wedge](#2-problem-statement-and-validated-wedge)
3. [Product scope — in and out](#3-product-scope--in-and-out)
4. [Personas, ICP, and core user flows](#4-personas-icp-and-core-user-flows)
5. [System architecture (high level)](#5-system-architecture-high-level)
6. [Backend design — modular monolith](#6-backend-design--modular-monolith)
7. [Domain model and database schema](#7-domain-model-and-database-schema)
8. [REST API surface (v1)](#8-rest-api-surface-v1)
9. [Background jobs and scheduler design](#9-background-jobs-and-scheduler-design)
10. [WhatsApp integration design](#10-whatsapp-integration-design)
11. [Payment integration design](#11-payment-integration-design)
12. [Multi-tenancy and security](#12-multi-tenancy-and-security)
13. [DPDP and healthcare compliance](#13-dpdp-and-healthcare-compliance)
14. [Observability, logging, and audit](#14-observability-logging-and-audit)
15. [Testing strategy](#15-testing-strategy)
16. [Frontend design (deferred to Phase 2)](#16-frontend-design-deferred-to-phase-2)
17. [Infrastructure and DevOps](#17-infrastructure-and-devops)
18. [Cost and unit economics](#18-cost-and-unit-economics)
19. [Phased build plan — first 120 days](#19-phased-build-plan--first-120-days)
20. [Backend-first implementation roadmap (sprint level)](#20-backend-first-implementation-roadmap-sprint-level)
21. [Risk register](#21-risk-register)
22. [Glossary](#22-glossary)

---

## 1. Executive summary

ClinicPulse is **not a clinic management system**. It is a **revenue-leakage and patient-retention workflow product** for small Indian outpatient clinics, starting with **dental and physiotherapy clinics with 1–5 doctors/chairs and 20–80 appointments per day**.

The product automates three things over WhatsApp:

1. **Appointment confirmation and reminder** (reduce no-shows)
2. **Follow-up recovery** (bring patients back for recall visits, treatment plans, session packages)
3. **Payment collection** (pending dues, package installments)

Everything else — full EMR, prescriptions, telemedicine, ABDM, multi-branch, AI — is explicitly out of scope for v1.

The architecture is a **Spring Boot modular monolith** on PostgreSQL with a React/Next.js admin UI in Phase 2. WhatsApp messaging is delegated to a BSP (Interakt, Gupshup, or AiSensy). Payments are delegated to Razorpay or Cashfree. Hosting starts on Render or Railway, migrating to AWS Mumbai when revenue justifies it.

**Backend is built first**, because:
- Sales demos work with Postman + a thin internal admin UI for the concierge pilot phase.
- The first 3–5 pilot clinics will be operated by the founder; they do not need a polished UI.
- Building the backend forces clean domain modelling before any UI bias creeps in.

---

## 2. Problem statement and validated wedge

### 2.1 The problem (validated)

Small Indian outpatient clinics lose revenue because **appointments, follow-ups, billing, and patient communication are scattered across phone calls, paper registers, Excel sheets, and personal WhatsApp**. The receptionist is the bottleneck. The clinic owner cannot see where revenue leaks.

For dental clinics specifically:
- A 6-month cleaning recall is the single highest-ROI workflow in dentistry; most clinics do it by manual memory, not systematically.
- Multi-visit treatment plans (RCT, crown, implant, aligners) leak when patients miss the second or third visit.
- Pending payments on installment plans go unfollowed.

For physiotherapy clinics:
- Session packages (e.g., 10-session lower-back package) silently die when patients miss session 4.
- Renewal of packages is rarely proactively prompted.

### 2.2 The wedge (validated)

> **We help dental and physiotherapy clinics recover missed follow-ups and reduce no-shows over WhatsApp, without changing how the clinic works today.**

We do not sell software features. We sell **recovered revenue** and **receptionist time saved**.

### 2.3 What we are not

We are explicitly **not**:
- a clinic management system (CMS / HMS)
- an EMR / EHR
- a telemedicine platform
- a patient discovery marketplace (Practo)
- a doctor-facing prescription writer (HealthPlix)
- an ABDM-native Health OS (Eka Care)
- a hospital ERP

This narrowness is the strategy. It is what makes us different.

---

## 3. Product scope — in and out

### 3.1 In scope (MVP, v1.0)

| # | Capability | Why |
|---|---|---|
| 1 | Clinic registration and user management (owner, doctor, receptionist) | Required for tenant isolation |
| 2 | Patient registry (name, phone, age, gender, optional notes) | Required for all messaging |
| 3 | Appointment scheduling with status lifecycle | Core entity |
| 4 | WhatsApp appointment confirmation, reminder, and reply handling (Y/N → confirm/reschedule) | Reduces no-shows |
| 5 | Visit completion with follow-up date setting | Triggers follow-up recovery |
| 6 | Follow-up recovery automation (auto WA reminder when due) | Core wedge |
| 7 | Simple billing: line items, discount, paid/unpaid, payment mode | Required for owner ROI |
| 8 | Razorpay/Cashfree payment links + status tracking | Faster collection |
| 9 | Owner report: appointments, no-shows, follow-ups recovered, revenue collected, pending dues | Sales weapon |
| 10 | CSV import of existing patient list | Onboarding requirement |
| 11 | Patient consent capture (WA, audit-logged) | DPDP requirement |
| 12 | Audit log of all PHI access | DPDP + trust |
| 13 | Tenant-scoped data isolation | Multi-tenant safety |

### 3.2 In scope (v1.1, fast follow within 60 days of v1.0)

- Google Review automation (post-visit ask)
- Specialty-pack: dental recall (6-month cleaning) workflow
- Specialty-pack: physio session-package tracker
- Receipt PDF generation
- WhatsApp template variant per regional language (Hindi + 1 regional)

### 3.3 Explicitly out of scope (do not build)

- Full EMR / clinical notes editor / ICD coding
- Prescription drug database / interactions
- Telemedicine / video consultation
- AI diagnosis / AI prescription / AI triage
- Insurance claims / TPA workflows
- Lab / pharmacy / inventory modules
- Patient-facing mobile app (use WhatsApp + web links)
- Multi-branch hospital ERP
- ABDM HIP / HIU integration (Phase 3+)
- Native iOS / Android app for staff (mobile web works)

### 3.4 Future (Phase 3+, conditional on ≥20 paying clinics)

- ABDM HFR / HPR registration assist (no integration, just guided setup)
- ABDM HIP for issuing care context (only if customers ask)
- AI-generated visit summaries (doctor-approved)
- AI-drafted follow-up messages
- Marketing campaigns module (specialty-targeted recalls)
- Multi-branch support
- Native mobile apps (only if mobile web pain is real)

---

## 4. Personas, ICP, and core user flows

### 4.1 Ideal Customer Profile (ICP)

| Attribute | Value |
|---|---|
| Specialty | Dental (primary) or Physiotherapy (secondary) |
| Size | 1–5 doctors/chairs |
| Daily volume | 20–80 patient interactions |
| Location | Urban or semi-urban (tier-1 / tier-2 metro) |
| Staffing | Has at least one full-time receptionist |
| Current tools | WhatsApp Business (personal), paper register or Excel, sometimes a basic appointment app |
| Pain | Knows they lose follow-ups; can't quantify how many |
| Buying power | Owner-doctor; decision in 1 meeting, not committee |

### 4.2 Personas

| Persona | Role | Primary needs | Pays? |
|---|---|---|---|
| **Owner-doctor** (e.g., Dr. Suresh, 38, runs Smile Dental, Bengaluru) | Decision maker, sometimes also treats | See revenue, no-show %, recovered follow-ups, pending payments | Yes — pays |
| **Associate doctor** | Treats patients, may share clinic | See today's queue, last visit summary | No |
| **Receptionist / front-desk** | Daily user | Fast patient lookup, today's list, mark arrived/done, send reminders | No |
| **Patient** | Recipient of messages | Confirm, reschedule, pay | No — but blocks the value if they don't engage |

Design priority order: **Receptionist > Owner > Doctor > Patient.**

### 4.3 Core user flows

#### Flow A — Daily front-desk (receptionist)
1. Logs in on tablet / phone browser.
2. Sees today's appointments grouped by status (Confirmed / Pending / Arrived / Done / No-show).
3. New patient walks in → search by phone → not found → 1-tap add → 1-tap book appointment for today.
4. WA confirmation auto-sent.
5. Patient arrives → tap "Arrived".
6. Doctor done → receptionist taps "Completed", sets follow-up date (optional), records bill amount, payment status.
7. End of day → receptionist sees a "Pending payment reminders" and "Follow-ups due tomorrow" list; can bulk-send.

#### Flow B — Follow-up recovery (automated)
1. Visit marked completed with `followUpDate = 2026-12-15`.
2. A scheduled job at T-3 days, T-1 day, and T-day sends a WA utility template:
   *"Hello {{name}}, Dr. {{doctor}} has recommended a follow-up on {{date}}. Reply 1 to book a slot."*
3. Patient replies "1" → webhook routes to receptionist's queue → receptionist confirms slot → patient gets confirmation.
4. If no reply after T+3 days, status flips to `MISSED_FOLLOWUP` and shows up in owner's leakage report.

#### Flow C — Payment collection
1. Visit completed, bill ₹2,500, paid ₹1,000 → pending ₹1,500.
2. Receptionist taps "Send payment link" → Razorpay link generated → WA utility template sent.
3. Patient pays → Razorpay webhook → invoice status updates → no further reminder.
4. If unpaid after 48h → auto reminder once. After 7d → flagged in owner report.

#### Flow D — Owner weekly report
- Cron-generated Monday morning WhatsApp summary to owner:
  > "Last week: 142 appointments, 12 no-shows (8.5%), 18 follow-ups booked (₹X recovered), ₹Y pending payment. Open dashboard: link"

---

## 5. System architecture (high level)

```
┌───────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                    │
│  Receptionist mobile-web (React) │ Owner dashboard │ Postman/cURL  │
└────────────────────────┬──────────────────────────────────────────┘
                         │ HTTPS (JWT)
┌────────────────────────▼──────────────────────────────────────────┐
│                    ClinicPulse Backend                             │
│        Spring Boot 3 modular monolith (single deployable)         │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐        │
│  │  auth    │ clinic   │ patient  │  appt    │ followup │        │
│  ├──────────┼──────────┼──────────┼──────────┼──────────┤        │
│  │ billing  │ comms    │ payments │ reporting│  audit   │        │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘        │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ Scheduler (Spring Scheduling, later Quartz)            │      │
│  └────────────────────────────────────────────────────────┘      │
└─────┬──────────────┬──────────────┬──────────────┬───────────────┘
      │              │              │              │
      ▼              ▼              ▼              ▼
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│PostgreSQL│   │  Redis   │   │ WA BSP   │   │ Razorpay │
│ (RDS or  │   │ (queue+  │   │(Interakt │   │ /Cashfree│
│  managed)│   │  cache)  │   │ /Gupshup)│   │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
      │
      ▼
┌──────────────┐
│  S3-compat   │  (receipts, optional file uploads)
│   storage    │
└──────────────┘
```

### Why modular monolith, not microservices

| Reason | Detail |
|---|---|
| Speed of iteration | One repo, one deploy, one DB. Schema changes are atomic. |
| Cost | Single VM at $20–40/mo runs the first 50 clinics. |
| Team size | One developer for the first 12 months. |
| Domain boundaries are still emerging | Forcing them into network boundaries now is premature. |
| Migration path is open | Modules are package-isolated; can extract `comms` or `payments` later if proven hot. |

The existing `healthcare-hospital-system` workspace already taught the team microservices patterns; ClinicPulse intentionally uses a monolith.

---

## 6. Backend design — modular monolith

### 6.1 Tech stack

| Layer | Choice | Notes |
|---|---|---|
| Language | Java 21 LTS | Existing Spring Boot expertise |
| Framework | Spring Boot 3.3+ | Web, Security, Data JPA, Validation, Actuator |
| Build | Maven | Match existing workspace convention |
| DB | PostgreSQL 16 | JSONB for flexible specialty fields |
| Migration | Flyway | Versioned SQL |
| Persistence | Spring Data JPA (Hibernate) | jOOQ only if dynamic reporting becomes painful |
| Auth | Spring Security + JWT (access + refresh) | Stateless |
| Validation | Jakarta Validation | DTO-level |
| Caching / queue | Redis (Phase 1.5) | Defer until scheduler load demands it |
| Scheduler | Spring `@Scheduled` initially, Quartz when multi-instance | Distributed lock via Redis later |
| HTTP client | Spring `RestClient` or `WebClient` | For BSP and Razorpay |
| Docs | springdoc-openapi | Auto-generated Swagger UI |
| Testing | JUnit 5, Testcontainers, MockMvc, REST Assured | Postgres testcontainer for integration |
| Containerization | Docker (multi-stage) | Match existing convention |
| Observability | Micrometer + Prometheus + Grafana Cloud free tier | Logs to stdout, shipped via host agent |

### 6.2 Package structure

```
com.clinicpulse
├── ClinicPulseApplication.java
├── common/                       # shared kernel — no domain logic
│   ├── api/                      # ApiResponse, PageResponse, ErrorResponse
│   ├── exception/                # GlobalExceptionHandler, domain exceptions
│   ├── security/                 # JwtFilter, TenantContext, CurrentUser
│   ├── tenancy/                  # TenantInterceptor, Hibernate filter
│   ├── audit/                    # AuditAware, AuditLogger
│   ├── time/                     # Clock bean, time utilities
│   └── util/                     # Phone normalizer, money utility
│
├── auth/                         # MODULE — sign-up, login, refresh, user CRUD
│   ├── api/                      # AuthController, UserController
│   ├── domain/                   # User, Role, Permission
│   ├── repo/                     # UserRepository
│   ├── service/                  # AuthService, UserService, JwtService
│   └── dto/
│
├── clinic/                       # MODULE — clinic entity, settings, subscription
│   ├── api/
│   ├── domain/                   # Clinic, ClinicSettings, Subscription
│   ├── repo/
│   ├── service/
│   └── dto/
│
├── patient/                      # MODULE
│   ├── api/
│   ├── domain/                   # Patient, Consent
│   ├── repo/
│   ├── service/                  # PatientService, ConsentService, ImportService
│   └── dto/
│
├── appointment/                  # MODULE
│   ├── api/
│   ├── domain/                   # Appointment, AppointmentStatus
│   ├── repo/
│   ├── service/
│   └── dto/
│
├── visit/                        # MODULE — completed appointment + notes (minimal)
│   ├── api/
│   ├── domain/                   # Visit
│   ├── repo/
│   ├── service/
│   └── dto/
│
├── followup/                     # MODULE
│   ├── api/
│   ├── domain/                   # FollowUpTask, FollowUpStatus
│   ├── repo/
│   ├── service/
│   ├── scheduler/                # FollowUpReminderJob
│   └── dto/
│
├── billing/                      # MODULE
│   ├── api/
│   ├── domain/                   # Invoice, InvoiceLine, Payment
│   ├── repo/
│   ├── service/
│   └── dto/
│
├── payments/                     # MODULE — payment gateway abstraction
│   ├── api/                      # webhooks
│   ├── gateway/                  # PaymentGateway interface
│   │   ├── razorpay/             # RazorpayGateway
│   │   └── cashfree/             # CashfreeGateway (Phase 1.5)
│   ├── service/
│   └── dto/
│
├── comms/                        # MODULE — WhatsApp, SMS fallback
│   ├── api/                      # webhooks (delivery, replies)
│   ├── provider/                 # MessagingProvider interface
│   │   ├── interakt/
│   │   └── gupshup/              # Phase 1.5
│   ├── template/                 # MessageTemplate, TemplateRenderer
│   ├── domain/                   # MessageLog, MessageStatus
│   ├── repo/
│   ├── service/                  # MessagingService, OptInService
│   └── dto/
│
├── reporting/                    # MODULE — owner dashboard data
│   ├── api/
│   ├── service/                  # ReportService (read-only queries)
│   └── dto/
│
└── audit/                        # MODULE — append-only audit log
    ├── api/                      # admin-only viewer
    ├── domain/                   # AuditEvent
    ├── repo/
    └── service/
```

### 6.3 Module boundaries — rules

1. **No cross-module entity references.** `appointment` does not import `patient.domain.Patient`. It holds `patientId` and asks `PatientService` (via a thin spring bean interface in `patient.api`) for what it needs.
2. **No cross-module repository calls.** Each module owns its tables.
3. **Public API per module** lives in a `*.api` subpackage. Everything else is package-private if possible.
4. **DTOs cross module boundaries**, not entities.
5. **Cross-module orchestration** (e.g., "complete visit → create invoice → schedule follow-up") lives in a thin `application` service that calls module public APIs in order, inside a transaction.

This keeps the migration path to microservices open. If `comms` becomes hot, it can be extracted with a known contract.

---

## 7. Domain model and database schema

### 7.1 Entity overview (ER)

```
Clinic (1) ──── (N) User
   │
   ├── (N) Patient ──── (N) Consent
   │       │
   │       └── (N) Appointment ──── (0/1) Visit ──── (0/1) Invoice ──── (N) Payment
   │                                    │
   │                                    └── (0/N) FollowUpTask
   │
   ├── (1) ClinicSettings
   ├── (1) Subscription
   └── (N) MessageLog
```

### 7.2 Tables (PostgreSQL DDL sketch)

All tables include:
- `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- `clinic_id UUID NOT NULL` (except `clinic` itself)
- `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- `created_by UUID`, `updated_by UUID`

```sql
-- clinic
CREATE TABLE clinic (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name            TEXT NOT NULL,
  specialty       TEXT NOT NULL CHECK (specialty IN ('DENTAL','PHYSIO','OTHER')),
  phone           TEXT NOT NULL,
  address         TEXT,
  city            TEXT,
  state           TEXT,
  timezone        TEXT NOT NULL DEFAULT 'Asia/Kolkata',
  status          TEXT NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE, PILOT, SUSPENDED
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- clinic_settings  (1:1 with clinic)
CREATE TABLE clinic_settings (
  clinic_id                 UUID PRIMARY KEY REFERENCES clinic(id) ON DELETE CASCADE,
  wa_business_phone         TEXT,                          -- the BSP-registered number
  wa_provider               TEXT,                          -- INTERAKT, GUPSHUP
  wa_api_key_encrypted      TEXT,
  default_appointment_mins  INT NOT NULL DEFAULT 15,
  reminder_offsets_minutes  INT[] NOT NULL DEFAULT '{1440, 120}',
  followup_offsets_days     INT[] NOT NULL DEFAULT '{-3, -1, 0}',
  google_review_url         TEXT,
  language_preference       TEXT NOT NULL DEFAULT 'en'
);

-- subscription
CREATE TABLE subscription (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  plan            TEXT NOT NULL,                          -- PILOT, STARTER, GROWTH
  amount_inr      NUMERIC(10,2) NOT NULL,
  billing_cycle   TEXT NOT NULL DEFAULT 'MONTHLY',
  starts_on       DATE NOT NULL,
  ends_on         DATE,
  status          TEXT NOT NULL DEFAULT 'ACTIVE',
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- app_user  (the table name "user" is reserved in postgres)
CREATE TABLE app_user (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  full_name       TEXT NOT NULL,
  phone           TEXT NOT NULL,
  email           TEXT,
  password_hash   TEXT NOT NULL,
  role            TEXT NOT NULL CHECK (role IN ('OWNER','DOCTOR','RECEPTIONIST','ADMIN')),
  status          TEXT NOT NULL DEFAULT 'ACTIVE',
  last_login_at   TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (clinic_id, phone)
);

-- patient
CREATE TABLE patient (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  full_name       TEXT NOT NULL,
  phone_e164      TEXT NOT NULL,                          -- +91XXXXXXXXXX, normalized
  gender          TEXT CHECK (gender IN ('M','F','O','U')),
  date_of_birth   DATE,
  city            TEXT,
  source          TEXT,                                   -- WALKIN, PHONE, WHATSAPP, REFERRAL
  notes           TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (clinic_id, phone_e164)
);
CREATE INDEX idx_patient_clinic_phone ON patient (clinic_id, phone_e164);
CREATE INDEX idx_patient_clinic_name ON patient (clinic_id, lower(full_name));

-- consent
CREATE TABLE consent (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  patient_id      UUID NOT NULL REFERENCES patient(id) ON DELETE CASCADE,
  channel         TEXT NOT NULL,                          -- WHATSAPP, SMS, EMAIL
  purpose         TEXT NOT NULL,                          -- APPT_REMINDERS, FOLLOWUPS, PAYMENTS, MARKETING
  status          TEXT NOT NULL,                          -- GRANTED, REVOKED
  captured_via    TEXT NOT NULL,                          -- WA_REPLY, WEB_FORM, IN_PERSON
  captured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  evidence        JSONB                                   -- raw WA reply payload etc.
);
CREATE INDEX idx_consent_patient ON consent (patient_id);

-- appointment
CREATE TABLE appointment (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  patient_id      UUID NOT NULL REFERENCES patient(id),
  doctor_id       UUID REFERENCES app_user(id),
  scheduled_at    TIMESTAMPTZ NOT NULL,
  duration_mins   INT NOT NULL DEFAULT 15,
  status          TEXT NOT NULL DEFAULT 'BOOKED',
                  -- BOOKED, CONFIRMED, ARRIVED, COMPLETED, CANCELLED, NO_SHOW
  reason          TEXT,
  source          TEXT,                                   -- PHONE, WALKIN, WHATSAPP, RECALL
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_appt_clinic_day ON appointment (clinic_id, scheduled_at);
CREATE INDEX idx_appt_status ON appointment (clinic_id, status, scheduled_at);

-- visit
CREATE TABLE visit (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  appointment_id  UUID NOT NULL UNIQUE REFERENCES appointment(id),
  patient_id      UUID NOT NULL REFERENCES patient(id),
  doctor_id       UUID REFERENCES app_user(id),
  completed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  notes           TEXT,                                   -- free text only, no diagnosis
  followup_date   DATE,                                   -- null = no follow-up
  followup_reason TEXT                                    -- short label only
);

-- followup_task
CREATE TABLE followup_task (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  patient_id      UUID NOT NULL REFERENCES patient(id),
  source_visit_id UUID REFERENCES visit(id),
  due_date        DATE NOT NULL,
  status          TEXT NOT NULL DEFAULT 'PENDING',
                  -- PENDING, REMINDED, BOOKED, MISSED, CANCELLED
  last_reminded_at TIMESTAMPTZ,
  reminder_count  INT NOT NULL DEFAULT 0,
  notes           TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_followup_due ON followup_task (clinic_id, due_date, status);

-- invoice
CREATE TABLE invoice (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  patient_id      UUID NOT NULL REFERENCES patient(id),
  visit_id        UUID REFERENCES visit(id),
  invoice_number  TEXT NOT NULL,                          -- INV-2026-000123
  issued_on       DATE NOT NULL DEFAULT CURRENT_DATE,
  subtotal_inr    NUMERIC(10,2) NOT NULL DEFAULT 0,
  discount_inr    NUMERIC(10,2) NOT NULL DEFAULT 0,
  total_inr       NUMERIC(10,2) NOT NULL DEFAULT 0,
  paid_inr        NUMERIC(10,2) NOT NULL DEFAULT 0,
  status          TEXT NOT NULL DEFAULT 'UNPAID',
                  -- UNPAID, PARTIAL, PAID, VOID
  UNIQUE (clinic_id, invoice_number)
);

CREATE TABLE invoice_line (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id      UUID NOT NULL REFERENCES invoice(id) ON DELETE CASCADE,
  description     TEXT NOT NULL,
  quantity        INT NOT NULL DEFAULT 1,
  unit_price_inr  NUMERIC(10,2) NOT NULL,
  line_total_inr  NUMERIC(10,2) NOT NULL
);

CREATE TABLE payment (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id          UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  invoice_id         UUID NOT NULL REFERENCES invoice(id),
  amount_inr         NUMERIC(10,2) NOT NULL,
  mode               TEXT NOT NULL,                       -- CASH, UPI, CARD, LINK
  gateway            TEXT,                                -- RAZORPAY, CASHFREE
  gateway_ref        TEXT,
  status             TEXT NOT NULL,                       -- INITIATED, SUCCESS, FAILED
  paid_at            TIMESTAMPTZ,
  raw_payload        JSONB
);

-- message_template (clinic-overridable)
CREATE TABLE message_template (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id       UUID REFERENCES clinic(id) ON DELETE CASCADE, -- null = system default
  code            TEXT NOT NULL,
                  -- APPT_CONFIRM, APPT_REMINDER, FOLLOWUP_DUE, PAYMENT_LINK, REVIEW_ASK
  language        TEXT NOT NULL DEFAULT 'en',
  channel         TEXT NOT NULL DEFAULT 'WHATSAPP',
  provider_template_name TEXT NOT NULL,                   -- Meta-approved template id
  body_preview    TEXT NOT NULL,
  variables       TEXT[] NOT NULL DEFAULT '{}',
  active          BOOLEAN NOT NULL DEFAULT TRUE,
  UNIQUE (COALESCE(clinic_id, '00000000-0000-0000-0000-000000000000'::uuid), code, language)
);

-- message_log (every outbound + inbound message)
CREATE TABLE message_log (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id          UUID NOT NULL REFERENCES clinic(id) ON DELETE CASCADE,
  patient_id         UUID REFERENCES patient(id),
  direction          TEXT NOT NULL CHECK (direction IN ('OUT','IN')),
  channel            TEXT NOT NULL DEFAULT 'WHATSAPP',
  template_code      TEXT,
  related_entity     TEXT,                                -- APPOINTMENT, FOLLOWUP, INVOICE
  related_id         UUID,
  provider           TEXT,
  provider_message_id TEXT,
  status             TEXT,                                -- QUEUED, SENT, DELIVERED, READ, FAILED, REPLIED
  body_rendered      TEXT,
  cost_inr           NUMERIC(8,3),                        -- conversation cost tracking
  error_message      TEXT,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_msglog_clinic_date ON message_log (clinic_id, created_at);
CREATE INDEX idx_msglog_related ON message_log (related_entity, related_id);

-- audit_event
CREATE TABLE audit_event (
  id              BIGSERIAL PRIMARY KEY,
  clinic_id       UUID,
  actor_user_id   UUID,
  actor_role      TEXT,
  action          TEXT NOT NULL,                          -- PATIENT_VIEW, INVOICE_UPDATE, etc.
  entity_type     TEXT,
  entity_id       UUID,
  ip_address      INET,
  user_agent      TEXT,
  metadata        JSONB,
  occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_clinic_time ON audit_event (clinic_id, occurred_at DESC);
CREATE INDEX idx_audit_entity ON audit_event (entity_type, entity_id);
```

### 7.3 Schema rules

- **Every tenant-scoped query MUST include `clinic_id`.** Enforced by Hibernate `@Filter`, tested by integration tests.
- **Soft delete only where it matters** (patient, invoice). Hard-delete elsewhere on DPDP erasure request.
- **All money fields are `NUMERIC(10,2)`** in INR. Never `float`/`double`.
- **All timestamps are `TIMESTAMPTZ`**, stored UTC, rendered in `clinic.timezone` (default `Asia/Kolkata`).
- **Phone numbers are normalized to E.164** (`+91XXXXXXXXXX`) at the API boundary, never stored otherwise.

---

## 8. REST API surface (v1)

### 8.1 Conventions

- Base path: `/api/v1`
- Auth: `Authorization: Bearer <jwt>` (except `/auth/login`, `/auth/refresh`, webhooks)
- All responses: `{"data": ..., "meta": ...}` or `{"error": {"code", "message", "details"}}`
- Pagination: `?page=0&size=20&sort=scheduledAt,desc`
- Idempotency: `Idempotency-Key` header on POST that creates/changes external state (payments, messages)
- Tenant: resolved from JWT `clinicId` claim; never trusted from request body

### 8.2 Endpoints (v1.0)

#### Auth
```
POST   /api/v1/auth/login                 { phone, password }
POST   /api/v1/auth/refresh               { refreshToken }
POST   /api/v1/auth/logout
GET    /api/v1/auth/me
```

#### Clinic & settings (OWNER only)
```
GET    /api/v1/clinic
PUT    /api/v1/clinic
GET    /api/v1/clinic/settings
PUT    /api/v1/clinic/settings
GET    /api/v1/clinic/subscription
```

#### Users (OWNER, ADMIN)
```
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/{id}
PUT    /api/v1/users/{id}
DELETE /api/v1/users/{id}
POST   /api/v1/users/{id}/reset-password
```

#### Patients
```
GET    /api/v1/patients?q=phoneOrName&page=0&size=20
POST   /api/v1/patients
GET    /api/v1/patients/{id}
PUT    /api/v1/patients/{id}
DELETE /api/v1/patients/{id}                 # DPDP erasure
POST   /api/v1/patients/import               # multipart CSV
GET    /api/v1/patients/{id}/consents
POST   /api/v1/patients/{id}/consents
POST   /api/v1/patients/{id}/consents/{cid}/revoke
GET    /api/v1/patients/{id}/timeline        # appointments + visits + messages
```

#### Appointments
```
GET    /api/v1/appointments?date=2026-06-17&status=BOOKED
POST   /api/v1/appointments
GET    /api/v1/appointments/{id}
PUT    /api/v1/appointments/{id}
POST   /api/v1/appointments/{id}/transition  { to: CONFIRMED|ARRIVED|COMPLETED|NO_SHOW|CANCELLED }
POST   /api/v1/appointments/{id}/send-reminder   # manual override
```

#### Visits
```
POST   /api/v1/visits                         # created on COMPLETED transition or manually
GET    /api/v1/visits/{id}
PUT    /api/v1/visits/{id}                    # notes, follow-up date
```

#### Follow-ups
```
GET    /api/v1/followups?from=...&to=...&status=PENDING
POST   /api/v1/followups
GET    /api/v1/followups/{id}
PUT    /api/v1/followups/{id}
POST   /api/v1/followups/{id}/mark-booked    { appointmentId }
POST   /api/v1/followups/{id}/cancel
POST   /api/v1/followups/{id}/send-reminder  # manual override
```

#### Billing
```
GET    /api/v1/invoices?status=UNPAID
POST   /api/v1/invoices
GET    /api/v1/invoices/{id}
PUT    /api/v1/invoices/{id}
POST   /api/v1/invoices/{id}/payments        # record manual payment
POST   /api/v1/invoices/{id}/payment-link    # generate Razorpay link + send WA
GET    /api/v1/invoices/{id}/pdf             # receipt PDF
```

#### Reporting (OWNER)
```
GET    /api/v1/reports/overview?from=...&to=...
GET    /api/v1/reports/no-shows?from=...&to=...
GET    /api/v1/reports/followups?from=...&to=...
GET    /api/v1/reports/revenue?from=...&to=...
GET    /api/v1/reports/messages?from=...&to=...
```

#### Webhooks (no auth, signed)
```
POST   /api/v1/webhooks/wa/{provider}        # delivery + inbound reply
POST   /api/v1/webhooks/payments/{gateway}   # razorpay / cashfree
```

#### Admin (internal only, not exposed to clinics)
```
GET    /api/v1/admin/clinics
POST   /api/v1/admin/clinics
PUT    /api/v1/admin/clinics/{id}/status
GET    /api/v1/admin/audit?clinicId=...
```

### 8.3 Sample request/response

`POST /api/v1/appointments`
```json
{
  "patientId": "8f3a...",
  "doctorId": "21b0...",
  "scheduledAt": "2026-06-20T11:30:00+05:30",
  "durationMins": 30,
  "reason": "Cleaning",
  "source": "PHONE"
}
```

Response `201`:
```json
{
  "data": {
    "id": "0e91...",
    "status": "BOOKED",
    "scheduledAt": "2026-06-20T11:30:00+05:30",
    "patient": { "id": "8f3a...", "name": "Ananya R.", "phone": "+919xxxxx" },
    "doctor": { "id": "21b0...", "name": "Dr. Suresh" },
    "reminders": {
      "scheduled": ["2026-06-19T11:30:00+05:30", "2026-06-20T09:30:00+05:30"]
    }
  },
  "meta": { "requestId": "req_..." }
}
```

---

## 9. Background jobs and scheduler design

### 9.1 Job catalogue

| Job | Cadence | Purpose |
|---|---|---|
| `AppointmentReminderJob` | every 5 min | For each `BOOKED`/`CONFIRMED` appt where `now()` is at any of `clinic.reminder_offsets` before `scheduled_at`, send template `APPT_REMINDER`. |
| `FollowUpReminderJob` | every 30 min | For each `PENDING` `followup_task` due in any `clinic.followup_offsets` window, send `FOLLOWUP_DUE`. Increment `reminder_count`. |
| `NoShowMarkerJob` | every 15 min | Flip `ARRIVED?=false, status=BOOKED` past 60 min → `NO_SHOW`. Configurable per clinic. |
| `PaymentReminderJob` | daily 11:00 IST | Unpaid invoices > 48h → 1 reminder. > 7d → flag in owner report (no further messages). |
| `OwnerWeeklyReportJob` | Mon 08:30 IST | Send WA summary to clinic owner. |
| `MessageRetryJob` | every 2 min | Re-send `FAILED` messages with retryable error codes, up to 3 attempts. |
| `ConsentExpiryJob` | daily 02:00 IST | Optional — re-prompt opt-in after 12 months if message activity has paused. |
| `AuditRetentionJob` | weekly | Archive `audit_event` older than 7 years to cold storage. |

### 9.2 Implementation

- **Phase 1 (single instance):** Spring `@Scheduled` with `@SchedulerLock` (ShedLock + JDBC) so it remains safe when the app accidentally runs on 2 instances.
- **Phase 1.5 (multi-instance):** Migrate to Quartz cluster on PostgreSQL or move queue-based jobs to Redis Streams.
- **Phase 2 (high volume):** Move to a worker process (separate JAR, same codebase, `--spring.profiles.active=worker`) so web pods are not blocked.

### 9.3 Reliability rules

- Every outbound message goes through `MessagingService.send(...)`, which:
  1. Writes a `message_log` row with status `QUEUED`.
  2. Calls the provider.
  3. Updates status to `SENT` (with provider id) or `FAILED` (with error).
  4. Webhook later flips to `DELIVERED` / `READ` / `REPLIED`.
- Jobs never call providers directly — always through the service.
- All jobs are idempotent: re-running them produces no duplicate messages because of a unique `(related_entity, related_id, template_code, scheduled_for_minute_bucket)` constraint enforced in code.

---

## 10. WhatsApp integration design

### 10.1 Provider strategy

Do **not** use the WhatsApp Cloud API directly in v1. Use a BSP that has:
- Pre-approved healthcare appointment-reminder templates
- Indian INR billing
- Webhooks for delivery + replies
- Dashboard for template management

Recommended: **Interakt** or **Gupshup** for v1. Add a second provider in v1.5 to avoid lock-in.

### 10.2 Provider abstraction

```java
public interface MessagingProvider {
    SendResult send(SendRequest request);
    void verifyAndProcessWebhook(WebhookPayload payload);
    String name();      // "INTERAKT", "GUPSHUP"
}
```

`MessagingService` holds a `Map<String, MessagingProvider>` and picks based on `clinic_settings.wa_provider`. Default is `INTERAKT`.

### 10.3 Template catalogue (v1)

| Code | Category | Body | Variables |
|---|---|---|---|
| `APPT_CONFIRM` | Utility | "Hello {{1}}, your appointment with Dr. {{2}} at {{3}} is confirmed for {{4}} at {{5}}. Reply 1 to confirm, 2 to reschedule." | name, doctor, clinic, date, time |
| `APPT_REMINDER` | Utility | "Reminder: your appointment with Dr. {{1}} is on {{2}} at {{3}}. Please reach 10 min early." | doctor, date, time |
| `FOLLOWUP_DUE` | Utility | "Hello {{1}}, Dr. {{2}} has recommended a follow-up on {{3}}. Reply 1 to book a slot." | name, doctor, date |
| `PAYMENT_LINK` | Utility | "Your pending amount of ₹{{1}} for your visit on {{2}}. Pay here: {{3}}" | amount, date, link |
| `REVIEW_ASK` | Utility | "Thank you for visiting {{1}}. If you had a good experience, please rate us: {{2}}" | clinic, googleUrl |

All templates must be approved by Meta via the BSP before first send. Translations are separate template submissions.

### 10.4 Inbound reply handling

- BSP webhook → `POST /api/v1/webhooks/wa/{provider}` → signature verify → enqueue → `InboundMessageHandler`.
- Handler parses text:
  - `1`, `Y`, `Yes`, `yes` → if pending appointment → mark `CONFIRMED`. If pending follow-up → create a placeholder appointment in a "request" queue for the receptionist to slot.
  - `2`, `N`, `No`, `Reschedule` → mark appointment for receptionist attention.
  - Anything else → log, mark unread for receptionist.
- Replies inside the **24-hour customer service window** can be answered with free-form messages (no template cost).

### 10.5 Opt-in and DPDP

- A patient is added → consent is **NOT** assumed.
- First message sent only after either:
  - Receptionist marks "verbal consent given" with a timestamp + their user id (recorded in `consent` table), **or**
  - Patient self-opts-in via a WhatsApp keyword (rare in v1).
- Every outbound message includes the implicit footer "Reply STOP to unsubscribe" (BSP-managed).
- On STOP, all future messages for that patient are suppressed; recorded in `consent.status = REVOKED`.

---

## 11. Payment integration design

### 11.1 Gateway abstraction

```java
public interface PaymentGateway {
    PaymentLink createLink(CreateLinkRequest req);
    PaymentStatus fetchStatus(String gatewayRef);
    void verifyAndProcessWebhook(WebhookPayload payload);
    String name();      // "RAZORPAY", "CASHFREE"
}
```

### 11.2 Flow

1. Receptionist requests "Send payment link" for invoice `inv_X`.
2. `PaymentService.createLink(invoice)` → calls `RazorpayGateway.createLink` → stores `payment` row with status `INITIATED`, gateway reference, and link URL.
3. `MessagingService.send(PAYMENT_LINK, patient, {amount, date, link})`.
4. Patient pays. Razorpay calls our webhook.
5. Webhook handler verifies signature, finds `payment` by `gateway_ref`, marks `SUCCESS`, updates invoice `paid_inr`, recomputes `status`.
6. If invoice fully paid, suppress any further reminders.

### 11.3 Reconciliation

- Daily 23:00 IST job pulls Razorpay settled-transactions list and reconciles against `payment` rows.
- Mismatches surface in admin dashboard, not auto-corrected.

### 11.4 Pricing capture

- We do **not** take a cut of payments in v1 (avoids RBI aggregator licensing concerns). We may consider it in Phase 3 with appropriate licensing.

---

## 12. Multi-tenancy and security

### 12.1 Tenancy model

**Shared schema, shared database, discriminator column.** Every table (except `clinic`) has `clinic_id`. This is the simplest model and is appropriate for ≤200 clinics. Migrate to schema-per-tenant only if a large enterprise customer demands it.

Enforcement:
- `TenantContext` (ThreadLocal) is set by `JwtAuthenticationFilter` from the JWT `clinicId` claim.
- Hibernate `@Filter("tenant")` is enabled in all entity classes and activated in a `@Component @Aspect TenantFilterEnabler`.
- A custom `@TenantScoped` annotation on service methods asserts `TenantContext` is set; tests fail if any endpoint is missing it.
- Repository methods that take `clinicId` explicitly are preferred; the filter is defense in depth.

### 12.2 Auth

- Passwords: `bcrypt`, cost 12.
- JWT: access token 15 min, refresh token 7 days, rotated on use, family-tracked to detect theft.
- Claims: `sub` (userId), `clinicId`, `role`, `iat`, `exp`, `jti`.
- Refresh tokens stored hashed in `refresh_token` table; revoke-by-jti supported.

### 12.3 Authorization

- Spring Security `@PreAuthorize` with role expressions.
- Role matrix:

| Capability | OWNER | DOCTOR | RECEPTIONIST | ADMIN |
|---|---|---|---|---|
| View patients | ✅ | ✅ | ✅ | ✅ |
| Edit patients | ✅ | — | ✅ | ✅ |
| Delete patient (DPDP) | ✅ | — | — | ✅ |
| Create appointment | ✅ | ✅ | ✅ | ✅ |
| Complete visit | ✅ | ✅ | ✅ | — |
| Create invoice | ✅ | — | ✅ | ✅ |
| Send payment link | ✅ | — | ✅ | ✅ |
| View revenue reports | ✅ | — | — | ✅ |
| View no-show reports | ✅ | ✅ | ✅ | ✅ |
| Manage users | ✅ | — | — | ✅ |
| Configure clinic settings | ✅ | — | — | ✅ |
| View audit log | ✅ | — | — | ✅ |

### 12.4 Transport and storage security

- TLS 1.2+ everywhere, HSTS enabled.
- Database connection requires TLS.
- Secrets in env vars + Doppler/AWS Secrets Manager (never in repo, never in logs).
- WA API keys encrypted at rest using a KMS-backed envelope key.
- Backups encrypted, 30-day retention, restore drill quarterly.

### 12.5 Rate limiting

- Bucket4j per IP and per user for auth endpoints.
- Per-clinic outbound message rate limit (configurable; default 60/min) to prevent WA quality-rating damage.

---

## 13. DPDP and healthcare compliance

### 13.1 Lawful basis and notice

- We are a **Data Processor** for patient data; the clinic is the **Data Fiduciary**.
- Each clinic signs a Data Processing Agreement at onboarding.
- We provide a sample patient notice the clinic must display: short, plain Hindi/English, listing purposes (appointments, follow-ups, payments).

### 13.2 Consent

- Captured at three points: in-person form (clinic responsibility), WhatsApp opt-in keyword, web link.
- Stored in `consent` table with channel, purpose, status, evidence.
- Revocable from a public link in every WhatsApp message footer.

### 13.3 Data minimization rules

- No diagnosis text in WhatsApp message bodies.
- No prescription images sent via WhatsApp from the system in v1.
- `visit.notes` is free text the clinic can use, but our reporting never aggregates note content.
- We do not ask for Aadhaar / ABHA in v1.

### 13.4 Data subject rights

- Patient export: `GET /api/v1/patients/{id}/export` → JSON bundle of all data.
- Patient erasure: `DELETE /api/v1/patients/{id}` → hard-deletes patient + appointments + visits + messages + payments. Invoice retained for tax law (7 years) but `patient_id` is set to `NULL` and personal fields blanked. Action audit-logged with reason.

### 13.5 Breach handling

- Defined `SECURITY-INCIDENT.md` runbook (separate doc).
- 72-hour clinic + Data Protection Board notification process documented.
- Incident severity matrix and on-call rotation defined.

### 13.6 What we explicitly do not do

- No training of ML models on patient data.
- No data sale, ever, even aggregated/anonymized in v1.
- No cross-clinic data joins (each clinic sees only its own).
- No transfer of data outside India in v1 (DB, files, backups all in `ap-south-1` / Mumbai region).

---

## 14. Observability, logging, and audit

### 14.1 Logs

- Structured JSON via Logback.
- Every log line carries `requestId`, `clinicId`, `userId`, `action`.
- Patient PII (name, phone) is **never** logged at INFO/DEBUG; only patient `id`.
- Logs shipped to Grafana Loki (free tier) or BetterStack.

### 14.2 Metrics

- Micrometer → Prometheus.
- Key business metrics published as gauges/counters:
  - `clinicpulse.messages.sent{template, status}`
  - `clinicpulse.followups.due`
  - `clinicpulse.followups.recovered`
  - `clinicpulse.payments.received{gateway, status}`
  - `clinicpulse.appointments.no_shows`

### 14.3 Tracing

- OpenTelemetry SDK, exported to Grafana Tempo (later). Skip in v1.0.

### 14.4 Audit log

- See `audit_event` schema in §7.
- Every read of a patient by a non-owner role is audit-logged.
- Audit is append-only; no UPDATE/DELETE permitted in DB role for the app user.

---

## 15. Testing strategy

| Layer | Tool | Target |
|---|---|---|
| Unit | JUnit 5 + Mockito | All services, ≥80% line coverage on domain logic |
| Integration | Spring Boot Test + Testcontainers (Postgres) | All repositories and controllers, real DB |
| Contract | WireMock for Razorpay + BSP | Verify request shape, signature handling |
| End-to-end | REST Assured | Smoke flows: book appointment → send → reply → recover follow-up |
| Multi-tenancy | Dedicated test suite | Asserts that user from clinic A cannot read any row from clinic B |
| Security | OWASP ZAP baseline scan in CI | Catch common misconfigurations |

CI gate: PR cannot merge if integration tests fail or multi-tenancy suite fails.

---

## 16. Frontend design (deferred to Phase 2)

We will build the receptionist UI **after** the backend is usable via Postman and one concierge pilot is live.

### 16.1 Tech

- Next.js 15 (App Router) + React + TypeScript
- Tailwind CSS + Headless UI
- TanStack Query for server state
- Auth via JWT in HTTP-only cookie (recommended) or memory + refresh
- Mobile-first design: 360×640 target

### 16.2 Screen list (v1)

1. Login
2. Today (receptionist landing): today's queue, +Appointment FAB
3. Patient search + add
4. Patient detail + timeline
5. Appointment detail
6. Visit completion modal
7. Follow-ups due
8. Invoices & payments
9. Owner report
10. Settings (templates, hours, users)

UX north star: **a receptionist with 1 hour of training can run a 40-patient day.**

---

## 17. Infrastructure and DevOps

### 17.1 Environments

| Env | Purpose | Hosting |
|---|---|---|
| `local` | Dev | Docker Compose (Postgres + Redis + app) |
| `staging` | Internal demos, CI deploys | Render or Railway, free/cheap tier |
| `production` | Real clinics | Render (Phase 1) → AWS Mumbai ECS Fargate + RDS (Phase 2) |

### 17.2 Repo layout

```
clinic-pulse/
├── backend/                  # Spring Boot monolith (this design)
│   ├── pom.xml
│   ├── Dockerfile
│   └── src/
├── frontend/                 # Next.js (Phase 2)
├── infra/                    # Terraform when we move to AWS
├── ops/                      # Runbooks, scripts
└── docs/
    ├── DESIGN.md             # this file
    ├── API.md                # auto-generated from OpenAPI
    ├── DPDP.md               # compliance specifics
    └── ONBOARDING.md         # clinic onboarding checklist
```

### 17.3 CI/CD

- GitHub Actions.
- On PR: build, unit, integration, lint, security scan, OpenAPI diff.
- On merge to `main`: deploy to staging.
- Manual approve → deploy to production.
- Database migrations: Flyway runs on app boot; in prod, migrations are reviewed in PR.

### 17.4 Backups and DR

- RDS automated backups, 30-day retention.
- Daily logical dump (`pg_dump`) to S3 with 90-day retention.
- DR drill: monthly restore to a scratch environment, verify smoke tests pass.

---

## 18. Cost and unit economics

### 18.1 Per-clinic monthly cost (worked, at small scale)

Assume 750 appointments/month, 3 WA messages each = 2,250 utility messages.

| Item | Cost |
|---|---|
| WhatsApp utility messages (2,250 × ₹0.16) | ₹360 |
| BSP platform fee (allocated, ~10 clinics share Interakt Growth ₹2,499) | ₹250 |
| Hosting (allocated; Render Pro $25/mo / 10 clinics × ₹85) | ₹212 |
| Postgres (allocated) | ₹100 |
| Payment gateway markup (we don't take a cut in v1) | ₹0 |
| Support + ops (allocated) | ₹400 |
| **Total COGS / clinic** | **~₹1,322** |

### 18.2 Pricing tiers (revised from the original plan)

| Tier | Price | Target | Gross margin |
|---|---|---|---|
| **Pilot** (30 days) | ₹999 | First touch | break-even |
| **Standard** (self-serve) | ₹3,499/mo | Most clinics | ~62% |
| **Growth** (managed) | ₹6,999/mo | Owner who wants done-for-you recall campaigns | ~80% |
| **Pro** (multi-doctor + specialty pack) | ₹12,999/mo | 3+ chair clinics | ~85% |

Setup / data-migration: ₹5,000 one-time after pilot.

### 18.3 Target year-1 outcome

- 10 paying clinics × ₹4,500 ARPU = **₹45,000 MRR**
- COGS ~₹13,000
- Gross profit ~₹32,000/mo
- Founder salary intentionally zero in year 1

---

## 19. Phased build plan — first 120 days

| Phase | Days | Goal | Output |
|---|---|---|---|
| **0. Validation** | 1–14 | Talk to 25–30 clinics; quantify pain | Filled CRM, 5 pilot commits |
| **1. Concierge** | 15–44 | Run 3 clinics manually | Recovered-revenue report per clinic; learnt workflow |
| **2. Backend MVP** | 30–80 (overlaps) | Ship backend covering 13 v1 capabilities | Postman-usable API, internal admin UI |
| **3. Frontend MVP** | 60–100 | Receptionist UI for the 3 pilots | Mobile web app, live for 3 clinics |
| **4. First 10 paying** | 90–150 | Founder-led sales | 10 paying clinics, ₹45K MRR |

The user's question — *"is it OK to build the backend first?"* — answer: **yes, and it is the correct order**, because the concierge pilots can run on Postman + Google Sheets while the backend takes shape.

---

## 20. Backend-first implementation roadmap (sprint level)

Each sprint = 1 calendar week. Solo developer assumed.

### Sprint 0 — Project scaffolding (3 days)
- Initialise `backend/` Spring Boot 3 Maven project, Java 21
- Add Spring Web, Data JPA, Security, Validation, Actuator, springdoc, Flyway, Testcontainers
- Set up `application.yaml` per profile (`local`, `staging`, `prod`)
- Docker Compose: app + Postgres 16 + Redis
- Skeleton CI in GitHub Actions: build + test on PR
- Decide repo location: `c:\Users\320219651\spring-projects-hub\clinic-pulse\backend`

**Definition of done:** `./mvnw spring-boot:run` returns 200 on `/actuator/health`.

### Sprint 1 — Auth & clinic & multi-tenancy
- Tables: `clinic`, `clinic_settings`, `app_user`, `subscription`
- Endpoints: `/auth/login`, `/auth/refresh`, `/auth/me`, `/clinic`, `/clinic/settings`, `/users`
- `TenantContext`, JWT filter, Hibernate tenant filter
- Multi-tenancy test suite (must be green before any further sprint merges)
- Seed admin user, seed 1 demo clinic

**DoD:** A request with clinic A's JWT cannot read any row tagged with clinic B.

### Sprint 2 — Patient + Consent + Import
- Tables: `patient`, `consent`
- Endpoints listed in §8 for patients and consents
- CSV import endpoint with column mapping + validation report
- Phone normalization utility (E.164, +91 default)
- Unit + integration tests

**DoD:** Can import 200 patients from a Practo CSV export; consent capture and revoke work end-to-end.

### Sprint 3 — Appointments + Visits + status lifecycle
- Tables: `appointment`, `visit`
- Endpoints + state machine for appointment status
- Visit completion creates an audit event
- Owner can see today's list filtered by status
- Tests for invalid transitions (cannot go `COMPLETED → BOOKED`)

**DoD:** Concierge use case "book → confirm → arrive → complete → set follow-up" works via Postman.

### Sprint 4 — Comms module (WhatsApp scaffolding)
- Tables: `message_template`, `message_log`
- `MessagingProvider` interface + `InteraktProvider` implementation
- `MessagingService.send()` with idempotency
- Webhook endpoint for delivery + replies, signature verification
- Manual `/appointments/{id}/send-reminder` endpoint
- Seed 5 default templates

**DoD:** Sending `APPT_CONFIRM` for a test appointment delivers a real WhatsApp message in staging and the delivery webhook updates `message_log.status`.

### Sprint 5 — Scheduler + automated reminders
- ShedLock + `AppointmentReminderJob` + `NoShowMarkerJob`
- Idempotency keys to prevent duplicate sends within a bucket
- Reminder offsets driven by `clinic_settings`
- Metrics exposed on `/actuator/prometheus`

**DoD:** A booked appointment 25h in the future receives exactly one T-24h reminder and one T-2h reminder, even after job restarts.

### Sprint 6 — Follow-ups
- Tables: `followup_task`
- Endpoints + `FollowUpReminderJob`
- Auto-creation of `followup_task` on visit completion if `followup_date` set
- Reply handling: "1" on follow-up reminder creates a "request" row for receptionist (no auto-booking in v1)

**DoD:** Visit completed today with `followup_date = +30d` → reminder fires at T-3, T-1, T-0 → patient replies "1" → receptionist sees it in queue.

### Sprint 7 — Billing + Razorpay
- Tables: `invoice`, `invoice_line`, `payment`
- Endpoints listed in §8 for invoices/payments
- `RazorpayGateway` implementation
- Payment link generation + WA `PAYMENT_LINK` template
- Webhook handling + reconciliation job
- PDF receipt generation (`openhtmltopdf` or `iText community`)

**DoD:** Receptionist creates invoice, sends link, simulates Razorpay payment in test mode, invoice flips to `PAID`, no further reminders sent.

### Sprint 8 — Reporting + owner weekly summary
- Read-only `ReportService` with SQL aggregations
- `OwnerWeeklyReportJob`
- Endpoints `/reports/overview`, `/reports/no-shows`, `/reports/followups`, `/reports/revenue`, `/reports/messages`

**DoD:** Owner receives a real WA summary on a chosen Monday with non-trivial numbers.

### Sprint 9 — Audit, DPDP, hardening
- `audit_event` writes from a Spring AOP aspect on every patient/visit/invoice access
- Patient export endpoint
- Patient erasure endpoint with invoice anonymization
- Rate limiting on auth + per-clinic outbound messages
- Run OWASP ZAP baseline; fix all `High` findings

**DoD:** Independent reviewer can fetch+verify a patient bundle, then delete it, and confirm via audit log.

### Sprint 10 — Internal admin UI + onboarding workflow
- Tiny internal-only Next.js admin (NOT for clinics) to add a clinic, view audit logs, see message volumes
- Onboarding script: create clinic → create owner → run CSV import → activate subscription

**DoD:** A new clinic can be onboarded by the founder in <30 minutes end-to-end without DB access.

After Sprint 10 the backend is feature-complete for v1.0. Phase 2 is the receptionist Next.js frontend.

---

## 21. Risk register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | WhatsApp template rejected by Meta | High | High | Use BSP-pre-approved healthcare templates; keep messages neutral, no diagnosis |
| 2 | BSP account quality drop → throttling | Medium | High | Strict opt-in, per-clinic rate limit, monitor `quality_rating` daily |
| 3 | Free competitor (Eka Care) undercuts us | High | Medium | Compete on managed service + dental specialty depth, not on feature parity |
| 4 | DPDP rules tighten unexpectedly | Medium | Medium | Compliance-first design from day 1; legal review before Phase 3 |
| 5 | Clinic churn after 2 months | Medium | High | Concierge phase proves ROI before onboarding; weekly owner report keeps value visible |
| 6 | Founder bandwidth (solo dev + sales) | High | High | Phased plan; do not start frontend until backend is operational |
| 7 | Receptionist resistance | High | Medium | UX designed for them; founder shadows reception for 1 day at first 3 clinics |
| 8 | Razorpay account onboarding delays | Medium | Low | Apply on Day 1; have manual UPI fallback |
| 9 | Postgres scaling | Low (in v1) | Low | Single db handles 200+ clinics easily; revisit at scale |
| 10 | Cash-culture clinics under-report revenue | High | Low | Support "Cash, not billed" visit option; don't force every visit through invoicing |

---

## 22. Glossary

| Term | Meaning |
|---|---|
| **ABDM** | Ayushman Bharat Digital Mission — India's national digital health framework |
| **ABHA** | Ayushman Bharat Health Account — a citizen's national health ID |
| **HFR / HPR** | Health Facility Registry / Healthcare Professionals Registry (under ABDM) |
| **BSP** | Business Solution Provider — Meta-authorized reseller of WhatsApp Business Platform |
| **DPDP** | Digital Personal Data Protection Act, 2023 (India) |
| **HIP / HIU** | Health Information Provider / User (ABDM roles) |
| **ICP** | Ideal Customer Profile |
| **MRR** | Monthly Recurring Revenue |
| **OPD** | Outpatient Department |
| **PHI** | Protected Health Information |
| **PII** | Personally Identifiable Information |
| **RCT** | Root Canal Treatment (dental) |
| **RMP** | Registered Medical Practitioner |
| **Utility template** | WhatsApp template category for transactional messages (appointment, payment) |

---

## Appendix A — Open decisions awaiting input

1. **Specialty for v1:** Dental confirmed as primary. Confirm whether physiotherapy is v1.1 or remains an explicit Phase 3.
2. **BSP choice:** Interakt vs Gupshup vs AiSensy — needs a 1-day spike on pricing + template approval speed.
3. **Hosting:** Render (Phase 1) vs Railway vs direct AWS Mumbai — Render recommended for solo-dev simplicity.
4. **Brand name:** "ClinicPulse" is the working name; trademark check pending.
5. **First city:** Bengaluru recommended (high density of dental clinics, founder proximity assumed). Confirm.

## Appendix B — Reference repo files in this workspace

These existing projects in `spring-projects-hub/` can be cannibalised for patterns:

- [healthcare-hospital-system](healthcare-hospital-system/) — Spring Boot microservices, gives proven patterns for entity mapping, exception handling, Docker compose, init scripts. Treat as a code library, not as architecture to copy.
- [fhir-api-application](fhir-api-application/) — Spring Boot + GraphQL, not used in v1 but the JPA conventions and `application.yaml` style transfer cleanly.
- [spring-rag-app](spring-rag-app/) — only relevant when Phase 3 AI summaries are added.

The new project should live at:
`c:\Users\320219651\spring-projects-hub\clinic-pulse\backend`

---

**End of design document.**
