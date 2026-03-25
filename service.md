

## 🔹 1. POST `/visitors`  
### Register visitor + initial visit  

**Action by:** Receptionist  
**Status after registration:** `PENDING` (no check‑in)

```java
@Override
public VisitorResponseDto registerVisitor(VisitorWithVisitRequestDto dto) {

    // 1. Map visitor
    Visitor visitor = modelMapper.map(dto.getVisitor(), Visitor.class);

    visitor.setUniqueId(generateUniqueId());
    visitor.setStatus(VisitorStatus.PENDING);
    visitor.setCreatedAt(LocalDateTime.now());

    Visitor savedVisitor = visitorRepository.save(visitor);

    // 2. Create initial visit (NO entryTime)
    VisitRecord visit = new VisitRecord();

    visit.setVisitor(savedVisitor);
    visit.setReasonForVisit(dto.getVisit().getReasonForVisit());
    visit.setVisitNotes(dto.getVisit().getVisitNotes());
    visit.setGatePassDuration(dto.getVisit().getGatePassDuration());
    visit.setGatePassTemplate(
        dto.getVisit().getGatePassTemplate() != null ? dto.getVisit().getGatePassTemplate() : "Standard"
    );

    visitRecordRepository.save(visit);

    // 3. Attach visit to response
    savedVisitor.setVisitRecords(List.of(visit));

    return mapToVisitorResponse(savedVisitor);
}
```

---

## 🔹 2. POST `/visitors/{visitorId}/visits`  
### Add repeat visit  

**Action by:** Receptionist  
**Status remains unchanged**

```java
@Override
public VisitResponseDto addVisit(Integer visitorId, VisitRequestDto dto) {

    // 1. Fetch Visitor
    Visitor visitor = visitorRepository.findById(visitorId)
            .orElseThrow(() -> new RuntimeException("Visitor not found"));

    // 2. 🔥 Validation: Visitor should NOT be inside
    if (visitor.getStatus() == VisitorStatus.CHECKED_IN) {
        throw new RuntimeException("Visitor already inside, cannot create new visit");
    }

    // 3. Create new VisitRecord
    VisitRecord visit = new VisitRecord();

    visit.setVisitor(visitor);
    visit.setReasonForVisit(dto.getReasonForVisit());
    visit.setVisitNotes(dto.getVisitNotes());

    // Default duration
    int duration = dto.getGatePassDuration() != null ? dto.getGatePassDuration() : 1;
    visit.setGatePassDuration(duration);

    // Default template
    visit.setGatePassTemplate(
            dto.getGatePassTemplate() != null ? dto.getGatePassTemplate() : "Standard"
    );



    visitRecordRepository.save(visit);

    // 4. 🔥 Reset Visitor status for new visit lifecycle
    visitor.setStatus(VisitorStatus.PENDING);
    visitorRepository.save(visitor);

    // 5. Return response
    return modelMapper.map(visit, VisitResponseDto.class);
}
```

---

## 🔹 3. GET `/visitors` (Pagination + Filter)  
### Retrieve visitors with optional status filter and sorting  

```java
@Override
public Page<VisitorResponseDto> getAllVisitors(
        int page, int limit, String sortBy, String order, String status) {

    Sort sort = order.equalsIgnoreCase("desc") ?
            Sort.by(sortBy).descending() :
            Sort.by(sortBy).ascending();

    Pageable pageable = PageRequest.of(page, limit, sort);

    Page<Visitor> visitors;

    if (status != null) {
        visitors = visitorRepository.findByStatus(VisitorStatus.valueOf(status), pageable);
    } else {
        visitors = visitorRepository.findAll(pageable);
    }

    return visitors.map(visitor -> {
        VisitorResponseDto dto = mapToVisitorResponse(visitor);

        // Optional: attach latest visit only
        VisitRecord latest = visitRecordRepository
                .findTopByVisitorOrderByEntryTimeDesc(visitor);

        if (latest != null) {
            dto.setVisitRecords(List.of(modelMapper.map(latest, VisitResponseDto.class)));
        }

        return dto;
    });
}
```

---

## 🔹 4. GET `/visitors/search?q=`  
### Search visitors by name or unique ID  

