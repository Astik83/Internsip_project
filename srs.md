

## 1. Problem Understanding

### Purpose
Automate visitor registration, gate pass generation, entry/exit tracking, and reporting. Replace manual logs with a digital pass that includes a configurable duration, a countdown timer, and template selection, streamlining security operations.

### Target Users & Their Goals
- **Receptionist** – Manually register visitors, generate unique visitor IDs.
- **Security Officer** – Search visitors, generate gate passes (set duration, choose template), record check‑in/out, view status.
- **Administrator** – Manage system configuration, user roles, view reports, export data.
- **System Administrator** – Full system control, audit logs.

---

## 2. Core Features

| Module | Features |
|--------|----------|
| **Visitor Registration** | Manual form with name, company, reason, contact, email; unique visitor ID generation. |
| **Gate Pass Generation** | Display visitor info (name, company, ID, date/time), duration selector (hours), template selection (Standard / Premium), countdown timer, print button. |
| **Entry/Exit Tracking** | Security officer records check‑in and check‑out; automatic timestamp and status update. |
| **Visitor Search & Display** | Search by name or ID; paginated table with status color coding; detailed view. |
| **Reporting & Export** | Dashboard with visitor counts (daily/weekly/monthly); CSV export. |
| **Role Management** | Define roles (Receptionist, Security Officer, Administrator, System Admin). |
| **Configuration** | System defaults: default duration, email notification recipient, expiry handling. |

---

## 3. Actors / Roles

| Role | Responsibilities |
|------|------------------|
| **Receptionist** | Register visitors (only). |
| **Security Officer** | Search visitors; generate gate passes (set duration, choose template); record check‑in/out; view visitor status. |
| **Administrator** | Manage users, configure system settings, export data, view reports. |
| **System Administrator** | Full access, audit logs, user account management. |

---

## 4. Entity Identification

### Visitor
- `visitor_id` (PK) – auto‑generated unique ID (e.g., VIS-YYYYMMDD-XXXX)
- `name`
- `company`
- `reason_for_visit` (predefined list)
- `contact_number`
- `email`
- `notes` (optional)
- `created_at`

### Visit
- `visit_id` (PK)
- `visitor_id` (FK)
- `status` (Checked In, Checked Out, Expired)
- `entry_time` (timestamp)
- `exit_time` (timestamp, nullable)
- `gate_pass_duration_hours` (selected by security officer at check‑in)
- `gate_pass_issued_at` (timestamp)
- `gate_pass_valid_until` (timestamp = issued_at + duration hours)
- `template_type` (Standard / Premium)

### GatePass (optional – to log print actions)
- `pass_id` (PK)
- `visit_id` (FK, one‑to‑one)
- `printed_at` (timestamp)
- `printed_by` (FK to user)

### User
- `user_id` (PK)
- `username`
- `password_hash`
- `role` (Receptionist, Security Officer, Administrator, System Admin)
- `is_active`

### AuditLog
- `log_id` (PK)
- `user_id` (FK)
- `action`
- `entity_type`
- `entity_id`
- `timestamp`
- `details` (JSON)

### Configuration
- `config_id` (PK)
- `config_key` (unique)
- `config_value`

---

## 5. Relationships
- **Visitor ↔ Visit** – one‑to‑many.
- **Visit ↔ GatePass** – one‑to‑one (optional).
- **User ↔ AuditLog** – one‑to‑many.

---

## 6. Business Rules & Constraints

1. **Visitor Registration**
   - Required fields: name, company, reason, contact, email.
   - Contact number and email must be valid formats.
   - Visitor ID is system‑generated, unique, and consistent format.

2. **Gate Pass Generation**
   - Pass is generated only by a Security Officer at the time of check‑in.
   - Duration is set by the officer (in hours, e.g., 1–24).
   - Pass validity is computed as current time + duration.
   - Template selection (Standard / Premium) is stored for audit.
   - Printed pass includes visitor name, company, ID, date/time, and remaining time (calculated client‑side).

3. **Entry/Exit**
   - Check‑in creates a Visit record with entry_time = NOW(), status = Checked In.
   - Check‑out updates exit_time and status = Checked Out.
   - A visitor cannot have two active visits (status Checked In or Expired without check‑out) simultaneously.
   - If a pass expires before check‑out, status becomes Expired (via scheduled job).

4. **Roles & Permissions**
   - Receptionist: only `POST /visitors`.
   - Security Officer: `GET /visitors` (with search), `GET /visitors/{id}`, `POST /visits`, `PUT /visits/{visitId}/checkout`, `POST /visits/{visitId}/print`.
   - Administrator: `GET /reports/*`, `GET/PUT /config`, CRUD `/users`.
   - System Administrator: full access.

