
# Visitor Management and Gate Pass Automation System – Backend System Design (Revised)

This design aligns with the provided requirements, including the simplified Gate Pass Generation component shown in the UI mockup.

---

## 1. Problem Understanding

### Purpose
Automate visitor registration, gate pass generation, entry/exit tracking, and reporting. The system eliminates paper logs, provides a digital pass with configurable duration and visual timer, and streamlines security operations.

### Target Users & Their Goals
- **Receptionist** – Manually register visitors, set gate pass duration, select template, and print passes.
- **Security Officer** – Search visitors, view status, record check‑in/out.
- **Administrator** – Configure system settings, manage user roles, export reports.
- **System Administrator** – Full system control, audit logs.

---

## 2. Core Features

| Module | Features |
|--------|----------|
| Visitor Registration | Manual form with name, company, reason, contact, email; unique visitor ID generation. |
| Gate Pass Generation | Display visitor info (name, company, ID, date/time), duration selector (hours), template selection (Standard / Premium), countdown timer, print button. |
| Entry/Exit Tracking | Security officer records check‑in and check‑out; automatic timestamp and status update. |
| Visitor Search & Display | Search by name or ID; table with status color coding; detailed view. |
| Reporting & Export | Dashboard with visitor counts (daily/weekly/monthly); CSV export. |
| Role Management | Define roles (Receptionist, Security Officer, Administrator, System Admin). |
| Configuration | System defaults: default duration, email notification recipient, expiry handling. |

---

## 3. Actors / Roles

| Role | Responsibilities |
|------|------------------|
| Receptionist | Register visitors; generate and print gate passes (set duration, choose template). |
| Security Officer | Search visitors; record check‑in/out; view visitor status. |
| Administrator | Manage users, configure system settings, export data, view reports. |
| System Administrator | Full access, audit logs, user account management. |

---

## 4. Entity Identification

- **Visitor**
  - `visitor_id` (PK) – auto‑generated unique ID
  - `name`
  - `company`
  - `reason_for_visit` (predefined list)
  - `contact_number`
  - `email`
  - `notes` (optional)
  - `created_at`

- **Visit**
  - `visit_id` (PK)
  - `visitor_id` (FK)
  - `status` (Checked In, Checked Out, Expired)
  - `entry_time` (timestamp)
  - `exit_time` (timestamp, nullable)
  - `gate_pass_duration_hours` (selected by receptionist)
  - `gate_pass_issued_at` (timestamp)
  - `gate_pass_valid_until` (timestamp = issued_at + duration hours)
  - `template_type` (Standard / Premium) – stored for audit, but printing handled frontend

- **GatePass** (optional – can be derived from Visit; kept separate for tracking prints)
  - `pass_id` (PK)
  - `visit_id` (FK, one‑to‑one)
  - `printed_at` (timestamp)
  - `printed_by` (FK to user)

- **User**
  - `user_id` (PK)
  - `username`
  - `password_hash`
  - `role` (Receptionist, Security Officer, Administrator, System Admin)
  - `is_active`

- **AuditLog**
  - `log_id` (PK)
  - `user_id` (FK)
  - `action`
  - `entity_type`
  - `entity_id`
  - `timestamp`
  - `details` (JSON)

- **Configuration**
  - `config_id` (PK)
  - `config_key` (unique)
  - `config_value`

---

## 5. Relationships

- `Visitor` ↔ `Visit` – one‑to‑many.
- `Visit` ↔ `GatePass` – one‑to‑one (optional).
- `User` ↔ `AuditLog` – one‑to‑many.

---

## 6. Business Rules & Constraints

### 1. Visitor Registration
- Required fields: name, company, reason, contact, email.
- Contact number and email must be valid formats.
- Visitor ID is system‑generated (e.g., `VIS-YYYYMMDD-XXXX`).

### 2. Gate Pass Generation
- Pass can only be generated after visitor registration.
- Duration can be set by receptionist (in hours).
- Pass validity is computed as current time + duration.
- Template selection is stored (Standard / Premium) but printing is frontend‑driven.
- Printed pass includes visitor name, company, ID, date/time, and remaining time (calculated).

