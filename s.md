

# Visitor Management & Gate Pass Automation – Final Technical Specification

## 1. Core Entities, Attributes, and Relationships

The data model consists of five main entities: **Role**, **User**, **Visitor**, **VisitRecord**, and **SystemConfig**.  
The key design decision is that every **visit** (check‑in) has its own `reasonForVisit` and `visitNotes`, allowing a single visitor to have multiple visits with different purposes.

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

### 1.3 Visitor (Permanent Information)
| Attribute         | Type     | Description                              |
|-------------------|----------|------------------------------------------|
| `VisitorID` (PK)  | Integer  | Internal unique identifier               |
| `UniqueID`        | String   | Auto‑generated ID for gate passes (e.g., `V202503250001`) |
| `Name`            | String   | Full name of the visitor                 |
| `Company`         | String   | Company name                             |
| `ContactNumber`   | String   | Phone number (validated format)          |
| `Email`           | String   | Email address (validated)                |
| `Notes`           | String   | General notes about the visitor (e.g., “VIP client”) |
| `Status`          | String   | `Pending`, `Checked‑In`, `Checked‑Out`, `Expired` |
| `CreatedAt`       | DateTime | Registration timestamp                   |
| `CreatedBy` (FK)  | Integer  | References `User` who registered them    |

**Relationships:** Many `Visitors` can be created by one **User**; one `Visitor` can have many **VisitRecords**.

---

### 1.4 VisitRecord (Visit‑Specific Information)
| Attribute               | Type     | Description                              |
|-------------------------|----------|------------------------------------------|
| `RecordID` (PK)         | Integer  | Unique identifier for each visit instance|
| `VisitorID` (FK)        | Integer  | References `Visitor`                     |
| `ReasonForVisit`        | String   | Reason for this specific visit (e.g., “Business Meeting”, “Training”) |
| `VisitNotes`            | String   | Optional notes for this specific visit   |
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

The UI is role‑based, showing only relevant features for each user type.

### 2.1 Receptionist UI
- **Dashboard:**  
  - Prominent “Quick Register New Visitor” button.  
  - List of “Today’s Registered Visitors” with statuses (Pending, Checked‑In).  
  - Small search bar for quick lookups.

- **Visitor Registration Form:**  
  - Card‑based form with fields: **Name**, **Company**, **Contact Number**, **Email**, **General Notes** (optional).  
  - Separate section for **Visit Details**: **Reason for Visit** (dropdown), **Visit Notes** (optional), and **Gate Pass Duration** (dropdown: 1h, 2h, 4h, Custom).  
  - After submission, auto‑generated `UniqueID` is displayed.

- **Visitor Management:**  
  - Table view of all visitors they created.  
  - **Edit Visitor** modal to update permanent visitor details (name, company, contact, email, general notes) – *not* visit‑specific fields.  
  - Option to **Add New Visit** for a repeat visitor, allowing a new reason and duration.

### 2.2 Security Officer UI
- **Dashboard:**  
  - Large, prominent search bar with autocomplete (US 02).  
  - Live feed of recent check‑ins.  
  - Widget: “Currently on Premises: X”.

- **Visitor Search Panel:**  
  - Table of search results with sorting & pagination.  
  - Each row has an “Action” button → “View & Print Gate Pass” for the **latest visit**.

- **Visitor Record Display:**  
  - Detailed view with large `Visitor Name` and `UniqueID`.  
  - Color‑coded status indicator (Green = Checked‑In, Red = Expired, Gray = Pending) (US 09).  
  - **Check‑In / Check‑Out** buttons (US 06).  
  - Gate pass section with **Print** button (US 05), countdown timer, and “Expired” indicator (US 11).  
  - Option to send email notification upon check‑in (US 07).  
  - History of past visits (each with its own reason, dates, and duration).

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

## 3. Gate Pass Content & Design

When the **Print Gate Pass** button is clicked, the system generates a professional, print‑friendly document containing the following information:

