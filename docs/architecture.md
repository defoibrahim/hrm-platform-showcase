# HRM Platform architecture

This page expands on the [README](../README.md). Everything here comes from the private source code, its configuration and the live OpenAPI spec. No code is reproduced.

## Module map

The application is one NestJS process made of feature modules. Each module owns its TypeORM entities, so module boundaries match the data model.

```mermaid
flowchart TB
  APP[AppModule] --> CFG[ConfigModule<br/>database + JWT config]
  APP --> DBM[DatabaseModule<br/>TypeORM / MySQL]
  APP --> AUTH[AuthModule]
  APP --> USERS[UsersModule]
  APP --> COMP[CompaniesModule]
  APP --> DEPT[DepartmentsModule]
  APP --> SHIFT[ShiftsModule]
  APP --> ATT[AttendanceModule]
  APP --> LT[LeavetypeModule]
  APP --> LB[LeaveBalanceModule]
  APP --> LR[LeaveRequestModule]
  APP --> SETUP[SetupModule]
  APP --> THR[ThrottlerModule<br/>sign-in rate limits]
  APP --> POL[Access policy<br/>role · ownership · company<br/>called by every feature service]

  AUTH --- JWTS[JWT strategy]
  AUTH --- GS[Google OAuth strategy]
  ATT --- HELP[AttendanceHelper<br/>shift resolution, status, breaks, summaries]
  SETUP -. uses .-> COMP & DEPT & SHIFT & AUTH
```

| Module | Responsibility | Base route |
| --- | --- | --- |
| Auth | Login, logout (token revocation), email check, profile, admin-created users, user update and removal, user listing, department assignment; registration and Google sign-in are off unless configured | `/api/v1/auth` |
| Companies | A super admin manages every company; an admin reads and renames their own | `/api/v1/companies` |
| Departments | Department CRUD within the admin's company; names unique per company; a department with people in it cannot be deleted | `/api/v1/departments` |
| Shifts | Shift CRUD within the admin's company: company-wide or department-scoped, start and end time, grace period | `/api/v1/shifts` |
| Attendance | Check-in and check-out and breaks for the signed-in user; live status, history, daily and weekly summaries, department and company views for permitted readers | `/api/v1/attendances` |
| Leave types / balances / requests | Leave catalogue per company, balances per user and type, and requests (always created `PENDING`, overlap check, approved or rejected by an admin of the same company) | `/api/v1/leave-types`, `/leave-balances`, `/leave-requests` |
| Setup | Initialisation status, one-time bootstrap of the first super admin (needs a token held on the server), and super-admin-only onboarding of a company with its first department, shift and user | `/api/v1/setup` |

## Request pipeline

These cross-cutting concerns are registered once, globally, rather than per controller:

```mermaid
sequenceDiagram
  autonumber
  participant C as Client
  participant H as Helmet + CORS
  participant V as URI versioning (/api/v1)
  participant G as JwtAuthGuard (global)
  participant R as RolesGuard (per controller/route)
  participant P as ValidationPipe (global)
  participant Ctl as Controller / Service
  participant A as Access policy
  participant I as ResponseInterceptor
  participant F as AllExceptionsFilter

  C->>H: HTTPS request
  H->>V: security headers applied
  V->>G: route resolved
  alt route is @Public()
    G-->>R: skip token check
  else
    G->>G: verify Bearer JWT (Passport, pinned algorithm)
    G->>G: reload user, check token version
  end
  R->>R: compare user role with @Roles(...)
  R->>P: whitelist + transform DTO
  P->>Ctl: typed, validated input + caller
  Ctl->>A: may this caller act on this record?
  A-->>Ctl: allow, or 403
  Ctl-->>I: result
  I-->>C: { status: "success", message, code, data }
  Ctl--xF: thrown exception
  F-->>C: { status: "error", message, code, data: [], path, timestamp }
```

- **Secure by default:** forgetting a decorator leaves a route protected rather than open.
- **Identity from the database:** the JWT strategy re-loads the user on every request, so the role and company come from the database. A per-user token version revokes older tokens on logout or password change, and a deleted user's token gets 401.
- **One place for authorization:** services pass the caller to a single access policy, which checks role, ownership and company before anything is read or written. Ids in a request body never decide who the action is for.
- **Rate-limited sign-in:** login, email check, registration and bootstrap are limited per client IP and per account.
- **One envelope:** clients always read `status`, `message`, `code` and `data`, whether the call succeeded or failed. For validation errors the filter returns the first message as a single string.
- **No leaked secrets in responses:** `ClassSerializerInterceptor` respects `@Exclude()` on the user's password hash and token version.

## Check-in flow

```mermaid
flowchart TD
  A[POST /attendances/check-in<br/>user = caller from the token] --> B{User exists?}
  B -- no --> E1[404 User not found]
  B -- yes --> C{Assigned to a department?}
  C -- no --> E2[400 Not assigned to any department]
  C -- yes --> D{Already checked in today?}
  D -- yes --> E3[400 Attendance already exists]
  D -- no --> S{Department shift exists?}
  S -- yes --> T[Use department shift]
  S -- no --> U{Company-wide shift exists?}
  U -- no --> E4[400 No shift assigned]
  U -- yes --> T2[Use company-wide shift]
  T & T2 --> K{now > shift start + grace period?}
  K -- yes --> L[Save attendance as LATE]
  K -- no --> P[Save attendance as PRESENT]
```

Breaks follow the same pattern. A break is only allowed when today's attendance exists, the user has not checked out, and no other break is open. Ending a break closes the one open break.

## Reporting endpoints

