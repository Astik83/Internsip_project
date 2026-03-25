

## 📦 1. VisitorRepository

```java
@Repository
public interface VisitorRepository extends JpaRepository<Visitor, Integer> {

    // 🔹 Pagination + Status Filter
    Page<Visitor> findByStatus(VisitorStatus status, Pageable pageable);

    // 🔹 Search by name OR uniqueId
    List<Visitor> findByNameContainingIgnoreCaseOrUniqueIdContaining(String name, String uniqueId);

    // 🔹 Optional: exact search by uniqueId
    Optional<Visitor> findByUniqueId(String uniqueId);
}
```

---

## 📦 2. VisitRecordRepository

```java
@Repository
public interface VisitRecordRepository extends JpaRepository<VisitRecord, Integer> {

    // 🔹 Get latest visit for a visitor
    VisitRecord findTopByVisitorOrderByEntryTimeDesc(Visitor visitor);

    // 🔹 Get all visits sorted (used in GET /visitors/{id})
    List<VisitRecord> findByVisitorOrderByEntryTimeDesc(Visitor visitor);
}
```

---

## 🔥 3. Advanced Queries (Recommended)

### ✅ 3.1 Latest Visit using JPQL (More control)

```java
@Query("SELECT v FROM VisitRecord v WHERE v.visitor = :visitor ORDER BY v.entryTime DESC")
List<VisitRecord> findLatestVisit(@Param("visitor") Visitor visitor, Pageable pageable);
```

**Usage:**  
```java
PageRequest.of(0, 1) // get only latest
```

### ✅ 3.2 Search with JPQL (Better control)

```java
@Query("SELECT v FROM Visitor v WHERE LOWER(v.name) LIKE LOWER(CONCAT('%', :q, '%')) OR v.uniqueId LIKE CONCAT('%', :q, '%')")
List<Visitor> searchVisitors(@Param("q") String query);
```

### ✅ 3.3 Expired Visits (for Scheduler)

```java
@Query("SELECT vr FROM VisitRecord vr WHERE vr.gatePassExpiryTime < :now AND vr.exitTime IS NULL")
List<VisitRecord> findExpiredVisits(@Param("now") LocalDateTime now);
```

### ✅ 3.4 Active Visits (Optional)

```java
@Query("SELECT vr FROM VisitRecord vr WHERE vr.exitTime IS NULL")
List<VisitRecord> findActiveVisits();
```

---

## 🔥 4. Pagination + Sorting (Already Supported)

Because you extend `JpaRepository<Visitor, Integer>`, you automatically get:

```java
findAll(Pageable pageable);
```

✔ No need to write manually.

---

## 🔥 5. Optional Native Query (Interview Bonus ⭐)

```java
@Query(
  value = "SELECT * FROM visitors v WHERE v.name LIKE %:q% OR v.unique_id LIKE %:q%",
  nativeQuery = true
)
List<Visitor> searchVisitorsNative(@Param("q") String query);
```

---

## 🔷 Final Repository Structure

```
repository/
 ├── VisitorRepository
 └── VisitRecordRepository
```

---

## 🔥 Important Concepts (Interview Ready)

### ✅ Method Naming Magic

```java
findByNameContainingIgnoreCase
```

👉 Spring automatically converts to SQL.

### ✅ When to use `@Query`?

- Complex logic  
- Joins  
- Performance tuning  

### ✅ When to use `Pageable`?

- Pagination  
- Sorting  
- Large data handling  

--