| Field               | Description                                                                 | Example                     |
|---------------------|-----------------------------------------------------------------------------|-----------------------------|
| **Visitor Name**    | Full name of the visitor                                                    | John Smith                  |
| **Unique ID**       | Auto‑generated alphanumeric ID (displayed prominently)                      | V202503250001               |
| **Company**         | Visitor’s company name                                                      | Acme Inc.                   |
| **Reason for Visit**| Purpose of the visit (from the visit record)                                | Business Meeting            |
| **Date & Time**     | Check‑in date and time (entry time)                                         | 25 Mar 2025, 10:30 AM       |
| **Duration**        | Allotted gate pass duration (set by receptionist)                           | 4 hours                     |
| **Expiry Time**     | Calculated as entry time + duration                                         | 25 Mar 2025, 2:30 PM        |
| **Status**          | “Active” or “Expired” (visual indicator)                                    | Active (green badge)        |
| **Host/Contact**    | (Optional) Name of the person the visitor is meeting, or department         | Mr. Robert (Sales)          |
| **QR Code / Barcode**| For quick scanning at security gates (enhances automation)                 | [QR code image]             |

The printed pass includes a **countdown timer** on the digital version (optional) and clearly indicates the expiry time. The QR code encodes the visitor’s `uniqueId` or a URL that can be scanned for quick lookup.

---

## 4. API Endpoints with Request/Response Examples

All endpoints are prefixed with `/api/v1/`. Authentication is performed via JWT (in `Authorization` header).  
`{visitorId}` refers to the internal primary key; `uniqueId` is the auto‑generated alphanumeric identifier displayed on gate passes. `{recordId}` is the primary key of `VisitRecord`.

### 4.1 Authentication & User Management

| Method | Endpoint | Description | Request Body (JSON) | Response (JSON) |
|--------|----------|-------------|---------------------|-----------------|
| `POST` | `/auth/login` | Authenticate a user | `{ "email": "string", "password": "string" }` | `{ "token": "string", "user": { ... } }` |
| `GET` | `/users` | (Admin) List all users | — | `{ "users": [ { "userId": 1, "name": "...", "email": "...", "role": { "roleId": 1, "roleName": "Receptionist" } } ] }` |
| `POST` | `/users` | (Admin) Create a new user | `{ "name": "string", "email": "string", "password": "string", "roleId": integer }` | `{ "userId": integer, "name": "...", "email": "...", "role": { ... } }` |
| `PUT` | `/users/{userId}` | (Admin) Update user | `{ "name": "string (opt)", "email": "string (opt)", "roleId": integer (opt) }` | Updated user object |
| `GET` | `/roles` | (Admin) Get all roles | — | `{ "roles": [ { "roleId": 1, "roleName": "Receptionist", "description": "..." } ] }` |

### 4.2 Visitor Management

| Method | Endpoint | Description | Request Body (JSON) | Response (JSON) |
|--------|----------|-------------|---------------------|-----------------|
| `POST` | `/visitors` | Register a new visitor **with an initial visit** (US 01, US 04). | `{ "name": "string (req)", "company": "string (req)", "contactNumber": "string (req)", "email": "string (req)", "notes": "string (opt)", "reasonForVisit": "string (req)", "visitNotes": "string (opt)", "gatePassDuration": integer (opt) }` | Visitor object with `visitRecords` array containing the initial visit. `uniqueId` auto‑generated. |
| `POST` | `/visitors/{visitorId}/visits` | Add a **new visit** for an existing visitor (repeat visit). | `{ "reasonForVisit": "string (req)", "visitNotes": "string (opt)", "gatePassDuration": integer (opt) }` | The newly created `VisitRecord` object. |
| `GET` | `/visitors` | Paginated list of visitors | Query params: `page`, `limit`, `sortBy`, `order`, `status` (opt) | `{ "visitors": [ ... ], "total": integer, "page": integer, "limit": integer }` |
| `GET` | `/visitors/search?q={term}` | Search by name or `uniqueId` (US 02) | Query param: `q` | `{ "visitors": [ { "visitorId": integer, "name": "...", "uniqueId": "...", "company": "...", "status": "..." } ] }` |
| `GET` | `/visitors/{visitorId}` | Get detailed visitor info (including all visit records) | — | Full visitor object with `visitRecords` list. |
| `PUT` | `/visitors/{visitorId}` | Update **permanent visitor info** (US 13) – not visit‑specific. | `{ "name": "string (opt)", "company": "string (opt)", "contactNumber": "string (opt)", "email": "string (opt)", "notes": "string (opt)" }` | Updated visitor object. |
| `PUT` | `/visitors/{visitorId}/visits/{recordId}` | Update a specific visit (e.g., reason, notes) – optional. | `{ "reasonForVisit": "string (opt)", "visitNotes": "string (opt)", "gatePassDuration": integer (opt) }` | Updated `VisitRecord` object. |

### 4.3 Visit & Gate Pass Management

