## 1. Role Entity

```java
@Entity
@Table(name = "roles")
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer roleId;

   @Enumerated(EnumType.STRING)
   @Column(nullable = false, unique = true
   private RoleType roleName;

    private String description;

    // Optional (if you want JSON permissions later)
    // @Column(columnDefinition = "JSON")
    // private String permissions;

    // One Role → Many Users (optional mapping)
    @OneToMany(mappedBy = "role", fetch = FetchType.LAZY)
    private List<User> users = new ArrayList<>();

    // getters & setters
}
```

## 2. User Entity

```java
@Entity
@Table(name = "roles")
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer roleId;

    @Column(nullable = false, unique = true)
    private String roleName;

    private String description;

    // Optional (if you want JSON permissions later)
    // @Column(columnDefinition = "JSON")
    // private String permissions;

    // One Role → Many Users (optional mapping)
    @OneToMany(mappedBy = "role", fetch = FetchType.LAZY)
    private List<User> users = new ArrayList<>();

    // getters & setters
}
```


## 📦 3. Visitor Entity

```java
@Entity
@Table(name = "visitors")
public class Visitor {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer visitorId;

    @Column(nullable = false, unique = true)
    private String uniqueId;

    @Column(nullable = false)
    private String name;

    private String company;

    @Column(nullable = false)
    private String contactNumber;

    @Column(nullable = false)
    private String email;

    private String notes;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private VisitorStatus status;

    private LocalDateTime createdAt;

    @ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "created_by", nullable = false)
private User createdBy;

    // One Visitor → Many VisitRecords
    @OneToMany(mappedBy = "visitor", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<VisitRecord> visitRecords = new ArrayList<>();

    // getters & setters
}
```

---

## 📦 4. VisitRecord Entity

```java
@Entity
@Table(name = "visit_records")
public class VisitRecord {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer recordId;

    // Many VisitRecords → One Visitor
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "visitor_id", nullable = false)
    private Visitor visitor;

    @Column(nullable = false)
    private String reasonForVisit;

    private String visitNotes;

    private Integer gatePassDuration;

    private String gatePassTemplate;

    private LocalDateTime entryTime;

    private LocalDateTime exitTime;

    private LocalDateTime gatePassExpiryTime;

    @Enumerated(EnumType.STRING)
private VisitorStatus statusAtTime;

    // getters & setters
}
```

---

## 🏷️ 3. VisitorStatus Enum

Using an enum is always better than a plain string.

```java
public enum VisitorStatus {
    PENDING,
    CHECKED_IN,
    CHECKED_OUT,
    EXPIRED
}
```

---

## 🔄 4. Prevent Infinite JSON Loop

Because of the bidirectional relationship, add `@JsonIgnore` on the `visitor` field in `VisitRecord`:

```java
import com.fasterxml.jackson.annotation.JsonIgnore;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "visitor_id", nullable = false)
@JsonIgnore
private Visitor visitor;
```

---

## ⚙️ 5. Optional Industry-Level Improvements

### ✅ Auto Timestamp

Add a `@PrePersist` method to set the creation timestamp automatically.

```java
@PrePersist
public void prePersist() {
    this.createdAt = LocalDateTime.now();
}
```

### ✅ Default Visitor Status

Set a default status before persisting.

```java
@PrePersist
public void setDefaultStatus() {
    if (this.status == null) {
        this.status = VisitorStatus.PENDING;
    }
}
```

### ✅ Default GatePass Template

Handle default values in the service layer:

```java
if (visit.getGatePassTemplate() == null) {
    visit.setGatePassTemplate("Standard");
}
```

---

## 🧩 6. Final Relationship (ER Design)

```
User (1) ────< Visitor (M) ────< VisitRecord (M)
```

- **One Visitor** → Many Visits
- **Many Visits** → One Visitor

---

## 📌 7. Important Notes

- ❌ **Don’t** add `List<Visit>` in request DTOs.
- ❌ **Don’t** expose Entity directly in API responses.
- ✔ **Always** use DTOs for request/response.


