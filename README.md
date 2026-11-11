<p align="center">
  <img src="docs/banner.svg" alt="HRM Platform banner: multi-company HR backend for attendance, shifts, breaks and leave" width="100%">
</p>

<h1 align="center">HRM Platform</h1>

<p align="center">
  A multi-company HR backend: organisations, departments and shifts, daily attendance with late detection and breaks, and leave management, served as a versioned REST API for web and mobile clients.
</p>

<p align="center">
  <img alt="Status: Live" src="https://img.shields.io/badge/status-live-brightgreen?style=flat-square">
  <img alt="Platform: REST API" src="https://img.shields.io/badge/platform-REST%20API-blue?style=flat-square">
  <img alt="NestJS 11" src="https://img.shields.io/badge/NestJS-11-E0234E?style=flat-square&logo=nestjs&logoColor=white">
  <img alt="TypeScript" src="https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white">
  <img alt="MySQL" src="https://img.shields.io/badge/MySQL-TypeORM-00758F?style=flat-square&logo=mysql&logoColor=white">
  <img alt="OpenAPI 3" src="https://img.shields.io/badge/OpenAPI-3.0-85EA2D?style=flat-square&logo=swagger&logoColor=black">
</p>

<p align="center">
  <a href="https://hrm.everbuild.cloud/docs"><b>Live API docs</b></a> ·
  <a href="https://portfolio.everbuild.cloud"><b>Portfolio</b></a> ·
  <a href="https://linkedin.com/in/dafallah-ibrahim"><b>LinkedIn</b></a>
</p>

> This is a showcase repository. The source code is private (team project in a private GitHub organisation). It documents what the product does, how it's built and my role. Happy to walk through the code in an interview.

## Screenshots