| Method | Endpoint | Description | Request Body (JSON) | Response (JSON) |
|--------|----------|-------------|---------------------|-----------------|
| `POST` | `/visitors/{visitorId}/visits/{recordId}/checkin` | Record entry time for a specific visit (US 06). May trigger email notification (US 07). | `{ "gatePassDuration": integer (opt, overrides), "gatePassTemplate": "string (opt)" }` | Updated `VisitRecord` with `entryTime`, `gatePassExpiryTime`. Visitor status updated to `Checked‑In`. |
| `PUT` | `/visitors/{visitorId}/visits/{recordId}/checkout` | Record exit time for a specific visit (US 06). | — | Updated `VisitRecord` with `exitTime`. Visitor status updated to `Checked‑Out` (if no other active visits). |
| `POST` | `/visitors/{visitorId}/visits/{recordId}/gate-pass` | Generate / regenerate gate pass for a specific visit (US 05, US 10). | `{ "duration": integer (opt, hours), "template": "string (opt)" }` | `{ "gatePassData": { "visitorName": "...", "visitorId": "...", "company": "...", "reason": "...", "dateTime": "...", "expiryTime": "...", "status": "Active" } }` |
| `GET` | `/visitors/{visitorId}/visits/{recordId}/gate-pass` | Get latest gate pass for a specific visit (includes expiry status, US 11). | — | Same as above; `status` may be `"Expired"` if expiry time passed. |

### 4.4 Reporting & Configuration

| Method | Endpoint | Description | Request Body (JSON) | Response (JSON) |
|--------|----------|-------------|---------------------|-----------------|
| `GET` | `/reports/visitors` | Visitor count trends based on visit records (US 12) | Query params: `groupBy` = `day`\|`week`\|`month`, `startDate`, `endDate` | `{ "data": [ { "period": "2025-03-25", "count": 15 }, ... ] }` |
| `GET` | `/reports/visitors/export` | Export visit data to CSV (US 08). | Query params: `startDate`, `endDate`, `status` (opt) | Returns CSV file (Content‑Disposition: attachment) |
| `GET` | `/config` | Get system configuration | — | `{ "defaultGatePassDuration": 4, "notificationEmail": "security@company.com", "allowedDurations": [1,2,4,8], "gatePassTemplates": ["Standard","Premium"] }` |
| `PUT` | `/config` | (Admin) Update system configuration (US 10) | `{ "defaultGatePassDuration": integer (opt), "notificationEmail": "string (opt)", "allowedDurations": [integer] (opt), "gatePassTemplates": ["string"] (opt) }` | Updated config object |

---

## 5. Data Type Notes
- **Timestamps:** ISO 8601 format (e.g., `"2025-03-25T14:30:00Z"`).  
- **IDs:** `visitorId`, `userId`, `recordId` are integers (or UUIDs).  
- **UniqueID:** Alphanumeric, e.g., `"V202503250001"`.  
- **Status Values:** `"Pending"`, `"Checked‑In"`, `"Checked‑Out"`, `"Expired"`.  
- **ReasonForVisit:** Predefined options (e.g., “Business Meeting”, “Client Visit”, “Interview”, “Delivery”) or free text.

---

## 6. Validation & Exception Messages (Key‑Value Pairs)

| Error Key              | Message                                                                 |
|------------------------|-------------------------------------------------------------------------|
| `MISSING_FIELD`        | "The field '{field}' is required."                                       |
| `INVALID_EMAIL`        | "Please provide a valid email address."                                  |
| `INVALID_PHONE`        | "Phone number must contain at least 10 digits."                          |
| `INVALID_DURATION`     | "Gate pass duration must be one of the allowed values: {allowedList}."   |
| `DUPLICATE_UNIQUE_ID`  | "System error: could not generate a unique ID. Please try again."        |
| `ROLE_NOT_FOUND`       | "Role with ID {roleId} does not exist."                                  |
| `UNAUTHORIZED`         | "You do not have permission to perform this action."                     |
| `VISITOR_NOT_FOUND`    | "Visitor with ID {visitorId} does not exist."                            |
| `ALREADY_CHECKED_IN`   | "Visitor is already checked in."                                         |
| `NOT_CHECKED_IN`       | "Visitor is not currently checked in."                                   |
| `EMAIL_FAILED`         | "Email notification could not be sent. Please check email configuration."|
| `INVALID_DATE_RANGE`   | "End date must be after start date."                                     |

---

This specification provides a complete foundation for development, aligning the data model, UI, API contracts, gate pass design, and error handling with the original requirements.
