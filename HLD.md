# SmartSure — High Level Design (HLD)

## 1. System Overview

SmartSure is a microservices-based insurance management platform. It allows users to buy vehicle and home insurance policies, file and track claims, and enables admins to manage users, policies, claims, and generate reports.

---

## 2. Architecture Style

**Microservices + Event-Driven Architecture**

- Each domain (Identity, Policy, Claims, Admin) is an independent service with its own database
- Services communicate asynchronously via RabbitMQ using the publish/subscribe pattern
- A single API Gateway (Ocelot) is the entry point for all client requests
- No direct service-to-service HTTP calls — all cross-service data sync happens through events

---

## 3. System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
│                   Angular 19 SPA (Port 4200)                    │
│         User Portal  │  Admin Portal  │  Auth Pages             │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTP (JWT Bearer)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway — Ocelot (Port 5083)             │
│              Route-based reverse proxy + Rate Limiting          │
└────┬──────────────┬──────────────┬──────────────┬──────────────-┘
     │              │              │              │
     ▼              ▼              ▼              ▼
┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│Identity │  │ Policy   │  │ Claims   │  │  Admin   │  │ AI (RAG) │
│API      │  │ API      │  │ API      │  │  API     │  │ Service  │
│:5001    │  │ :5152    │  │ :5008    │  │  :5113   │  │ :5000    │
└────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │            │             │              │              │
     ▼            ▼             ▼              ▼              ▼
┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│Identity │  │ Policy   │  │ Claims   │  │  Admin   │  │ Ollama   │
│   DB    │  │   DB     │  │   DB     │  │   DB     │  │ LLM      │
└─────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘

                    ┌─────────────────────┐
                    │  RabbitMQ (Port 5672)│
                    │  MassTransit Pub/Sub │
                    └─────────────────────┘
                    ↑ Published by services
                    ↓ Consumed by services
