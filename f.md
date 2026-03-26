# project structure

```text
com.vms
│
├── controller
│   ├── AuthController.java
│   ├── UserController.java
│   ├── RoleController.java
│   ├── VisitorController.java
│   ├── VisitController.java
│   ├── ReportController.java
│   └── ConfigController.java
│
├── service
│   ├── AuthService.java
│   ├── UserService.java
│   ├── RoleService.java
│   ├── VisitorService.java
│   ├── VisitService.java
│   ├── ReportService.java
│   └── ConfigService.java
│
├── service/impl
│   ├── AuthServiceImpl.java
│   ├── UserServiceImpl.java
│   ├── RoleServiceImpl.java
│   ├── VisitorServiceImpl.java
│   ├── VisitServiceImpl.java
│   ├── ReportServiceImpl.java
│   └── ConfigServiceImpl.java
│
├── repository
│   ├── UserRepository.java
│   ├── RoleRepository.java
│   ├── VisitorRepository.java
│   └── VisitRecordRepository.java
│
├── entity
│   ├── User.java
│   ├── Role.java
│   ├── Visitor.java
│   └── VisitRecord.java
│
├── dto
│   ├── auth
│   │   ├── LoginRequest.java
│   │   └── LoginResponse.java
│   │
│   ├── user
│   │   └── UserDTO.java
│   │
│   ├── role
│   │   └── RoleDTO.java
│   │
│   ├── visitor
│   │   ├── VisitorRequestDto.java
│   │   ├── VisitorResponseDto.java
│   │   └── VisitorWithVisitRequestDto.java
│   │
│   ├── visit
│   │   ├── VisitRequestDto.java
│   │   └── VisitResponseDto.java
│
├── enums
│   ├── VisitorStatus.java
│   └── RoleType.java (optional)
```

---

# 🧠 LAYER RESPONSIBILITY (VERY IMPORTANT)

| Layer          | Responsibility       |
| -------------- | -------------------- |
| **Entity**     | Database tables      |
| **DTO**        | API request/response |
| **Repository** | DB operations        |
| **Service**    | Business logic       |
| **Controller** | API endpoints        |

---

# 🎯 CONTROLLER ↔ SERVICE ↔ API MAPPING

---

## 🔐 1. AUTH MODULE

| API                | Controller     | Service     |
| ------------------ | -------------- | ----------- |
| POST `/auth/login` | AuthController | AuthService |

👉 Responsibility:

* Authenticate user
* Generate JWT

---

## 👥 2. USER MODULE

| API                   | Controller     | Service     |
| --------------------- | -------------- | ----------- |
| GET `/users`          | UserController | UserService |
| POST `/users`         | UserController | UserService |
| PUT `/users/{userId}` | UserController | UserService |

👉 Responsibility:

* Create user
* Update user
* Fetch users

---

## 🏷️ 3. ROLE MODULE

| API          | Controller     | Service     |
| ------------ | -------------- | ----------- |
| GET `/roles` | RoleController | RoleService |

👉 Responsibility:

* Fetch roles

---

## 👤 4. VISITOR MODULE (Permanent Data)

👉 Handles **visitor profile (WHO)**

| API                         | Controller        | Service        |
| --------------------------- | ----------------- | -------------- |
| POST `/visitors`            | VisitorController | VisitorService |
| GET `/visitors`             | VisitorController | VisitorService |
| GET `/visitors/search`      | VisitorController | VisitorService |
| GET `/visitors/{visitorId}` | VisitorController | VisitorService |
| PUT `/visitors/{visitorId}` | VisitorController | VisitorService |

👉 Responsibility:

* Register visitor (with first visit)
* Update visitor info
* Search & list visitors
* Get visitor details

---

## 🧾 5. VISIT MODULE (Visit Actions)

👉 Handles **visit activity (WHAT happened)**

| API                                               | Controller      | Service      |
| ------------------------------------------------- | --------------- | ------------ |
| POST `/visitors/{id}/visits`                      | VisitController | VisitService |
| PUT `/visitors/{id}/visits/{recordId}`            | VisitController | VisitService |
| POST `/visitors/{id}/visits/{recordId}/checkin`   | VisitController | VisitService |
| PUT `/visitors/{id}/visits/{recordId}/checkout`   | VisitController | VisitService |
| POST `/visitors/{id}/visits/{recordId}/gate-pass` | VisitController | VisitService |
| GET `/visitors/{id}/visits/{recordId}/gate-pass`  | VisitController | VisitService |

👉 Responsibility:

* Add visit
* Update visit
* Check-in / Check-out
* Gate pass generation

---

## 📊 6. REPORT MODULE

| API                            | Controller       | Service       |
| ------------------------------ | ---------------- | ------------- |
| GET `/reports/visitors`        | ReportController | ReportService |
| GET `/reports/visitors/export` | ReportController | ReportService |

👉 Responsibility:

* Visitor analytics
* Export data

---

## ⚙️ 7. CONFIG MODULE

| API           | Controller       | Service       |
| ------------- | ---------------- | ------------- |
| GET `/config` | ConfigController | ConfigService |
| PUT `/config` | ConfigController | ConfigService |

👉 Responsibility:

* System configuration

---

# 🔥 GOLDEN RULE (REMEMBER ALWAYS)

👉 **Visitor = WHO (profile data)**  
👉 **Visit = WHAT (activity/event)**

---

## ✔ Use VisitorService when:

* Name, email, company, notes
* Listing / searching visitors

---

## ✔ Use VisitService when:

* Check-in / Check-out
* Gate pass
* Visit history

---

# 🧠 FINAL MEMORY SHORTCUT

| Question                    | Go To          |
| --------------------------- | -------------- |
| Is it about person?         | VisitorService |
| Is it about visit activity? | VisitService   |
| Is it about system user?    | UserService    |
| Is it about login?          | AuthService    |

---
