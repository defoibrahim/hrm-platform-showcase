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

  AUTH --- JWTS[JWT strategy]
  AUTH --- GS[Google OAuth strategy]
  ATT --- HELP[AttendanceHelper<br/>shift resolution, status, breaks, summaries]
  SETUP -. uses .-> COMP & DEPT & SHIFT & AUTH
```

| Module | Responsibility | Base route |
| --- | --- | --- |
| Auth | Registration, login, email check, profile, user listing, department assignment, Google sign-in | `/api/v1/auth` |
| Companies | Company CRUD (admin roles) | `/api/v1/companies` |
| Departments | Department CRUD under a company (admin roles) | `/api/v1/departments` |
| Shifts | Shift CRUD: company-wide or department-scoped, start and end time, grace period | `/api/v1/shifts` |
| Attendance | Check-in and check-out, live status, history, breaks, daily and weekly summaries, department and company views | `/api/v1/attendances` |
| Leave types / balances / requests | Leave catalogue per company, balances per user and type, and requests with an overlap check | `/api/v1/leave-types`, `/leave-balances`, `/leave-requests` |
| Setup | One-call tenant onboarding | `/api/v1/setup` |

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
  participant I as ResponseInterceptor
  participant F as AllExceptionsFilter

  C->>H: HTTPS request
  H->>V: security headers applied
  V->>G: route resolved
  alt route is @Public()
    G-->>R: skip token check
  else
    G->>G: verify Bearer JWT (Passport)
  end
  R->>R: compare user role with @Roles(...)
  R->>P: whitelist + transform DTO
  P->>Ctl: typed, validated input
  Ctl-->>I: result
  I-->>C: { status: "success", message, code, data }
  Ctl--xF: thrown exception
  F-->>C: { status: "error", message, code, data: [], path, timestamp }
```

- **Secure by default:** forgetting a decorator leaves a route protected rather than open.
- **One envelope:** clients always read `status`, `message`, `code` and `data`, whether the call succeeded or failed. For validation errors the filter returns the first message as a single string.
- **No leaked secrets in responses:** `ClassSerializerInterceptor` respects `@Exclude()` on the user's password hash.

## Check-in flow

```mermaid
flowchart TD
  A[POST /attendances/check-in] --> B{User exists?}
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
    uuid department_id FK
    uuid leave_type_id FK
  }

  COMPANY ||--o{ DEPARTMENT : has
  COMPANY ||--o{ SHIFT : defines
  COMPANY ||--o{ LEAVE_TYPE : defines
  COMPANY ||--o{ LEAVE_REQUEST : receives
  DEPARTMENT |o--o{ USER : employs
  DEPARTMENT |o--o{ SHIFT : scopes
  USER ||--o{ ATTENDANCE : records
  SHIFT ||--o{ ATTENDANCE : "applies to"
  ATTENDANCE ||--o{ BREAK : contains
  USER ||--o{ LEAVE_BALANCE : holds
  LEAVE_TYPE ||--o{ LEAVE_BALANCE : "measured in"
  USER ||--o{ LEAVE_REQUEST : submits
  LEAVE_TYPE ||--o{ LEAVE_REQUEST : "is of"
```

Referential behaviour is set on the relations. Deleting a company cascades to its departments, leave types and leave requests. Deleting a department sets the department to null on users and shifts instead of deleting them. Deleting a user cascades to their leave balances and requests.

## Deployment

```mermaid
flowchart LR
  DEV[Push to main] --> CI[GitHub Actions<br/>Node 22 · npm ci · nest build]
  CI -->|on success| SSH[SSH deploy job]
  SSH --> VPS

  subgraph VPS[Ubuntu VPS]
    direction TB
    PULL["git pull (fast-forward only)"] --> INST[npm ci + build]
    INST --> PM2[pm2 reload ecosystem config<br/>autorestart · memory cap]
    PM2 --> HC{Health check on /docs}
    HC -- fail --> LOGS[Print PM2 logs, fail the job]
  end

  NGINX[Nginx · TLS] --> PM2
```

- **Zero-touch deploys:** a push to `main` builds in CI first, and the deploy job only runs after a successful build.
- **Process supervision:** PM2 runs the compiled app with auto-restart and a memory ceiling, and `pm2 save` persists the process list.
- **Fail loudly:** a failed health check after reload fails the pipeline and prints recent logs.