### 3. Entry/Exit
- Check‑in records `entry_time` and status → “Checked In”.
- Check‑out records `exit_time` and status → “Checked Out”.
- If pass expires before check‑out, status becomes “Expired” (batch job or on‑demand).
- A checked‑in visitor cannot be checked in again without checking out first.

### 4. Roles & Permissions
- Only Receptionists can generate/print passes.
- Security Officers can check in/out.
- Administrators can view reports and export data.

### 5. Expiry
- After validity period, gate pass is considered expired; the system prevents check‑in.
- A scheduled job updates status of expired visits daily.

---

## 7. Database Design (Tables)

### `visitors`

| Column | Type | Constraints |
|--------|------|-------------|
| `visitor_id` | VARCHAR(20) | PK |
| `name` | VARCHAR(100) | NOT NULL |
| `company` | VARCHAR(100) | NOT NULL |
| `reason_for_visit` | VARCHAR(50) | NOT NULL |
| `contact_number` | VARCHAR(20) | NOT NULL |
| `email` | VARCHAR(100) | NOT NULL |
| `notes` | TEXT | |
| `created_at` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |

### `visits`

| Column | Type | Constraints |
|--------|------|-------------|
| `visit_id` | BIGINT | PK AUTO_INCREMENT |
| `visitor_id` | VARCHAR(20) | FK → `visitors(visitor_id)` |
| `status` | ENUM('Checked In','Checked Out','Expired') | NOT NULL |
| `entry_time` | TIMESTAMP | NULL |
| `exit_time` | TIMESTAMP | NULL |
| `gate_pass_duration_hours` | INT | NOT NULL |
| `gate_pass_issued_at` | TIMESTAMP | NOT NULL |
| `gate_pass_valid_until` | TIMESTAMP | NOT NULL |
| `template_type` | VARCHAR(20) | DEFAULT 'Standard' |

### `gate_passes` (optional – to log print actions)

| Column | Type | Constraints |
|--------|------|-------------|
| `pass_id` | BIGINT | PK AUTO_INCREMENT |
| `visit_id` | BIGINT | FK → `visits(visit_id)`, UNIQUE |
| `printed_at` | TIMESTAMP | NOT NULL |
| `printed_by` | BIGINT | FK → `users(user_id)` |

### `users`

| Column | Type | Constraints |
|--------|------|-------------|
| `user_id` | BIGINT | PK AUTO_INCREMENT |
| `username` | VARCHAR(50) | UNIQUE NOT NULL |
| `password_hash` | VARCHAR(255) | NOT NULL |
| `role` | VARCHAR(50) | NOT NULL |
| `is_active` | BOOLEAN | DEFAULT TRUE |

### `audit_logs`

| Column | Type | Constraints |
|--------|------|-------------|
| `log_id` | BIGINT | PK AUTO_INCREMENT |
| `user_id` | BIGINT | FK → `users(user_id)` |
| `action` | VARCHAR(100) | NOT NULL |
| `entity_type` | VARCHAR(50) | NOT NULL |
| `entity_id` | VARCHAR(100) | NOT NULL |
| `timestamp` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |
| `details` | JSON | |

### `configuration`

| Column | Type | Constraints |
|--------|------|-------------|
| `config_id` | BIGINT | PK AUTO_INCREMENT |
| `config_key` | VARCHAR(100) | UNIQUE NOT NULL |
| `config_value` | TEXT | NOT NULL |
| `description` | VARCHAR(255) | |

---

## 8. Workflow / Process Flow

### Gate Pass Generation
1. Receptionist registers a visitor → Visitor record created.
2. Receptionist selects Gate Pass Generation component.
3. System fetches visitor details (name, company, visitor ID) and current timestamp.
4. Receptionist sets duration (hours) and selects template (Standard/Premium).
5. System calculates `valid_until = now + duration`.
6. Creates a Visit record with status `'Active'` (pass generated but not yet checked in).
7. System stores `gate_pass_duration_hours`, `gate_pass_issued_at`, `gate_pass_valid_until`, and `template_type` in `visits` table.
8. Optional: log print in `gate_passes` table.
9. Frontend displays the countdown timer based on `valid_until` and provides a print button.