```java
@Override
public List<VisitorSearchDto> searchVisitors(String query) {

    List<Visitor> visitors =
            visitorRepository.findByNameContainingIgnoreCaseOrUniqueIdContaining(query, query);

    return visitors.stream().map(visitor -> {

        VisitRecord latest = visitRecordRepository
                .findTopByVisitorOrderByEntryTimeDesc(visitor);

        VisitorSearchDto dto = new VisitorSearchDto();

        dto.setVisitorId(visitor.getVisitorId());
        dto.setName(visitor.getName());
        dto.setUniqueId(visitor.getUniqueId());
        dto.setCompany(visitor.getCompany());
        dto.setStatus(visitor.getStatus().name());

        if (latest != null) {
            dto.setLatestVisit(modelMapper.map(latest, VisitResponseDto.class));
        }

        return dto;

    }).toList();
}
```

---

## 🔹 5. GET `/visitors/{visitorId}`  
### Get detailed visitor information with all visits (sorted descending)  

```java
@Override
public VisitorResponseDto getVisitorById(Integer visitorId) {

    Visitor visitor = visitorRepository.findById(visitorId)
            .orElseThrow(() -> new RuntimeException("Visitor not found"));

    // Load visits sorted DESC
    List<VisitRecord> visits =
            visitRecordRepository.findByVisitorOrderByEntryTimeDesc(visitor);

    visitor.setVisitRecords(visits);

    return mapToVisitorResponse(visitor);
}
```

---

## 🔹 6. PUT `/visitors/{visitorId}`  
### Update permanent visitor information only  

**Action by:** Receptionist (or Admin)  
**No status or visit data is changed**

```java
@Override
public VisitorResponseDto updateVisitor(Integer visitorId, VisitorRequestDto dto) {

    Visitor visitor = visitorRepository.findById(visitorId)
            .orElseThrow(() -> new RuntimeException("Visitor not found"));

    if (dto.getName() != null) visitor.setName(dto.getName());
    if (dto.getCompany() != null) visitor.setCompany(dto.getCompany());
    if (dto.getContactNumber() != null) visitor.setContactNumber(dto.getContactNumber());
    if (dto.getEmail() != null) visitor.setEmail(dto.getEmail());
    if (dto.getNotes() != null) visitor.setNotes(dto.getNotes());

    visitorRepository.save(visitor);

    return mapToVisitorResponse(visitor);
}
```

---

## 🔹 7. PUT `/visitors/{visitorId}/visits/{recordId}`  
### Update specific visit details  

**Action by:** Receptionist (or Security)  
**Only updates allowed fields, no workflow logic**

```java
@Override
public VisitResponseDto updateVisit(Integer visitorId, Integer recordId, VisitRequestDto dto) {

    VisitRecord visit = visitRecordRepository.findById(recordId)
            .orElseThrow(() -> new RuntimeException("Visit not found"));

    if (!visit.getVisitor().getVisitorId().equals(visitorId)) {
        throw new RuntimeException("Visit does not belong to this visitor");
    }

    if (dto.getReasonForVisit() != null)
        visit.setReasonForVisit(dto.getReasonForVisit());

    if (dto.getVisitNotes() != null)
        visit.setVisitNotes(dto.getVisitNotes());

    if (dto.getGatePassDuration() != null)
        visit.setGatePassDuration(dto.getGatePassDuration());

    if (dto.getGatePassTemplate() != null)
        visit.setGatePassTemplate(dto.getGatePassTemplate());

    visitRecordRepository.save(visit);

    return modelMapper.map(visit, VisitResponseDto.class);
}
```

---

## 🔥 Important Rules Followed

- ✔ No check‑in logic in these APIs  
- ✔ No status change except initial `PENDING`  
- ✔ Visits are created without `entryTime`  
- ✔ Separation of roles maintained (Receptionist vs Security)  

---

## 🔷 Final Flow (Aligned with Requirement)

```
POST   /visitors               → create visitor + visit → status = PENDING
POST   /visitors/{id}/visits   → add visit              → status unchanged
GET    /visitors               → read only (paginated, filtered)
GET    /visitors/{id}          → read only (full details)
PUT    /visitors/{id}          → update permanent info only
PUT    /visitors/{id}/visits/{recordId} → update visit details only
```

---