```

---

## 4. Services

### 4.1 Identity Service (Port 5001)
Handles all authentication and user management.

**Responsibilities:**
- User registration and login (email/password + Google OAuth)
- JWT token generation using RSA private key
- OTP-based email verification
- Token blacklisting (logout)
- Admin user management

**Key Events Published:**
- `UserRegisteredEvent` → consumed by Admin, Policy
- `UserLoggedInEvent` → consumed by Admin (audit)
- `UserRoleChangedEvent` → consumed by Admin

**Database:** `SmartSure_IdentityDB`
- Tables: `Users`, `Roles`, `OtpCodes`

---

### 4.2 Policy Service (Port 5152)
Handles insurance catalog, policy lifecycle, and payments.

**Responsibilities:**
- Insurance type and subtype catalog management
- Policy purchase (vehicle & home) with IDV calculation
- Policy cancellation
- Premium and payment recording
- Vehicle and home details storage

**Key Events Published:**
- `PolicyCreatedEvent` → consumed by Admin, Claims
- `PolicyCancelledEvent` → consumed by Admin, Claims

**Key Events Consumed:**
- `UserRegisteredEvent` → mirrors user for policy holder lookup

**Database:** `SmartSure_PolicyDB`
- Tables: `Policies`, `InsuranceTypes`, `InsuranceSubTypes`, `VehicleDetails`, `HomeDetails`, `Payments`, `PolicyDocuments`

---

### 4.3 Claims Service (Port 5008)
Handles the full claim lifecycle.

**Responsibilities:**
- Claim initiation and validation against active policies
- Claim status tracking (Submitted → Under Review → Approved/Rejected)
- Claim history recording
- Document upload (MEGA storage)
- Validates claim amount does not exceed IDV
- Blocks claims on cancelled policies

**Key Events Published:**
- `ClaimSubmittedEvent` → consumed by Admin
- `ClaimApprovedEvent` → consumed by Admin
- `ClaimRejectedEvent` → consumed by Admin
- `ClaimStatusChangedEvent` → consumed by Admin

**Key Events Consumed:**
- `PolicyCreatedEvent` → mirrors policy into `ValidPolicies` table
- `PolicyCancelledEvent` → marks `ValidPolicy` as Cancelled

**Database:** `SmartSure_ClaimsDB`
- Tables: `Claims`, `ClaimHistory`, `ClaimDocuments`, `ValidPolicies`

---

### 4.4 Admin Service (Port 5113)
Read-side aggregator for the admin portal. Maintains mirrored data from all other services.

**Responsibilities:**
- Admin dashboard KPIs (total claims, policies, revenue, active users)
- Policy management (view all, cancel)
- Claims management (approve, reject, mark under review)
- User management (view, deactivate)
- Report generation (CSV + PDF)
- Audit log tracking
- Email notifications on claim approval/rejection and policy cancellation

**Key Events Consumed:**
- `UserRegisteredEvent`, `UserLoggedInEvent`, `UserRoleChangedEvent`
- `PolicyCreatedEvent`, `PolicyCancelledEvent`
- `ClaimSubmittedEvent`, `ClaimApprovedEvent`, `ClaimRejectedEvent`, `ClaimStatusChangedEvent`

**Database:** `SmartSure_AdminDB`
- Tables: `AdminUsers`, `AdminPolicies`, `AdminClaims`, `AuditLogs`, `Reports`

---

### 4.5 API Gateway (Port 5083)
Ocelot-based reverse proxy.

**Responsibilities:**
- Single entry point for all frontend requests
- Route-based forwarding to downstream services
- Rate limiting (configurable per route)
- No authentication — JWT validation is done by each downstream service

**Route Mapping:**

| Upstream Path | Downstream Service |
|---|---|
| `/api/auth/**` | Identity API :5001 |
| `/api/policy/**` | Policy API :5152 |
| `/api/dashboard` | Policy API :5152 |
| `/api/claims/**` | Claims API :5008 |
| `/api/admin/**` | Admin API :5113 |
| `/api/ai/**` | AI Service :5000 |

---

### 4.6 AI Service (Port 5000)
Python Flask-based RAG (Retrieval-Augmented Generation) chatbot.

**Responsibilities:**
- Upload and index `.txt` / `.md` knowledge base documents
- Chunk documents and build a local token-based search index
- Retrieve relevant context for user questions via cosine similarity
- Generate answers using a local Ollama LLM (llama3.2:1b)
- Provide document management (list, delete)

**Tech Stack:** Python 3.10+, Flask, Ollama, Local file-based index

**API Endpoints:**
- `GET /api/ai/health` → health check + model info
- `POST /api/ai/upload` → upload and index a document
- `POST /api/ai/ask` → ask a question (RAG pipeline)
- `GET /api/ai/documents` → list indexed documents
- `DELETE /api/ai/documents/{id}` → remove a document

---

## 5. Event Flow Diagrams

### 5.1 User Registration
```
User registers → Identity API
  → saves to IdentityDB
  → publishes UserRegisteredEvent
      → Admin Service: mirrors user into AdminUsers
      → Policy Service: stores user as PolicyHolder
  → sends verification email (Gmail SMTP)
```

### 5.2 Buy Policy
```
User buys policy → Policy API
  → calculates IDV and premium
  → saves to PolicyDB
  → publishes PolicyCreatedEvent
      → Admin Service: mirrors into AdminPolicies
      → Claims Service: mirrors into ValidPolicies
```

### 5.3 File a Claim
```
User files claim → Claims API
  → validates policy exists in ValidPolicies
  → validates policy is not Cancelled
  → validates claim amount ≤ IDV
  → saves claim to ClaimsDB
  → publishes ClaimSubmittedEvent
      → Admin Service: mirrors into AdminClaims
```

### 5.4 Admin Approves/Rejects Claim
```
Admin approves/rejects → Admin API
  → updates AdminClaim status
  → publishes ClaimApprovedEvent / ClaimRejectedEvent
      → Admin Service (self): sends email to user
      → Claims Service: updates Claim status
```

### 5.5 Admin Cancels Policy
```
Admin cancels → Policy API (PUT /api/policy/policies/{id}/cancel)
  → updates Policy status to Cancelled in PolicyDB
  → publishes PolicyCancelledEvent
      → Admin Service: updates AdminPolicy status + sends email to user
      → Claims Service: updates ValidPolicy status to Cancelled
```

---

## 6. Security Design

| Concern | Approach |
|---|---|
| Authentication | JWT with RSA-256 (asymmetric keys) |
| Key storage | RSA keys in `.env` file, never committed to source control |
| Authorization | Role-based (`User`, `Admin`) enforced per endpoint |
| Token invalidation | In-memory blacklist on logout |
| CORS | Restricted to `http://localhost:4200` |
| Secrets | All credentials in `.env`, empty strings in `appsettings.json` |

---

## 7. Database Design (Per Service)

Each service owns its database — no shared DB, no cross-DB joins in application code.

```
SmartSure_IdentityDB    → Users, Roles, OtpCodes
SmartSure_PolicyDB      → Policies, InsuranceTypes, InsuranceSubTypes,
                          VehicleDetails, HomeDetails, Payments
SmartSure_ClaimsDB      → Claims, ClaimHistory, ClaimDocuments, ValidPolicies
SmartSure_AdminDB       → AdminUsers, AdminPolicies, AdminClaims,
                          AuditLogs, Reports
```

Cross-service data consistency is maintained through **event-driven mirroring** — each service keeps a local copy of the data it needs from other services, updated via RabbitMQ consumers.

---

## 8. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | Angular 19, Bootstrap 5, Bootstrap Icons |
| API Gateway | ASP.NET Core 10, Ocelot |
| Backend Services | ASP.NET Core 10, Clean Architecture |
| ORM | Entity Framework Core 10 |
| Database | SQL Server 2019+ / LocalDB |
| Message Broker | RabbitMQ + MassTransit 8 |
| Authentication | JWT (RSA-256), Google OAuth 2.0 |
| Payment | Razorpay (Test Mode) |
| Email | MailKit (Gmail SMTP) |
| File Storage | MEGA Cloud Storage |
| PDF Generation | QuestPDF |
| AI | Python Flask, Ollama (llama3.2:1b), RAG pipeline |
| Logging | Serilog (file + console) |
| Containerization | Docker (RabbitMQ only) |

---

## 9. IDV Calculation Logic

IDV (Insured Declared Value) is calculated on the frontend at policy purchase time and back-calculated from the stored premium when displaying.

**Vehicle IDV:**
```
Age = Current Year - Manufacturing Year
Depreciation:
  < 1 yr  → 5%
  < 2 yr  → 15%
  < 3 yr  → 20%
  < 4 yr  → 30%
  < 5 yr  → 40%
  ≥ 5 yr  → 50%

IDV = Listed Price × (1 - Depreciation)
Annual Premium = IDV × 2%
High Mileage Surcharge (> 15,000 km/yr) = +20%
```

**Home IDV:**
```
Age = Current Year - Year Built
Depreciation:
  < 5 yr  → 0%
  < 10 yr → 10%
  < 20 yr → 20%
  < 30 yr → 30%
  ≥ 30 yr → 40%

IDV = Property Value × (1 - Depreciation)
Annual Premium = IDV × 0.1%
Security System Discount = -10%
```

---

## 10. Deployment (Local)

```
Docker Desktop (RabbitMQ container)
    +
SQL Server (local, Windows Auth)
    +
5 × dotnet run (Identity, Policy, Claims, Admin, Gateway)
    +
npx ng serve (Frontend)
```

All managed via `start-all.ps1` / `stop-all.ps1` scripts.