### Check‑in
1. Visitor arrives with printed pass (or security officer looks up by name/ID).
2. Security officer finds visitor record, clicks “Check In”.
3. System updates `visit.entry_time = NOW()`, `status = 'Checked In'`.
4. Audit log records action.

### Check‑out
1. Visitor leaves; security officer selects “Check Out”.
2. System updates `exit_time` and `status = 'Checked Out'`.

### Expiry Handling
- A scheduled job runs daily to find visits where `valid_until < NOW()` and `status` is `'Active'` or `'Checked In'`; set status to `'Expired'`.
- During check‑in, system also checks if `valid_until` is in the past; if yes, reject with “Pass expired”.

### Reporting
- Administrator selects date range; backend queries visits with filters.
- Returns counts and optionally detailed data for CSV export.

---

## 9. API Design (High‑Level)

All endpoints under `/api/v1`

| Endpoint | Method | Purpose | Request | Response |
|----------|--------|---------|---------|----------|
| `/visitors` | POST | Register visitor | VisitorDTO (name, company, reason, contact, email, notes) | 201 Created with visitor_id |
| `/visitors/{id}` | GET | Get visitor details | – | VisitorDetailDTO (includes current visit info if any) |
| `/visitors/search` | GET | Search by name or ID | q query param | List of VisitorSummaryDTO |
| `/visits` | POST | Generate gate pass | VisitRequestDTO (visitor_id, duration_hours, template_type) | VisitDTO (includes visitor info, valid_until) |
| `/visits/{visitId}/checkin` | PUT | Record check‑in | – | 200 OK with updated visit |
| `/visits/{visitId}/checkout` | PUT | Record check‑out | – | 200 OK |
| `/reports/visitors` | GET | Get visitor counts | from, to, groupBy (day/week/month) | ReportDTO |
| `/reports/export` | GET | Export CSV | same filters | CSV file |
| `/config` | GET / PUT | Configuration (PUT) | ConfigDTO | 200 OK |
| `/users` | CRUD | User management (admin only) | UserDTO | 200/201 |

All responses follow standard wrapper. Error responses include appropriate HTTP status codes.

---

## 10. Security & Access Control

- **Authentication**: JWT (stateless).
- **Authorization**: Role‑based with `@PreAuthorize`.
  - **Receptionist**: can POST `/visitors`, POST `/visits`, view own registrations.
  - **Security Officer**: can GET `/visitors/search`, GET `/visitors/{id}`, PUT `/visits/{visitId}/checkin`, PUT `/visits/{visitId}/checkout`.
  - **Administrator**: can GET `/reports/*`, GET/PUT `/config`, manage users.
- **Password**: BCrypt hashing.
- **Validation**: Bean validation on DTOs.
- **Protection**: CORS, rate limiting, SQL injection prevention via JPA.

---

## 11. Edge Cases & Exception Handling

| Scenario | Handling |
|----------|----------|
| Visitor already has an active pass | Prevent duplicate active visits; return conflict error. |
| Check‑in after expiry | Reject; show message “Gate pass expired on [date]”. |
| Check‑in without gate pass | Not possible – passes are generated for all visitors. |
| Print failure | Log error; frontend can retry. |
| Invalid duration | Validate (1–24 hours, default 3). |
| Search with no results | Return empty list; frontend shows “No visitors found”. |
| Unauthorized access | 401 Unauthorized. |

---

## 12. Logging & Monitoring

### Logging
- All service methods logged via AOP.
- Audit logs stored in `audit_logs` table.
- Security events (login failures, permission violations) logged at INFO.
- Exceptions logged at ERROR.

### Monitoring
- Spring Boot Actuator endpoints.
- Metrics: visitor registrations, gate passes generated, check‑in/out counts.
- Alerts: high error rate, database connection issues.

### Caching
- Cache visitor search results for 5 minutes.

---