- **Status:** checked in or out, hours worked so far (live until check-out) and today's shift.
- **History:** a query-builder query with an optional `startDate`/`endDate` range and `page`/`limit` pagination, returning total pages.
- **Daily summary:** every record in a month, with display-ready date and time strings and total hours.
- **Weekly summary:** records grouped by week number for a year, adding up hours worked and counting `ABSENT` days.
- **Department / company views:** every attendance record for a date, filtered through the user → department → company relations.

## Data model

```mermaid
erDiagram
  COMPANY {
    uuid id PK
    string name UK
  }
  DEPARTMENT {
    uuid id PK
    string name
    uuid company_id FK
  }
  USER {
    uuid id PK
    string email UK
    string first_name
    string last_name
    enum role "employee | manager | admin | super_admin"
    boolean status
    int token_version "bumped to revoke tokens"
    uuid department_id FK "nullable"
  }
  SHIFT {
    uuid id PK
    string name
    time start_time
    time end_time
    int grace_period "minutes, default 15"
    uuid company_id FK
    uuid department_id FK "nullable = company-wide"
  }
  ATTENDANCE {
    uuid id PK
    date date
    timestamp check_in
    timestamp check_out
    enum status "PRESENT | ABSENT | LATE | HALF_DAY"
    uuid user_id FK
    uuid shift_id FK
  }
  BREAK {
    uuid id PK
    timestamp break_start
    timestamp break_end "null while open"
    uuid attendance_id FK
  }
  LEAVE_TYPE {
    uuid id PK
    string name
    text description
    int default_days
    uuid company_id FK
  }
  LEAVE_BALANCE {
    uuid id PK
    int balance
    uuid user_id FK
    uuid leave_type_id FK
  }
  LEAVE_REQUEST {
    uuid id PK
    date start_date
    date end_date
    string status "PENDING | APPROVED | REJECTED"
    text reason
    uuid user_id FK
    uuid company_id FK
    uuid department_id FK "nullable"
    uuid leave_type_id FK
  }

  COMPANY ||--o{ DEPARTMENT : has
  COMPANY ||--o{ SHIFT : defines
  COMPANY ||--o{ LEAVE_TYPE : defines
  COMPANY ||--o{ LEAVE_REQUEST : receives
  DEPARTMENT |o--o{ USER : employs
  DEPARTMENT |o--o{ SHIFT : scopes
  DEPARTMENT |o--o{ LEAVE_REQUEST : "tagged on"
  USER ||--o{ ATTENDANCE : records
  SHIFT ||--o{ ATTENDANCE : "applies to"
  ATTENDANCE ||--o{ BREAK : contains
  USER ||--o{ LEAVE_BALANCE : holds
  LEAVE_TYPE ||--o{ LEAVE_BALANCE : "measured in"
  USER ||--o{ LEAVE_REQUEST : submits
  LEAVE_TYPE ||--o{ LEAVE_REQUEST : "is of"
```

Referential behaviour is set on the relations. Deleting a company cascades to its departments, leave types and leave requests. Deleting a department that still has people in it is refused (409), so nobody loses their company; otherwise the department is set to null on its shifts and leave requests. Deleting a user cascades to their leave balances and requests.

## Front end

The web client is a Next.js 15 App Router app (React 19, strict TypeScript) with one route per area: dashboard, attendance, leave, employees, departments, shifts, company and profile.

- **Server state** lives in TanStack React Query. Each screen is a set of queries and mutations over one Axios client that adds the bearer token, unwraps the API's response envelope and turns a 401 into "session ended": clear the session, go to `/login`.
- **UI** is Tailwind CSS with shadcn/ui components on Radix primitives. Navigation and actions are filtered by role, but only as a convenience: the API makes every decision.
- **Sign-in** is two steps (does the email exist, then the password) and runs in the browser, so the API's rate limits apply per visitor. Sign-out calls the API's logout to revoke the token.
- **Demo accounts** are read from the container's environment at request time, never baked into the image. Their sessions are treated as shared, so the profile page does not offer a name or password change for them.

## Deployment

```mermaid
flowchart LR
  DEV[Push to main] --> CI[GitHub Actions<br/>Node 22 · npm ci · nest build<br/>unit + e2e tests on MySQL]
  CI -->|on success| SSH[SSH deploy job]
  SSH --> VPS

  subgraph VPS[Ubuntu VPS]
    direction TB
    PULL["git pull (fast-forward only)"] --> INST[npm ci + build]
    INST --> PM2[pm2 reload ecosystem config<br/>autorestart · memory cap]
    PM2 --> HC{Health check on /docs}
    HC -- fail --> LOGS[Print PM2 logs, fail the job]
  end

  NGINX[Nginx · TLS] -->|"/api/ · /docs"| PM2
```

The front end has its own pipeline:

```mermaid
flowchart LR
  PR[Pull request or push to main] --> CHK[Lint · typecheck · next build]
  PR --> DK[Docker image build]
  PR --> E2E[Playwright smoke test<br/>real API + MySQL in CI]
  CHK & DK & E2E -->|main only| IMG[Image to GHCR<br/>+ release copy]
  IMG --> UPD[update script on the VPS<br/>pull or load the image · recreate container]
  UPD --> CT[Next.js container<br/>localhost only · memory and CPU caps]
  NG[Nginx · TLS] -->|"/"| CT
```

Nothing is built on the server: the VPS only pulls or loads a finished image.

- **Zero-touch deploys:** a push to `main` builds and tests in CI first, and the deploy job only runs after both pass.
- **Process supervision:** PM2 runs the compiled app with auto-restart and a memory ceiling, and `pm2 save` persists the process list.
- **Fail loudly:** a failed health check after reload fails the pipeline and prints recent logs.