5. **Expiry**
   - A scheduled job runs periodically (e.g., every 5 minutes) to set status to Expired for any Checked In visit where `valid_until < NOW()`.
   - During check‑in attempt, the system also ensures that no active visit exists (already covered by duplicate rule).

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
| `visitor_id` | VARCHAR(20) | FK → visitors(visitor_id) |
| `status` | ENUM('Checked In','Checked Out','Expired') | NOT NULL |
| `entry_time` | TIMESTAMP | NOT NULL |
| `exit_time` | TIMESTAMP | NULL |
| `gate_pass_duration_hours` | INT | NOT NULL |
| `gate_pass_issued_at` | TIMESTAMP | NOT NULL |
| `gate_pass_valid_until` | TIMESTAMP | NOT NULL |
| `template_type` | VARCHAR(20) | DEFAULT 'Standard' |

### `gate_passes` (optional – to log print actions)

| Column | Type | Constraints |
|--------|------|-------------|
| `pass_id` | BIGINT | PK AUTO_INCREMENT |
| `visit_id` | BIGINT | FK → visits(visit_id) UNIQUE |
| `printed_at` | TIMESTAMP | NOT NULL |
| `printed_by` | BIGINT | FK → users(user_id) |

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
| `user_id` | BIGINT | FK → users(user_id) |
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

## 8. JPA Repository Methods with Native Queries

### VisitorRepository

```java
public interface VisitorRepository extends JpaRepository<Visitor, String> {

    // Find all visitors with pagination and optional search (name or ID)
    @Query(value = "SELECT v.visitor_id, v.name, v.company, " +
                   "COALESCE(vs.status, 'Pending') as status " +
                   "FROM visitors v " +
                   "LEFT JOIN ( " +
                   "    SELECT visitor_id, status, " +
                   "           ROW_NUMBER() OVER (PARTITION BY visitor_id ORDER BY visit_id DESC) as rn " +
                   "    FROM visits " +
                   ") vs ON v.visitor_id = vs.visitor_id AND vs.rn = 1 " +
                   "WHERE (:search IS NULL OR v.visitor_id LIKE CONCAT('%', :search, '%') " +
                   "       OR v.name LIKE CONCAT('%', :search, '%')) " +
                   "ORDER BY v.name ASC " +
                   "LIMIT :limit OFFSET :offset",
           countQuery = "SELECT COUNT(*) FROM visitors v " +
                        "WHERE (:search IS NULL OR v.visitor_id LIKE CONCAT('%', :search, '%') " +
                        "       OR v.name LIKE CONCAT('%', :search, '%'))",
           nativeQuery = true)
    Page<Object[]> findVisitorsWithStatus(@Param("search") String search,
                                          @Param("limit") int limit,
                                          @Param("offset") int offset,
                                          Pageable pageable);

    // Get visitor details with current visit
    @Query(value = "SELECT v.visitor_id, v.name, v.company, v.reason_for_visit, " +
                   "v.contact_number, v.email, v.notes, v.created_at, " +
                   "vs.visit_id, vs.status, vs.entry_time, vs.exit_time, " +
                   "vs.gate_pass_valid_until, vs.gate_pass_duration_hours, vs.template_type " +
                   "FROM visitors v " +
                   "LEFT JOIN visits vs ON v.visitor_id = vs.visitor_id " +
                   "AND vs.visit_id = (SELECT visit_id FROM visits WHERE visitor_id = v.visitor_id " +
                   "                   ORDER BY visit_id DESC LIMIT 1) " +
                   "WHERE v.visitor_id = :visitorId",
           nativeQuery = true)
    Optional<Object[]> findVisitorDetail(@Param("visitorId") String visitorId);

    // Autocomplete suggestions (name or ID)
    @Query(value = "SELECT visitor_id, name FROM visitors " +
                   "WHERE visitor_id LIKE CONCAT(:q, '%') OR name LIKE CONCAT(:q, '%') " +
                   "LIMIT 10", nativeQuery = true)
    List<Object[]> findSuggestions(@Param("q") String q);
}
```

### VisitRepository

```java
public interface VisitRepository extends JpaRepository<Visit, Long> {

    // Check if a visitor has an active visit (Checked In or Expired without check-out)
    @Query(value = "SELECT COUNT(*) FROM visits " +
                   "WHERE visitor_id = :visitorId " +
                   "AND status IN ('Checked In', 'Expired')",
           nativeQuery = true)
    int countActiveVisitsByVisitorId(@Param("visitorId") String visitorId);

    // Get the latest visit for a visitor (for status display)
    @Query(value = "SELECT * FROM visits WHERE visitor_id = :visitorId " +
                   "ORDER BY visit_id DESC LIMIT 1",
           nativeQuery = true)
    Optional<Visit> findLatestByVisitorId(@Param("visitorId") String visitorId);

    // Update expired visits
    @Modifying
    @Query(value = "UPDATE visits SET status = 'Expired' " +
                   "WHERE status = 'Checked In' AND gate_pass_valid_until < NOW()",
           nativeQuery = true)
    int updateExpiredVisits();
}
```

