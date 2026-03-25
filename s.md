
# Visitor Management & Gate Pass Automation – Technical Specification

## 1. Core Entities, Attributes, and Relationships

The data model is built around four main entities:

### 1.1 Role
| Attribute       | Type     | Description                              |
|-----------------|----------|------------------------------------------|
| `RoleID` (PK)   | Integer  | Unique identifier                        |
| `RoleName`      | String   | e.g., Receptionist, Security Officer, Administrator |
| `Description`   | String   | Brief description of the role            |
| `Permissions`   | JSON     | (Optional) Fine‑grained access rights    |

**Relationships:** One `Role` can be assigned to many **Users**.

---

### 1.2 User (System User)
| Attribute         | Type     | Description                              |
|-------------------|----------|------------------------------------------|
| `UserID` (PK)     | Integer  | Unique identifier                        |
| `Name`            | String   | Full name of the user                    |
| `Email`           | String   | Unique, used for login                   |
| `PasswordHash`    | String   | Securely stored password                 |
| `RoleID` (FK)     | Integer  | References `Role`                         |
| `CreatedAt`       | DateTime | Timestamp of account creation            |
| `LastLogin`       | DateTime | Last successful login time               |

**Relationships:** Many `Users` belong to one **Role**; one `User` can register many **Visitors**.

---

### 1.3 Visitor
| Attribute         | Type     | Description                              |
|-------------------|----------|------------------------------------------|
| `VisitorID` (PK)  | Integer  | Internal unique identifier               |
| `UniqueID`        | String   | Auto‑generated ID for gate passes (e.g., `V202503250001`) |
| `Name`            | String   | Full name of the visitor                 |
| `Company`         | String   | Company name                             |
| `ContactNumber`   | String   | Phone number (validated format)          |
| `Email`           | String   | Email address (validated)                |
| `ReasonForVisit`  | String   | Predefined or free‑text reason           |
| `Notes`           | String   | Optional additional information          |
| `Status`          | String   | `Pending`, `Checked‑In`, `Checked‑Out`, `Expired` |
| `CreatedAt`       | DateTime | Registration timestamp                   |
| `CreatedBy` (FK)  | Integer  | References `User` who registered them    |

**Relationships:** Many `Visitors` can be created by one **User**; one `Visitor` can have many **VisitRecords**.

---

### 1.4 VisitRecord
| Attribute               | Type     | Description                              |
|-------------------------|----------|------------------------------------------|
| `RecordID` (PK)         | Integer  | Unique identifier for each visit instance|
| `VisitorID` (FK)        | Integer  | References `Visitor`                     |
| `GatePassDuration`      | Integer  | Duration in hours (e.g., 1, 4)           |
| `GatePassTemplate`      | String   | e.g., `Standard`, `Premium`              |
| `EntryTime`             | DateTime | Time when visitor checked in             |
| `ExitTime`              | DateTime | Time when visitor checked out            |
| `GatePassExpiryTime`    | DateTime | Calculated as `EntryTime + Duration`     |
| `StatusAtTime`          | String   | Snapshot of visitor status at check‑in   |

**Relationships:** Many `VisitRecords` belong to one **Visitor**. This table captures each individual visit.

---

### Entity‑Relationship Diagram (Conceptual)
```
Role ────< User ────< Visitor ────< VisitRecord
```
- One **Role** → many **Users**
- One **User** → many **Visitors**
- One **Visitor** → many **VisitRecords**

---

## 2. User Interface by Role

The application will have role‑specific dashboards to show only the features relevant to each user type.

### 2.1 Receptionist UI
- **Dashboard:**  
  - Prominent “Quick Register New Visitor” button.  
  - List of “Today’s Registered Visitors” with statuses (Pending, Checked‑In).  
  - Small search bar for quick lookups.

- **Visitor Registration Form:**  
  - Card‑based form with fields: Name, Company, Reason for Visit (dropdown), Contact Number, Email, Notes.  
  - After submission, auto‑generated `UniqueID` is displayed.  
  - Option to set **Gate Pass Duration** (dropdown: 1h, 2h, 4h, Custom).

- **Visitor Management:**  
  - Table view of all visitors they created.  
  - **Edit Visitor** modal to update contact details or reason for visit (US 13).

### 2.2 Security Officer UI
- **Dashboard:**  
  - Large, prominent search bar with autocomplete (US 02).  
  - Live feed of recent check‑ins.  
  - Widget: “Currently on Premises: X”.

- **Visitor Search Panel:**  
  - Table of search results with sorting & pagination.  
  - Each row has an “Action” button → “View & Print Gate Pass”.

- **Visitor Record Display:**  
  - Detailed view with large `Visitor Name` and `UniqueID`.  
  - Color‑coded status indicator (Green = Checked‑In, Red = Expired, Gray = Pending) (US 09).  
  - **Check‑In / Check‑Out** buttons (US 06).  
  - Gate pass section with **Print** button (US 05), countdown timer, and “Expired” indicator (US 11).  
  - Option to send email notification upon check‑in (US 07).

### 2.3 System Administrator UI
- **Dashboard:**  
  - Charts for visitor trends (line/bar) grouped by day/week/month (US 12).  
  - “Export to CSV” button for reports (US 08).

- **User Management:**  
  - List of system users.  
  - Form to add new users, assign a **Role** (US 03).  
  - Interface to configure global settings: default gate pass duration, allowed durations, notification email (US 10).

- **Reporting & Audit:**  
  - Dedicated reports section with filters (date range, status) and export options.

---

## 3. API Endpoints with Request/Response Examples

All endpoints are prefixed with `/api/v1/`. Authentication is performed via JWT (in `Authorization` header).  
`{visitorId}` refers to the internal primary key; `uniqueId` is the auto‑generated alphanumeric identifier displayed on gate passes.

