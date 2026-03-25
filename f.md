
# Project Structure

Below is the directory layout of the Visitor Management System Spring Boot application, along with a brief description of each component.

```
src/main/java/com/example/visitormanagement/
│
├── VisitorManagementApplication.java          # Main Spring Boot entry point
│
├── config/                                     # Configuration classes
│   ├── SecurityConfig.java                     # Spring Security + JWT setup
│   ├── JwtAuthenticationFilter.java            # JWT request filter
│   ├── WebConfig.java                          # CORS, MVC configuration
│   └── SwaggerConfig.java                      # API documentation (OpenAPI)
│
├── controller/                                 # REST API endpoints
│   ├── AuthController.java                     # /auth/login, /auth/refresh
│   ├── UserController.java                     # /users (admin only)
│   ├── VisitorController.java                  # /visitors
│   ├── VisitController.java                    # /visitors/{id}/visits
│   ├── GatePassController.java                 # /visitors/{id}/visits/{rid}/gate-pass
│   ├── ReportController.java                   # /reports
│   └── ConfigController.java                   # /config
│
├── service/                                    # Business logic
│   ├── AuthService.java                        # Authentication logic
│   ├── UserService.java                        # User management
│   ├── VisitorService.java                     # Visitor operations
│   ├── VisitService.java                       # Visit record operations
│   ├── GatePassService.java                    # Gate pass generation
│   ├── ReportService.java                      # Reporting & export
│   └── EmailService.java                       # Email notifications
│
├── repository/                                 # Spring Data JPA repositories
│   ├── RoleRepository.java
│   ├── UserRepository.java
│   ├── VisitorRepository.java
│   ├── VisitRecordRepository.java
│   └── SystemConfigRepository.java
│
├── entity/                                     # JPA entities (domain objects)
│   ├── Role.java
│   ├── User.java
│   ├── Visitor.java
│   ├── VisitRecord.java
│   └── SystemConfig.java
│
├── dto/                                        # Data Transfer Objects
│   ├── request/                                # Incoming payloads
│   │   ├── LoginRequest.java
│   │   ├── UserCreateRequest.java
│   │   ├── UserUpdateRequest.java
│   │   ├── VisitorCreateRequest.java          # includes reason, duration
│   │   ├── VisitorUpdateRequest.java
│   │   ├── VisitCreateRequest.java
│   │   ├── VisitUpdateRequest.java
│   │   ├── CheckinRequest.java
│   │   └── ConfigUpdateRequest.java
│   └── response/                               # Outgoing payloads
│       ├── LoginResponse.java
│       ├── UserResponse.java
│       ├── VisitorResponse.java
│       ├── VisitRecordResponse.java
│       ├── GatePassResponse.java
│       ├── ReportDataResponse.java
│       └── ConfigResponse.java
│
├── enums/                                      # Enumerations
│   ├── RoleName.java
│   ├── VisitorStatus.java
│   └── GatePassTemplate.java
│
├── exception/                                  # Custom exceptions & global handler
│   ├── BusinessException.java                  # Base custom exception
│   ├── ResourceNotFoundException.java
│   ├── UnauthorizedException.java
│   ├── InvalidRequestException.java
│   └── GlobalExceptionHandler.java             # @ControllerAdvice
│
├── security/                                   # Security utilities
│   ├── JwtTokenProvider.java                   # JWT generation/validation
│   └── SecurityUtils.java                      # Helper methods (get current user)
│
├── util/                                       # Utility classes
│   ├── UniqueIdGenerator.java                  # Generate visitor uniqueId
│   ├── DateTimeUtils.java                      # Timestamp formatting
│   ├── CsvExportUtil.java                      # CSV generation logic
│   ├── ValidationUtils.java                    # Custom validators (phone, email)
│   └── Constants.java                          # Application constants
│
└── validator/                                  # Custom validation annotations (optional)
    ├── PhoneNumber.java
    └── PhoneNumberValidator.java
```

## Package Overview

| Package          | Purpose |
|------------------|---------|
| **`config`**     | Spring configuration classes: security, JWT filter, CORS, and OpenAPI (Swagger) docs. |
| **`controller`** | REST endpoints handling HTTP requests. |
| **`service`**    | Core business logic, transactional operations, and external integrations (email). |
| **`repository`** | Spring Data JPA repositories for database access. |
| **`entity`**     | JPA entities mapping to database tables. |
| **`dto`**        | Data Transfer Objects for request validation and response serialization. |
| **`enums`**      | Enumerations for role names, visitor statuses, and gate pass templates. |
| **`exception`**  | Custom exception classes and a global exception handler (`@ControllerAdvice`). |
| **`security`**   | JWT token generation/validation and security utilities. |
| **`util`**       | Helper classes for ID generation, date/time formatting, CSV export, and validation. |
| **`validator`**  | Custom validation annotations (e.g., phone number validator). |


```