### GatePassRepository

```java
public interface GatePassRepository extends JpaRepository<GatePass, Long> {

    // Log a print action
    @Query(value = "INSERT INTO gate_passes (visit_id, printed_at, printed_by) " +
                   "VALUES (:visitId, NOW(), :userId)",
           nativeQuery = true)
    @Modifying
    int logPrint(@Param("visitId") Long visitId, @Param("userId") Long userId);
}
```

### ReportRepository (Custom)

```java
public interface ReportRepository {

    // Get visitor counts grouped by day/week/month
    @Query(value = "SELECT DATE(entry_time) as date, COUNT(*) as count " +
                   "FROM visits " +
                   "WHERE entry_time BETWEEN :from AND :to " +
                   "GROUP BY DATE(entry_time) " +
                   "ORDER BY date",
           nativeQuery = true)
    List<Object[]> getDailyVisitorCounts(@Param("from") LocalDateTime from,
                                          @Param("to") LocalDateTime to);

    // For weekly/monthly grouping, similar queries with DATE_FORMAT
    @Query(value = "SELECT DATE_FORMAT(entry_time, '%Y-%u') as week, COUNT(*) as count " +
                   "FROM visits " +
                   "WHERE entry_time BETWEEN :from AND :to " +
                   "GROUP BY week " +
                   "ORDER BY week",
           nativeQuery = true)
    List<Object[]> getWeeklyVisitorCounts(@Param("from") LocalDateTime from,
                                           @Param("to") LocalDateTime to);

    @Query(value = "SELECT DATE_FORMAT(entry_time, '%Y-%m') as month, COUNT(*) as count " +
                   "FROM visits " +
                   "WHERE entry_time BETWEEN :from AND :to " +
                   "GROUP BY month " +
                   "ORDER BY month",
           nativeQuery = true)
    List<Object[]> getMonthlyVisitorCounts(@Param("from") LocalDateTime from,
                                            @Param("to") LocalDateTime to);
}
```

---

## 9. Workflow / Process Flow (with API Calls)

### Step 1: Visitor Registration (Receptionist)
- **API:** `POST /api/v1/visitors`  
  **Request:**
  ```json
  {
    "name": "John Doe",
    "company": "Acme Corp",
    "reasonForVisit": "Business Meeting",
    "contactNumber": "+1234567890",
    "email": "john.doe@acme.com",
    "notes": "Meeting with IT team"
  }
  ```
  **Response (201 Created):**
  ```json
  {
    "visitorId": "VIS-20250325-001",
    "name": "John Doe",
    "company": "Acme Corp",
    "createdAt": "2025-03-25T09:00:00"
  }
  ```

### Step 2: Security Officer Searches for Visitor (Initial Table Load)
- **API:** `GET /api/v1/visitors?page=0&size=10&sort=name,asc`  
  **Response (200 OK):**
  ```json
  {
    "content": [
      {
        "visitorId": "VIS-20250325-001",
        "name": "John Doe",
        "company": "Acme Corp",
        "status": "Pending"
      },
      {
        "visitorId": "VIS-20250325-002",
        "name": "Jane Smith",
        "company": "Beta Inc",
        "status": "Checked In"
      }
    ],
    "pageable": { "pageNumber": 0, "pageSize": 10 },
    "totalPages": 5,
    "totalElements": 50
  }
  ```

### Step 3: Gate Pass Generation (Check‑in)
- **API:** `POST /api/v1/visits`  
  **Request:**
  ```json
  {
    "visitorId": "VIS-20250325-001",
    "durationHours": 3,
    "templateType": "Standard"
  }
  ```
  **Response (201 Created):**
  ```json
  {
    "visitId": 101,
    "visitorId": "VIS-20250325-001",
    "visitorName": "John Doe",
    "company": "Acme Corp",
    "entryTime": "2025-03-25T09:00:00",
    "validUntil": "2025-03-25T12:00:00",
    "durationHours": 3,
    "templateType": "Standard",
    "status": "Checked In"
  }
  ```

### Step 4: Print Gate Pass (Optional Logging)
- **API:** `POST /api/v1/visits/{visitId}/print`  
  **Response:**
  ```json
  { "message": "Print action logged" }
  ```