All screenshots are real captures of the live, public Swagger UI at [hrm.everbuild.cloud/docs](https://hrm.everbuild.cloud/docs). The API has no public front end, so the docs are the public face of the product. No requests were executed against it.

<p align="center">
  <img src="docs/screenshots/api-docs-overview.png" alt="Swagger UI landing page for the HRM API, showing the OpenAPI 3.0 badge, the Authorize button and the first endpoint groups" width="100%">
</p>

<p align="center">
  <img src="docs/screenshots/attendance-endpoints.png" alt="Attendance endpoint group: check-in, check-out, per-user, department and company views, live status, today, history, shift info, break start and end, and daily and weekly summaries" width="49%">
  <img src="docs/screenshots/leave-management-endpoints.png" alt="Leave endpoint groups: leave types, leave balances and leave requests" width="49%">
</p>

<p align="center">
  <img src="docs/screenshots/attendance-history-operation.png" alt="Expanded attendance history operation showing the user id path parameter and the startDate, endDate, page and limit query parameters" width="49%">
  <img src="docs/screenshots/auth-endpoints.png" alt="Auth endpoint group: register, login, email check, profile, user listing, department assignment and Google sign-in" width="49%">
</p>

<p align="center">
  <img src="docs/screenshots/organization-endpoints.png" alt="Companies, Departments and Shifts endpoint groups with full CRUD operations" width="100%">
</p>

<p align="center">
  <img src="docs/screenshots/login-operation.png" alt="Expanded login operation showing the JSON request body example with placeholder email and password" width="100%">
</p>

<p align="center">
  <img src="docs/screenshots/authorize-bearer-jwt.png" alt="Swagger Authorize dialog for the bearer JWT security scheme" width="60%">
</p>

## Features

**Organisation setup**
- Companies, departments and shifts, each with create, read, update and delete endpoints.
- Orchestrated onboarding (`POST /setup/complete`, authenticated): one call creates a company, its first department, a shift and the first user, then assigns that user to the department with a role.
- Shifts belong to a company and can optionally be scoped to a department. Each has start and end times and a grace period (15 minutes by default).

**Attendance**
- Check-in and check-out, limited to one attendance record per user per day.
- Status is decided on the server at check-in: `PRESENT`, or `LATE` once the shift's grace period has passed.
- Breaks: start and end a break, list an attendance record's breaks and get the active one. Only one break can be open at a time, and no break can start after check-out.
- Endpoints for the mobile app: current status (checked in or out, hours worked so far, today's shift), today's record, and the user's shift with its late threshold.
- Paginated attendance history with an optional date range.
- Daily summary for a month (formatted check-in and check-out times and total hours) and weekly summary for a year (hours worked and missed days per week).
- Attendance views per department and per company for a given date.

**Leave management**
- Leave types per company, each with a default number of days.
- Leave balances per user and leave type, with one balance per pair.
- Leave requests per user and leave type, created as `PENDING` by default (status enum `PENDING` / `APPROVED` / `REJECTED`). A new request is rejected if it overlaps one of the user's pending requests. Approval endpoints are not part of this API yet.

**Identity and access**
- Email and password registration and login, with bcrypt-hashed passwords and JWT access tokens.
- Google OAuth 2.0 sign-in through Passport.
- Four roles: `employee`, `manager`, `admin`, `super_admin`. Company and department management and department assignment are limited to admins.

## Architecture

```mermaid
flowchart LR
  subgraph Clients
    M[Mobile client<br/>planned]
    W[Web front end]
    S[Swagger UI /docs]
  end

  M -.->|HTTPS| NG[Nginx<br/>TLS termination]
  W & S -->|HTTPS| NG
  NG --> APP

  subgraph APP[NestJS 11 API, run by PM2]
    direction TB
    G[Global pipeline<br/>Helmet · CORS · /api/v1 versioning<br/>ValidationPipe · JwtAuthGuard · RolesGuard<br/>ResponseInterceptor · AllExceptionsFilter]
    G --> AUTH[Auth<br/>JWT · Google OAuth]
    G --> ORG[Companies · Departments · Shifts]
    G --> SETUP[Setup<br/>one-call onboarding]
    G --> ATT[Attendance<br/>check-in/out · breaks · summaries]
    G --> LV[Leave<br/>types · balances · requests]
  end

  APP -->|TypeORM| DB[(MySQL)]
  GOOGLE[Google OAuth 2.0] -.-> AUTH

  GH[GitHub Actions<br/>build on push to main] -->|SSH deploy:<br/>npm ci · build · pm2 reload · health check| APP
```

### Data model

```mermaid
erDiagram
  COMPANY ||--o{ DEPARTMENT : has
  COMPANY ||--o{ SHIFT : defines
  COMPANY ||--o{ LEAVE_TYPE : defines
  COMPANY ||--o{ LEAVE_REQUEST : receives
  DEPARTMENT |o--o{ USER : employs
  DEPARTMENT |o--o{ SHIFT : "optionally scopes"
  DEPARTMENT |o--o{ LEAVE_REQUEST : "tagged on"
  USER ||--o{ ATTENDANCE : records
  SHIFT ||--o{ ATTENDANCE : "applies to"
  ATTENDANCE ||--o{ BREAK : contains
  USER ||--o{ LEAVE_BALANCE : holds
  LEAVE_TYPE ||--o{ LEAVE_BALANCE : "measured in"
  USER ||--o{ LEAVE_REQUEST : submits
  LEAVE_TYPE ||--o{ LEAVE_REQUEST : "is of"
```

All primary keys are UUIDs. The live OpenAPI spec lists 56 operations in 10 groups. [docs/architecture.md](docs/architecture.md) covers the request pipeline, the check-in flow and the deployment in more depth.

### Key engineering decisions

- **Endpoints are protected by default.** The JWT guard is registered as a global `APP_GUARD`, so every route needs a token unless it is explicitly marked `@Public()`. Only register, login, email check, Google sign-in and a hello/test endpoint are public. Admin-only routes add a `RolesGuard` with `@Roles(...)` on top.
- **Every response has the same shape.** A global interceptor wraps successes as `{ status, message, code, data }`, and a catch-all exception filter returns errors in the same shape with `path` and `timestamp`. Mobile and web clients parse one format.
- **Attendance rules live on the server.** Check-in looks up the user's department shift and falls back to the company-wide shift (one with no department). It rejects a second check-in on the same day and works out `PRESENT` or `LATE` from the shift start and grace period. Clients never send a status.
- **Breaks are child records of attendance.** Storing them this way makes rules such as one open break at a time, no break before check-in and no break after check-out simple checks against the day's record.
- **Onboarding is one orchestrated call.** One authenticated call (`POST /setup/complete`) runs the company, department, shift and auth services in sequence to create the company, its first department, a shift and the first user, instead of duplicating that logic.
- **The API is versioned and hardened from day one.** It uses URI versioning (`/api/v1`), Helmet security headers, a whitelisting `ValidationPipe` with implicit type conversion, and `ClassSerializerInterceptor`, so password hashes (`@Exclude`) are never serialised.

## Tech stack

| Layer | Tech |
| --- | --- |
| Runtime and framework | Node.js 22, NestJS 11, TypeScript 5 |
| Data | MySQL via TypeORM 0.3 (`mysql2` driver), UUID primary keys |
| Auth | Passport JWT (`@nestjs/jwt`), Google OAuth 2.0 (`passport-google-oauth20`), bcrypt |
| Validation and serialisation | class-validator, class-transformer, global `ValidationPipe` |
| Security | Helmet (CSP, HSTS and other headers), CORS, role guard, TLS at the Nginx proxy |
| API docs | `@nestjs/swagger` (OpenAPI 3.0), Swagger UI at `/docs` |
| CI/CD | GitHub Actions: build on push to `main`, then SSH deploy |
| Hosting | Ubuntu VPS, Nginx reverse proxy, PM2 (auto-restart, memory cap) |

## My role

I co-developed the backend with one other developer, in a private GitHub organisation repository. From the git history:

- **API foundation:** Helmet and CORS setup, URI versioning, Swagger/OpenAPI setup, global validation, and environment-based configuration for the database and JWT.
- **Authentication:** JWT and Passport login and registration, the Google OAuth 2.0 strategy and callback, and user deletion.
- **Access control:** department CRUD with role-based access, linked to companies.
- **Onboarding:** the `setup` module (company, department, shift and first user in one call).
- **Attendance for mobile:** current status, today's record, paginated history, shift info, break management (start, end, list, active break) and the daily and weekly summary endpoints.
- **Delivery:** the GitHub Actions pipeline that builds and deploys to the VPS, plus (with my own automation tooling) the PM2 ecosystem config and the post-deploy health check.

The other developer scaffolded the project and the database connection. They built the company, department and user modules, the response interceptor and exception filter, shifts, the first check-in and check-out flow, and the leave module (types, balances, requests).

## Status

- **Live:** the API and its documentation are at [hrm.everbuild.cloud/docs](https://hrm.everbuild.cloud/docs).
- **Development:** September 2025 to February 2026 (first commit 16 Sep 2025, latest 4 Feb 2026).
- **Clients:** the attendance endpoints were designed for a mobile client (planned; the diagram shows it dashed), and a web front end lives in a separate private repository.

---

<p align="center">© 2026 Dafallah Ibrahim (documentation and screenshots). HRM Platform is a team project, shared with permission. Source code is private.</p>
