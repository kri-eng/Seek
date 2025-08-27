# Seek — Caregiver/Agency Marketplace (Code Private)

> **Note:** The source code for this project is intentionally private. This README documents the product, architecture, decisions, and includes a starter scaffold so stakeholders and collaborators can understand the project without direct code exposure.

---

## Overview

**Seek** is a two-sided marketplace that connects **home care agencies** with **qualified caregivers**.  

- **Caregivers (CGs):** Create profiles, verify credentials, list availability, apply for jobs, track shifts & documents.  
- **Agencies:** Post jobs with requirements (skills, languages, schedules, pay ranges, POC hours), review matches, hire caregivers, schedule shifts, and track compliance.  
- **Admins:** Manage organizations, monitor compliance, review audits, and oversee EVV records.  

---

## Features

- Secure authentication (JWT, RBAC by role)  
- Caregiver profiles (skills, certifications, availability, radius, language support)  
- Job posting and applications  
- Matching engine (rules-based scoring: skills, schedule fit, distance, language, pay alignment)  
- EVV (Electronic Visit Verification) events with check-in/out  
- Document upload & compliance tracking (expiry reminders, audit logs)  
- Notifications (email in MVP; SMS/push later)  
- Audit logs for sensitive actions  

---

## Project Structure

```bash
seek/
├─ src/
│  ├─ index.ts          # Entry point
│  ├─ server.ts         # Express app setup
│  ├─ config.ts         # Env loader
│  ├─ routes/           # REST routes
│  │   ├─ auth.routes.ts
│  │   ├─ caregivers.routes.ts
│  │   ├─ jobs.routes.ts
│  │   └─ shifts.routes.ts
│  ├─ controllers/      # Request handlers
│  ├─ services/         # Business logic
│  ├─ db/               # Prisma/Knex setup
│  ├─ middleware/       # Auth, logging, validation
│  └─ utils/            # Helpers (logger, etc.)
├─ prisma/
│  └─ schema.prisma     # Database schema
├─ package.json
├─ tsconfig.json
├─ .env.example
└─ README.md