### Step 5: Check‑out
- **API:** `PUT /api/v1/visits/{visitId}/checkout`  
  **Response:**
  ```json
  {
    "visitId": 101,
    "visitorId": "VIS-20250325-001",
    "visitorName": "John Doe",
    "company": "Acme Corp",
    "entryTime": "2025-03-25T09:00:00",
    "exitTime": "2025-03-25T11:30:00",
    "validUntil": "2025-03-25T12:00:00",
    "durationHours": 3,
    "templateType": "Standard",
    "status": "Checked Out"
  }
  ```

### Step 6: Reporting (Administrator)
- **API:** `GET /api/v1/reports/visitors?from=2025-03-01&to=2025-03-31&groupBy=day`  
  **Response:**
  ```json
  {
    "labels": ["2025-03-25", "2025-03-26"],
    "data": [12, 8]
  }
  ```
- **Export CSV:** `GET /api/v1/reports/export?from=2025-03-01&to=2025-03-31` → CSV file.

---

## 10. API Design (Complete with Examples)

All endpoints under `/api/v1`

| Endpoint | Method | Purpose | Request Example | Response Example |
|----------|--------|---------|-----------------|------------------|
| `/visitors` | POST | Register visitor | `{ "name": "...", "company": "...", "reasonForVisit": "...", "contactNumber": "...", "email": "...", "notes": "..." }` | 201 Created `{ "visitorId": "...", "name": "...", "company": "...", "createdAt": "..." }` |
| `/visitors` | GET | Paginated list of visitors (optional search) | `?q=John&page=0&size=10&sort=name,asc` | `Page<VisitorSummaryDTO>` |
| `/visitors/suggest` | GET | Autocomplete suggestions | `?q=John` | `[ { "visitorId": "...", "name": "..." } ]` |
| `/visitors/{id}` | GET | Full visitor details | – | `VisitorDetailDTO` (includes current visit) |
| `/visits` | POST | Check‑in & generate pass | `{ "visitorId": "...", "durationHours": 3, "templateType": "Standard" }` | 201 Created `VisitDTO` |
| `/visits/{visitId}/checkout` | PUT | Check‑out | – | `VisitDTO` with updated status |
| `/visits/{visitId}/print` | POST | Log print action | – | `{ "message": "Print action logged" }` |
| `/reports/visitors` | GET | Visitor counts | `?from=...&to=...&groupBy=day/week/month` | `{ "labels": [...], "data": [...] }` |
| `/reports/export` | GET | CSV export | same filters | CSV file |
| `/config` | GET / PUT | System configuration | `ConfigDTO` | `ConfigDTO` |
| `/users` | CRUD | User management (admin) | `UserDTO` | 200/201 |

---

## 11. Security & Access Control
- **Authentication:** JWT (stateless) with refresh token.
- **Authorization:** Role‑based (RBAC) using Spring Security method‑level annotations (`@PreAuthorize`).
- **Password:** BCrypt hashing.
- **Validation:** Bean validation on all DTOs.
- **Protection:** CORS configured, rate limiting on public endpoints, SQL injection prevented by JPA (native queries use parameter binding).

---

## 12. Edge Cases & Exception Handling

| Scenario | Handling |
|----------|----------|
| Visitor already has an active visit | `POST /visits` returns `409 Conflict` with message “Visitor already has an active pass. Please check out first.” |
| Check‑in with expired pass (if ever attempted) | Not possible because duplicate active check prevents it; the old visit would already be Expired or Checked Out. |
| Invalid duration (out of range) | `POST /visits` returns `400 Bad Request` with validation error. |
| Search returns no results | `200 OK` with empty array. |
| Unauthorized access | `401 Unauthorized`; no extra info. |
| Database failure | Circuit breaker returns generic error, logs details. |

---

## 13. Logging & Monitoring

### Logging
- All service method entry/exit via AOP.
- Audit logs stored in `audit_logs` table (CRUD, check‑in/out, prints).
- Security events (login failures, permission violations) logged at INFO.
- Exceptions logged at ERROR with stack trace.

### Monitoring
- Spring Boot Actuator endpoints: `health`, `metrics`, circuit breakers.
- Custom metrics: `visitor_registration_count`, `gate_pass_generated_count`, `checkin_count`, `checkout_count`.
- Alerts: high error rate (>5%), circuit breaker open, database connection pool issues.

### Caching
- Cache visitor search results for 5 minutes using Spring Cache (Redis or in‑memory).

---

This design fully supports the gate pass generation UI, provides clear API examples, defines a consistent workflow, and includes all necessary native queries for JPA repositories.
