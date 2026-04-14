# SmartSure — Insurance Management Platform

A full-stack microservices-based insurance management system built with Angular 19, ASP.NET Core 10, SQL Server, RabbitMQ, and MassTransit.

---

## Architecture Overview

```
Angular SPA (:4200)
      │
      ▼
Ocelot API Gateway (:5083)
      │
      ├── Identity API  (:5001)  →  IdentityDB
      ├── Policy API    (:5152)  →  PolicyDB
      ├── Claims API    (:5008)  →  ClaimsDB
      ├── Admin API     (:5113)  →  AdminDB
      └── AI Service    (:5000)  →  Ollama LLM

All services communicate asynchronously via RabbitMQ + MassTransit
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Angular 19, Bootstrap 5 |
| API Gateway | ASP.NET Core 10, Ocelot |
| Backend | ASP.NET Core 10, Clean Architecture |
| ORM | Entity Framework Core 10 |
| Database | SQL Server (LocalDB or full) |
| Message Broker | RabbitMQ + MassTransit 8 |
| Auth | JWT RS256, Google OAuth 2.0 |
| Email | MailKit (Gmail SMTP) |
| File Storage | MEGA Cloud |
| AI | Python Flask, Ollama (llama3.2:1b) |
| Logging | Serilog |
| Payment | Razorpay (Test Mode) |

---

## Service Ports

| Service | Port |
|---------|------|
| Frontend | http://localhost:4200 |
| API Gateway | http://localhost:5083 |
| Identity API | http://localhost:5001 |
| Policy API | http://localhost:5152 |
| Claims API | http://localhost:5008 |
| Admin API | http://localhost:5113 |
| AI Service | http://localhost:5000 |
| RabbitMQ Management | http://localhost:15672 |

---

## Prerequisites

| Tool | Version |
|------|---------|
| .NET SDK | 10.0+ |
| Node.js | 18+ |
| SQL Server | 2019+ or LocalDB |
| Docker Desktop | Latest (for RabbitMQ) |
| Python | 3.10+ (for AI service) |
| Ollama | Latest |

---

## Setup Guide

### Step 1 — Install Frontend Dependencies

```bash
cd frontend
npm install
cd ..
```

### Step 2 — Start RabbitMQ

```bash
docker-compose up -d
```

RabbitMQ Management UI → http://localhost:15672
- Username: `smartsure`
- Password: `smartsure_dev`

### Step 3 — Configure Environment

Copy `.env.example` to `.env` and fill in your credentials:

```bash
copy .env.example .env
```

Key values to set:

```env
# SQL Server
ConnectionStrings__IdentityDb=Server=(localdb)\MSSQLLocalDB;Database=SmartSure_IdentityDB;Trusted_Connection=True;TrustServerCertificate=True
ConnectionStrings__PolicyDb=Server=(localdb)\MSSQLLocalDB;Database=SmartSure_PolicyDB;Trusted_Connection=True;TrustServerCertificate=True
ConnectionStrings__ClaimsDb=Server=(localdb)\MSSQLLocalDB;Database=SmartSure_ClaimsDB;Trusted_Connection=True;TrustServerCertificate=True
ConnectionStrings__AdminDb=Server=(localdb)\MSSQLLocalDB;Database=SmartSure_AdminDB;Trusted_Connection=True;TrustServerCertificate=True

# JWT RSA Keys
JwtSettings__PrivateKeyContent=...
JwtSettings__PublicKeyContent=...

# Google OAuth
GoogleAuth__ClientId=...
GoogleAuth__ClientSecret=...

# Gmail SMTP
MailSettings__UserName=your@gmail.com
MailSettings__Password=your-app-password

# MEGA Storage
MegaOptions__Email=your@email.com
MegaOptions__Password=yourpassword

# Razorpay (Test Mode)
Razorpay__KeyId=rzp_test_...
Razorpay__KeySecret=...
```

> Databases and tables are created automatically on first service startup — no manual SQL needed.

### Step 4 — Generate RSA Keys (first time only)

```bash
dotnet run --project backend/tools/GenKeys/GenKeys.csproj
```

Copy the output into `.env` under `JwtSettings__PrivateKeyContent` and `JwtSettings__PublicKeyContent`.

### Step 5 — Run All Services

```powershell
.\start-all.ps1
```

If you get a script execution error:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

Stop all services:

```powershell
.\stop-all.ps1
```

---

## Default Admin Account

| Field | Value |
|-------|-------|
| Email | admin@smartsure.com |
| Password | Admin@123 |

---

## Project Structure

```
SmartSure/
├── frontend/                    # Angular 19 SPA
├── backend/
│   ├── gateway/                 # Ocelot API Gateway
│   ├── services/
│   │   ├── identity/            # Auth, JWT, Google OAuth
│   │   ├── policy/              # Policies, Catalog, Razorpay
│   │   ├── claims/              # Claims, Documents
│   │   └── admin/               # Dashboard, Reports, Audit
│   └── shared/                  # Shared contracts, JWT, MassTransit
├── ollama_rag_app/              # Python AI service
├── .env.example                 # Credentials template
├── docker-compose.yml           # RabbitMQ container
├── start-all.ps1                # Start all services
└── stop-all.ps1                 # Stop all services
```

---

## Swagger / API Docs

| Service | URL |
|---------|-----|
| Identity | http://localhost:5001/swagger |
| Policy | http://localhost:5152/swagger |
| Claims | http://localhost:5008/swagger |
| Admin | http://localhost:5113/swagger |

---

## Design Documents

| Document | Description |
|----------|-------------|
| `HLD.md` | High Level Design — architecture, services, event flows |
| `LLD.md` | Low Level Design — entities, interfaces, endpoints |
| `SmartSure-HLD-Diagram.md` | HLD Mermaid diagram script |
| `SmartSure-LLD-Diagram.md` | LLD Mermaid diagram script |
| `SmartSure-RabbitMQ-Flow.md` | RabbitMQ event flow diagram |
| `SmartSure-ER-dbdiagram.md` | ER diagram (dbdiagram.io DBML) |
| `SmartSure-Usecase-Diagram.md` | Use case diagram |

---

## Security Notice

> All secrets are stored in `.env` — never commit this file to version control.
> For production, use a dedicated secret manager (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault).