### 3.1 Authentication & User Management

| Method | Endpoint | Description | Request Body (JSON) | Response (JSON) |
|--------|----------|-------------|---------------------|-----------------|
| `POST` | `/auth/login` | Authenticate a user | `{ "email": "string", "password": "string" }` | `{ "token": "string", "user": { ... } }` |
| `GET` | `/users` | (Admin) List all users | — | `{ "users": [ { "userId": 1, "name": "...", "email": "...", "role": { "roleId": 1, "roleName": "Receptionist" } } ] }` |
| `POST` | `/users` | (Admin) Create a new user | `{ "name": "string", "email": "string", "password": "string", "roleId": integer }` | `{ "userId": integer, "name": "...", "email": "...", "role": { ... } }` |
| `PUT` | `/users/{userId}` | (Admin) Update user | `{ "name": "string (opt)", "email": "string (opt)", "roleId": integer (opt) }` | Updated user object |
| `GET` | `/roles` | (Admin) Get all roles | — | `{ "roles": [ { "roleId": 1, "roleName": "Receptionist", "description": "..." } ] }` |

### 3.2 Visitor Management

| Method | Endpoint | Description | Request Body (JSON) | Response (JSON) |
|--------|----------|-------------|---------------------|-----------------|
| `POST` | `/visitors` | Register a new visitor (US 01, US 04) | `{ "name": "string (req)", "company": "string (req)", "reasonForVisit": "string (req)", "contactNumber": "string (req)", "email": "string (req, valid)", "notes": "string (opt)", "gatePassDuration": integer (opt, hours) }` | Full visitor object (includes auto‑generated `uniqueId`) |
| `GET` | `/visitors` | Paginated list of visitors | Query params: `page`, `limit`, `sortBy`, `order`, `status` (opt) | `{ "visitors": [ ... ], "total": integer, "page": integer, "limit": integer }` |
| `GET` | `/visitors/search?q={term}` | Search by name or `uniqueId` (US 02) | Query param: `q` | `{ "visitors": [ { "visitorId": integer, "name": "...", "uniqueId": "...", "company": "...", "status": "..." } ] }` |
| `GET` | `/visitors/{visitorId}` | Get detailed visitor info | — | Full visitor object (includes latest `visitRecord` if exists) |
| `PUT` | `/visitors/{visitorId}` | Update visitor info (US 13) | `{ "name": "string (opt)", "company": "string (opt)", "reasonForVisit": "string (opt)", "contactNumber": "string (opt)", "email": "string (opt)", "notes": "string (opt)" }` | Updated visitor object |

### 3.3 Visit & Gate Pass Management

| Method | Endpoint | Description | Request Body (JSON) | Response (JSON) |
|--------|----------|-------------|---------------------|-----------------|
| `POST` | `/visitors/{visitorId}/checkin` | Record entry time (US 06). May trigger email notification (US 07). | `{ "gatePassDuration": integer (opt, hours), "gatePassTemplate": "string (opt, e.g., 'Standard')" }` | `{ "visitRecordId": integer, "visitorId": integer, "entryTime": "ISO timestamp", "gatePassDuration": integer, "gatePassTemplate": "...", "gatePassExpiryTime": "ISO timestamp", "status": "Checked‑In" }` |
| `PUT` | `/visitors/{visitorId}/checkout` | Record exit time (US 06) | — | `{ "visitRecordId": integer, "exitTime": "ISO timestamp", "status": "Checked‑Out" }` |
| `POST` | `/visitors/{visitorId}/gate-pass` | Generate / regenerate gate pass (US 05, US 10) | `{ "duration": integer (opt, hours), "template": "string (opt)" }` | `{ "gatePassData": { "visitorName": "...", "visitorId": "...", "company": "...", "dateTime": "...", "expiryTime": "...", "status": "Active" } }` |
| `GET` | `/visitors/{visitorId}/gate-pass` | Get latest gate pass (includes expiry status, US 11) | — | Same as above; `status` may be `"Expired"` if expiry time passed |

### 3.4 Reporting & Configuration

| Method | Endpoint | Description | Request Body (JSON) | Response (JSON) |
|--------|----------|-------------|---------------------|-----------------|
| `GET` | `/reports/visitors` | Visitor count trends (US 12) | Query params: `groupBy` = `day`\|`week`\|`month`, `startDate`, `endDate` (ISO dates) | `{ "data": [ { "period": "2025-03-25", "count": 15 }, ... ] }` |
| `GET` | `/reports/visitors/export` | Export visitor data to CSV (US 08) | Query params: `startDate`, `endDate`, `status` (opt) | Returns CSV file (Content‑Disposition: attachment) |
| `GET` | `/config` | Get system configuration | — | `{ "defaultGatePassDuration": 4, "notificationEmail": "security@company.com", "allowedDurations": [1,2,4,8], "gatePassTemplates": ["Standard","Premium"] }` |
| `PUT` | `/config` | (Admin) Update system configuration (US 10) | `{ "defaultGatePassDuration": integer (opt), "notificationEmail": "string (opt)", "allowedDurations": [integer] (opt), "gatePassTemplates": ["string"] (opt) }` | Updated config object |

---

## 4. Data Type Notes
- **Timestamps:** ISO 8601 format (e.g., `"2025-03-25T14:30:00Z"`).  
- **IDs:** `visitorId` and `userId` are integers (or UUIDs).  
- **UniqueID:** Alphanumeric, e.g., `"V202503250001"`.  
- **Status Values:** `"Pending"`, `"Checked‑In"`, `"Checked‑Out"`, `"Expired"`.

---

This specification provides a solid foundation for development. It clearly defines the data model, role‑based interfaces, and the exact API contracts needed to implement the Visitor Management and Gate Pass Automation system.
```
